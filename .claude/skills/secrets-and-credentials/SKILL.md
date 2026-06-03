---
name: secrets-and-credentials
description: Use when storing, fetching, or rotating any kind of secret — app config secrets, tenant OAuth tokens, vendor API keys, encryption keys. Use when designing the tenant credential vault, integrating Cloud KMS or Secret Manager, debugging "where does this key live," or auditing secret access.
---

# secrets-and-credentials

## When to use this skill

Triggered for any code path touching a secret: storing an OAuth token a user just granted, decrypting it to make an API call, rotating an encryption key, reading a service config secret, debugging an integration that returns 401. Also during review — every `str` parameter that *could* be a secret is worth pausing on.

## Core principles

1. **Two homes by kind:**
   - **App config secrets** (DB password, Anthropic key, Twilio token) → **Google Secret Manager**, mounted as env vars by Cloud Run at startup.
   - **Tenant secrets** (each customer's OAuth tokens, their API keys) → **envelope encryption** with Cloud KMS, ciphertext stored in AlloyDB, decrypted per-request.
2. **Envelope encryption: KMS wraps the key, the key wraps the data.** The KMS-wrapped Data Encryption Key (DEK) is stored alongside the ciphertext. Decryption is one KMS call (unwrap) + one local AES decrypt. Encryption is symmetric.
3. **Fetch, use, drop.** Tenant secrets are decrypted at the call site, used immediately, and dropped at the end of the request. No process-wide cache. No request memo. Memory zeroization is impractical in Python — minimize the window instead.
4. **`SecretStr` at every boundary.** Pydantic's `SecretStr` hides values from `repr`, `str`, and most accidental log lines. Call `.get_secret_value()` only at the point of use.
5. **Audit every access.** A row per tenant credential read: who, when, which credential, for which operation. This is the difference between "we had an incident" and "we know exactly what was touched."
6. **Rotation is built in.** KMS keys version; ciphertext records the version. Re-encrypting on rotation is a background job, not a panic.
7. **Least privilege.** The IAM identity for the app service can `decrypt` with the KMS key but not `destroy` or `setIamPolicy`. Audit-log access can `read` audits but not `decrypt`. Separate identities for separate surfaces.

## Always

### App config secrets via Secret Manager → env

Cloud Run service config mounts secrets as environment variables. Pydantic-settings reads them at startup. See `python-configuration`.

```python
class Settings(BaseSettings):
    anthropic_api_key: SecretStr
    twilio_auth_token: SecretStr
    kms_key_name: str  # "projects/.../cryptoKeys/tenant-secrets"
```

`SecretStr` ensures these values don't leak via logs or `repr(settings)`.

### Tenant credentials: envelope encryption schema

```sql
CREATE TABLE tenant_credentials (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    integration     TEXT NOT NULL,           -- 'gmail', 'gcal', 'github', ...
    purpose         TEXT NOT NULL,           -- 'read', 'send', etc.
    -- The envelope
    ciphertext      BYTEA NOT NULL,          -- AES-GCM encrypted secret JSON
    nonce           BYTEA NOT NULL,          -- 12-byte AES-GCM nonce
    wrapped_dek     BYTEA NOT NULL,          -- KMS-encrypted DEK
    kms_key_version TEXT NOT NULL,           -- 'projects/.../cryptoKeyVersions/3'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    rotated_at      TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ              -- for OAuth tokens with expiry
);
CREATE INDEX tenant_credentials_lookup
    ON tenant_credentials (tenant_id, integration, purpose);

ALTER TABLE tenant_credentials ENABLE ROW LEVEL SECURITY;
ALTER TABLE tenant_credentials FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON tenant_credentials
    USING (tenant_id = current_setting('app.tenant_id', true)::uuid)
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);
```

RLS still applies — even within tenant data, secrets get the same isolation as everything else.

### The vault interface (swap-ready)

```python
from typing import Protocol
from pydantic import SecretStr

class CredentialVault(Protocol):
    async def store(
        self,
        *,
        ctx: TenantContext,
        integration: str,
        purpose: str,
        secret: dict[str, Any],
        expires_at: datetime | None = None,
    ) -> None: ...

    async def fetch(
        self,
        *,
        ctx: TenantContext,
        integration: str,
        purpose: str,
    ) -> dict[str, Any]: ...      # returns the decrypted dict; caller drops after use
```

The Protocol is the swap point. The KMS-backed implementation is one file.

### KMS-backed implementation

```python
import os
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
from google.cloud import kms_v1

class KmsCredentialVault:
    def __init__(self, kms: kms_v1.KeyManagementServiceAsyncClient, key_name: str, db: AsyncSession) -> None:
        self._kms = kms
        self._key_name = key_name
        self._db = db

    async def store(self, *, ctx, integration, purpose, secret, expires_at=None) -> None:
        # 1. Generate a fresh DEK
        dek = os.urandom(32)  # 256-bit
        nonce = os.urandom(12)

        # 2. Encrypt the secret payload with the DEK
        aes = AESGCM(dek)
        ciphertext = aes.encrypt(nonce, json.dumps(secret).encode(), associated_data=str(ctx.tenant_id).encode())

        # 3. Wrap the DEK with KMS
        wrap = await self._kms.encrypt(request={"name": self._key_name, "plaintext": dek})

        # 4. Persist envelope
        row = TenantCredentialRow(
            tenant_id=ctx.tenant_id,
            integration=integration,
            purpose=purpose,
            ciphertext=ciphertext,
            nonce=nonce,
            wrapped_dek=wrap.ciphertext,
            kms_key_version=wrap.name,
            expires_at=expires_at,
        )
        self._db.add(row)
        await self._db.flush()
        await self._audit("store", ctx, integration, purpose, row.id)

    async def fetch(self, *, ctx, integration, purpose) -> dict[str, Any]:
        row = await self._db.scalar(
            select(TenantCredentialRow).where(
                TenantCredentialRow.integration == integration,
                TenantCredentialRow.purpose == purpose,
            )  # RLS scopes to tenant
        )
        if row is None:
            raise CredentialNotFound(integration=integration, purpose=purpose)

        # 1. Unwrap DEK with KMS
        unwrap = await self._kms.decrypt(request={"name": self._key_name, "ciphertext": row.wrapped_dek})
        dek = unwrap.plaintext

        # 2. Decrypt payload locally
        aes = AESGCM(dek)
        plaintext = aes.decrypt(row.nonce, row.ciphertext, associated_data=str(ctx.tenant_id).encode())
        await self._audit("fetch", ctx, integration, purpose, row.id)
        return json.loads(plaintext)
```

The DEK never goes to disk in plaintext. KMS sees only the wrapped DEK; the app sees the unwrapped DEK for the duration of one decrypt call.

### Use-and-drop in the call site

```python
async def call_gmail_api(ctx: TenantContext, vault: CredentialVault, query: str) -> list[Message]:
    creds = await vault.fetch(ctx=ctx, integration="gmail", purpose="read")
    access_token = SecretStr(creds["access_token"])
    try:
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                "https://gmail.googleapis.com/...",
                headers={"Authorization": f"Bearer {access_token.get_secret_value()}"},
                params={"q": query},
            )
        resp.raise_for_status()
        return parse_messages(resp.json())
    finally:
        del creds                    # explicit (won't zeroize, but reduces residency)
        del access_token
```

The `del` doesn't truly wipe Python memory, but it removes the references, allowing earlier GC. The bigger win is *not extending the lifetime* by stashing in a class attribute or a cache.

### Rotation

```python
# Background job: re-encrypt with the latest KMS key version
async def rotate_tenant_credentials(vault: KmsCredentialVault, ctx: TenantContext) -> int:
    rows = await db.execute(select(TenantCredentialRow))
    count = 0
    for row in rows.scalars():
        decrypted = await vault.fetch(ctx=ctx, integration=row.integration, purpose=row.purpose)
        await vault.store(ctx=ctx, integration=row.integration, purpose=row.purpose, secret=decrypted, expires_at=row.expires_at)
        await db.delete(row)
        count += 1
    return count
```

Run as an Inngest/durable job per tenant when key rotation triggers (manual, scheduled, or post-incident).

### Audit log

```sql
CREATE TABLE credential_audit (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id     UUID NOT NULL,
    user_id       UUID,                    -- nullable for system access
    operation     TEXT NOT NULL,           -- 'store' | 'fetch' | 'rotate' | 'destroy'
    integration   TEXT NOT NULL,
    purpose       TEXT NOT NULL,
    credential_id UUID NOT NULL,           -- the tenant_credentials row
    request_id    TEXT,                    -- correlate with logs
    at            TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Every `vault.fetch` writes a row. Retention follows compliance policy (likely 1 year minimum). This table is RLS-protected by tenant but admin-readable for security reviews.

## Never

- Plaintext tenant secrets in the database. *Why:* the whole point of the vault. A `LIKE` query against a leak finds nothing if everything is encrypted.
- Plaintext secrets in environment variables for tenant data. *Why:* env vars are per-process, not per-tenant. App config goes in env; tenant config goes in the vault.
- Logging a secret. Even at `debug`. *Why:* "It's only debug" until it isn't. `SecretStr` hides via `repr`, but you can still leak by calling `.get_secret_value()` in an f-string.
- `model_dump()` of a Pydantic model containing `SecretStr` and logging the result. *Why:* `model_dump()` reveals secrets by default. Use `model_dump(mode='json')` (which masks) or `exclude={"api_key"}`.
- Caching decrypted secrets across requests. *Why:* extends residency, leaks across tenant boundaries if cache key is wrong. Decrypt per request.
- Hardcoded API keys, even in tests. *Why:* committed once, leaked forever. Use a `.env.test` with stubs, or build the secret in a fixture.
- `os.environ["X"]` in application code for any secret. *Why:* bypasses validation and `SecretStr`. Go through `Settings` (see `python-configuration`).
- Service-account JSON keys on disk. *Why:* high-value loot if your filesystem is breached. Use Workload Identity Federation for GitHub Actions and Cloud Run's runtime service account for the app.
- Granting `BYPASSRLS` to the role that reads credentials. *Why:* defeats the tenant-isolation layer of the vault.
- Storing the decrypted secret on a class attribute. *Why:* lives for the lifetime of the instance, which is the lifetime of the worker. Use locals; drop them.
- Custom crypto. *Why:* AES-GCM via `cryptography` is the standard. Don't write your own cipher modes, KDFs, or key wrappers.
- Reusing a nonce. *Why:* GCM nonce reuse with the same key catastrophically breaks confidentiality and authenticity. Use `os.urandom(12)` per encryption.
- Same DEK across multiple tenants' rows. *Why:* a leaked DEK then decrypts every tenant's data. Generate per-row.
- Decrypting in middleware. *Why:* couples auth path to KMS; one KMS outage takes down auth. Decrypt only where the secret is used.

## Pitfalls

- **KMS quota under burst load.** Default `cryptoKeys` quota is generous but finite. If every request decrypts, you hit the per-key QPS cap. Cache the *wrapped* DEK lookup (the DB row), or batch operations per request.
- **Forgetting `associated_data` on GCM.** Without binding the ciphertext to the tenant_id, an attacker who can write to the table could swap a row between tenants. Always pass `associated_data=str(ctx.tenant_id).encode()`.
- **KMS key version drift between encryption and decryption.** Store `kms_key_version` on each row. Decryption uses that version explicitly (KMS finds it by name).
- **`SecretStr` leaked via JSON encoder.** `json.dumps(model.model_dump())` includes the secret as plain string. Either `mode="json"` (masks) or explicit `exclude` set.
- **Audit log under load becoming the bottleneck.** Insert audit rows async/batched via Pub/Sub if synchronous insert becomes a hot path. Don't drop audit; defer it.
- **OAuth token refresh logic touching plaintext.** When refreshing an OAuth token, fetch ciphertext, decrypt, call `/token` endpoint to refresh, encrypt new tokens, store. Don't keep a long-lived refresh client with cached secrets.
- **Backups containing ciphertext + wrapped DEK.** Backups are RLS-bypassed by definition. Encrypt backups at rest (GCP does), and ensure KMS keys aren't included — separate the backup of the encrypted data from the key material.
- **Deleting a tenant doesn't delete their KMS key.** Best practice: when offboarding, run a deletion job that explicitly removes credential rows. The KMS key wrapping is shared (single key, per-row DEKs), so don't destroy the key.
- **In-memory secrets surviving exceptions.** A traceback can include local variables (e.g., `pytest --tb=long` or some loggers). Configure loggers to never include locals (`structlog` doesn't by default; verify your custom processors).
- **`del secret` not actually clearing.** Python won't zeroize. The realistic mitigation is short residency: don't hold secrets in variables longer than necessary. If you genuinely need wipe semantics, use a `bytearray` and overwrite with zeros — but accept this is best-effort.
- **Audit `user_id` missing for system-driven access.** Background jobs may not have a user; set `user_id=None` and log `actor="system"` instead.
- **Vault tests that pass with mocks.** Tests must use the real KMS (or a local emulator like the `google-cloud-kms` test stub) plus a real Postgres. Otherwise the envelope correctness isn't proven.

## Verification

Before declaring secrets work done:

- `rg "os\.environ" src/` — only inside `config.py` if anywhere. No app code reads raw env.
- `rg "SecretStr" src/` shows all sensitive fields use it.
- A test asserts `repr(settings)` does not contain any known secret value.
- A test stores a secret, fetches it, asserts the plaintext matches; asserts the DB row contains only ciphertext + wrapped DEK.
- A test in the vault confirms KMS is called once per encrypt and once per decrypt; the DEK is fresh per encrypt.
- A test confirms tampering with `associated_data` (e.g., swapping the tenant_id in the row) causes decryption to fail with an authentication error.
- A migration test asserts the `tenant_credentials` table has RLS enabled + forced + policy.
- The audit table has a row per `vault.fetch` in a smoke test (`assert len(audit_rows) == N`).
- IAM review: only the app's service account has `cloudkms.cryptoKeyEncrypterDecrypter` on the tenant-secrets key; only admin accounts have higher-privilege roles.
- A rotation job test re-encrypts a row and the old ciphertext no longer decrypts with the new version.
- Logs from a vault operation contain `operation`, `integration`, `tenant_id` — and *not* any secret material.
- `model_dump` of any model containing a `SecretStr` is masked when logged (verified by a test).

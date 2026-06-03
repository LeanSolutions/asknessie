---
name: python-configuration
description: Use when reading or adding any kind of configuration — env vars, app settings, feature flags, per-environment overrides, or per-tenant config. Use when wiring pydantic-settings, deciding whether something is build-time or runtime, choosing between Secret Manager and KMS, or refactoring scattered os.environ reads.
---

# python-configuration

## When to use this skill

Triggered any time configuration crosses a module boundary or comes from outside the process: environment variables, files, Secret Manager, KMS, feature flags, per-tenant settings, build-time constants. If a value is hard-coded but probably shouldn't be, this skill applies.

## Core principles

1. **There is exactly one way a value enters the app.** A field in a `Settings` model. Not `os.environ`, not `os.getenv`, not a global, not a passed-around dict. One source, typed, validated at startup.
2. **Fail at startup, not at request time.** Missing config kills the process during `lifespan` startup — never lets a request through with an invalid `Settings`.
3. **Three categories of config, each with its own home.**
   - **App config** (region, log level, feature flags, model name): `Settings` from env, loaded from Secret Manager in prod.
   - **Tenant secrets** (their OAuth tokens, API keys): Cloud KMS envelope encryption, ciphertext in Postgres, fetched per-request. See `secrets-and-credentials`.
   - **User preferences** (notification channels, voice settings): regular Postgres rows, not "config."
4. **Build-time vs runtime is a deliberate choice.** Anything that changes per deploy without a code change is runtime. Anything that defines the shape of the build (Python version, dependency pins) is build-time and lives in `pyproject.toml`.
5. **Settings are immutable per process.** Build once during lifespan startup, freeze, inject via FastAPI dependencies. Reload requires a restart — and that's fine because Cloud Run rolls instances cheaply.

## Always

- Define settings as a single `Settings(BaseSettings)` (Pydantic v2 via `pydantic-settings`). One file: `src/config.py`.

  ```python
  from functools import lru_cache
  from pydantic import Field, SecretStr
  from pydantic_settings import BaseSettings, SettingsConfigDict

  class Settings(BaseSettings):
      model_config = SettingsConfigDict(
          env_file=".env",
          env_file_encoding="utf-8",
          env_nested_delimiter="__",
          extra="forbid",            # unknown env vars are a bug, not a feature
          frozen=True,
      )

      env: Literal["dev", "staging", "prod"] = "dev"
      log_level: Literal["debug", "info", "warning", "error"] = "info"

      database_url: SecretStr = Field(..., description="AlloyDB DSN")
      redis_url: SecretStr

      anthropic_api_key: SecretStr
      anthropic_vertex_project: str
      anthropic_vertex_region: str = "us-east5"

      kms_key_name: str = Field(..., description="projects/.../cryptoKeys/...")

  @lru_cache(maxsize=1)
  def get_settings() -> Settings:
      return Settings()  # raises on missing/invalid → process dies at startup
  ```

- Inject settings as a FastAPI dependency, never import the module-level instance into business code:

  ```python
  SettingsDep = Annotated[Settings, Depends(get_settings)]

  @router.post("/threads")
  async def create_thread(settings: SettingsDep, ...): ...
  ```

- Use `SecretStr` for anything sensitive. It hides the value from `repr`/`str` so accidental log lines don't leak it. Call `.get_secret_value()` only at the point of use.

- Use `Literal[...]` for known-set fields (`env`, `log_level`, region). Pydantic validates the value; Pyright narrows downstream.

- Use nested settings models when grouping related fields:

  ```python
  class TwilioConfig(BaseModel):
      account_sid: str
      auth_token: SecretStr
      from_number: str

  class Settings(BaseSettings):
      twilio: TwilioConfig
  ```
  Env vars become `TWILIO__ACCOUNT_SID`, etc. via `env_nested_delimiter="__"`.

- Validate cross-field invariants with `@model_validator(mode="after")`. Example: prod must require `kms_key_name`; dev allows a stub.

- In prod on GCP, source secrets from **Secret Manager**, mounted as env vars by Cloud Run. Local dev uses `.env`. Same `Settings` class — only the source changes.

- Feature flags live in a remote store (PostHog or Statsig), not in `Settings`. `Settings` carries the SDK key; the flag values come from the SDK.

## Never

- `os.environ[...]` or `os.getenv(...)` in application code. *Why:* bypasses validation, hides what the app actually needs to run, and makes the env contract invisible.
- A module-level `settings = Settings()` imported and reused across the app. *Why:* hides dependency, breaks testability, makes overriding in tests painful.
- Mutating a `Settings` object at runtime. *Why:* causes spooky-action-at-a-distance bugs. Use a restart, or model the changing thing as state (DB row), not config.
- Putting tenant-specific values in `Settings`. *Why:* `Settings` is per-process, tenants are per-request. Tenant data belongs in the DB, vault, or `TenantContext`.
- Putting business logic defaults in `Settings` (e.g., "max_retries = 3"). *Why:* operational knobs and product constants are different things. Product constants live next to the code that uses them and change with code review, not deploys.
- Logging the `Settings` object. *Why:* even with `SecretStr`, you risk leaking via debugging or accidents. Log a redacted summary at startup (which env, which region, which features on) — never the object.
- Reading `.env` in prod. *Why:* `.env` is for local dev. Prod uses Secret Manager → Cloud Run env. Make `.env` optional and guard it.
- Different settings classes per environment. *Why:* one class, env-driven values. Branching schemas is how dev-vs-prod drift starts.

## Pitfalls

- **Silent string-to-bool conversion.** `BOOL_FLAG=False` becomes the string `"False"` which is truthy. Use `bool` typed fields in Pydantic and pass `"true"/"false"/"1"/"0"` — Pydantic parses these correctly. Never `if os.getenv("FLAG")`.
- **`SecretStr` printed accidentally.** `f"{settings}"` is safe; `f"{settings.api_key}"` shows `'**********'`. But `settings.model_dump()` reveals secrets by default — use `model_dump(mode="json")` and configure `SecretStr` to serialize masked, or exclude secret fields explicitly.
- **`@lru_cache` + multi-worker confusion.** Each worker has its own cache. Don't rely on this for "single instance globally" semantics. It's "single instance per process," which is what you want.
- **`extra="ignore"` instead of `"forbid"`.** Hides typos in env var names. Use `forbid` so `DATBASE_URL` (typo) raises at startup instead of running with the default.
- **Tests can't override settings cleanly.** Solution: `app.dependency_overrides[get_settings] = lambda: test_settings`. Never monkey-patch `os.environ` after import.
- **Putting `redis_url` and `database_url` as plain `str`.** They contain credentials. Use `SecretStr`.
- **Computing derived config at import time.** Anything like `BASE_URL = f"https://{os.environ['HOST']}"` at module top-level fails imports in tests and crashes Cloud Run if the env var is missing. Compute inside `Settings` via `@computed_field` or `@model_validator`.

## Verification

Before declaring config work done:

- `uv run pyright` passes — `Settings` is fully typed, no `Any`.
- Process refuses to start when a required env var is missing. Test this manually: `unset DATABASE_URL && uv run python -c "from src.config import get_settings; get_settings()"` should raise a clear validation error.
- `extra="forbid"` is set; an unknown env var raises.
- No `os.environ` or `os.getenv` calls exist in `src/` (verify with `rg "os\.(environ|getenv)" src/` — only acceptable hit is inside `config.py` if at all).
- Secrets use `SecretStr`, not `str`.
- Settings are injected via `Depends(get_settings)`, not imported as a module-level singleton (verify with `rg "from .*config import settings" src/` — should find nothing).
- Startup log line shows redacted summary of effective config (env, region, features on) — never the raw object.

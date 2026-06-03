---
name: in-house-utilities
description: Use whenever building or extending a DIY utility we chose over a SaaS — LLM client wrapper, durable executor, notify dispatcher, feature flags evaluator, credential vault, event tracker, STT/TTS clients. This is the meta-skill that keeps our small utilities from growing into 2000-LOC frameworks.
---

# in-house-utilities

## When to use this skill

Triggered whenever a DIY utility is the subject: adding a method, adding a configuration knob, adding a "just one more" feature, adding a plugin point, adding a vendor-feature parity field. Also during review — every PR that grows the surface of an in-house utility deserves a moment of "is this earning its weight?"

## Why this skill exists

We deliberately chose to build small utilities instead of bringing in vendors (Inngest, Helicone, Knock, PostHog). The cost-benefit only works if the utilities **stay small**. The failure mode isn't writing the utility — it's the slow drift where each release adds one knob, one configuration option, one "in case we need it" hook, until it's a worse version of the SaaS we were avoiding.

This skill is the immune system against that drift.

## Core principles

1. **One Protocol per utility, named after the role.** `LLMClient`, `DurableExecutor`, `NotifyDispatcher`, `FlagEvaluator`, `CredentialVault`, `STTClient`. The Protocol is the contract; the implementation is invisible to callers.
2. **Implementation lives in `lib/` or `integrations/`.** One file, one concept. `lib/durable.py`. Not `lib/durable/engine/core/abstract.py`.
3. **Public surface stays under ~10 symbols.** Five is better. The Protocol, one or two concrete classes, a couple of errors, a factory function. If the public API grows past 10 in a quarter, push back hard.
4. **Build the 80%, not feature parity.** We're not trying to be Inngest. We're trying to be "good enough that Inngest isn't worth $X/month yet." When you find yourself implementing the *next* tier of features to match the vendor, that's the signal to bring the vendor back.
5. **No configuration for one consumer.** Configuration knobs exist for users who need different behavior. We are one user. Hardcode. If a second consumer ever asks, refactor then.
6. **No plugin systems for one plugin.** A registry of one is a list of one. Inline it.
7. **The swap-out interface is sacred.** Every change to the Protocol is a deliberate, deliberate, deliberate decision. Implementation changes are cheap; interface changes ripple.
8. **Document the exit ramp.** Each utility's docstring names the vendor it's standing in for and the trigger to swap. Not "if we get bigger" — a specific trigger.
9. **Tests against the Protocol, not the implementation.** If a test imports the concrete class and verifies it called a private method, refactor it to use the Protocol.

## Always

### Define the Protocol next to the implementation

```python
# lib/durable.py
from typing import Protocol, Callable, Awaitable
from uuid import UUID

class DurableContext(Protocol):
    job_id: UUID
    tenant_id: TenantId
    attempt: int
    async def step[T](self, name: str, fn: Callable[[], Awaitable[T]]) -> T: ...
    async def sleep(self, name: str, duration: timedelta) -> None: ...
    async def wait_for_event(self, name: str, predicate: Callable[[dict], bool], timeout: timedelta) -> dict: ...

class DurableExecutor(Protocol):
    """Durable function executor.

    Stand-in for Inngest. Swap when: per-tenant concurrency contention exceeds DIY,
    fan-out complexity grows past two levels, or reliability incidents become recurring.
    Swap target: Inngest Python SDK. Adapter is one file (~150 LOC).
    """
    def function(self, name: str, *, max_attempts: int = 5) -> Callable: ...
    async def enqueue(self, fn_name: str, args: dict, *, tenant_id: TenantId, idempotency_key: str | None = None) -> UUID: ...
    async def cancel(self, job_id: UUID) -> None: ...

# concrete impl in the same file:
class PostgresDurableExecutor:
    # ... see python-durable-execution skill
```

The Protocol's docstring captures: what vendor it stands in for, the swap trigger, and the swap path. Future readers (and your future self) need this to make the decision when the time comes.

### Keep the surface tiny

```python
# lib/flags.py — entire public surface
from typing import Protocol

class FlagEvaluator(Protocol):
    async def is_enabled(self, flag: str, ctx: TenantContext) -> bool: ...

class PostgresFlagEvaluator:
    def __init__(self, db: AsyncSession, redis: Redis) -> None: ...
    async def is_enabled(self, flag: str, ctx: TenantContext) -> bool: ...

class FlagNotFound(Exception): ...

def get_flag_evaluator(...) -> FlagEvaluator: ...
```

Four public symbols. Adding a fifth requires asking "do I need this." The CRUD admin endpoint for flags is a separate concern — it goes in the API router, not in this utility.

### Tests target the Protocol

```python
# tests/lib/test_durable.py
async def test_executor_resumes_from_step_after_crash(executor: DurableExecutor):
    # No mention of PostgresDurableExecutor. The test works against any impl.
    @executor.function("test_resume")
    async def run(ctx, args):
        a = await ctx.step("a", lambda: random.random())
        b = await ctx.step("b", lambda: raise_once())   # raises on first call
        return {"a": a, "b": b}
    ...
```

When we swap to Inngest later, these tests still apply.

### Document the swap-out next to the implementation

```python
# lib/llm_client.py

class LLMClient(Protocol):
    """LLM gateway: cache + retry + cost log + per-tenant rate limiting.

    Stand-in for: Helicone / Portkey.
    Swap when: we want per-prompt caching analytics, multi-provider fallback orchestration,
              or model A/B testing UI.
    Swap target: Helicone (in front of Vertex AI). Adapter rewires `complete()` and emits
                 the same llm_calls row. ~100 LOC.
    """
```

This is the single most important paragraph for any in-house utility. It tells the next reader (or auditor) what we built and why, without needing to read the conversation that produced it.

### Strict additions discipline

Before adding a public method, function, class, or kwarg to a utility, check:

| Question | If yes |
|---|---|
| Does any caller in the repo need it today? | Add it. |
| Is it for a feature we're building right now? | Add it. |
| Is it for a hypothetical future caller or a "what if"? | Don't add it. |
| Is it for parity with the vendor we're not using? | Don't add it. |
| Is it a configuration knob with no current caller varying it? | Don't add it. Hardcode the default. |
| Is it a plugin/hook for one plugin? | Don't add it. Inline. |

Three yes-add and four no-add. The defaults skew correct.

### "Vendor parity creep" — the canary

When a PR description includes language like:

- "matches what Inngest does..."
- "for parity with PostHog..."
- "Helicone supports this so we should too..."

That's the canary. Stop. Ask: are we trying to be the vendor, or build the 80% that defers the vendor? If we're building parity, the vendor probably belongs back in the stack. Either bring it back or push back on the addition.

### Know the exit ramp

For each utility, the exit ramp answers three questions:

1. **What's the trigger?** Concrete signal that "we should swap." E.g., "p99 durable step latency exceeds 5 minutes" or "we're spending more than $200/mo on KMS calls."
2. **What's the target?** The named vendor we'd swap to.
3. **What's the cost?** Approximate LOC for the adapter and the migration time.

If you can't write these for a utility, you don't know if it's worth keeping in-house. Write them; revisit annually.

## Never

- Adding a configuration class with more than 5 fields to a utility. *Why:* configuration sprawl. If it has 6 knobs, it's a framework, not a utility.
- Adding a plugin/registry system for a single plugin. *Why:* future-proofing imagination. Inline; refactor when (if) a second plugin shows up.
- Matching vendor features one-for-one. *Why:* we chose DIY to defer the vendor; matching it negates the choice.
- Abstract base classes when one concrete implementation exists. *Why:* "what if we have another implementation" is rarely a real concern. `Protocol` lets you add one later without ABCs.
- Generic utility names like `utils`, `helpers`, `common`. *Why:* see `python-code-architecture`. Name by concept.
- Importing the concrete implementation in another module's code (vs. the Protocol). *Why:* couples callers to the impl. Inject the Protocol; let the wiring layer construct.
- "Internal" sub-packages (e.g., `lib/durable/_internals/`). *Why:* if one file isn't enough, you're overbuilding. Resist for as long as possible; refactor when truly needed.
- Adding telemetry inside a utility instead of structured logs that flow to the standard pipeline. *Why:* one telemetry pipeline, not per-utility. See `python-logging` and `python-llm-observability`.
- Building a "config" or "settings" mini-framework inside a utility. *Why:* `python-configuration` is the source of truth. The utility takes its config via constructor, period.
- Versioning the Protocol (`DurableExecutorV2`). *Why:* the Protocol changes in lockstep with the codebase; we're not exposing it as a public API for external consumers.
- Writing wrappers around the Protocol that "just add convenience." *Why:* the convenience method becomes the new interface that everyone uses, the Protocol drifts, swap-readiness dies.
- Caching results at the utility level when the caller could cache. *Why:* utility-level caching adds invalidation complexity, lifetime ambiguity, test surprises. Push to the caller unless the cache is *intrinsic* to the utility's job (e.g., the LLM gateway's prompt cache).
- Implementing a feature because "we'll definitely need it." *Why:* YAGNI. Implement when the first real caller asks.

## Pitfalls

- **Slow growth.** Each PR adds one method. Over 6 months: 20 methods. No PR seems wrong individually. Defense: monthly review of utility surface area. If the public surface grew, ask "did we earn it?"
- **Tests anchoring to implementation.** Tests that import the concrete class and check internal state are why "implementation changes are cheap" stops being true. Refactor tests to target the Protocol.
- **Hidden coupling via globals.** A utility that secretly reads a global config or environment leaks the assumption that "this only runs in our app." Future swap targets don't have that global. Pass dependencies; don't read globals.
- **"Just one more thing" syndrome on the LLM client.** LLM client wrappers attract feature creep — retry policies, multi-provider fallback, response transforms, dynamic prompt building. Resist; each one is a 100-LOC addition that adds a maintenance surface.
- **Cascading configuration.** The utility takes a config, which takes a sub-config, which... End up reinventing dependency-injection containers. Stop at constructor args.
- **Default values that drift from production.** Production runs with `concurrency=5`, dev with `concurrency=2`. Defaults in the utility encode "what's right for prod." Don't put per-env defaults in the utility; pass them at wiring time.
- **Logs leaking utility internals.** A utility that logs every step trace into general logs floods the system. Logs that need to flow elsewhere (LLM observability, audit) go via the dedicated pipelines; general logs are events.
- **Forgetting the swap-out doc.** Six months in, a new dev finds `lib/durable.py` and can't tell why it's DIY vs Inngest. The docstring is the only durable place that information lives.
- **Building an admin UI for the utility.** Internal admin tooling is fine; building a *generic* admin UI inside the utility's package is overbuild. Admin lives in `api/admin/`.
- **Mid-PR scope expansion.** "Now that I'm here, I'll also add..." Stop. Open a separate PR with separate justification. Each addition stands on its own.
- **Building before deciding.** "We might want to defer Langfuse" → start building LLM tracing. Decide first (see CLAUDE.md deferred decisions), build second. The decision constrains the build.

## Verification

Before declaring in-house-utility work done — and recurring quarterly as a hygiene check:

- Each utility has a Protocol whose docstring names: the vendor it stands in for, the swap trigger, the swap target.
- Each utility's public surface (Protocol + concrete + errors + factory) is ≤10 symbols. List them.
- `rg "class .*Config|class .*Settings" lib/` — utility config classes have ≤5 fields each.
- No utility imports another utility's concrete class — only Protocols.
- Tests for each utility target the Protocol, not the concrete (no internal-state assertions, no concrete imports in test imports).
- No `_internal` or `_private` sub-packages within `lib/<utility>/`.
- Every utility has a one-paragraph "exit ramp" comment with trigger, target, and rough adapter LOC.
- Code review checklist for utility PRs: "Does this addition have a current caller? Is this configuration knob varied in production? Is this for parity with the vendor we're avoiding?"
- A "vendor parity audit" run quarterly: look at each utility, list the features the corresponding vendor has that we don't, and confirm none of them are bothering us. If they are, swap. If they aren't, keep moving.
- Total LOC for each utility tracked. If `lib/durable.py` is 1,200 LOC, that's no longer a small utility — it's a framework. Decide consciously: do we keep growing, or swap?

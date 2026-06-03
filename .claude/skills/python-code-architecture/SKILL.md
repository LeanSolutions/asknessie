---
name: python-code-architecture
description: Use when deciding where code lives — class vs function vs module, designing Protocols for polymorphism, organizing dependency injection without a framework, applying SOLID principles in Python-native ways, choosing composition over inheritance, deciding when to split a module or merge two. Counterpart to `pythonic-style` (which covers statement-level idioms).
---

# python-code-architecture

## When to use this skill

Triggered for any structural decision: do I make this a class? where does this function go? should I extract a base class? do I need an interface? how do I inject this dependency? Also during review — every new file, every new abstraction, every inheritance relationship is a structural call.

## Core principles

1. **Functions first. Classes earn the privilege.** A class needs to justify itself with state, polymorphism, or a clear conceptual identity. Stateless logic is a function. Two stateless functions sharing a helper is two functions plus a private helper, not a class with two methods.
2. **Composition over inheritance.** Inheritance is for sharing implementation that you actually share. For polymorphism, use `Protocol`. For "I want my class to have these capabilities," put the capabilities in attributes.
3. **Protocols for interfaces; classes for implementations.** Structural typing matches Python's runtime. Reserve `ABC` for the rare case where you want to share implementation through inheritance *and* enforce subclass contracts.
4. **DI is constructor args.** At the edge (FastAPI route), inject via `Depends`. Below the edge, services accept their dependencies in `__init__`. We don't need a DI container; constructors are the container.
5. **Modules are the unit of cohesion.** A module is a *concept*. Files named `utils.py`, `helpers.py`, `common.py` are unsorted-laundry signals — split them.
6. **Domain by feature, not by layer.** `agents/`, `channels/`, `services/notify.py` is better than `services/`, `controllers/`, `repositories/` for our shape. We don't grow horizontally fast enough to justify layer-folders.
7. **SOLID translated.** SRP at module level (one reason to change per module). OCP via Protocol + new implementations. LSP just means "your Protocol implementation actually honors the contract." ISP: small focused Protocols. DIP: depend on Protocols, not concrete classes.

## Always

### Choose function vs class deliberately

```python
# Function — stateless, deterministic
def compute_cost(model: str, prompt_tokens: int, completion_tokens: int) -> Decimal:
    rate = MODEL_RATES[model]
    return Decimal(prompt_tokens) * rate.input + Decimal(completion_tokens) * rate.output

# Class — has state across calls
class LLMClient:
    def __init__(self, vertex: VertexClient, vault: CredentialVault) -> None:
        self._vertex = vertex
        self._vault = vault
        self._semaphore = asyncio.Semaphore(20)

    async def complete(self, prompt: str, ctx: TenantContext) -> Completion: ...
```

The semaphore + injected clients justify the class. If `LLMClient` had no state and no injected deps, it should be a module-level function.

### Use `Protocol` for polymorphism, plain classes for implementations

```python
class Tool(Protocol):
    name: str
    description: str
    args_schema: type[BaseModel]
    async def execute(self, args: BaseModel, ctx: TenantContext) -> Any: ...

# Implementations don't inherit — they match the shape
class GmailSearch:
    name = "gmail_search"
    description = "Search Gmail"
    args_schema = GmailSearchArgs
    def __init__(self, vault: CredentialVault) -> None:
        self._vault = vault
    async def execute(self, args: GmailSearchArgs, ctx: TenantContext) -> list[Message]: ...

class CalendarLookup:
    name = "calendar_lookup"
    # ... same shape, no shared base
```

Adding a new tool is adding a new class. No base class, no registration ceremony.

### Inject dependencies via constructor

```python
class ThreadService:
    def __init__(
        self,
        threads: ThreadRepository,
        notify: NotifyDispatcher,
        clock: Clock,
    ) -> None:
        self._threads = threads
        self._notify = notify
        self._clock = clock

    async def archive(self, thread_id: ThreadId, ctx: TenantContext) -> None:
        thread = await self._threads.get(thread_id, ctx)
        thread.archive(at=self._clock.now())
        await self._threads.save(thread, ctx)
        await self._notify.send(...)
```

`ThreadService` doesn't import a global `db` session or a singleton notifier. Its dependencies are explicit. Test substitution is "pass a different `notify`," not "monkey-patch a module."

### Wire at the edge with `Depends`

```python
def get_thread_service(
    threads: Annotated[ThreadRepository, Depends(get_thread_repo)],
    notify: Annotated[NotifyDispatcher, Depends(get_notify)],
    clock: Annotated[Clock, Depends(get_clock)],
) -> ThreadService:
    return ThreadService(threads, notify, clock)

@router.post("/threads/{tid}/archive")
async def archive_thread(
    tid: ThreadId,
    svc: Annotated[ThreadService, Depends(get_thread_service)],
    ctx: TenantDep,
) -> None:
    await svc.archive(tid, ctx)
```

FastAPI's `Depends` is the DI container. No `wire`, no `inject`, no registration class.

### Organize by domain, not by layer

```
src/
  agents/                # the agent loop and tool registry
    loop.py
    tools/
      gmail_search.py
      calendar_lookup.py
    errors.py
  channels/              # one folder per channel
    email/
    sms/
    web/
  threads/               # threads as a feature
    service.py
    repository.py
    models.py
    errors.py
  notifications/
  billing/
  lib/                   # cross-cutting utilities (durable, vault, flags)
    durable.py
    vault.py
    flags.py
```

Not:

```
src/
  services/              # everything's service
  repositories/          # everything's repo
  models/                # everything's model
  controllers/           # everything's controller
```

Layered folders make "find me everything about threads" require four `cd`s. Domain folders make a feature change a single-folder change.

### Apply SRP at module level

If `notifications/router.py` has functions for routing, batching, quiet-hours, *and* templating, that's four reasons to change. Split:

```
notifications/
  router.py        # event → channel decision
  batcher.py       # debounce / coalesce
  quiet_hours.py   # time-window logic
  templates.py     # rendering
```

Each can change independently. Each is small. Each is testable in isolation.

### Use `__init__.py` to define the public surface

```python
# notifications/__init__.py
from .router import NotifyDispatcher
from .errors import NotificationError, ChannelUnavailable

__all__ = ["NotifyDispatcher", "NotificationError", "ChannelUnavailable"]
```

Callers import `from notifications import NotifyDispatcher`. Internal modules (`batcher`, `templates`) stay implementation detail.

## Never

- Inheritance for code reuse. *Why:* mixins create MRO confusion; "extends" relationships are rarely true subtype-of relationships. Prefer composition (a `ThreadService` *has-a* `Repository`) or a small Protocol.
- `class Helper:` with `@staticmethod` methods and no state. *Why:* that's a module. Move the functions to module level.
- `class FooService` with no `__init__` dependencies and only one method. *Why:* that's a function. Make it a function.
- `utils.py`, `helpers.py`, `common.py`, `misc.py`. *Why:* grab bags accumulate junk. If utilities are cross-cutting, name them: `time.py`, `ids.py`, `crypto.py`. If they belong to one feature, put them there.
- Singletons via module globals (`db = AsyncSession()` at module top). *Why:* binds at import time, breaks test substitution, leaks across requests. Build inside `lifespan`, inject via `Depends`.
- DI containers (`dependency-injector`, `inject`, `punq`). *Why:* FastAPI `Depends` + constructors covers us. A container is overhead for the scale we're at.
- `ABC` where `Protocol` works. *Why:* forces inheritance. We rarely need to share implementation.
- `class FactoryFactory` or `AbstractFactoryRegistry`. *Why:* enterprise pattern that's wrong for Python. If you're tempted, the actual answer is usually "a dict of constructors."
- Cyclic imports "solved" by putting imports inside functions. *Why:* the cycle is a design smell. Restructure so the dependency goes one direction.
- Putting business logic in `__init__.py`. *Why:* `__init__` is for re-exports. Logic at module-import time runs unpredictably.
- "Service locator" globals like `Registry.get("tool")`. *Why:* hides dependencies, makes test substitution painful. Pass the dependency.
- Inheritance chains deeper than two. *Why:* by depth 3, the LSP violations and MRO surprises start. If you find yourself there, refactor to composition.

## Pitfalls

- **`Protocol` with `Self` interactions.** Newer Pyright handles this; older versions get confused with `self` in Protocol method signatures. Sometimes you need `from typing import Self` and explicit annotation.
- **Mixins with conflicting method names.** MRO resolves left-to-right; if two mixins define `save`, the order in the class statement determines which wins. Surprising. Prefer composition or rename methods.
- **Circular imports at module top-level.** Often caused by type annotations that reference classes from another module. Use `if TYPE_CHECKING:` to import the type for annotations only, then string-quote the type in signatures.
- **Service classes that should be modules.** Symptom: the class has no `__init__` dependencies, all methods are `@staticmethod`, and no state. Refactor to a module of functions.
- **Modules with 20+ public functions.** Usually a sign of insufficient SRP. Split by concept (e.g., `text.py` and `time.py` instead of one `utils.py` with 30 functions).
- **`from x import *`.** Pollutes the namespace, makes "where does this come from" undebuggable. Always explicit imports.
- **`Protocol` covariance/contravariance errors.** A method returning a generic param is covariant; a method accepting one is contravariant. Pyright complains; the fix is variance markers (`T_co`, `T_contra`) or rethinking the interface.
- **`__init__.py` re-exporting in a way that breaks tooling.** Some IDEs and Pyright handle re-exports via `__all__` differently. Test that "Go to Definition" works on the re-exported symbol.
- **DI through factory functions becoming a religion.** It's fine to construct directly in the wiring layer (`lifespan` or `Depends`). Not everything needs a `make_foo()` function.
- **Splitting a module too aggressively.** Three files of 30 lines each is worse than one file of 90 lines. Split when a concept *separates*, not because of size alone.
- **Naming a module after its layer rather than its concept.** `services/notify_service.py` is redundant. `notifications/router.py` is concept-named, more honest.

## Verification

Before declaring organization work done:

- `rg "^class .*\(ABC\)" src/` — review every hit. Each one should justify the `ABC` (shared implementation, enforced subclass contract). Otherwise switch to `Protocol`.
- `rg "from .* import \*" src/` returns nothing.
- No file named `utils.py`, `helpers.py`, `common.py`, `misc.py` in `src/`.
- No class with only `@staticmethod` methods and no `__init__`.
- No global mutable state at module top-level (verify with a grep for `= []`, `= {}`, `= AsyncSession()` etc. at module scope).
- Service classes have at least one constructor dependency or instance state; otherwise refactor to functions.
- Circular imports broken with `if TYPE_CHECKING:` rather than function-local imports.
- Each `src/<feature>/` directory has a focused concept; module names within are concept-named, not layer-named.
- `from <feature> import <symbol>` works without importing internals.

---
name: python-typing
description: Use when adding or refining type hints, designing typed interfaces (Protocol, Generic), narrowing types, making illegal states unrepresentable, or resolving Pyright strict-mode errors. Use whenever a function signature, class attribute, or boundary contract is being written or changed.
---

# python-typing

## When to use this skill

Triggered for any change involving the type system: signatures, return types, generics, protocols, narrowing, `Annotated` metadata, discriminated unions, Pyright suppressions. Also during review — every unannotated function and every `Any` is a design choice worth questioning.

## Core principles

1. **Types are documentation the machine enforces.** A correctly-typed signature is a contract Pyright verifies on every save. Untyped code is a contract written in invisible ink.
2. **Make illegal states unrepresentable.** Use discriminated unions, `NewType`, narrow `Literal`, and small sum types instead of "string with these expected values." If a value is invalid, the type system should reject it before runtime does.
3. **Protocols over ABCs.** Structural typing matches Python's duck-typed runtime. Reserve inheritance for sharing implementation, not for declaring interfaces.
4. **`Annotated` is for type + metadata.** FastAPI dependencies, Pydantic validators, doc strings on parameters — all flow through `Annotated[T, ...]`, not subclasses or sidecar registries.
5. **`Any` is a confession, not a tool.** Every `Any` is "I gave up here." Sometimes that's correct (untyped third-party); usually it's a hole. Justify with a comment.
6. **Strict mode, no exceptions.** Pyright `strict` from day one. Loosening later is harder than tightening early.

## Always

### Modern syntax (Python 3.13)

```python
# Yes
def get(ids: list[str], meta: dict[str, int]) -> User | None: ...
def first(xs: list[T]) -> T | None: ...

# No
from typing import List, Dict, Optional, Union
def get(ids: List[str], meta: Dict[str, int]) -> Optional[User]: ...
```

No `from __future__ import annotations` — not needed in 3.13. Ruff's `UP` rules will flag it.

### PEP 695 generics (3.12+) — the new way

```python
# 3.12+ generic function syntax
def first[T](xs: list[T]) -> T | None:
    return xs[0] if xs else None

# Generic class
class Repository[T]:
    async def get(self, id: UUID) -> T | None: ...

# Generic type alias
type Result[T] = tuple[T, None] | tuple[None, Exception]
```

Prefer this over `TypeVar` + `Generic[T]` for new code. `TypeVar` is still fine where you need explicit bounds or contravariance.

### `NewType` for identity-carrying primitives

```python
from typing import NewType
from uuid import UUID

TenantId = NewType("TenantId", UUID)
UserId = NewType("UserId", UUID)
ThreadId = NewType("ThreadId", UUID)

def send_message(thread_id: ThreadId, user_id: UserId) -> None: ...

# Pyright catches this:
send_message(user_id, thread_id)  # error: arguments swapped
```

The same UUID at runtime; different types at typecheck time. Cheap, catches the highest-impact bug class (mixing IDs).

### `Protocol` for interfaces

```python
from typing import Protocol

class Tool(Protocol):
    name: str
    description: str
    args_schema: type[BaseModel]
    async def execute(self, args: BaseModel, ctx: TenantContext) -> Any: ...

# Any class with these members satisfies Tool — no inheritance required.
class GmailSearch:
    name = "gmail_search"
    description = "Search Gmail messages"
    args_schema = GmailSearchArgs
    async def execute(self, args: GmailSearchArgs, ctx: TenantContext) -> list[Message]: ...
```

`Protocol` is structural; `ABC` is nominal. Use `ABC` only when you also want to share implementation via mixins (which is rare in our codebase).

### Discriminated unions for variant types

```python
from typing import Literal

class TextPart(BaseModel):
    kind: Literal["text"] = "text"
    text: str

class ImagePart(BaseModel):
    kind: Literal["image"] = "image"
    url: str
    mime: str

class ToolCallPart(BaseModel):
    kind: Literal["tool_call"] = "tool_call"
    tool: str
    args: dict[str, Any]

MessagePart = TextPart | ImagePart | ToolCallPart

def render(part: MessagePart) -> str:
    match part.kind:
        case "text":  return part.text          # narrowed to TextPart
        case "image": return f"<image {part.url}>"
        case "tool_call": return f"<call {part.tool}>"
```

Pyright (and Pydantic) discriminate on the `kind` field automatically. Add a new variant and `match` is an error until you handle it — exhaustiveness for free.

### `Annotated` for type + metadata

```python
from typing import Annotated
from fastapi import Depends
from pydantic import Field

# FastAPI dependency injection
DbDep = Annotated[AsyncSession, Depends(get_db)]
TenantDep = Annotated[TenantContext, Depends(get_tenant_context)]

@router.post("/threads")
async def create_thread(db: DbDep, ctx: TenantDep, body: CreateThreadIn) -> ThreadOut: ...

# Pydantic field metadata
class CreateThreadIn(BaseModel):
    title: Annotated[str, Field(min_length=1, max_length=200)]
    initial_message: Annotated[str, Field(min_length=1, max_length=10_000)]
```

`Annotated` keeps the type primary and the metadata secondary — readable in signatures, machine-readable for tooling.

### `TypeGuard` and `TypeIs` (PEP 742, 3.13+) for narrowing

```python
from typing import TypeIs

def is_text_part(p: MessagePart) -> TypeIs[TextPart]:
    return p.kind == "text"

if is_text_part(part):
    # Pyright narrows part to TextPart here AND
    # narrows to MessagePart - TextPart in the else branch
    print(part.text)
```

Use `TypeIs` (3.13+) where possible — it narrows both branches, unlike the older `TypeGuard` which only narrows the truthy branch.

### `Self` for chainable / fluent return types

```python
from typing import Self

class QueryBuilder:
    def where(self, **kwargs: Any) -> Self:
        ...
        return self

    def limit(self, n: int) -> Self:
        ...
        return self

# Subclass methods return the subclass type — not the base
class TenantQueryBuilder(QueryBuilder): ...
TenantQueryBuilder().where(x=1).limit(10)  # Pyright: TenantQueryBuilder, not QueryBuilder
```

### `Final` for constants and frozen attributes

```python
from typing import Final

MAX_TOKENS_PER_TURN: Final = 8000
DEFAULT_MODEL: Final[str] = "claude-sonnet-4-6"

class Settings:
    env: Final[str]
    def __init__(self, env: str) -> None:
        self.env = env
```

`Final` says "this won't be reassigned." Pyright enforces it.

### `ParamSpec` for decorator types

```python
from typing import ParamSpec, TypeVar
from collections.abc import Callable
from functools import wraps

P = ParamSpec("P")
R = TypeVar("R")

def with_tenant_logging[**P, R](fn: Callable[P, R]) -> Callable[P, R]:
    @wraps(fn)
    async def inner(*args: P.args, **kwargs: P.kwargs) -> R:
        logger.info("call_start", fn=fn.__name__)
        return await fn(*args, **kwargs)
    return inner
```

PEP 695 also adds `[**P, R]` syntax. Use it for new decorators.

### `TypedDict` and `Unpack` for keyword arg contracts

```python
from typing import TypedDict, Unpack

class CreateThreadKw(TypedDict, total=False):
    title: str
    initial_message: str
    tags: list[str]

def create_thread(**kwargs: Unpack[CreateThreadKw]) -> Thread: ...
```

Use sparingly — Pydantic models are usually a better answer at boundaries. `TypedDict` is for internal kwargs contracts.

## Never

- `typing.List`, `typing.Dict`, `typing.Optional`, `typing.Union`. *Why:* deprecated in favor of built-in generics and `|`. Ruff's `UP` rules flag them.
- `from __future__ import annotations`. *Why:* not needed in 3.13. Causes confusion with runtime introspection (Pydantic, FastAPI).
- `Any` without `# pyright: ignore[reason]` and an inline comment explaining why. *Why:* every `Any` is a typecheck hole. Force the writer to justify it.
- `# type: ignore`. *Why:* unscoped suppression. Use `# pyright: ignore[ruleName]` so future readers know what was silenced.
- `cast(T, value)` to silence Pyright instead of fixing the source. *Why:* `cast` is fine when you have information Pyright can't see (after `isinstance` won't work — e.g., a checked invariant). It's not fine as a "make the error go away" hammer.
- Inheriting from `ABC` to define an interface. *Why:* use `Protocol`. ABCs force inheritance; Protocols allow structural matching.
- `TypeVar` without bounds when bounds exist. *Why:* `T = TypeVar("T", bound=BaseModel)` catches misuse at the call site; bare `T` accepts anything and loses information.
- `Optional[X]`. *Why:* use `X | None`. Same meaning, modern syntax, consistent with other unions.
- `Union[X, Y]`. *Why:* use `X | Y`.
- Untyped `**kwargs: Any` on public functions. *Why:* defeats type checking for callers. Use `TypedDict` + `Unpack`, or accept a Pydantic model.
- `*args: Any`. *Why:* same reason. If you're collecting variadic args, type them.
- `# type: ignore[attr-defined]` to access dynamic attributes. *Why:* usually means the type is wrong upstream. Fix the source type, don't paper over it.
- Module-level type aliases without `type` or `TypeAlias`. *Why:* `Vector = list[float]` is a variable, not an alias, in some interpretations. Use `type Vector = list[float]` (PEP 695) or `Vector: TypeAlias = list[float]`.

## Pitfalls

- **Forward references inside a class.** A method that references the enclosing class needs `Self` (3.11+) or a string literal: `def clone(self) -> "MyClass":`.
- **Pydantic v2 generics.** Pydantic models with `Generic[T]` work, but the field type often needs `Annotated` to surface validators. When in doubt, check Pydantic v2 docs for the specific generic pattern.
- **Pyright "could not be resolved" on an installed package.** Almost always a venv-mismatch in the editor, not a type issue. Check VSCode's Python interpreter points at uv's `.venv`.
- **`TypeGuard` narrows only the truthy branch.** If you write `if not is_text(p): handle(p)`, Pyright does *not* narrow `p` in that branch. Use `TypeIs` (3.13+) which narrows both branches.
- **Variance in `Protocol`.** Methods returning a generic param are covariant; methods accepting one are contravariant. Pyright will complain about variance violations — usually the fix is `T_co` or `T_contra` explicit variance.
- **Cycles in module-level types.** If `models/user.py` references `Thread` and `models/thread.py` references `User`, import cycles. Break with `if TYPE_CHECKING:` and string literals for the type, or restructure modules.
- **`Final` on a mutable container.** `x: Final[list[int]] = [1]` means the *binding* can't change — but `x.append(2)` still works. Use `tuple` or `frozenset` for true immutability.
- **`Self` in `Protocol`.** Works in 3.11+, but some older third-party stubs may not handle it. Usually fine; if Pyright complains about a third-party Protocol, file an upstream issue.
- **`NewType` lost across boundaries.** Pydantic, JSON, DB rows — when a `TenantId` round-trips through serialization, you get back a `UUID`. Re-wrap explicitly: `TenantId(row.tenant_id)`.
- **Pyright complaining about `model_validate` return type.** Sometimes Pydantic's stubs return `Self` and Pyright can't infer the subclass. Use `Model.model_validate(data)` explicitly typed: `obj: Model = Model.model_validate(data)`.
- **Decorator that wraps `async def` losing the type.** Without `ParamSpec`, the wrapped function loses its signature. Always use `ParamSpec` + `TypeVar` for decorator definitions, or PEP 695 `[**P, R]`.
- **`Literal` with a runtime-computed string.** `Literal[some_var]` doesn't work — `Literal` requires actual literal values at typecheck time. Use an `Enum` or `StrEnum` for runtime-dynamic constrained values.
- **Pydantic discriminated union without `Field(discriminator=...)`.** Pydantic *can* discriminate from `Literal` tags automatically, but adding `model_config = ConfigDict(discriminator="kind")` (or `Field(discriminator="kind")` on the union) makes it explicit and faster.

## Verification

Before declaring typing work done:

- `uv run pyright` returns zero errors and zero warnings.
- `rg "from typing import.*\b(List|Dict|Optional|Union)\b" src/` returns nothing.
- `rg "from __future__ import annotations" src/` returns nothing.
- `rg "Any\b" src/` — review every hit. Each one either has a `# pyright: ignore` with a comment, or is at a boundary with genuinely untyped input.
- `rg "# type: ignore" src/` returns nothing (we use `# pyright: ignore[rule]` if anything).
- IDs use `NewType` and Pyright catches a deliberately-swapped argument in a test.
- A discriminated union test confirms exhaustiveness: adding a new variant without handling it produces a Pyright error.
- A `Protocol` test confirms an unrelated class with the right shape satisfies it without inheritance.
- No `ABC` in `src/` unless paired with shared implementation that's actually inherited.

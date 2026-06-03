---
name: python-domain-design
description: Use when modeling business concepts — designing entities, value objects, services, aggregates. Use when deciding where logic lives (in the model vs in a service), separating domain code from infrastructure (DB, HTTP, vendors), or naming concepts to match how the business talks about them.
---

# python-domain-design

## When to use this skill

Triggered for any modeling change: adding a new business concept, deciding what's an entity vs value object, where invariants enforce, when to introduce a service, naming. Also during review — the difference between an anemic data structure and a rich domain model is exactly the kind of choice that compounds.

## Core principles

1. **Domain at the center, infrastructure at the edges.** Domain code knows about business rules. It does not import SQLAlchemy, httpx, or FastAPI. Repositories adapt between the two.
2. **Value objects: identity-less, immutable, equal-by-value.** Money, EmailAddress, DateRange, ToolName. Two `Money(USD, 100)` are equal. They have no ID, can't change in place, and validation happens at construction.
3. **Entities: identity that persists.** A `Thread` is an entity. Two threads with identical content are still two threads. Equality is by ID, not attributes. Mutation is allowed but must go through methods that enforce invariants.
4. **Aggregates: consistency boundaries.** An aggregate is a cluster of objects that change together within a single transaction. One root entity is the public face; the rest are reached through it. Cross-aggregate consistency is eventual, via events.
5. **Services for cross-entity workflows.** Logic that doesn't naturally belong on one entity lives in a service. A service coordinates entities, repositories, and integrations.
6. **Rich models over anemic ones.** If `Thread.archive()` is the rule, it lives on `Thread`. If `ThreadService.archive(thread)` is the rule and `Thread` only has data, you've separated logic from the data it protects — invariants drift.
7. **Domain language matches the business.** Use the words customers use. Naming "Thread" if customers say "Thread"; never "Conversation" or "MessageSession" because it sounded more general.
8. **Pydantic models for shape, methods for behavior.** Pydantic gives validation; methods give invariants. Both belong on the domain model.

## Always

### Value objects: `frozen=True` Pydantic models

```python
from decimal import Decimal
from pydantic import BaseModel, ConfigDict, model_validator

class Money(BaseModel):
    model_config = ConfigDict(frozen=True)
    amount: Decimal
    currency: Literal["USD", "EUR", "GBP"]

    @model_validator(mode="after")
    def non_negative(self) -> "Money":
        if self.amount < 0:
            raise ValueError("Money cannot be negative")
        return self

    def __add__(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("currency mismatch")
        return Money(amount=self.amount + other.amount, currency=self.currency)
```

Frozen + validation = a value type. Pyright treats `Money(100, "USD")` and `Money(100, "USD")` as equal. Mutation would raise.

### Entities: classes with identity and behavior

```python
@dataclass
class Thread:
    id: ThreadId
    tenant_id: TenantId
    title: str
    status: Literal["active", "archived"]
    created_at: datetime
    archived_at: datetime | None

    def archive(self, at: datetime) -> None:
        if self.status == "archived":
            raise ThreadAlreadyArchived(self.id)
        self.status = "archived"
        self.archived_at = at

    def __eq__(self, other: object) -> bool:
        return isinstance(other, Thread) and self.id == other.id

    def __hash__(self) -> int:
        return hash(self.id)
```

The entity owns the rule ("can't archive twice"). External code calls `thread.archive(now)`; it can't bypass the check by setting `status` directly (well, it can — Python — but it shouldn't, and review catches it).

Use `@dataclass` for entities (mutable, identity-based) and Pydantic frozen models for value objects. The semantics are different; the syntax should make that obvious.

### Aggregates: a root + components, one transaction

```python
@dataclass
class Thread:
    id: ThreadId
    tenant_id: TenantId
    title: str
    messages: list[Message]   # owned by the Thread aggregate
    status: ThreadStatus

    def add_message(self, msg: Message) -> None:
        if self.status == "archived":
            raise CannotAddToArchivedThread(self.id)
        if len(self.messages) >= MAX_MESSAGES_PER_THREAD:
            raise ThreadMessageLimitExceeded(self.id)
        self.messages.append(msg)
```

`Message` is reachable only through `Thread`. A repository load returns `Thread` with messages already attached. Mutations to messages go through `Thread` methods. A single DB transaction saves the whole aggregate.

A separate aggregate — `User`, `Tenant`, `Subscription` — is a separate transaction.

### Services for cross-entity / cross-aggregate workflows

```python
class ConversationService:
    def __init__(
        self,
        threads: ThreadRepository,
        agent: Agent,
        notify: NotifyDispatcher,
        clock: Clock,
    ) -> None:
        self._threads = threads
        self._agent = agent
        self._notify = notify
        self._clock = clock

    async def reply_to_message(
        self,
        thread_id: ThreadId,
        user_msg: UserMessage,
        ctx: TenantContext,
    ) -> AgentReply:
        thread = await self._threads.get(thread_id, ctx)
        thread.add_message(user_msg)
        reply = await self._agent.respond(thread, ctx)
        thread.add_message(reply)
        await self._threads.save(thread, ctx)
        await self._notify.send(...)
        return reply
```

The service orchestrates entities + integrations. It doesn't have business rules of its own — those live on the entities. It coordinates.

### Domain models stay infrastructure-free

```python
# threads/models.py — pure domain
@dataclass
class Thread:
    id: ThreadId
    title: str
    messages: list[Message]
    ...
    def archive(self, at: datetime) -> None: ...

# threads/repository.py — infrastructure adapter
class ThreadRepository:
    def __init__(self, db: AsyncSession) -> None:
        self._db = db

    async def get(self, id: ThreadId, ctx: TenantContext) -> Thread:
        row = await self._db.execute(...)
        return _row_to_thread(row)

    async def save(self, thread: Thread, ctx: TenantContext) -> None:
        await self._db.merge(_thread_to_row(thread))
```

`Thread` doesn't know about SQLAlchemy. `ThreadRepository` knows about both. The domain stays testable in isolation; the adapter handles persistence.

### Domain events for cross-aggregate reactions

```python
@dataclass(frozen=True)
class ThreadArchived:
    thread_id: ThreadId
    tenant_id: TenantId
    archived_at: datetime

class Thread:
    def archive(self, at: datetime) -> ThreadArchived:
        if self.status == "archived":
            raise ThreadAlreadyArchived(self.id)
        self.status = "archived"
        self.archived_at = at
        return ThreadArchived(self.id, self.tenant_id, at)
```

The service collects events emitted by the aggregate and publishes them after the transaction commits. Other aggregates (billing, notifications) subscribe.

### Domain errors typed (see `python-exception-handling`)

```python
# threads/errors.py
class ThreadError(Exception): ...

class ThreadNotFound(ThreadError):
    def __init__(self, thread_id: ThreadId) -> None:
        super().__init__(f"thread not found")
        self.thread_id = thread_id

class ThreadAlreadyArchived(ThreadError):
    def __init__(self, thread_id: ThreadId) -> None:
        super().__init__("thread already archived")
        self.thread_id = thread_id
```

Domain raises domain errors. The API boundary translates to HTTP. Services don't construct HTTPException.

## Never

- Importing SQLAlchemy, FastAPI, or vendor SDKs inside a domain model. *Why:* couples the domain to infrastructure. Now the model can't be tested without a DB session.
- Anemic models: `class Thread(BaseModel): id, title, status` + `ThreadService.archive(thread)` that flips `thread.status = "archived"` externally. *Why:* invariants live somewhere other than the data. They drift.
- Mutating entity state by reaching past methods (`thread.status = "archived"`). *Why:* bypasses the rule. Python doesn't enforce; the team does. Code review catches.
- Cross-aggregate updates in one transaction. *Why:* breaks the consistency boundary. Use a domain event + a follow-up handler that's eventually consistent.
- Domain code calling `await db.execute(...)`. *Why:* coupling. Repositories own DB; domain calls repository methods (or, more often, the service calls the repo and passes data to the domain).
- Naming domain concepts after CRUD operations (`UserCreateService`). *Why:* CRUD is infrastructure. Use the business name (`UserOnboarding`, `UserService`).
- Storing computed/derived values that can drift. *Why:* `thread.message_count` separate from `len(thread.messages)` will drift. Compute on read, or recompute on every mutation.
- Pydantic models that are both DB row + API response + domain model. *Why:* the three have different invariants. They look similar; they're not. Pay the duplication cost — it's small, and the alternative is field-creep across layers.
- One giant `User` entity that owns Threads, Messages, Notifications, Billing. *Why:* aggregate too big. Each becomes its own root.
- Putting infrastructure concerns in the entity (`thread.send_to_kafka()`). *Why:* domain depends on Kafka. Use a domain event; let the service publish.
- "Manager" suffix (`ThreadManager`). *Why:* what does it manage? Be specific: `ThreadRepository`, `ThreadArchiveService`. Or just use the entity's methods.
- Hexagonal/clean-architecture purity for its own sake. *Why:* if you're writing five files to enable one feature change, you've over-architected. Domain/infrastructure split should *reduce* friction, not add it.

## Pitfalls

- **Value object that should be an entity.** Sign: it has an ID, or two of them with identical content are conceptually different. Promote to entity.
- **Entity that should be a value object.** Sign: never mutated, no useful ID, equal by content. Demote to value object.
- **Aggregate root too big.** Sign: saving a single change locks the whole tree; other writes wait. Split: pull the lower-frequency parts into their own aggregate, communicate via events.
- **Aggregate root too small.** Sign: an "atomic" business operation requires two saves with no transaction. Either merge them or accept eventual consistency *deliberately*.
- **Invariants checked in two places.** Sign: the domain method checks "can't archive twice" *and* the API handler does too. Pick one (the domain). Trust the domain.
- **Pydantic model mirroring DB schema exactly.** Sign: every column is a field, with no methods. You've made a DB row, not a domain model. The repository's job is the mapping; the domain shouldn't reflect the schema 1:1.
- **Domain method that takes a DB session.** Sign: domain coupled to infrastructure. Pass data, not handles. The repository loads, the domain mutates in memory, the repository saves.
- **Events with too much data.** A domain event should carry IDs and key facts, not the whole entity. Subscribers re-load if they need more.
- **Naming drift.** Code says "Conversation," product says "Thread," support says "Chat." Pick one. Refactor everything to use it. Future joiners will thank you.
- **`@dataclass(frozen=True)` for entities.** Then mutation requires re-construction, and identity becomes fragile. Use `@dataclass` for entities; `frozen=True` only for value objects.
- **`__eq__` on entity by all fields instead of just ID.** Two threads with identical content are still different threads. Equality is by ID.
- **Domain event with a vendor type in it.** A `ThreadArchived` event carrying a `TwilioMessageSid` ties the domain to the vendor. Store the ID, not the vendor type.

## Verification

Before declaring domain work done:

- `rg "from sqlalchemy" src/threads/models.py` — and any other domain models — returns nothing. (Or wherever your domain folder is.)
- `rg "HTTPException" src/<feature>/` (domain folder) returns nothing.
- Each entity has at least one method that enforces an invariant, not just data fields with external mutation.
- Each value object is `frozen=True` and has a validator (Pydantic) or an `__post_init__` check (dataclass).
- Domain errors live in `<feature>/errors.py` and inherit from a single per-feature base.
- A test creates an entity, calls a method that should fail (e.g., archive a deleted thread), and asserts the typed error.
- A test verifies a value object is immutable: assigning a field raises (Pydantic) or is a TypeError (frozen dataclass).
- A domain event test verifies the event carries only IDs and facts, no live references to the entity or DB row.
- Repositories return domain objects, not DB rows; services accept domain objects, not DB rows.
- Aggregate boundaries are explicit in code review: "this transaction touches one aggregate root."

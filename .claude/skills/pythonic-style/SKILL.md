---
name: pythonic-style
description: Use whenever writing or reviewing Python code at the statement level — checking, iterating, formatting strings, opening files, comparing to None, destructuring values, building collections. Use when the code "works" but reads like Java/C# translated into Python. Counterpart to `python-code-architecture` (which covers structural anti-Java patterns).
---

# pythonic-style

## When to use this skill

Triggered whenever code is being written or reviewed at the statement-and-expression level. Different from `python-code-architecture` (which is about *structural* anti-Java patterns like FactoryFactory and DI containers). This skill is about *line-by-line* idioms: how you check, how you iterate, how you format, how you open a resource.

A piece of code can be perfectly architected and still read like Java. This skill stops that.

## Core principles

1. **EAFP over LBYL.** "Easier to Ask Forgiveness than Permission." Try the thing; handle the exception. Don't check-then-act in two steps that can race.
2. **Truthy/falsy is the language.** Empty collections, `None`, `0`, `""` are all falsy. Use `if items:` not `if len(items) > 0:`. Know the exceptions (see Pitfalls).
3. **Iterate the thing, not its indices.** `for x in xs` is the default. `enumerate(xs)` when you also need the index. `range(len(xs))` is a Java reflex; almost always wrong.
4. **Context managers own resources.** `with open(...) as f` owns the file's lifecycle. `try/finally: f.close()` is the same thing, longer and easier to break.
5. **Comprehensions for transforms, generators for streams, loops for side effects.** Each has one job. A comprehension with a side effect is a smell. A loop that builds a list is a missed comprehension.
6. **Destructure everything.** Tuple unpacking in returns, parameters, loop bodies, assignments. Reading `a, b = pair` is faster than reading `a = pair[0]; b = pair[1]`.
7. **Plain attributes by default; `@property` only when computed or invariant-bearing.** `get_name()` and `set_name()` are Java rituals. Python attributes *are* the interface.
8. **One way to write each thing.** f-strings for formatting, `pathlib` for paths, `is None` for None checks, `match` for variant destructuring. The alternatives exist; we use the modern one.
9. **`assert` is for development invariants, not runtime validation.** Python runs with `-O` in some prod configs and asserts vanish. If the check must happen in prod, `raise`.
10. **Modern syntax over historical.** `class Foo:` not `class Foo(object):`. `X | None` not `Optional[X]`. `list[str]` not `List[str]`.

## Always

### EAFP over LBYL

```python
# Pythonic
try:
    user = users[user_id]
except KeyError as err:
    raise UserNotFound(user_id) from err

# Java-style (LBYL — races, double-lookup)
if user_id in users:
    user = users[user_id]
else:
    raise UserNotFound(user_id)
```

```python
# Pythonic
try:
    value = int(s)
except ValueError:
    value = 0

# Java-style
if s.isdigit():
    value = int(s)
else:
    value = 0
```

EAFP is also the only correct pattern under concurrency — the `in` check and the `[]` access aren't atomic.

### Truthiness

```python
# Pythonic
if items:
    process(items)

if not data:
    raise EmptyPayload()

# Java-style
if len(items) > 0: ...
if items != [] and items is not None: ...
if data == "" or data is None: ...
```

When the empty-vs-None distinction matters, be explicit:

```python
if data is None:
    raise MissingPayload()
if not data:
    raise EmptyPayload()
```

### Iteration idioms

```python
# Pythonic
for msg in messages:
    process(msg)

for i, msg in enumerate(messages):
    log(i, msg)

for k, v in headers.items():
    write(k, v)

for a, b in zip(left, right, strict=True):
    pair(a, b)

# Java-style
for i in range(len(messages)):
    msg = messages[i]
    process(msg)

for k in headers.keys():
    v = headers[k]
    ...
```

`strict=True` on `zip` (3.10+) catches accidental length mismatch — use it whenever the lengths *should* be equal.

### Context managers for resources

```python
# Pythonic
with open(path) as f:
    data = f.read()

async with db.session() as s:
    rows = await s.execute(...)

with asyncio.timeout(10):
    result = await call()

# Java-style
f = open(path)
try:
    data = f.read()
finally:
    f.close()
```

If you find yourself writing `try/finally` to clean up, the answer is almost always a context manager. Write a `@contextmanager` if one doesn't exist.

### Comprehensions, generators, loops — pick the right one

```python
# Transform: comprehension
upper_titles = [t.title.upper() for t in threads]
by_id = {t.id: t for t in threads}
unique_models = {call.model for call in calls}

# Stream (lazy): generator expression
total_tokens = sum(c.tokens for c in calls)

# Side effects: loop
for thread in threads:
    await notify(thread)

# Java-style: loop building a list
upper_titles = []
for t in threads:
    upper_titles.append(t.title.upper())
```

Rule of thumb: if the loop's body ends in `.append`, `.add`, or `dict[k] = v`, it wants to be a comprehension.

### Destructuring

```python
# Function returning a pair
def split(name: str) -> tuple[str, str]: ...

first, last = split(full_name)

# Iterating dict
for thread_id, message_count in counts.items(): ...

# Unpacking from match
match part:
    case TextPart(text=t):     # destructure the field directly
        return t
    case ImagePart(url=u, mime=m):
        return f"<{m}@{u}>"
```

### Attributes, not getters/setters

```python
# Pythonic
class Thread:
    def __init__(self, id: ThreadId, title: str) -> None:
        self.id = id
        self.title = title

thread.title = "renamed"           # just assign

# When you need invariants or computed values, use @property
class Thread:
    @property
    def is_archived(self) -> bool:
        return self.status == "archived"

# Java-style — no!
class Thread:
    def __init__(...): self._title = title
    def get_title(self) -> str: return self._title
    def set_title(self, v: str) -> None: self._title = v
```

If a field needs validation on every set, that's a real case for `@property` + setter. Otherwise, plain attributes.

### Strings, paths, comparisons — modern only

```python
# Pythonic
msg = f"thread {tid} archived by {user_id}"
path = Path("/tmp") / "asknessie" / "logs.txt"
if value is None: ...
if not value: ...

# Historical
msg = "thread {} archived by {}".format(tid, user_id)
msg = "thread %s archived by %s" % (tid, user_id)
path = os.path.join("/tmp", "asknessie", "logs.txt")
if value == None: ...                              # ruff flags
if value != None: ...
```

### Pattern matching for sum types

```python
match part:
    case TextPart(text=t):
        return t
    case ImagePart(url=u):
        return f"<image {u}>"
    case ToolCallPart(tool=name, args=a):
        return run_tool(name, a)
    # Pyright will error if a new variant is added and not handled
```

For simple tag-dispatch, `match` reads better than a chain of `isinstance` checks.

### `assert` vs `raise`

```python
# Pythonic — runtime validation that must happen
def archive(thread: Thread) -> None:
    if thread.status == "archived":
        raise ThreadAlreadyArchived(thread.id)
    ...

# Pythonic — development-only invariant
def _internal_helper(state: dict[str, Any]) -> None:
    assert "tenant_id" in state, "tenant_id missing from state"  # never reached if our code is correct
    ...
```

`assert` disappears under `python -O`. Don't use it for input validation, auth checks, or anything a user could trigger. Use it to catch "this shouldn't be possible" bugs during development.

### Pythonic "small" wins

- `enumerate(xs, start=1)` when humans count from 1.
- `dict.get(k, default)` instead of `if k in d: ... else default`.
- `dict.setdefault(k, []).append(...)` for grouping.
- `collections.Counter(iterable)` for histograms.
- `collections.defaultdict(list)` when grouping by key.
- `any()` / `all()` for short-circuit checks instead of building bools in a loop.
- `sorted(xs, key=...)` over manual sort.
- Walrus `:=` when it removes a redundant variable:
  ```python
  if (match := pattern.search(line)) is not None:
      handle(match.group(1))
  ```
- Use `*` and `**` in calls: `f(*args)`, `f(**kwargs)`, `{**a, **b}`.
- Multiple assignment: `x = y = 0`, `a, b = b, a` for swap.

## Never

- `for i in range(len(xs)):` followed by `xs[i]`. *Why:* `for x in xs:` is the same thing, half the noise. If you need the index, `enumerate`.
- `if obj is not None and obj != "":`. *Why:* `if obj:` covers both. Be explicit only when the distinction matters semantically.
- `if x == None:` or `if x != None:`. *Why:* Pyright + ruff flag this. `None` is a singleton; `is`/`is not` is the correct check.
- `class Foo(object):`. *Why:* Python 2 leftover. All classes inherit from `object` in 3.
- `"hello {}".format(name)` or `"hello %s" % name`. *Why:* f-strings are clearer, faster, and ruff (`UP032`, `UP031`) will rewrite them.
- `os.path.join`, `os.path.exists`, `os.path.dirname`. *Why:* `pathlib.Path` is one type with methods. `Path(a) / b`, `path.exists()`, `path.parent`.
- `try: ... finally: f.close()`. *Why:* `with` is the same logic, automatic, exception-safe.
- `list(map(fn, xs))` or `list(filter(fn, xs))`. *Why:* comprehension is clearer: `[fn(x) for x in xs]`, `[x for x in xs if fn(x)]`.
- `if len(xs) == 0:` / `if len(xs) > 0:`. *Why:* `if not xs:` / `if xs:`. Truthiness.
- `dict.has_key(k)` (gone in 3, but the instinct sometimes remains as `if k in d.keys():`). *Why:* `if k in d:`.
- `get_x()` / `set_x()` pairs on a class with no invariants. *Why:* Java ritual. Use plain attributes; `@property` for computed.
- `Builder().with_x(1).with_y(2).build()`. *Why:* `Foo(x=1, y=2)` with keyword args and sensible defaults.
- Empty interface-style ABCs (`class Repository(ABC): pass` with abstract methods only). *Why:* use `Protocol`. See `python-code-architecture`, `python-typing`.
- `assert` for runtime validation (auth, user input, business rules). *Why:* `python -O` strips asserts. Use `if not ...: raise ...`.
- `lambda x: x.attr` when `attrgetter("attr")` or a simple `def` would do. *Why:* lambdas are great for one-liners passed to `key=`, `sorted`, etc. Multi-statement lambda hacks are a code smell.
- `*args, **kwargs` on a public function when you know the args. *Why:* hides the contract. Type the parameters; let Pyright check callers.
- `__init__` boilerplate when `@dataclass` or Pydantic works. *Why:* repetition, drift, no `__repr__`. Use the tools.
- `class Singleton:` patterns. *Why:* a module is a singleton. Or use DI (FastAPI `Depends`).
- Mutating a list while iterating. *Why:* skips or duplicates elements. Iterate over a copy (`for x in list(xs):`) or build a new list.
- Default mutable arguments: `def f(x: list = []):`. *Why:* same list shared across all calls. Bugs. Use `None` sentinel:
  ```python
  def f(x: list[int] | None = None) -> ...:
      if x is None:
          x = []
  ```

## Pitfalls

- **Truthiness traps.** `0` is falsy. `0.0` is falsy. `""` is falsy. `False` is falsy. `Decimal("0")` is falsy. If the value `0` is meaningful in your domain, `if value:` skips it incorrectly. Use `if value is None:` when None vs zero matters.
- **EAFP raising the wrong exception.** Catching `KeyError` to map to `UserNotFound` is fine. Catching `Exception` to do the same hides bugs. Catch narrowly — see `python-exception-handling`.
- **`with` and async.** Sync `with` doesn't work with async resources. Use `async with`. Pyright catches most of these.
- **Mutating during iteration.** `for x in xs: xs.remove(x)` is the canonical bug. Iterate over a snapshot (`for x in list(xs):`) or build the result list with a comprehension.
- **Default mutable args.** Module-level default lists/dicts get shared across calls. The `None` sentinel pattern is the fix.
- **`match` falling through unintentionally.** `match` doesn't fall through like C `switch`, but if your patterns overlap, the *first match wins*. Order patterns specific-to-general.
- **`match` exhaustiveness with non-`Literal` types.** Pattern matching against a string variable doesn't get exhaustiveness checks — only against discriminated unions with `Literal` tags. See `python-typing`.
- **`@property` accidentally expensive.** A property looks like an attribute but runs code. If callers do `obj.expensive` in a loop, they pay the cost every iteration. Cache (`functools.cached_property`) or rename to `get_expensive()` so the cost is visible.
- **`is` vs `==` on small ints/strings.** `is` checks identity. `42 is 42` happens to be True due to small-int caching, but it's not guaranteed. Use `==` for value equality, `is` for identity (and `None`).
- **f-string accidentally executing.** `f"{obj}"` calls `__str__`; if `__str__` has side effects, they happen at log time. Usually fine; occasionally surprising.
- **f-string in logging.** `logger.info(f"x={x}")` formats eagerly even if the log level skips it. Use `logger.info("event", x=x)` (structured) — see `python-logging`.
- **Tuple unpacking with mismatched lengths.** Raises `ValueError` at runtime. `zip(..., strict=True)` (3.10+) gives you the same protection earlier and clearer.
- **`pathlib.Path` and JSON.** `Path` doesn't JSON-serialize by default. Convert to `str` at the boundary.
- **Walrus inside comprehension scoping.** `[y for x in xs if (y := f(x)) > 0]` binds `y` per-iteration; reads cleanly but can confuse if you forget the scope. Use sparingly.
- **`for ... else`.** The `else` clause runs if the loop completes without `break`. Useful, but rare; if a reader has to look up the semantics, you've lost.
- **Using `dict()` instead of `{}`.** `{}` is faster and idiomatic. `dict()` is for constructing from kwargs (`dict(a=1, b=2)`).
- **`return None` at the end of a function that already returns explicitly.** Redundant; Python returns None implicitly. Ruff catches.

## Verification

Before declaring style work done — most of this is automated via ruff, so a clean `just verify` covers a lot. But verify directly:

- `rg "range\(len\(" src/` returns nothing.
- `rg "== None|!= None" src/` returns nothing.
- `rg "\.format\(|%s|%d" src/` returns nothing in f-string contexts. (`%` may appear in math; check hits.)
- `rg "os\.path\." src/` returns nothing (use `pathlib`).
- `rg "class \w+\(object\):" src/` returns nothing.
- `rg "try:\s*\n.*\nfinally:\s*\n\s*.*\.close\(\)" src/` returns nothing (use `with`).
- `rg "list\(map\(|list\(filter\(" src/` returns nothing (use comprehensions).
- `rg "if len\(.*\) (==|>|<) 0" src/` returns nothing (use truthiness).
- `rg "def get_\w+\(self\) -> .*: return self\._\w+" src/` returns nothing (plain attrs or `@property`).
- `rg "= \[\],?\s*$" src/` checked for default mutable args (false positives possible; manually review).
- `rg "@property" src/` — for each, verify it earns its keep (computed or invariant-bearing).
- Ruff rules `UP`, `SIM`, `PIE`, `RUF` all pass with zero violations.
- A code-review pass on a non-trivial PR: at least one reviewer specifically looks for line-level Java-isms.

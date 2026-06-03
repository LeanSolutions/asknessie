---
name: python-developer-experience
description: Use when setting up or modifying the Python toolchain — uv, ruff, pyright, pre-commit, the task runner, debugger, dev server, IDE integration. Use when onboarding a new app/package in the monorepo, when adding/removing dependencies, when the local dev loop feels slow or inconsistent, or when "what command do I run" isn't obvious.
---

# python-developer-experience

## When to use this skill

Triggered whenever the toolchain is the subject: pyproject.toml changes, new dependencies, lint/format/typecheck config, pre-commit hooks, the task runner, IDE settings, "how do I run X." Not for application code — that's other skills' territory.

## Core principles

1. **One tool per job, and the same job everywhere.** `uv` for everything env/dep-related. `ruff` for everything lint+format. `pyright` for typecheck. `pytest` for tests. `just` for orchestration. No alternatives running in parallel.
2. **The dev loop is one command.** `just dev` runs. `just test` tests. `just verify` is the green-bar gate before commit. If you need three commands to know if your change is good, the toolchain is broken.
3. **Local dev and CI run the same commands.** CI calls `just verify`. Pre-commit calls subsets. No "passes locally fails in CI" because of tool drift.
4. **No global Python installations.** uv manages the interpreter. `python` on PATH is irrelevant. `uv run` is the entry point.
5. **Strict from the start, loosen never.** Pyright strict, ruff with the broadest sensible ruleset, full diagnostic output. It's easier to suppress one rule than to retrofit strictness onto a permissive codebase.
6. **Fast feedback or it's not happening.** Pre-commit hooks must run under 2s on changed files. Pyright incremental, ruff is already instant. If feedback gets slow, fix it before adding rules.

## Always

### `pyproject.toml` shape

```toml
[project]
name = "asknessie-api"
version = "0.0.0"
requires-python = ">=3.13"
dependencies = [
    "fastapi",
    "pydantic[email]",
    "pydantic-settings",
    "sqlalchemy[asyncio]",
    "asyncpg",
    "alembic",
    "structlog",
    "httpx",
    "anthropic[vertex]",
    "google-cloud-aiplatform",
    "granian",
]

[dependency-groups]
dev = [
    "pytest",
    "pytest-asyncio",
    "pytest-cov",
    "respx",
    "ruff",
    "pyright",
    "polyfactory",
    "testcontainers[postgres]",
]

[tool.uv]
package = true

[tool.ruff]
line-length = 100
target-version = "py313"

[tool.ruff.lint]
select = [
    "E", "F", "W",     # pyflakes + pycodestyle
    "I",               # isort
    "B",               # bugbear
    "UP",              # pyupgrade
    "SIM",             # simplify
    "RUF",             # ruff-specific
    "ASYNC",           # async pitfalls
    "S",               # security (bandit subset)
    "TID",             # tidy imports
    "PT",              # pytest style
    "PIE",             # misc
    "N",               # naming
]
ignore = ["E501"]      # line length is the formatter's job

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101"]  # asserts are fine in tests

[tool.pyright]
include = ["src", "tests"]
pythonVersion = "3.13"
typeCheckingMode = "strict"
reportMissingImports = "error"
reportUnknownMemberType = "error"
reportUnknownVariableType = "error"

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
addopts = "-ra --strict-markers --strict-config"
```

### `.python-version`

```
3.13
```

uv reads this and pins the interpreter. No `pyenv`.

### `justfile`

```just
default: verify

install:
    uv sync

dev:
    uv run granian --interface asgi src.main:app --reload

test *args:
    uv run pytest {{args}}

test-watch:
    uv run pytest --looponfail

lint:
    uv run ruff check .

fmt:
    uv run ruff format .
    uv run ruff check --fix .

typecheck:
    uv run pyright

verify: lint typecheck test

migrate:
    uv run alembic upgrade head

migration name:
    uv run alembic revision -m "{{name}}" --autogenerate

shell:
    uv run python
```

### `.pre-commit-config.yaml`

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: local
    hooks:
      - id: pyright
        name: pyright
        entry: uv run pyright
        language: system
        types: [python]
        pass_filenames: false
```

Pyright runs on the whole project (`pass_filenames: false`) because per-file checking misses cross-file type issues.

### VSCode `settings.json` (per workspace)

```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll.ruff": "explicit",
      "source.organizeImports.ruff": "explicit"
    }
  },
  "ruff.importStrategy": "fromEnvironment",
  "python.analysis.typeCheckingMode": "strict",
  "python.analysis.useLibraryCodeForTypes": true
}
```

### Dependency commands

```sh
uv add anthropic                # runtime dep
uv add --dev pytest-mock        # dev dep
uv remove some-package          # remove
uv sync                         # install from lock
uv lock --upgrade               # upgrade locks
uv tree                         # dep tree
```

## Never

- `pip install`, `pip freeze`, `requirements.txt`. *Why:* uv is the source of truth. `uv.lock` is the lockfile. A `requirements.txt` next to it is two sources, guaranteed to drift.
- `poetry`. *Why:* one tool. uv is faster, covers the same surface, and we picked it.
- `pyenv`. *Why:* uv manages the interpreter via `.python-version`. No second version manager.
- `black` or `isort`. *Why:* ruff format replaces both, faster, single config.
- `mypy`. *Why:* pyright has better inference, faster, integrates cleaner with editors. One typechecker.
- Hand-rolled `venv` or `python -m venv`. *Why:* `uv sync` creates `.venv` automatically. Manual venvs end up with mismatched Python versions.
- Globally installed CLI tools (`pip install --user black`). *Why:* version drift across machines. Use `uv tool install` or `uvx` for ephemeral runs.
- `Makefile` as a task runner. *Why:* tab-vs-space landmines, weak Windows story. `just` is the answer.
- A CI-only quality check. *Why:* if it's not in `just verify`, devs find out it's broken at PR time. Same gates locally and in CI.
- `# noqa` without a rule code (`# noqa: B008`). *Why:* unscoped silences hide future issues. Always scope the suppression.
- `# type: ignore` without a comment explaining why. *Why:* a future reader has no idea if it's safe to fix.

## Pitfalls

- **VSCode using the wrong Python.** If autocomplete is missing imports, VSCode is on a global interpreter. Check `python.defaultInterpreterPath` points to `.venv/bin/python` and the venv exists (`uv sync` first).
- **Pyright "could not be resolved" on installed packages.** Almost always because the editor's Python interpreter ≠ uv's `.venv`. Fix the interpreter, not the import.
- **`ruff check --fix` and `ruff format` order.** Run `ruff format` *first*, then `ruff check --fix`. Reverse order can create noise. The `fmt` recipe above does this correctly.
- **Pre-commit on a 500-file change hangs.** Use `pre-commit run --files <list>` for targeted runs; `pre-commit run --all-files` is for the initial bulk fix or CI.
- **`uv.lock` merge conflicts.** Don't hand-edit. Run `uv lock` after resolving the `pyproject.toml` conflict; commit both files together.
- **`from __future__ import annotations` left over from old Python.** Not needed in 3.13. Ruff's `UP` rules flag it; remove.
- **Pyright complaining about Pydantic v2 patterns.** Pydantic ships its own pyright plugin (`pydantic.mypy` historically, now built into Pyright via Pydantic's plugin). Ensure the Pydantic version is recent enough and that Pyright sees it via `useLibraryCodeForTypes: true`.
- **`uv run` not finding the script.** Means it's not declared as a project script or the venv isn't synced. `uv sync` then retry.
- **Stale `.venv` after Python version bump.** `rm -rf .venv && uv sync`. Don't try to upgrade the existing venv.
- **Pre-commit hooks pinned to old `rev:` values.** Run `pre-commit autoupdate` periodically; review the diff before committing.

## Verification

Before declaring toolchain work done:

- `just verify` passes from a clean clone: `git clean -fdx && uv sync && just verify`.
- `pre-commit run --all-files` passes.
- A new dev can complete: `git clone`, install uv (`curl -LsSf https://astral.sh/uv/install.sh | sh`), `cd apps/api`, `just install`, `just dev` — and have a running server in under 3 minutes.
- No `pip`, `poetry`, `pyenv`, `mypy`, `black`, `isort` referenced anywhere in the repo (`rg -i "pip install|poetry|pyenv|mypy|black|isort" --type-not lock`).
- `pyright` runs in under 5s incrementally on small changes (warm cache). If it's slower, investigate before adding rules.
- VSCode opens the project, picks up the uv venv automatically, shows ruff diagnostics on save.

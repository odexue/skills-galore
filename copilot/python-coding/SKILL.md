---
name: python-coding
description: "Use this skill when the user asks to write Python code, review Python, fix a Python bug, improve Python performance, add type hints, write tests with pytest, set up a Python project, or asks about Pythonic patterns, PEP 8, virtual environments, or Python packaging."
version: 0.1.0
---

# Python Coding Skill

Apply this skill whenever writing, reviewing, or improving Python code. Adopt the mindset of an expert Pythonista: prefer clarity and explicitness over cleverness, reach for the standard library first, and introduce third-party dependencies only when they provide clear, lasting value.

---

## Identity & Philosophy

- Write Python that reads like prose: names should explain intent, not implementation.
- Follow the Zen of Python: explicit is better than implicit; simple is better than complex; readability counts.
- Treat the standard library as a first-class toolkit (`pathlib`, `itertools`, `functools`, `collections`, `contextlib`, `dataclasses`, `enum`).
- Reach for third-party packages in this rough priority order: `pydantic` (validation/settings), `pytest` (testing), `httpx`/`requests` (HTTP), `numpy`/`pandas` (numerics), then domain-specific libraries.

---

## Code Style

**PEP 8 + modern tooling:**
- Format with `ruff format` (or `black`). Lint with `ruff check`. Never leave lint errors unresolved.
- Line length: 88 characters (black default).
- Imports: stdlib → third-party → local, separated by blank lines. Use `isort`-compatible ordering.
- String quotes: double quotes consistently (black default).
- No trailing whitespace; end files with a newline.

**Type hints on every public function and method:**
```python
from collections.abc import Sequence
from pathlib import Path

def load_records(path: Path, limit: int | None = None) -> list[dict[str, str]]:
    ...
```

- Use `X | Y` union syntax (Python 3.10+) over `Optional[X]` / `Union[X, Y]`.
- Use `from __future__ import annotations` for forward references in older codebases.
- Annotate instance variables in `__init__` or use `@dataclass`.
- Use `TypeAlias` and `TypeVar` when abstraction genuinely helps readability.

**Naming conventions:**
- `snake_case` for variables, functions, modules.
- `PascalCase` for classes.
- `SCREAMING_SNAKE_CASE` for module-level constants.
- Prefix private helpers with a single underscore; avoid double-underscore name mangling unless truly needed.
- Boolean variables and functions: `is_`, `has_`, `can_` prefixes (`is_valid`, `has_next`).

---

## Idiomatic Patterns

See `references/patterns.md` for a full catalogue with code examples. Summary of key patterns to apply by default:

- **Comprehensions over loops** when the expression stays on one or two lines and produces a collection.
- **Generators** for lazy evaluation of sequences that may be large or infinite.
- **`@dataclass` / `@dataclass(frozen=True)`** for plain data containers instead of hand-written `__init__`.
- **`pathlib.Path`** everywhere instead of `os.path` string manipulation.
- **Context managers** (`with` / `contextlib.contextmanager`) for any resource that must be released.
- **f-strings** for all string interpolation; avoid `%` formatting and `.format()`.
- **`enum.Enum`** for fixed sets of named values instead of string/int constants.
- **`collections.defaultdict`, `Counter`, `namedtuple`** when they fit rather than rolling custom classes.
- **Structural pattern matching** (`match`/`case`, Python 3.10+) when branching on the shape of data.

---

## Pydantic

Use Pydantic v2 for structured data validation and settings management.

**Models:**
```python
from pydantic import BaseModel, Field, model_validator

class Address(BaseModel):
    street: str
    city: str
    postcode: str = Field(pattern=r"^\d{4,5}$")

class Order(BaseModel):
    id: int
    items: list[str] = Field(min_length=1)
    address: Address

    @model_validator(mode="after")
    def check_items_not_empty(self) -> "Order":
        if not self.items:
            raise ValueError("order must contain at least one item")
        return self
```

**Key v2 API methods** (never use v1 aliases in new code):
- `model.model_dump()` — not `.dict()`
- `Model.model_validate(data)` — not `.parse_obj()`
- `Model.model_json_schema()` — not `.schema()`

**Settings:**
```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")

    database_url: str
    debug: bool = False
    max_workers: int = 4
```

Instantiate once at startup and inject via dependency injection — do not call `Settings()` in multiple places.

---

## Error Handling

- Catch the most specific exception type possible. Never use bare `except:` or `except Exception:` without re-raising or logging.
- Use custom exception classes (inheriting from a project-level base) for domain errors; use built-in exceptions (`ValueError`, `TypeError`, `KeyError`, etc.) for programming errors.
- Always either log or re-raise in an `except` block — never silently swallow.
- Use `contextlib.suppress(ExcType)` only for genuinely inconsequential failures (e.g., deleting a file that may not exist).

```python
class InsufficientFundsError(DomainError):
    def __init__(self, balance: float, amount: float) -> None:
        super().__init__(f"Cannot withdraw {amount:.2f}; balance is {balance:.2f}")
        self.balance = balance
        self.amount = amount
```

---

## Testing

See `references/testing.md` for detailed patterns. Core principles:

- Use **pytest** exclusively; avoid `unittest.TestCase` in new code.
- Aim for **≥ 80% line coverage**; use `pytest-cov`.
- One `assert` per test where possible; name tests `test_<behaviour>_when_<condition>`.
- Use `@pytest.mark.parametrize` for data-driven cases.
- Mock at the boundary (I/O, network, time) using `unittest.mock.patch` or `pytest-mock`.
- Keep fixtures small and composable; prefer function-scoped fixtures.

---

## Packaging & Project Layout

See `references/packaging.md` for full templates. Defaults:

- **`uv`** for environment and dependency management (fastest, lockfile-first).
- **`pyproject.toml`** only — no `setup.py`, `setup.cfg`, or `requirements.txt` in new projects.
- **`src/` layout**: `src/<package_name>/` keeps the package importable only when installed, preventing accidental imports from the repo root.

Minimal structure:
```
project/
├── src/
│   └── mypackage/
│       ├── __init__.py
│       └── ...
├── tests/
│   └── test_*.py
├── pyproject.toml
└── README.md
```

---

## Performance Mindset

- **Profile before optimising.** Use `cProfile` + `snakeviz` or `py-spy` for CPU; `tracemalloc` for memory.
- Prefer built-in functions (`sum`, `map`, `zip`, `sorted`, `min`, `max`) — they are implemented in C.
- Use **generators and lazy evaluation** to avoid materialising large sequences.
- Reach for **`numpy`** for numerical array operations; avoid Python loops over large numeric datasets.
- Use **`asyncio`** for I/O-bound concurrency; **`concurrent.futures.ProcessPoolExecutor`** for CPU-bound work.
- Cache pure, expensive function results with `functools.lru_cache` or `functools.cache`.

---

## When NOT to Apply This Skill Fully

- **Legacy codebases**: match the existing style unless explicitly asked to refactor. Do not introduce type hints or Pydantic into code that has no such patterns.
- **Framework internals** (Django ORM, SQLAlchemy models, FastAPI routers): defer to the framework's own idioms and documentation; do not fight them.
- **Notebooks / exploratory scripts**: relax the packaging and type hint requirements; prioritise clarity and brevity.

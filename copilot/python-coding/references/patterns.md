# Python Idiomatic Patterns Reference

A catalogue of patterns to apply when writing Python. Each section shows a preferred form and, where relevant, what to avoid.

---

## Comprehensions

**List comprehension** (preferred over `for` + `append`):
```python
# good
squares = [x * x for x in range(10) if x % 2 == 0]

# avoid
squares = []
for x in range(10):
    if x % 2 == 0:
        squares.append(x * x)
```

**Dict comprehension:**
```python
word_lengths = {word: len(word) for word in words}
```

**Set comprehension:**
```python
unique_domains = {email.split("@")[1] for email in emails}
```

**Avoid nested comprehensions** beyond two levels — extract into a function or use a loop for readability.

---

## Generators

Use generators when the full sequence is not needed simultaneously:
```python
def read_chunks(path: Path, size: int = 4096):
    with path.open("rb") as f:
        while chunk := f.read(size):
            yield chunk

# Generator expression (lazy list comprehension)
total = sum(len(line) for line in path.open())
```

Use `yield from` to delegate to a sub-generator:
```python
def flatten(nested: list) -> Generator[int, None, None]:
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)
        else:
            yield item
```

---

## Dataclasses

Prefer `@dataclass` over hand-written classes for data containers:
```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class Event:
    name: str
    timestamp: datetime
    tags: list[str] = field(default_factory=list)
    metadata: dict[str, str] = field(default_factory=dict)

@dataclass(frozen=True)   # immutable, hashable
class Point:
    x: float
    y: float
```

Use `__post_init__` for derived fields or validation:
```python
@dataclass
class Circle:
    radius: float

    def __post_init__(self) -> None:
        if self.radius <= 0:
            raise ValueError(f"radius must be positive, got {self.radius}")
```

---

## Pathlib

Replace all `os.path` usage with `pathlib.Path`:
```python
from pathlib import Path

config_dir = Path.home() / ".config" / "myapp"
config_dir.mkdir(parents=True, exist_ok=True)

data_file = config_dir / "data.json"
text = data_file.read_text(encoding="utf-8")
lines = data_file.read_text().splitlines()

for csv_file in Path("data").glob("**/*.csv"):
    process(csv_file)
```

---

## Context Managers

For resources that must be released (files, locks, DB connections):
```python
# Built-in
with open(path, encoding="utf-8") as f:
    content = f.read()

# Multiple contexts on one line
with src.open() as inp, dst.open("w") as out:
    out.write(inp.read())
```

Create reusable context managers with `contextlib.contextmanager`:
```python
from contextlib import contextmanager
import tempfile

@contextmanager
def temp_directory():
    with tempfile.TemporaryDirectory() as tmpdir:
        yield Path(tmpdir)
```

Use `contextlib.suppress` for inconsequential exceptions:
```python
from contextlib import suppress

with suppress(FileNotFoundError):
    cache_file.unlink()
```

---

## Enums

Use `enum.Enum` instead of string/int constants:
```python
from enum import Enum, auto

class Status(str, Enum):
    PENDING = "pending"
    RUNNING = "running"
    DONE = "done"
    FAILED = "failed"

class Direction(Enum):
    NORTH = auto()
    SOUTH = auto()
    EAST = auto()
    WEST = auto()
```

Inheriting from `str` allows direct comparison with string values from JSON/APIs.

---

## Collections

**`defaultdict`** — avoid key-existence checks:
```python
from collections import defaultdict

by_category: dict[str, list[str]] = defaultdict(list)
for item in items:
    by_category[item.category].append(item.name)
```

**`Counter`** — frequency counting:
```python
from collections import Counter

freq = Counter(words)
top_5 = freq.most_common(5)
```

**`namedtuple` / `typing.NamedTuple`** — lightweight immutable record:
```python
from typing import NamedTuple

class Coordinate(NamedTuple):
    lat: float
    lon: float
    alt: float = 0.0
```

**`deque`** — O(1) append/pop from both ends:
```python
from collections import deque

recent = deque(maxlen=100)   # automatically discards old entries
```

---

## Functools

**`lru_cache` / `cache`** — memoise pure functions:
```python
from functools import cache

@cache
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
```

**`partial`** — bind arguments to create specialised callables:
```python
from functools import partial

double = partial(operator.mul, 2)
values = list(map(double, range(10)))
```

**`reduce`** — fold a sequence (use sparingly; explicit loops are often clearer):
```python
from functools import reduce
import operator

product = reduce(operator.mul, numbers, 1)
```

---

## Structural Pattern Matching (Python 3.10+)

Use `match`/`case` when branching on the shape or type of data:
```python
def handle_event(event: dict) -> None:
    match event:
        case {"type": "click", "x": x, "y": y}:
            handle_click(x, y)
        case {"type": "key", "key": str(k)}:
            handle_key(k)
        case {"type": unknown}:
            raise ValueError(f"unknown event type: {unknown!r}")
```

---

## Walrus Operator (`:=`)

Use for `while` loops reading until exhaustion, and for avoiding repeated calls:
```python
# Reading chunks
while chunk := file.read(8192):
    process(chunk)

# Avoid calling expensive function twice
if result := expensive_computation(x):
    use(result)
```

---

## Pydantic v2 Patterns

**Basic model:**
```python
from pydantic import BaseModel, Field, field_validator

class Product(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    price: float = Field(gt=0)
    sku: str

    @field_validator("sku")
    @classmethod
    def sku_must_be_uppercase(cls, v: str) -> str:
        return v.upper()
```

**Nested models and serialisation:**
```python
class Cart(BaseModel):
    items: list[Product]
    discount: float = 0.0

cart = Cart.model_validate({"items": [...], "discount": 0.1})
data = cart.model_dump()          # dict
json_str = cart.model_dump_json() # JSON string
```

**Settings from environment:**
```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class AppSettings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_prefix="APP_",
        case_sensitive=False,
    )

    api_key: str
    timeout_seconds: int = 30
    allowed_origins: list[str] = []

settings = AppSettings()
```

**Custom types with `Annotated`:**
```python
from typing import Annotated
from pydantic import AfterValidator

def must_be_positive(v: float) -> float:
    if v <= 0:
        raise ValueError("must be positive")
    return v

PositiveFloat = Annotated[float, AfterValidator(must_be_positive)]
```

---

## Async Patterns

Use `asyncio` for I/O-bound concurrency:
```python
import asyncio
import httpx

async def fetch_all(urls: list[str]) -> list[str]:
    async with httpx.AsyncClient() as client:
        tasks = [client.get(url) for url in urls]
        responses = await asyncio.gather(*tasks)
        return [r.text for r in responses]
```

Use `async with` and `async for` with async context managers and iterators. Avoid `asyncio.sleep(0)` hacks — redesign instead.

---

## Logging

Use `logging` from the standard library; never `print` in library/production code:
```python
import logging

logger = logging.getLogger(__name__)

def process(data: list[str]) -> None:
    logger.debug("processing %d records", len(data))
    try:
        ...
    except ValueError as exc:
        logger.exception("failed to process record: %s", exc)
        raise
```

Configure logging once at the application entry point, not inside library modules.

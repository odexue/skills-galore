# Python Testing Reference

Testing patterns using pytest. Apply these when writing or improving tests.

---

## Project Setup

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--strict-markers -q"

[tool.coverage.run]
source = ["src"]
branch = true

[tool.coverage.report]
fail_under = 80
show_missing = true
```

Run tests:
```bash
uv run pytest
uv run pytest --cov                  # with coverage
uv run pytest -x                     # stop on first failure
uv run pytest -k "test_login"        # filter by name
uv run pytest tests/test_auth.py     # single file
```

---

## Naming Conventions

- Test files: `tests/test_<module>.py`
- Test functions: `test_<behaviour>_when_<condition>`
- Example: `test_withdraw_raises_when_insufficient_funds`

---

## Basic Structure

```python
def test_add_returns_sum_of_two_integers() -> None:
    result = add(2, 3)
    assert result == 5
```

One logical assertion per test. Multiple `assert` lines are acceptable when they all verify one outcome (e.g., checking multiple fields of a returned object).

---

## Parametrize

Use `@pytest.mark.parametrize` to avoid duplicated test logic:
```python
import pytest

@pytest.mark.parametrize("a, b, expected", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
    (100, -50, 50),
])
def test_add(a: int, b: int, expected: int) -> None:
    assert add(a, b) == expected
```

For exception cases:
```python
@pytest.mark.parametrize("value", [-1, 0, -100])
def test_sqrt_raises_for_non_positive(value: float) -> None:
    with pytest.raises(ValueError, match="must be positive"):
        sqrt(value)
```

---

## Fixtures

Define reusable test state in `conftest.py`:
```python
# tests/conftest.py
import pytest
from myapp.db import Database

@pytest.fixture
def db() -> Database:
    """In-memory database for each test."""
    database = Database(":memory:")
    database.migrate()
    yield database
    database.close()

@pytest.fixture
def user(db: Database) -> User:
    return db.users.create(name="Alice", email="alice@example.com")
```

**Scope** — use the narrowest scope that works:
| Scope | When to use |
|-------|-------------|
| `function` (default) | Isolated per test — safe for stateful resources |
| `module` | Expensive setup shared within one test file |
| `session` | Very expensive one-time setup (e.g., Docker container) |

---

## Mocking

Use `pytest-mock` (wraps `unittest.mock`) for cleaner syntax:
```python
def test_send_email_calls_smtp(mocker) -> None:
    mock_smtp = mocker.patch("myapp.mail.smtplib.SMTP")
    send_welcome_email("user@example.com")
    mock_smtp.return_value.__enter__.return_value.sendmail.assert_called_once()
```

Patch at the **import site**, not the definition site:
```python
# myapp/notifier.py imports `requests`
mocker.patch("myapp.notifier.requests.post", return_value=Mock(status_code=200))
```

Freeze time with `freezegun`:
```python
from freezegun import freeze_time

@freeze_time("2025-01-15 12:00:00")
def test_token_expiry() -> None:
    token = create_token(ttl_seconds=3600)
    assert token.expires_at.hour == 13
```

---

## Testing Exceptions

```python
def test_withdraw_raises_insufficient_funds() -> None:
    account = Account(balance=10.0)
    with pytest.raises(InsufficientFundsError) as exc_info:
        account.withdraw(50.0)
    assert exc_info.value.balance == 10.0
    assert exc_info.value.amount == 50.0
```

---

## Testing Async Code

```python
import pytest

@pytest.mark.asyncio
async def test_fetch_user_returns_data(mocker) -> None:
    mocker.patch("myapp.client.httpx.AsyncClient.get", return_value=Mock(json=lambda: {"id": 1}))
    user = await fetch_user(1)
    assert user["id"] == 1
```

Configure in `pyproject.toml`:
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

---

## Pydantic Model Testing

```python
import pytest
from pydantic import ValidationError

def test_product_rejects_negative_price() -> None:
    with pytest.raises(ValidationError) as exc_info:
        Product(name="Widget", price=-1.0, sku="W001")
    errors = exc_info.value.errors()
    assert any(e["loc"] == ("price",) for e in errors)

def test_product_upcases_sku() -> None:
    product = Product(name="Widget", price=9.99, sku="w001")
    assert product.sku == "W001"
```

---

## Property-Based Testing with Hypothesis

Use Hypothesis for edge cases that are hard to enumerate manually:
```python
from hypothesis import given, strategies as st

@given(st.integers(), st.integers())
def test_add_is_commutative(a: int, b: int) -> None:
    assert add(a, b) == add(b, a)

@given(st.text(min_size=1))
def test_slug_is_lowercase(text: str) -> None:
    assert slugify(text) == slugify(text).lower()
```

---

## Coverage

```bash
uv run pytest --cov=src --cov-report=term-missing --cov-report=html
```

Target ≥ 80% line coverage. Focus on:
- All happy paths
- All documented error/exception paths
- Edge cases: empty inputs, boundary values, None

Do not chase 100% coverage at the cost of test quality — untestable defensive code is a code smell.

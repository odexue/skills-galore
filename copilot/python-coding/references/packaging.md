# Python Packaging & Project Setup Reference

All new Python projects use `uv` + `pyproject.toml` + `src/` layout.

---

## Quick Start with uv

```bash
# Install uv (if not already installed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create a new project
uv init my-project
cd my-project

# Add dependencies
uv add pydantic pydantic-settings
uv add --dev pytest pytest-cov pytest-mock ruff

# Run code
uv run python src/my_project/main.py

# Run tests
uv run pytest

# Sync environment from lockfile
uv sync
```

---

## Project Structure (src/ layout)

```
my-project/
├── src/
│   └── my_project/
│       ├── __init__.py
│       ├── main.py
│       └── ...
├── tests/
│   ├── conftest.py
│   └── test_*.py
├── pyproject.toml
├── uv.lock              # commit to version control
└── README.md
```

The `src/` layout prevents accidental imports from the repo root and forces the package to be installed before it can be imported — catching packaging errors early.

---

## pyproject.toml Template

```toml
[project]
name = "my-project"
version = "0.1.0"
description = "Short description"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "pydantic>=2.0",
    "pydantic-settings>=2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-cov>=5.0",
    "pytest-mock>=3.0",
    "ruff>=0.4",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/my_project"]

[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]
ignore = []

[tool.ruff.format]
quote-style = "double"

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

---

## Dependency Management

```bash
# Add a runtime dependency
uv add httpx

# Add a development-only dependency
uv add --dev black

# Add with version constraint
uv add "pydantic>=2.5,<3"

# Remove a dependency
uv remove httpx

# Upgrade all dependencies
uv lock --upgrade

# Show installed packages
uv pip list
```

Always commit `uv.lock`. It ensures reproducible installs across machines and CI.

---

## Virtual Environments

`uv` manages virtual environments automatically in `.venv/`. No manual activation needed when using `uv run`.

```bash
# Run a command in the project's venv
uv run python script.py
uv run pytest

# Activate manually (when needed for IDE or shell use)
source .venv/bin/activate   # macOS/Linux
.venv\Scripts\activate      # Windows
```

---

## Common uv Commands

| Command | Purpose |
|---------|---------|
| `uv init <name>` | Create a new project |
| `uv add <pkg>` | Add a dependency |
| `uv add --dev <pkg>` | Add a dev dependency |
| `uv remove <pkg>` | Remove a dependency |
| `uv sync` | Install all deps from lockfile |
| `uv lock` | Regenerate lockfile |
| `uv run <cmd>` | Run a command in the project env |
| `uv pip install -e .` | Editable install (for IDE support) |
| `uv build` | Build wheel + sdist |
| `uv publish` | Publish to PyPI |

---

## Ruff Configuration

Ruff replaces `flake8`, `isort`, `pyupgrade`, and partially `pylint` in one fast tool.

```bash
# Format code
uv run ruff format .

# Lint
uv run ruff check .

# Lint and auto-fix
uv run ruff check --fix .
```

Recommended lint rules to enable (`select` in `pyproject.toml`):
- `E`, `F` — pycodestyle + pyflakes (baseline)
- `I` — isort (import ordering)
- `UP` — pyupgrade (modernise syntax)
- `B` — flake8-bugbear (likely bugs)
- `SIM` — flake8-simplify (cleaner patterns)

---

## Pre-commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

```bash
uv add --dev pre-commit
uv run pre-commit install
```

---

## CI with GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --all-extras
      - run: uv run ruff check .
      - run: uv run ruff format --check .
      - run: uv run pytest --cov
```

---

## Publishing to PyPI

```bash
# Build
uv build

# Publish (requires PyPI token)
uv publish

# Or publish to TestPyPI first
uv publish --publish-url https://test.pypi.org/legacy/
```

Set the token via environment variable:
```bash
export UV_PUBLISH_TOKEN=pypi-...
```

---

## Python Version Management

```bash
# Install a specific Python version
uv python install 3.12

# Pin project to a Python version
uv python pin 3.12

# List available versions
uv python list
```

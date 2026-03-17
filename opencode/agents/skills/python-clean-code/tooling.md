# Python Development Tooling (2024-2025)

## Package Manager: uv (Recommended)

Fast Rust-based package manager replacing pip, poetry, pipx.

### Installation
```bash
# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# Linux/macOS
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### Basic Usage
```bash
# Initialize project
uv init my-project
cd my-project

# Add dependencies
uv add requests pandas
uv add --dev pytest ruff mypy

# Install from requirements.txt
uv pip install -r requirements.txt

# Run with auto-sync
uv run python script.py

# Create virtual environment
uv venv
uv venv --python 3.11
```

## Linter + Formatter: Ruff (Recommended)

Replaces Black, Flake8, isort in one fast tool.

### Installation
```bash
uv add --dev ruff
# or
pip install ruff
```

### Usage
```bash
# Check code
ruff check .

# Fix auto-fixable issues
ruff check --fix .

# Format code (like Black)
ruff format .
```

### pyproject.toml Configuration
```toml
[tool.ruff]
target-version = "py310"
line-length = 100

[tool.ruff.lint]
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort
    "B",   # flake8-bugbear
    "C4",  # flake8-comprehensions
    "UP",  # pyupgrade
    "N",   # pep8-naming
    "SIM", # flake8-simplify
]
ignore = [
    "E501",  # line too long (handled by formatter)
]

[tool.ruff.format]
quote-style = "double"
indent-style = "space"

[tool.ruff.lint.isort]
known-first-party = ["my_package"]

[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401"]  # unused imports OK
"tests/**/*.py" = ["S101"]  # assert OK in tests
```

## Type Checker: mypy

### Installation
```bash
uv add --dev mypy
pip install mypy
```

### Usage
```bash
mypy src/
mypy src/ --strict
```

### pyproject.toml Configuration
```toml
[tool.mypy]
python_version = "3.10"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
no_implicit_optional = true
warn_redundant_casts = true
warn_unused_ignores = true
strict_equality = true

[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false
```

## Pre-commit Hooks

### Installation
```bash
uv add --dev pre-commit
pre-commit install
```

### .pre-commit-config.yaml
```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.7.0
    hooks:
      - id: mypy
        additional_dependencies: [types-requests]
        args: [--ignore-missing-imports]

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-toml
      - id: check-added-large-files
```

### Usage
```bash
# Run all hooks on all files
pre-commit run --all-files

# Run specific hook
pre-commit run ruff --all-files
```

## Project Structure (src Layout)

```
my-project/
├── src/
│   └── my_package/
│       ├── __init__.py
│       ├── module1.py
│       └── module2.py
├── tests/
│   ├── __init__.py
│   ├── test_module1.py
│   └── conftest.py
├── .pre-commit-config.yaml
├── pyproject.toml
├── README.md
└── LICENSE
```

## Complete pyproject.toml Template

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "my-package"
version = "0.1.0"
description = "My package description"
readme = "README.md"
requires-python = ">=3.10"
license = {text = "MIT"}
authors = [{name = "Your Name", email = "you@example.com"}]

dependencies = [
    "requests>=2.31.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0.0",
    "pytest-cov>=4.0.0",
    "ruff>=0.1.0",
    "mypy>=1.0.0",
    "pre-commit>=3.0.0",
]

[tool.setuptools.packages.find]
where = ["src"]

[tool.ruff]
target-version = "py310"
line-length = 100

[tool.ruff.lint]
select = ["E", "W", "F", "I", "B", "C4", "UP"]

[tool.mypy]
python_version = "3.10"
disallow_untyped_defs = true
```

## VSCode/Cursor Settings

```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
  "python.languageServer": "Pylance",
  "python.analysis.typeCheckingMode": "basic",
  
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll": "explicit",
      "source.organizeImports": "explicit"
    }
  },
  
  "ruff.enable": true,
  "python.testing.pytestEnabled": true,
  "python.testing.pytestArgs": ["tests"]
}
```

## GitHub Actions CI

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: astral-sh/setup-uv@v3
      
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      
      - name: Install dependencies
        run: |
          uv venv
          source .venv/bin/activate
          uv pip install -e ".[dev]"
      
      - name: Lint
        run: |
          source .venv/bin/activate
          ruff check .
          ruff format --check .
      
      - name: Type check
        run: |
          source .venv/bin/activate
          mypy src/
      
      - name: Test
        run: |
          source .venv/bin/activate
          pytest tests/ --cov=src
```

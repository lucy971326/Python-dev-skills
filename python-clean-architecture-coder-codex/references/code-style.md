# Python Code Style Standard

Use this reference when writing or reviewing Python code.

## Naming

| Item | Style | Example |
| --- | --- | --- |
| Class | `PascalCase` | `UserService`, `TaskRepository` |
| Function or variable | `snake_case` | `get_user`, `task_list` |
| Constant | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| File | `snake_case.py` | `task_controller.py` |
| Test file | `test_*.py` | `test_task_service.py` |
| Type-only helper file | `*_type.py` only when useful | `task_type.py` |

Avoid camelCase, unclear abbreviations, and names such as `tmp`, `data`, or `value` when a domain name is available.

## File Shape

Order Python files from top to bottom:

1. Imports.
2. Type definitions, Protocols, dataclasses, enums.
3. Functions and classes.
4. `if __name__ == "__main__":` block, only when needed.

Keep a file under about 300 lines. Split earlier if it mixes layers or responsibilities.

## Imports

Group imports in this order:

1. Standard library.
2. Third-party libraries.
3. Project imports.

Prefer absolute imports that match the existing project convention.

```python
from collections.abc import Sequence
from typing import Protocol

from fastapi import APIRouter
from pydantic import BaseModel

from entities.task import Task
from usecases.task_services import CreateTaskUseCase
```

Avoid relative imports such as `from ..services.user import UserService` unless the project already standardizes on them.

## Typing

Use type hints for public functions, methods, constructors, and tests.

```python
def greet(name: str) -> str:
    return f"Hello, {name}"
```

Use `Protocol` for dependency contracts consumed by use cases. If the dependency performs I/O in a FastAPI request path, make the Protocol async and keep the use case async too.

```python
class LLMProvider(Protocol):
    async def generate(self, prompt: str) -> str: ...
```

Use `@dataclass` or small value objects for stable domain data when no framework model is required.

```python
from dataclasses import dataclass


@dataclass
class Task:
    id: str
    title: str
    completed: bool = False
```

Do not use `Any` or `# type: ignore` unless there is a concrete reason. If a temporary ignore is necessary, add the narrowest ignore and a short reason.

## Formatting

Use the configured formatter, normally `ruff format`, instead of hand-aligning code.

General rules:

- Line length: 88 unless the project config says otherwise.
- One blank line between top-level functions when formatted that way by the tool.
- Two blank lines between top-level classes/functions if the formatter applies it.
- Operators and function parameters use normal spaces.
- Avoid negative or unusual formatting that fights the formatter.

## Docstrings and Comments

Use Google-style docstrings for public functions or classes whose purpose is not obvious.

```python
def calculate_tax(amount: Decimal, rate: Decimal) -> Decimal:
    """Calculate tax for a positive amount.

    Args:
        amount: Base amount.
        rate: Tax rate such as Decimal("0.10").

    Returns:
        Calculated tax.

    Raises:
        ValueError: If amount is negative.
    """
    if amount < 0:
        raise ValueError("金额不能为负")
    return amount * rate
```

Prefer clear names over comments. Add comments only to explain non-obvious decisions, not to narrate obvious code.

## Errors

Raise explicit errors and catch explicit errors.

```python
if not title.strip():
    raise ValidationError("标题不能为空")
```

Avoid:

```python
try:
    risky_operation()
except:
    pass
```

If catching a broad exception at a boundary, log useful context and re-raise or translate to a safe boundary response.

## Constants

Move repeated values and meaningful thresholds to constants.

```python
MAX_RETRY_COUNT = 3
DEFAULT_PAGE_SIZE = 20
```

Avoid unexplained magic numbers in conditions or call sites.

## Commit Messages

Use concise conventional prefixes:

```text
feat: add task completion
fix: handle missing task
docs: update architecture guide
style: format code
refactor: split task controller
test: add create task tests
chore: update tooling
```

Write the message in the language that matches the repository's existing commit history or the user's preference.

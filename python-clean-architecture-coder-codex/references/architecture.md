# Clean Architecture Development Standard

Use this reference as the main development contract. It merges the architecture, testing, error-handling, and logging documents into one set of rules for AI-generated Python application code.

## Layer Model

```text
adapters      -> receive external input and translate protocols
usecases      -> execute business actions and orchestrate dependencies
entities      -> hold stable core data and domain behavior
frameworks    -> implement external systems such as DB, HTTP infra, LLMs

Dependency direction: outer layers may depend inward. Inner layers must not depend outward.
```

Layer responsibilities:

| Layer | Responsibility | Examples |
| --- | --- | --- |
| `entities` | Stable domain data, invariants, simple behavior, business error types | `Task`, `User`, `Document`, `BusinessError` |
| `usecases` | Application actions and workflow orchestration | `CreateTaskUseCase`, `RetrieveDocsUseCase` |
| `adapters` | Entry points and format conversion | FastAPI routers, CLI handlers, request/response schemas |
| `frameworks` | Concrete external implementations | database repositories, OpenAI/LLM adapters, ChromaDB, logging, HTTP middleware |
| composition root | Wire concrete implementations into use cases | `main.py`, app factory, dependency overrides |

## Dependency Rules

Allowed examples:

```text
adapters -> usecases -> entities
frameworks -> usecases Protocols and entities
main.py -> every layer for wiring
```

Forbidden examples:

```text
entities -> usecases
entities -> adapters
usecases -> adapters
usecases -> frameworks.database.mysql_repo
usecases -> FastAPI, Pydantic request schemas, SQLAlchemy sessions, concrete SDK clients
```

`main.py` or an app factory is the exception: it may know every layer because its job is assembly.

## Protocol Placement

Define Protocols where they are consumed, not where they are implemented.

```text
Use case needs to store data   -> define Repository Protocol in usecases
Use case needs to call an LLM   -> define LLMProvider Protocol in usecases
Use case needs a clock or IDs   -> define Clock/IdGenerator Protocol in usecases
```

Concrete implementations live in `frameworks` and implement those Protocols implicitly.

```python
# usecases/task_repository.py
from typing import Protocol

from entities.task import Task


class TaskRepository(Protocol):
    async def save(self, task: Task) -> Task: ...
    async def find_by_id(self, task_id: str) -> Task | None: ...
```

Use sync Protocol methods only when the dependency is genuinely sync and the surrounding application is sync-first.

## Sync and Async

Choose sync or async at the architectural boundary, then keep the call chain consistent.

For FastAPI projects, prefer async use cases when a request path reaches I/O-bound dependencies such as databases, HTTP clients, LLM providers, queues, caches, or file stores:

```python
class CreateTaskUseCase:
    def __init__(self, task_repo: TaskRepository):
        self._repo = task_repo

    async def execute(self, title: str) -> Task:
        if not title.strip():
            raise ValidationError("标题不能为空")
        task = Task(title=title.strip())
        return await self._repo.save(task)
```

Controller endpoints should await async use cases:

```python
@router.post("", response_model=TaskResponseSchema)
async def create_task(
    schema: TaskCreateSchema,
    usecase: CreateTaskUseCase = Depends(get_create_usecase),
) -> TaskResponseSchema:
    task = await usecase.execute(schema.title)
    return _to_response(task)
```

Sync is acceptable when:

- The use case is pure CPU/domain logic and calls no async dependency.
- The existing project is consistently sync and uses sync framework APIs.
- The dependency is a small in-memory implementation used for tests or local demos.

Async rules:

- Do not call blocking database, network, or SDK clients directly inside `async def`.
- Do not use `asyncio.run()` inside FastAPI request handlers or use cases.
- Do not make an API async only for appearance. Async should reflect awaitable dependencies or a framework contract.
- If a blocking library is unavoidable, isolate it in `frameworks` and use the framework's threadpool/offloading pattern at the boundary.
- Keep test style aligned: async use cases need async tests.

## File Structure

Use this shape unless the existing project has a stronger local convention:

```text
project/
├── entities/
│   ├── task.py
│   └── exceptions.py
├── usecases/
│   ├── task_repository.py      # Protocol
│   ├── llm_provider.py         # Protocol
│   └── task_services.py        # Use case classes
├── adapters/
│   ├── controllers/
│   │   └── task_controller.py
│   └── serializers/
│       └── task_schema.py
├── frameworks/
│   ├── database/
│   │   ├── exceptions.py
│   │   └── in_memory_repo.py
│   ├── llm/
│   │   ├── exceptions.py
│   │   └── openai_adapter.py
│   ├── logging/
│   │   └── logger.py
│   └── http/
│       └── middleware.py
└── main.py
```

Keep files focused. If a file approaches 300 lines or mixes responsibilities, split by behavior or boundary.

## Use Case Rules

Use cases are concrete application actions. Prefer one class per action.

```python
class CreateTaskUseCase:
    def __init__(self, task_repo: TaskRepository):
        self._repo = task_repo

    async def execute(self, title: str) -> Task:
        if not title.strip():
            raise ValidationError("标题不能为空")
        task = Task(title=title.strip())
        return await self._repo.save(task)
```

Use cases may:

- Validate business input.
- Create or change entities.
- Call or await repository/provider Protocols.
- Raise business errors.
- Handle known system errors when there is a deliberate retry, fallback, or user-facing degradation.

Use cases must not:

- Import controllers, FastAPI, HTTP status codes, request/response schemas, loggers, database clients, or concrete external SDK adapters.
- Catch business errors only to rethrow the same meaning.
- Log normal business progress.

## Dependency Injection

Wire dependencies in the composition root.

```python
repo = InMemoryTaskRepository()
use_case = CreateTaskUseCase(task_repo=repo)
```

For FastAPI, controllers may expose dependency functions, while `main.py` or the app factory binds them to concrete implementations.

## Error Handling

Classify errors before handling them:

| Type | Meaning | Handling |
| --- | --- | --- |
| Business error | Expected domain/application failure | Raise in use cases, map at adapters |
| System error | External infrastructure failure | Retry, degrade, notify, or translate deliberately |
| Program error | Bug or broken invariant | Do not hide; let it surface |

Business errors belong near the stable core, usually `entities/exceptions.py`.

```python
class BusinessError(Exception):
    """Base class for expected business failures."""


class ValidationError(BusinessError):
    """Input or domain validation failed."""


class NotFoundError(BusinessError):
    """Requested resource does not exist."""


class ConflictError(BusinessError):
    """Requested operation conflicts with existing state."""
```

Controllers translate business errors to transport-specific responses:

| Business error | HTTP status |
| --- | --- |
| `ValidationError` | 400 |
| `NotFoundError` | 404 |
| `ConflictError` | 409 |
| `UnauthorizedError` | 401 |
| `ForbiddenError` | 403 |

Framework-specific failures belong under `frameworks/<system>/exceptions.py`.

Rules:

- Do not swallow business errors in use cases.
- Do not catch `Exception` unless the block adds real value and re-raises or translates safely.
- Do not turn programming bugs into fake success responses.
- Include useful context when logging errors at the boundary, but avoid secrets and tokens.

## Logging Boundaries

Core rule: business logic does not own logging; logging is an external observer.

| Layer | May log? | Rule |
| --- | --- | --- |
| `entities` | No | Keep pure and stable |
| `usecases` | No | Return results or raise errors |
| `adapters` | Yes | Log request/operation context and mapped errors |
| `frameworks` | Yes | Provide logging infrastructure and external call logs |

Recommended logging locations:

```text
frameworks/logging/logger.py
frameworks/logging/context_logger.py
frameworks/http/middleware.py
adapters/controllers/*.py
```

Prefer structured context over vague messages:

```python
logger.warning("业务验证失败", extra={"operation": "create_task", "title": title})
logger.error("系统错误", extra={"operation": "create_task"}, exc_info=True)
```

Log levels:

| Error/event | Level |
| --- | --- |
| Validation failure | `WARNING` |
| Not found | `WARNING` |
| System failure | `ERROR` |
| Unexpected exception | `ERROR` with exception info |
| Request start/end | `INFO` |

## Testing Strategy

Match tests to architecture boundaries.

| Target | Test type | Goal |
| --- | --- | --- |
| Use case | Unit test | Business logic, branching, errors |
| Repository/framework implementation | Integration test | Storage or external adapter behavior |
| Controller/API | API test | HTTP status, schema, error mapping |

Recommended layout:

```text
tests/
├── unit/
│   └── usecases/
│       └── test_create_task.py
├── integration/
│   └── repositories/
│       └── test_in_memory_repo.py
└── api/
    └── test_task_api.py
```

Testing priorities:

1. Cover use case business logic first.
2. Cover repository/provider implementations when persistence or external behavior changes.
3. Cover API behavior when request/response shape or status mapping changes.

Unit test use cases with mocked Protocol dependencies:

```python
from unittest.mock import AsyncMock

import pytest


@pytest.mark.anyio
async def test_create_task_empty_title_raises_validation_error() -> None:
    repo = AsyncMock(spec=TaskRepository)
    usecase = CreateTaskUseCase(repo)

    with pytest.raises(ValidationError):
        await usecase.execute("")
```

Integration test real repositories without mocking their internals:

```python
@pytest.mark.anyio
async def test_save_and_find_by_id() -> None:
    repo = InMemoryTaskRepository()
    task = Task(id="1", title="测试")

    saved = await repo.save(task)
    found = await repo.find_by_id("1")

    assert found is not None
    assert found.id == saved.id
```

API tests verify transport behavior:

```python
def test_create_task_empty_title_returns_400(client: TestClient) -> None:
    response = client.post("/tasks", json={"title": ""})

    assert response.status_code == 400
```

For fully async API tests, use the project's chosen async HTTP test client and mark the test consistently with `pytest.mark.anyio` or the existing pytest async plugin.

Test naming:

```text
test_<unit>_<scenario>_<expected_result>
```

Examples:

```text
test_create_task_success
test_create_task_empty_title_raises_validation_error
test_complete_task_not_found_raises_not_found_error
```

## AI Implementation Checklist

Before editing:

- Locate the behavior in the correct layer.
- Identify Protocols needed by the use case.
- Decide whether the call path should be async or sync, then keep Protocols, use cases, adapters, and tests consistent.
- Decide the test level that should prove the change.

During editing:

- Keep new imports consistent with dependency direction.
- Put DTO/schema conversion at adapters, not entities.
- Put external SDK calls in frameworks, behind usecase-owned Protocols.
- Use business errors for expected failures.
- Add boundary logs only in adapters/frameworks.

Before final response:

- Run the relevant narrow test.
- Run lint/format/type/test commands when configured.
- State any tool that could not be run and why.

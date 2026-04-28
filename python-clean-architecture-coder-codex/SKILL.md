---
name: python-clean-architecture-coder-codex
description: Use when Codex writes, modifies, reviews, or plans Python application code that should follow Clean Architecture, especially FastAPI/API projects with async use cases, entities, adapters, frameworks, dependency injection, repository/provider Protocols, structured error handling, logging boundaries, pytest tests, ruff, and gradual or strict mypy.
---

# Python Clean Architecture Coder

Use this skill to produce Python code that an AI agent can extend safely over time: clear layer boundaries, explicit dependencies, predictable tests, consistent style, and repeatable tool checks.

## Reference Loading

Load only what the task needs:

- Read `references/architecture.md` before changing application behavior, layer boundaries, use cases, repositories/providers, controllers, errors, logging, or tests.
- Read `references/code-style.md` before writing or reviewing Python code style, naming, imports, typing, docstrings, constants, or commits.
- Read `references/dev-tools.md` before editing `pyproject.toml`, lint/type/test configuration, CI commands, or local tooling.

If a task touches production code, `architecture.md` and `code-style.md` usually both apply. If the task changes tooling or verification, load `dev-tools.md` too.

## Coding Workflow

1. Identify the layer being changed: `entities`, `usecases`, `adapters`, `frameworks`, or composition root.
2. Keep dependencies pointing inward. Inner layers must not import outer layers.
3. Define Protocols on the consumer side, usually in `usecases`, not beside concrete implementations.
4. Choose sync or async deliberately. For FastAPI and I/O-bound dependencies, prefer async use cases and async Protocol methods end to end.
5. Keep `main.py` or the app factory as the composition root that wires concrete framework implementations into use cases.
6. Add or update tests at the matching level: use case unit tests first, repository integration tests when storage behavior changes, API tests when HTTP behavior changes.
7. Run the narrowest relevant verification first, then the full configured check command before claiming completion.

## Non-Negotiables

- Do not place FastAPI, database clients, logging infrastructure, or external SDK dependencies inside entities.
- Do not make use cases depend on controllers, schemas, FastAPI, concrete databases, or concrete LLM/API clients.
- Do not hide async I/O behind sync use case APIs unless the project is intentionally sync-only.
- Do not swallow unknown exceptions. Handle known business and system errors explicitly, then let unexpected bugs surface.
- Do not log from entities or use cases. Log at adapters and frameworks boundaries.
- Do not add broad abstractions until there is a real boundary, replacement point, or repeated dependency to isolate.

## Output Expectations

When implementing code, explain the layer placement briefly in the final response and name the verification command that passed. When reviewing code, lead with layer violations, hidden framework dependencies, error-handling mistakes, missing tests, and tool failures.

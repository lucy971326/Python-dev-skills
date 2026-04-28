# Python Development Tools Standard

Use this reference when configuring or running project tooling.

## Toolchain

Default stack:

```text
ruff + mypy + pytest
```

| Tool | Purpose |
| --- | --- |
| `ruff` | linting, formatting, import sorting, modernization |
| `mypy` | static type checking |
| `pytest` | unit, integration, and API tests |

For uv projects, prefer `uv add --dev ruff mypy pytest` and `uv run ...` commands.

## Suggested `pyproject.toml`

Use the project's Python version. For this repository, `requires-python = ">=3.13"` implies `py313`/`3.13`.

For new projects or already-typed projects, use strict mypy as the target state:

```toml
[tool.ruff]
line-length = 88
target-version = "py313"

[tool.ruff.lint]
select = [
    "E",
    "W",
    "F",
    "I",
    "B",
    "C4",
    "UP",
]
ignore = [
    "E501",
]

[tool.mypy]
python_version = "3.13"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_functions = ["test_*"]
addopts = "-v --tb=short"
```

For existing projects, adopt mypy gradually instead of enabling strict mode across the whole tree on day one:

```toml
[tool.mypy]
python_version = "3.13"
warn_return_any = true
warn_unused_configs = true
check_untyped_defs = true
disallow_untyped_defs = false

[[tool.mypy.overrides]]
module = [
    "entities.*",
    "usecases.*",
]
disallow_untyped_defs = true
```

Gradual mypy rules:

- Keep strict mypy as the destination, not necessarily the first commit.
- Start with `entities` and `usecases` because they define stable contracts.
- Expand strict overrides outward to `adapters` and `frameworks` after the core passes.
- Relax only the noisy rule or module that blocks adoption. Do not disable mypy entirely.
- Track every broad ignore or relaxed override as technical debt to remove.
- Prefer fixing new or touched files to creating a full-repo cleanup detour.

## Commands

For uv projects:

```bash
uv run ruff check .
uv run ruff check --fix .
uv run ruff format .
uv run mypy .
uv run pytest
```

Without uv:

```bash
ruff check .
ruff check --fix .
ruff format .
mypy .
pytest
```

Full local verification:

```bash
uv run ruff check . && uv run ruff format --check . && uv run mypy . && uv run pytest
```

If `ruff format --check` is unavailable in the installed version, run `uv run ruff format .` and inspect the resulting diff.

## Optional Pre-Commit Hooks

```yaml
repos:
  - repo: local
    hooks:
      - id: ruff-check
        name: ruff check
        entry: ruff check
        language: system
        types: [python]

      - id: ruff-format
        name: ruff format
        entry: ruff format
        language: system
        types: [python]

      - id: mypy
        name: mypy
        entry: mypy
        language: system
        types: [python]
```

## IDE Settings

Use editor integration only as a convenience. CI and local commands remain the source of truth.

```json
{
  "editor.formatOnSave": true,
  "python.analysis.typeCheckingMode": "basic",
  "ruff.enable": true
}
```

Use `"strict"` in the editor once the project has reached strict mypy or pyright parity. For legacy codebases, `"basic"` keeps feedback useful without flooding the editor.

## AI Verification Rules

- Run the narrowest relevant test while developing.
- Run `ruff check` and `ruff format` before finalizing Python edits.
- Run `mypy` when type signatures, Protocols, or data models change.
- Run `pytest` when behavior changes.
- Report any skipped command with the concrete reason, not a vague caveat.

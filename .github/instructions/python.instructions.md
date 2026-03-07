---
applyTo: "**/*.py"
description: "Python coding standards: PEP 8, async, type hints, AOS SDK patterns"
---

# Python Coding Standards

## Style & Formatting

- Follow **PEP 8** conventions (4-space indentation, 88-char line limit)
- Use **type hints** on all function signatures (`-> dict`, `List[str]`, etc.)
- Use `from __future__ import annotations` at the top of each module
- Write **Google-style docstrings** for public functions and classes
- Use double-quoted strings consistently

## Async Patterns

Azure Functions HTTP handlers are `async` functions using `await`:

```python
@app.route(route="realm/agents", methods=["GET"])
async def list_agents(req: func.HttpRequest) -> func.HttpResponse:
    registry = _load_registry()              # sync — file I/O, no await needed
    await _ensure_foundry_registration(registry)  # async — calls Foundry API
    return func.HttpResponse(json.dumps(...), mimetype="application/json")
```

- **Always `await`** coroutine calls (e.g. `_ensure_foundry_registration`)
- Use `asyncio_mode = "auto"` (configured in `pyproject.toml`) for pytest-asyncio
- Avoid blocking I/O in async functions

## Type Hints

```python
from typing import Any, Dict, List, Optional

async def _load_registry() -> AgentRegistry: ...
async def _ensure_foundry_registration(registry: AgentRegistry) -> None: ...
```

- Use `Dict`, `List`, `Any`, `Optional` from `typing` for Python 3.10 compatibility
- Use `Callable[[ArgType], ReturnType]` for function parameters

## Imports

Order: stdlib → third-party → local, with blank lines between groups:

```python
from __future__ import annotations

import json
import logging
from typing import Any, Dict, List, Optional

import azure.functions as func
from pydantic import BaseModel

from agent_config_schema import AgentRegistry, AgentRegistryEntry
```

## Logging

Use the module-level logger — never use `print()`:

```python
logger = logging.getLogger(__name__)
logger.info("Orchestration started: %s", orchestration_id)
```

## Error Handling

Raise `ValueError` with descriptive messages when required agents are unavailable:

```python
if not agent_ids:
    raise ValueError("No matching agents available in the catalog")
```

## Testing

```bash
pip install -e ".[dev]"
pytest tests/ -v                              # Run all tests
pytest tests/ -v -k "test_name"              # Run specific test
pylint src/                                   # Lint
```

- Use `pytest` classes (not `unittest.TestCase`)
- Use descriptive test method names: `test_<what>_<condition>`
- Test schema validation, registry loading, filtering, and negative assertions

## Tool Integration

```bash
pytest tests/ -v          # Validate correctness
pylint src/               # Enforce code quality
pip install -e ".[dev]"   # Install dependencies
```

## Related Documentation

→ **Repository spec**: `.github/specs/repository.md`  
→ **Azure Functions patterns**: `.github/instructions/azure-functions.instructions.md`  
→ **Conventional tools**: `.github/docs/conventional-tools.md`  
→ **Architecture**: `docs/architecture.md`  
→ **API reference**: `docs/api-reference.md`

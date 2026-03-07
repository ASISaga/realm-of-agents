---
applyTo: "src/**/*.py,function_app.py,tests/**/*.py"
description: "Azure Functions and Foundry Agent Service patterns for RealmOfAgents"
---

# Azure Functions & Foundry Agent Patterns

## Function App Setup

RealmOfAgents uses native `azure-functions` with a top-level `FunctionApp` instance:

```python
import azure.functions as func

app = func.FunctionApp()
```

No SDK-managed app wrapper — all HTTP trigger decorators are applied directly to `app`.

## HTTP Endpoint Pattern

```python
@app.function_name("list_agents")
@app.route(route="realm/agents", methods=["GET"])
async def list_agents(req: func.HttpRequest) -> func.HttpResponse:
    registry = _load_registry()
    await _ensure_foundry_registration(registry)
    payload = {"agents": [a.model_dump() for a in registry.get_enabled_agents()]}
    return func.HttpResponse(json.dumps(payload), mimetype="application/json")
```

- Always use `json.dumps(...)` + `mimetype="application/json"` for JSON responses
- Return `status_code=404` + error dict for missing resources
- Return `status_code=503` + error dict for health check failures

## Registry Loading Pattern

Load the registry once and cache it in a module-level variable (synchronous — no I/O awaiting needed):

```python
import os

_registry: Optional[AgentRegistry] = None

def _load_registry() -> AgentRegistry:
    global _registry
    if _registry is not None:
        return _registry
    registry_path = os.environ.get("AGENT_REGISTRY_PATH", "example_agent_registry.json")
    with open(registry_path, encoding="utf-8") as fh:
        data = json.load(fh)
    _registry = AgentRegistry(**data)
    return _registry
```

## Foundry Registration Pattern

Register enabled agents with the Foundry Agent Service once per function app lifetime:

```python
_foundry_registered: bool = False

async def _ensure_foundry_registration(registry: AgentRegistry) -> None:
    global _foundry_registered
    if _foundry_registered:
        return
    for entry in registry.get_enabled_agents():
        if _agent_manager is not None:
            await _agent_manager.register_agent(
                agent_id=entry.agent_id,
                purpose=entry.purpose,
                name=entry.agent_id,
                adapter_name=entry.adapter_name,
                capabilities=entry.capabilities,
            )
    _foundry_registered = True
```

- Registration is **idempotent** — guarded by `_foundry_registered`
- `_agent_manager` is `None` when `AgentOperatingSystem` is unavailable (stub mode)

## Agent Registry Schema

Use the Pydantic models from `agent_config_schema`:

```python
from agent_config_schema import AgentRegistry, AgentRegistryEntry

registry = AgentRegistry(**data)
enabled  = registry.get_enabled_agents()          # List[AgentRegistryEntry]
agent    = registry.get_agent("ceo")              # Optional[AgentRegistryEntry]
leaders  = registry.filter_by_type("LeadershipAgent")
```

## Validation

```bash
pytest tests/ -v                    # Run all tests
pylint src/                         # Lint
```

## Related Documentation

→ **Repository spec**: `.github/specs/repository.md`  
→ **Python standards**: `.github/instructions/python.instructions.md`  
→ **Architecture**: `docs/architecture.md`  
→ **API reference**: `docs/api-reference.md`  
→ **Migration to Foundry**: `docs/MIGRATION_TO_FOUNDRY.md`

# RealmOfAgents Repository Specification

**Version**: 1.0.0  
**Status**: Active  
**Last Updated**: 2026-03-07

## Overview

RealmOfAgents (`aos-realm-of-agents`) is the **agent catalog** for the Agent Operating System. It is a config-driven Azure Functions application that exposes an HTTP API for browsing available agents and registers them with the Foundry Agent Service. Agents are defined entirely in a JSON registry — no code changes are required to add or modify agents.

## Scope

- Repository role in the AOS ecosystem
- Technology stack and coding patterns
- Agent registry schema and configuration
- Testing and validation workflows
- Key design principles for agents and contributors

## Repository Role

| Concern | Owner |
|---------|-------|
| Agent catalog API (list/get agents, filter by type) | **RealmOfAgents** |
| Agent registry configuration (JSON) | **RealmOfAgents** |
| Foundry Agent Service registration (`FoundryAgentManager`) | **RealmOfAgents** + `AgentOperatingSystem` |
| Azure Functions scaffolding, HTTP triggers | `azure-functions` SDK |
| Agent orchestration, perpetual workflows, messaging | AOS Dispatcher / `aos-kernel` |
| Business workflow logic | `business-infinity` (client app) |

RealmOfAgents **serves the catalog**. It does not run orchestrations — client apps query it via `AOSClient.list_agents()`.

## Technology Stack

| Component | Technology |
|-----------|-----------|
| Runtime | Python 3.10+ |
| App framework | `azure-functions` — native `func.FunctionApp` |
| Agent registration | `AgentOperatingSystem.agents.FoundryAgentManager` |
| Agent schema | `pydantic` — `AgentRegistry` / `AgentRegistryEntry` |
| Hosting | Azure Functions |
| Tests | `pytest` + `pytest-asyncio` |
| Linter | `pylint` |
| Build / deploy | `azure.yaml` (Azure Developer CLI) |

## Directory Structure

```
realm-of-agents/
├── src/
│   ├── function_app.py            # Azure Functions HTTP endpoints
│   ├── agent_config_schema.py     # Pydantic models: AgentRegistry, AgentRegistryEntry
│   ├── example_agent_registry.json # Agent definitions (edit to add/remove agents)
│   ├── function_app_original.py   # Pre-Foundry reference implementation
│   └── host.json                  # Azure Functions host config
├── tests/
│   └── test_realm_of_agents.py    # pytest unit tests
├── docs/                          # Architecture, API reference, migration guides
├── pyproject.toml                 # Build config, dependencies, pytest settings
└── azure.yaml                     # Azure Developer CLI deployment config
```

## Core Patterns

### Agent Registry Entry

```json
{
    "agent_id": "ceo",
    "agent_type": "LeadershipAgent",
    "purpose": "Strategic leadership, vision setting, and executive decision-making",
    "adapter_name": "leadership",
    "capabilities": ["strategic_planning", "decision_making"],
    "config": {"decision_mode": "autonomous"},
    "enabled": true
}
```

### Pydantic Schema

```python
from agent_config_schema import AgentRegistry, AgentRegistryEntry

registry = AgentRegistry(**json.load(open("example_agent_registry.json")))
enabled = registry.get_enabled_agents()     # List[AgentRegistryEntry]
agent   = registry.get_agent("ceo")         # Optional[AgentRegistryEntry]
leaders = registry.filter_by_type("LeadershipAgent")
```

### HTTP Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/realm/agents` | List all enabled agents |
| GET | `/api/realm/agents?agent_type=CMOAgent` | Filter by agent type |
| GET | `/api/realm/agents/{agent_id}` | Get single agent |
| GET | `/api/realm/config` | Full registry (admin) |
| GET | `/api/health` | Health check |

### Foundry Registration

On first request, all enabled agents are registered with the Foundry Agent Service:

```python
await _agent_manager.register_agent(
    agent_id=entry.agent_id,
    purpose=entry.purpose,
    name=entry.agent_id,
    adapter_name=entry.adapter_name,
    capabilities=entry.capabilities,
)
```

Registration is idempotent — subsequent requests skip re-registration via the `_foundry_registered` flag.

## Testing Workflow

```bash
# Install dev dependencies
pip install -e ".[dev]"

# Run all tests
pytest tests/ -v

# Lint
pylint src/

# Specific test
pytest tests/test_realm_of_agents.py -v -k "test_c_suite_agents"
```

**CI**: GitHub Actions runs `pytest` across Python 3.10, 3.11, and 3.12 on every push/PR to `main`.

→ **CI workflow**: `.github/workflows/ci.yml`

## C-Suite Agents

The default `example_agent_registry.json` ships with five C-suite agents:

| Agent ID | Type | Adapter | Purpose |
|----------|------|---------|---------|
| `ceo` | `LeadershipAgent` | `leadership` | Strategic leadership and executive decision-making |
| `cfo` | `LeadershipAgent` | `finance` | Financial strategy, budgeting, and fiscal oversight |
| `cmo` | `CMOAgent` | `marketing` | Marketing strategy, brand management, market analysis |
| `coo` | `LeadershipAgent` | `operations` | Operations management and process optimisation |
| `cto` | `LeadershipAgent` | `technology` | Technology strategy, innovation, digital transformation |

## Related Repositories

| Repository | Role |
|-----------|------|
| [aos-client-sdk](https://github.com/ASISaga/aos-client-sdk) | Client SDK (queries this catalog) |
| [aos-dispatcher](https://github.com/ASISaga/aos-dispatcher) | AOS Orchestration API |
| [aos-kernel](https://github.com/ASISaga/aos-kernel) | OS kernel |
| [business-infinity](https://github.com/ASISaga/business-infinity) | Example client app |
| [purpose-driven-agent](https://github.com/ASISaga/purpose-driven-agent) | Agent base class |

## Key Design Principles

1. **Config-driven** — Agents defined in JSON; no code changes needed to add agents
2. **Catalog-only** — Does not run orchestrations; serves the agent registry
3. **Foundry-integrated** — Registers agents with Foundry Agent Service on startup
4. **Idempotent** — Registration runs once per function app lifetime

## References

→ **Agent framework**: `.github/specs/agent-intelligence-framework.md`  
→ **Conventional tools**: `.github/docs/conventional-tools.md`  
→ **Python coding standards**: `.github/instructions/python.instructions.md`  
→ **Azure Functions patterns**: `.github/instructions/azure-functions.instructions.md`  
→ **Architecture**: `docs/architecture.md`  
→ **API reference**: `docs/api-reference.md`  
→ **Migration to Foundry**: `docs/MIGRATION_TO_FOUNDRY.md`

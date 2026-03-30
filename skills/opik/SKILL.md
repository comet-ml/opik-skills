---
name: opik
description: Opik SDK reference — tracing, integrations, span types, entrypoint, AgentConfig, get_agent_config(), thread_id. Use for "add tracing", "instrument my code", "add entrypoint", "extract config".
---

# Opik SDK Reference

## Span Types

| Type | Use For |
|------|---------|
| `general` | Orchestration, entry points |
| `llm` | LLM API calls |
| `tool` | Tool execution, retrieval |
| `guardrail` | Safety/validation checks |

**Only** these four types are valid.

## Key Parameters

| Parameter | Purpose |
|-----------|---------|
| `entrypoint=True` | Marks main function for Local Runner UI triggering |
| `thread_id` | Groups multi-turn traces into a conversation |
| `opik.AgentConfig` | Externalizes config into managed Blueprints |
| `get_agent_config()` | Retrieves config with `latest=True` / `env="prod"` / `version="v1"` |

## Python Quick Start

```python
import opik
from typing import Annotated

class AgentConfig(opik.AgentConfig):
    model: Annotated[str, "LLM model"]
    temperature: Annotated[float, "Sampling temperature"]

@opik.track(entrypoint=True, name="my_agent")
def agent(query: str) -> str:
    """Run the agent.

    Args:
        query: User question.
    """
    return generate(query)

@opik.track(type="llm")
def generate(query: str) -> str:
    return llm_call(query)

result = agent("What is ML?")
opik.flush_tracker()
```

## TypeScript Quick Start

```typescript
import { track } from "opik";

const myAgent = track(
  { name: "my-agent", entrypoint: true, params: [{ name: "query", type: "string" }] },
  async (query: string) => { /* agent logic */ return result; }
);
```

TS requires explicit `params` — compilation strips param names/types.

## Configuration

```bash
export OPIK_API_KEY="your-api-key"
export OPIK_URL_OVERRIDE="https://www.comet.com/opik/api"  # Cloud
export OPIK_PROJECT_NAME="my-project"
export OPIK_WORKSPACE="my-workspace"  # Required for Cloud, "default" for self-hosted
```

TypeScript uses `OPIK_WORKSPACE_NAME` (note `_NAME` suffix).

## Framework Integrations

Use integrations over manual `@opik.track` — they capture tokens, model, cost automatically. Full list: `references/integrations.md`.

```python
from opik.integrations.openai import track_openai     # wrap-the-client
from opik.integrations.langchain import OpikTracer     # callback/tracer
from opik.integrations.crewai import track_crewai      # global enable
from opik.integrations.dspy import OpikCallback        # callback
from opik.integrations.adk import track_adk_agent_recursive  # agent-specific
```

## References

| Topic | File |
|-------|------|
| Python SDK (decorators, async, distributed, config, entrypoint) | `references/tracing-python.md` |
| TypeScript SDK (client, decorators, entrypoint, params) | `references/tracing-typescript.md` |
| REST API | `references/tracing-rest-api.md` |
| All integrations | `references/integrations.md` |
| Core concepts (traces, spans, threads, metadata) | `references/observability.md` |

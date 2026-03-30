---
name: instrument-python
description: Adding Opik observability to Python LLM apps — @opik.track, AgentConfig, get_agent_config(), entrypoint for Local Runner, thread_id for conversations.
---

# Instrument Python Agents

## 1. Add @opik.track

```python
import opik

@opik.track(entrypoint=True, name="my-agent")
def agent(query: str) -> str:
    """Run the agent.

    Args:
        query: The user's question.
    """
    context = retrieve(query)
    return generate(query, context)

@opik.track(type="tool")
def retrieve(query: str) -> list:
    return search_db(query)

@opik.track(type="llm")
def generate(query: str, context: list) -> str:
    return llm_call(query, context)
```

Span types: `general` (default), `llm`, `tool`, `guardrail`.

## 2. Extract Config + Retrieve at Runtime

```python
from typing import Annotated
import opik

class AgentConfig(opik.AgentConfig):
    model: Annotated[str, "LLM model"]
    temperature: Annotated[float, "Sampling temperature"]
    system_prompt: Annotated[str, "System prompt"]

client = opik.Opik()
client.create_agent_config_version(
    AgentConfig(model="gpt-4o", temperature=0.7, system_prompt="You are helpful."),
    project_name="my-agent",
)

@opik.track(entrypoint=True, project_name="my-agent")
def run_agent(question: str) -> str:
    cfg = client.get_agent_config(
        fallback=AgentConfig(model="gpt-4o", temperature=0.7, system_prompt="You are helpful."),
        project_name="my-agent",
        latest=True,  # or env="prod" or version="v1"
    )
    return openai_call(model=cfg.model, temperature=cfg.temperature)
```

`get_agent_config()` **must** be inside `@opik.track`. Selectors: `latest=True` | `env="prod"` | `version="v1"`.

## 3. Framework Integrations

```python
from opik.integrations.openai import track_openai    # OpenAI
from opik.integrations.anthropic import track_anthropic  # Anthropic
from opik.integrations.langchain import OpikTracer    # LangChain
from opik.integrations.crewai import track_crewai     # CrewAI
```

Don't double-wrap — use integration OR `@opik.track`, not both.

## 4. Thread ID (Conversations)

```python
@opik.track(entrypoint=True)
def handle_message(session_id: str, message: str) -> str:
    opik.update_current_trace(thread_id=session_id)
    return generate_response(session_id, message)
```

## Pitfalls

- Scripts need `opik.flush_tracker()` before exit
- Entrypoint needs docstring with `Args:` for Local Runner schema
- `get_agent_config()` outside `@opik.track` won't inject Blueprint metadata

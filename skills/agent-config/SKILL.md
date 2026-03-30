---
name: agent-config
description: Opik Agent Configuration — Blueprints, get_agent_config() with selectors, environment tags, Prompt/ChatPrompt fields, deploy_to(), MaskIDs, and config lifecycle.
---

# Agent Configuration (Blueprints)

Externalize tunable parameters into managed, version-controlled configs.

| Concept | Description |
|---------|-------------|
| **Config Dataclass** | Subclass of `opik.AgentConfig` with typed fields, no defaults |
| **Blueprint** | Immutable snapshot of a config version |
| **Environment Tag** | Label (`dev`/`staging`/`prod`) pointing to a Blueprint |
| **MaskID** | Temporary override for A/B testing (used by Optimizer) |

## Define and Publish

```python
from typing import Annotated
import opik

class MyConfig(opik.AgentConfig):
    temperature: Annotated[float, "Sampling temperature"]  # NO defaults
    model: str

client = opik.Opik()
v1 = client.create_agent_config_version(
    MyConfig(temperature=0.5, model="gpt-3.5"), project_name="my-project",
)
# Publishing identical values returns same version (dedup)
# Publishing different values creates a new version
```

## Retrieve at Runtime

> **`get_agent_config()` must be called inside `@opik.track`.** Accessing fields injects `agent_configuration` metadata into the trace.

```python
@opik.track(project_name="my-project")
def run_agent(user_input: str) -> str:
    cfg = client.get_agent_config(
        fallback=MyConfig(temperature=0.0, model="fallback"),
        project_name="my-project",
        latest=True,        # OR env="prod" OR version=v1
    )
    return f"{cfg.model} @ {cfg.temperature}: {user_input}"
```

Exactly **one** selector required: `latest=True` | `env="prod"` | `version="v1"`. The `fallback=` is used when backend is unreachable.

## Deploy to Environments

```python
cfg.deploy_to("prod")  # tag a version as production (can be called outside @track)
```

## Lifecycle

```
create_agent_config_version() → new Blueprint → test with Eval Suite → deploy_to("prod")
```

## Prompt-Typed Fields

```python
from opik.api_objects.prompt.text.prompt import Prompt
from opik.api_objects.prompt.chat.chat_prompt import ChatPrompt

class PromptConfig(opik.AgentConfig):
    system_prompt: Prompt      # text prompt — .format(user_input="hi") returns str
    messages: ChatPrompt       # chat prompt — .format(variables={...}) returns list[dict]
    temperature: float
```

Create prompts with `client.create_prompt()` / `client.create_chat_prompt()`, store in config fields. SDK serializes version ID and restores full object on retrieval.

## Mask Overrides (Optimizer)

Users don't create masks manually — the Optimizer + Local Runner handle this transparently.

```python
from opik.api_objects.agent_config.config import AgentConfigManager
from opik.api_objects.agent_config.context import agent_config_context

manager = AgentConfigManager(project_name="my-project", rest_client_=client.rest_client)
mask_id = manager.create_mask(parameters={"MyConfig.temperature": 0.9})

with agent_config_context(mask_id):
    run_agent("test")  # temperature=0.9, model unchanged from Blueprint
```

## What to Extract

**Extract:** model, temperature, top_p, max_tokens, system prompt, any tunable behavior parameter.
**Don't extract:** API keys (env vars), structural logic (code), true constants.

---
name: agent-config
description: Deep guide on Opik Agent Configuration — Blueprints (immutable config snapshots), get_agent_config() retrieval with selectors, environment tags, Prompt/ChatPrompt fields, deploy_to(), MaskIDs for A/B testing, and the full config lifecycle from development to production.
---

# Agent Configuration (Blueprints)

Opik's Agent Configuration system externalizes your agent's tunable parameters into managed, version-controlled configs.

## Core Concepts

| Concept | What It Is |
|---------|-----------|
| **Config Dataclass** | A Python subclass of `opik.AgentConfig` with typed fields |
| **Blueprint** | An immutable snapshot of a config version |
| **Environment Tag** | A label (`dev`, `staging`, `prod`) pointing to a specific Blueprint |
| **MaskID** | A temporary override for A/B testing without creating new Blueprints |
| **Fallback** | A local config instance used when the backend is unreachable |

## Creating a Config

Subclass `opik.AgentConfig` with typed fields. Use `Annotated` for descriptions. **No default values** — pass values at instantiation.

```python
from typing import Annotated
import opik

class MyConfig(opik.AgentConfig):
    temperature: Annotated[float, "Sampling temperature"]  # NO defaults
    model: str
```

**Rules:**
- Subclass `opik.AgentConfig` (NOT `@dataclass`)
- No default values on the class — pass values at instantiation
- All fields need type annotations

## Publishing Config Versions

```python
client = opik.Opik()

# Publish the first version
v1_name = client.create_agent_config_version(
    MyConfig(temperature=0.5, model="gpt-3.5"),
    project_name="my-project",
)
print("Published version:", v1_name)

# Publishing identical values is a no-op — same version name returned (dedup)
v1_again = client.create_agent_config_version(
    MyConfig(temperature=0.5, model="gpt-3.5"),
    project_name="my-project",
)
assert v1_again == v1_name  # dedup

# Publishing different values creates a new version
v2_name = client.create_agent_config_version(
    MyConfig(temperature=0.8, model="gpt-4"),
    project_name="my-project",
)
assert v2_name != v1_name
```

## Retrieving Config at Runtime

> **CRITICAL:** `get_agent_config()` must be called inside a function decorated with `@opik.track`. Accessing any field automatically injects `agent_configuration` metadata into the current trace.

```python
@opik.track(project_name="my-project")
def run_agent(user_input: str) -> str:
    config = client.get_agent_config(
        fallback=MyConfig(temperature=0.0, model="fallback"),  # used if backend unreachable
        project_name="my-project",
        # Exactly ONE of these selectors must be provided:
        latest=True,
        # env="prod",       # fetch version tagged with an environment
        # version=v1_name,  # fetch a specific version by name
    )
    return f"Running {config.model} at temperature {config.temperature} with: {user_input}"
```

### Selector Reference

| Selector | Usage | Description |
|----------|-------|-------------|
| `latest=True` | Development | Fetches the most recently published version |
| `env="prod"` | Production | Fetches the version tagged with the given environment |
| `version="v1_abc"` | Pinned | Fetches a specific version by name |

Exactly **one** selector must be provided. The `fallback=` parameter is always required — it's the config used when the backend is unreachable.

## Deploying to Environments

Use `deploy_to()` to tag a version with an environment. This can be called outside `@opik.track` — it's a management operation.

```python
@opik.track(project_name="my-project")
def fetch_v1():
    return client.get_agent_config(
        fallback=MyConfig(temperature=0.0, model="fallback"),
        project_name="my-project",
        version=v1_name,
    )

v1_cfg = fetch_v1()
v1_cfg.deploy_to("prod")  # tag v1 as the production version
opik.flush_tracker()
```

Now fetching by `env="prod"` returns v1, even if v2 is the latest:

```python
@opik.track(project_name="my-project")
def run_prod(user_input: str) -> str:
    prod_cfg = client.get_agent_config(
        fallback=MyConfig(temperature=0.0, model="fallback"),
        project_name="my-project",
        env="prod",
    )
    return f"{prod_cfg.model} @ {prod_cfg.temperature}: {user_input}"

print(run_prod("Hello!"))  # uses v1 (0.5, gpt-3.5) even though v2 is latest
opik.flush_tracker()
```

## Blueprint Lifecycle

```
Edit config → create_agent_config_version() → New Blueprint (immutable)
          → dev tag auto-moves to new Blueprint
          → Test with Evaluation Suite
          → PASS? → deploy_to("prod") → PROD tag moves to new Blueprint
          → FAIL? → Keep PROD on previous Blueprint
```

## Environment Tags

| Tag | Purpose |
|-----|---------|
| `dev` | Active development, latest changes |
| `staging` | Pre-production testing |
| `prod` | Production — what end users see |

## Prompt-Typed Fields

`opik.Prompt` (text) and `opik.ChatPrompt` (chat messages) can be stored as config fields. The SDK serializes the prompt version ID and restores a full prompt object on retrieval.

### Text Prompts

```python
from opik.api_objects.prompt.text.prompt import Prompt

prompt_v1 = client.create_prompt(
    name="system-prompt",
    prompt="You are a helpful assistant. The user said: {{user_input}}",
)

class PromptConfig(opik.AgentConfig):
    system_prompt: Prompt
    temperature: float

client.create_agent_config_version(
    PromptConfig(system_prompt=prompt_v1, temperature=0.3),
    project_name="my-project",
)

@opik.track(project_name="my-project")
def run_with_text_prompt(user_input: str) -> str:
    cfg = client.get_agent_config(
        fallback=PromptConfig(system_prompt=prompt_v1, temperature=0.0),
        project_name="my-project",
        latest=True,
    )
    assert isinstance(cfg.system_prompt, Prompt)
    # format() substitutes template variables and returns a string
    return cfg.system_prompt.format(user_input=user_input)
```

### Chat Prompts

```python
from opik.api_objects.prompt.chat.chat_prompt import ChatPrompt

chat_prompt_v1 = client.create_chat_prompt(
    name="chat-system-prompt",
    messages=[
        {"role": "system", "content": "You are a helpful assistant named {{assistant_name}}."},
        {"role": "user", "content": "{{user_input}}"},
    ],
)

class ChatPromptConfig(opik.AgentConfig):
    messages: ChatPrompt
    temperature: float

client.create_agent_config_version(
    ChatPromptConfig(messages=chat_prompt_v1, temperature=0.3),
    project_name="my-project",
)

@opik.track(project_name="my-project")
def run_with_chat_prompt(user_input: str) -> list:
    cfg = client.get_agent_config(
        fallback=ChatPromptConfig(messages=chat_prompt_v1, temperature=0.0),
        project_name="my-project",
        latest=True,
    )
    assert isinstance(cfg.messages, ChatPrompt)
    # format() substitutes template variables and returns a list of message dicts
    return cfg.messages.format(
        variables={"assistant_name": "Opik", "user_input": user_input},
    )
```

## Mask Overrides (for the Optimizer)

A mask overrides selected fields while leaving the rest unchanged. The Optimizer creates masks and sends them to the Local Runner, which applies them transparently. **Users don't create masks manually** — this is handled by the Optimizer + Local Runner pipeline.

```python
from opik.api_objects.agent_config.config import AgentConfigManager
from opik.api_objects.agent_config.context import agent_config_context

class MaskConfig(opik.AgentConfig):
    temperature: float
    model: str

client.create_agent_config_version(
    MaskConfig(temperature=0.5, model="gpt-4"),
    project_name="my-project",
)

manager = AgentConfigManager(
    project_name="my-project",
    rest_client_=client.rest_client,
)

# Override only temperature; model is left as-is from the backend Blueprint
mask_id = manager.create_mask(parameters={"MaskConfig.temperature": 0.9})

@opik.track(project_name="my-project")
def run_with_mask() -> None:
    result = client.get_agent_config(
        fallback=MaskConfig(temperature=0.0, model="fallback"),
        project_name="my-project",
        latest=True,
    )
    print(result.temperature)  # 0.9  <- from the mask
    print(result.model)        # gpt-4 <- unchanged from Blueprint

with agent_config_context(mask_id):
    run_with_mask()

opik.flush_tracker()
```

## What to Extract vs Not

### Extract (put in config)
- Model name, temperature, top_p, max_tokens
- System prompt / persona
- Any tunable parameter that affects agent behavior

### Don't Extract (keep in code/env)
- API keys and secrets -> env vars
- Structural logic -> code
- Truly constant values that never change

## Blueprint in Traces

Every trace includes `agent_configuration` metadata when `get_agent_config()` is used:
- Filter traces by Blueprint to compare config versions
- Roll back PROD tag if a Blueprint causes regression
- Track which config version produced each trace

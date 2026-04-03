---
name: instrument-agent-config
description: Add Opik AgentConfig to an existing codebase. Finds hardcoded parameters and existing config classes, converts them to versioned opik.AgentConfig, and wires up create/get at runtime. Does not require prior instrumentation — sets up tracing and environment config if missing. Use for "add agent config", "externalize my config", "add opik agent configuration", or "version my agent parameters".
argument-hint: "[file or directory path]"
compatibility: Works with Claude Code, OpenAI Codex, Cursor, and any Agent Skills-compatible tool. Requires a Python or TypeScript project.
allowed-tools:
  - Read
  - Edit
  - Write
  - Grep
  - Glob
  - Bash
---

# Instrument Agent Config — Add Opik AgentConfig to a Codebase

You are adding versioned agent configuration to an existing codebase using `opik.AgentConfig`. This externalizes all tunable parameters (models, temperatures, prompts, token limits) so they can be versioned, compared, and deployed through the Opik UI.

This skill does NOT assume the codebase has already been instrumented with Opik tracing. If tracing is missing, you will add the minimum required tracing to support agent configuration.

Follow these steps precisely.

## Step 1 — Scope

If `$ARGUMENTS` is provided, scope your work to those files or directories. Otherwise, discover the project root and work across the main application code.

## Step 2 — Detect Language

Determine the project language:

- **Python**: look for `*.py`, `pyproject.toml`, `requirements.txt`
- **TypeScript**: look for `*.ts`, `*.tsx`, `package.json`

## Step 3 — Find Tunable Parameters

Search the codebase for all tunable parameters that should be externalized. Look for:

### Existing config classes (migration targets)

Search for classes that hold tunable parameters. Look for names like `AgentConfig`, `Config`, `Settings`, `AgentSettings`, `ModelConfig`, `LLMConfig`, or any `@dataclass` / Pydantic `BaseModel` / plain class with fields like `model`, `temperature`, `system_prompt`, `max_tokens`, `top_p`.

**An existing config class is a migration target, not a reason to skip this step.**

If the class already inherits from `opik.AgentConfig`, audit it for completeness but do not re-migrate.

### Hardcoded parameters (extraction targets)

Search for hardcoded values that should be in config:

- **Model names**: strings like `"gpt-4o"`, `"claude-sonnet-4-20250514"`, `"gemini-pro"`, etc.
- **Temperature / top_p / max_tokens**: numeric values passed to LLM calls
- **Prompts and prompt templates**: system prompts, user prompt templates, few-shot examples
- **Any other runtime parameters** the user may want to compare or optimize

**Extract**: model, temperature, top_p, max_tokens, system prompt, all prompt templates, tunable parameters.
**Don't extract**: API keys, structural logic, true constants (e.g., retry counts that never change), framework boilerplate.

## Step 4 — Check for Existing Opik Setup

Before adding anything, check what's already in place:

1. **Opik imports**: is `import opik` or `@opik.track` already present?
2. **Entrypoint**: is there a function decorated with `@opik.track(entrypoint=True)`?
3. **Opik client**: is `opik.Opik()` already instantiated?
4. **Environment config**: does `.env`, `.env.local`, or `~/.opik.config` already have Opik vars?

Note what exists and what's missing — you'll fill the gaps in the following steps.

## Step 5 — Create or Convert the AgentConfig Class

### Python

#### If an existing config class was found — migrate it:

1. Replace the existing base (`@dataclass`, `BaseModel`, plain class) with `opik.AgentConfig`
2. Add `Annotated` type hints with descriptions to each field (if not already present)
3. Convert plain `str` prompt fields to `opik.Prompt`
4. Remove any defaults from the class fields — defaults go in the `DEFAULT_*` instance
5. Keep the class in its existing file unless it makes more sense to move it

Example migration from a dataclass:

```python
# BEFORE
from dataclasses import dataclass

@dataclass
class Config:
    model: str = "gpt-4o"
    temperature: float = 0.7
    system_prompt: str = "You are a helpful assistant."

# AFTER
from typing import Annotated
import opik

class AgentConfig(opik.AgentConfig):
    model: Annotated[str, "LLM model"]
    temperature: Annotated[float, "Sampling temperature"]
    system_prompt: Annotated[opik.Prompt, "Managed system prompt"]

DEFAULT_AGENT_CONFIG = AgentConfig(
    model="gpt-4o",
    temperature=0.7,
    system_prompt=opik.Prompt(
        name="agent-system-prompt",
        prompt="You are a helpful assistant.",
    ),
)
```

#### If no config class exists — create one:

Gather all hardcoded tunable parameters found in Step 3 and define a new `AgentConfig` class. Place it in a sensible location — either in the main module or in a dedicated `config.py` file if the project already uses that pattern.

```python
from typing import Annotated
import opik

class AgentConfig(opik.AgentConfig):
    model: Annotated[str, "LLM model"]
    temperature: Annotated[float, "Sampling temperature"]
    system_prompt: Annotated[opik.Prompt, "Managed system prompt"]

DEFAULT_AGENT_CONFIG = AgentConfig(
    model="gpt-4o",
    temperature=0.7,
    system_prompt=opik.Prompt(
        name="agent-system-prompt",
        prompt="You are a helpful assistant for {{product}}.",
    ),
)
```

### TypeScript

AgentConfig is Python-only. For TypeScript projects, inform the user that agent configuration is currently available in Python only. If the project has a Python backend, instrument that. Otherwise, suggest they track configuration manually via trace metadata until TypeScript support is added.

## Step 6 — Wire Up Config Publishing and Retrieval

### Publish at startup

Add a module-level `client` and `create_agent_config_version` call. Choose a meaningful `project_name` based on the agent's purpose.

```python
client = opik.Opik()
client.create_agent_config_version(
    DEFAULT_AGENT_CONFIG,
    project_name="<project-name>",
)
```

- Place this at module level or in a startup function that runs once
- Identical config values produce the same version (dedup) — safe to call on every start
- Use the same `project_name` consistently for publishing and retrieval

### Retrieve at runtime

Inside the entrypoint function, call `get_agent_config()`:

```python
@opik.track(entrypoint=True, project_name="<project-name>")
def run_agent(question: str) -> str:
    cfg = client.get_agent_config(
        fallback=DEFAULT_AGENT_CONFIG,
        project_name="<project-name>",
    )
    # Use cfg.model, cfg.temperature, cfg.system_prompt, etc.
    ...
```

**CRITICAL**: `get_agent_config()` **must** be called inside a `@opik.track`-decorated function. It raises an error otherwise.

### Update all call sites

Replace every reference to the old hardcoded values or old config object with the new `cfg` fields:

- `model="gpt-4o"` → `model=cfg.model`
- `temperature=0.7` → `temperature=cfg.temperature`
- `config.system_prompt` → `cfg.system_prompt.format(product="Opik")` (for `opik.Prompt` fields with template variables)

## Step 7 — Ensure Minimum Tracing is in Place

`get_agent_config()` requires `@opik.track` on the calling function. If the codebase isn't already instrumented:

1. Add `import opik` to the entrypoint file
2. Add `@opik.track(entrypoint=True, project_name="<project-name>")` to the main agent function
3. If it's a script (not a long-running server), add `opik.flush_tracker()` after the top-level call
4. You do NOT need to instrument every function — just the entrypoint is sufficient for agent config to work. If the user wants full tracing, they should run `/instrument`.

## Step 8 — Environment Config

Follow the setup decision tree:

1. **Check for existing `.env` / `.env.local` files and `dotenv` usage in code.**
   - If the project loads a `.env` file: **append** `OPIK_API_KEY` and `OPIK_WORKSPACE` to that same file (if missing). Do NOT create a separate config file.
   - If there is a `.env.example` or `.env.sample`: **also update it** with the new Opik vars.

2. **If no `.env` file exists:**
   - Python: create or update `~/.opik.config` (INI format):
     ```ini
     [opik]
     api_key=your-api-key
     url_override=https://www.comet.com/opik/api
     workspace=your-workspace
     ```
   - TypeScript: create `.env` or `.env.local`

3. **Never introduce a second config mechanism.** If the project already uses `.env`, don't also create `~/.opik.config`.
4. **Never overwrite existing values.**
5. **Set `project_name` in code**, not in env files.

## Step 9 — Install Dependencies

Print the install command but do NOT run it automatically:

**Python:**
```
pip install opik
```

**TypeScript:**
```
npm install opik
```

## Step 10 — Verify

After adding agent configuration, audit:

- [ ] All tunable parameters (models, temperatures, prompts, token limits) are in the `AgentConfig` class
- [ ] `AgentConfig` fields use `Annotated` type hints with descriptions
- [ ] Prompt fields use `opik.Prompt` (not plain strings)
- [ ] `DEFAULT_AGENT_CONFIG` instance has sensible defaults matching the original values
- [ ] `create_agent_config_version()` is called at startup
- [ ] `get_agent_config()` is called inside a `@opik.track`-decorated function
- [ ] All call sites reference `cfg.*` instead of hardcoded values
- [ ] The entrypoint has `@opik.track(entrypoint=True)`
- [ ] Script entrypoints call `opik.flush_tracker()` before exit
- [ ] No hardcoded API keys were introduced
- [ ] `project_name` is consistent across `@opik.track`, `create_agent_config_version`, and `get_agent_config`

## Anti-Patterns to Avoid

- **Skipping existing config classes**: An existing `@dataclass` or `BaseModel` with model/temperature/prompt fields MUST be migrated to `opik.AgentConfig` — not ignored.
- **Leaving hardcoded values**: Every tunable parameter in LLM calls should come from the config.
- **`get_agent_config()` outside `@opik.track`**: This will raise an error at runtime. Always call it inside a decorated function.
- **Inconsistent `project_name`**: Use the same project name in `@opik.track`, `create_agent_config_version`, and `get_agent_config`.
- **Plain string prompts**: Use `opik.Prompt` for prompt fields so they get versioning and template support.
- **Missing `fallback`**: Always pass `fallback=DEFAULT_AGENT_CONFIG` to `get_agent_config()` so the agent works even without a UI-published config.
- **Defaults on class fields**: Put defaults in the `DEFAULT_*` instance, not on the class field definitions.

## References

For detailed API signatures and advanced patterns, see:
- `../opik/references/tracing-python.md` — Python SDK reference
- `../opik/references/tracing-typescript.md` — TypeScript SDK reference
- `../opik/references/integrations.md` — All 40+ framework integrations
- `../opik/references/observability.md` — Core concepts (traces, spans, threads)

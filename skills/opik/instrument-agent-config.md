---
name: instrument-agent-config
description: Add Opik Config to an existing codebase. Finds hardcoded parameters and existing config classes, converts them to versioned opik.Config subclasses, and wires up get_or_create_config at runtime. Does not require prior instrumentation — sets up tracing and environment config if missing. Use for "add agent config", "externalize my config", "add opik config", or "version my agent parameters".
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

# Instrument Agent Config — Add Opik Config to a Codebase

You are adding versioned agent configuration to an existing codebase using `opik.Config`. This externalizes all tunable parameters (models, temperatures, prompts, token limits) so they can be versioned, compared, and deployed through the Opik UI.

This skill does NOT assume the codebase has already been instrumented with Opik tracing. If tracing is missing, you will add the minimum required tracing to support configuration.

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

If the class already inherits from `opik.Config`, audit it for completeness but do not re-migrate.

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

## Step 5 — Create or Convert the Config Class

### Python

#### If an existing config class was found — migrate it:

1. Replace the existing base (`@dataclass`, `BaseModel`, plain class) with `opik.Config`
2. Convert plain `str` prompt fields to `opik.Prompt` or `opik.ChatPrompt` (see below)
3. Remove any defaults from the class fields — defaults go in the `DEFAULT_CONFIG` instance
4. Keep the class in its existing file unless it makes more sense to move it

**Prompt field types:**
- Use `opik.Prompt` for single string-based system prompts or templates
- Use `opik.ChatPrompt` for multi-turn message templates (list of `{"role": ..., "content": ...}` messages)

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
import opik

class AgentConfig(opik.Config):
    model: str
    temperature: float
    system_prompt: opik.Prompt

DEFAULT_CONFIG = AgentConfig(
    model="gpt-4o",
    temperature=0.7,
    system_prompt=opik.Prompt(
        name="agent-system-prompt",
        prompt="You are a helpful assistant.",
    ),
)
```

Example with a chat prompt:

```python
import opik

class AgentConfig(opik.Config):
    model: str
    temperature: float
    chat_template: opik.ChatPrompt

DEFAULT_CONFIG = AgentConfig(
    model="gpt-4o",
    temperature=0.7,
    chat_template=opik.ChatPrompt(
        name="agent-chat-template",
        messages=[
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "Help me with {{task}}"},
        ],
    ),
)
```

#### If no config class exists — create one:

Gather all hardcoded tunable parameters found in Step 3 and define a new `AgentConfig` class. Place it in a sensible location — either in the main module or in a dedicated `config.py` file if the project already uses that pattern.

```python
import opik

class AgentConfig(opik.Config):
    model: str
    temperature: float
    system_prompt: opik.Prompt

DEFAULT_CONFIG = AgentConfig(
    model="gpt-4o",
    temperature=0.7,
    system_prompt=opik.Prompt(
        name="agent-system-prompt",
        prompt="You are a helpful assistant for {{product}}.",
    ),
)
```

### TypeScript

Define an interface for the config shape, then pass an object that satisfies it as the fallback. Use `Prompt` for string-based templates and `ChatPrompt` for multi-turn message templates:

```typescript
import { Opik, Prompt, ChatPrompt } from "opik";

interface AgentConfig {
  model: string;
  temperature: number;
  systemPrompt: Prompt;
  chatTemplate: ChatPrompt;
}

const client = new Opik();

const DEFAULT_CONFIG: AgentConfig = {
  model: "gpt-4o",
  temperature: 0.7,
  systemPrompt: await client.createPrompt({
    name: "agent-system-prompt",
    prompt: "You are a helpful assistant.",
  }),
  chatTemplate: await client.createChatPrompt({
    name: "agent-chat-template",
    messages: [
      { role: "system", content: "You are a helpful assistant." },
      { role: "user", content: "Help me with {{task}}" },
    ],
  }),
};
```

## Step 6 — Wire Up Config Retrieval

The primary entry point is `get_or_create_config` / `getOrCreateConfig`. Call it inside a tracked function, passing `fallback` as the local defaults. **Do not call `create_config` or `set_config_env` in application code** — those are admin/ops methods, not part of the normal agent runtime path.

### Python

```python
import opik

client = opik.Opik()

@opik.track(project_name="<project-name>")
def run_agent(question: str) -> str:
    cfg = client.get_or_create_config(
        fallback=DEFAULT_CONFIG,
        project_name="<project-name>",
    )
    # Use cfg.model, cfg.temperature, cfg.system_prompt, etc.
    ...
```

**How it works:**
- On first call when no config exists in the project, auto-creates one from `fallback` and returns it.
- On subsequent calls, returns the latest version tagged `"prod"` in the Opik UI.
- On backend failure, returns `fallback` with `is_fallback=True` (never breaks the agent).

**CRITICAL**: `get_or_create_config()` **must** be called inside a `@opik.track`-decorated function.

### TypeScript

```typescript
import { Opik, track } from "opik";

const client = new Opik();

const runAgent = track({ projectName: "<project-name>" }, async (question: string) => {
  const cfg = await client.getOrCreateConfig<AgentConfig>({
    fallback: DEFAULT_CONFIG,
    projectName: "<project-name>",
  });
  // Use cfg.model, cfg.temperature, etc.
  ...
});
```

**How it works:**
- On first call when no config exists in the project, auto-creates one from `fallback` and returns it.
- On subsequent calls, returns the latest version tagged `"prod"` in the Opik UI.
- On backend failure, returns `fallback` with `isFallback: true` (never breaks the agent).

**CRITICAL**: `getOrCreateConfig()` **must** be called inside a `track()`-wrapped function.

### Update all call sites

Replace every reference to the old hardcoded values or old config object with the new `cfg` fields:

- `model="gpt-4o"` → `model=cfg.model`
- `temperature=0.7` → `temperature=cfg.temperature`
- `config.system_prompt` → `cfg.system_prompt.format(product="Opik")` (for `opik.Prompt` fields with template variables)

## Step 7 — Ensure Minimum Tracing is in Place

`get_or_create_config()` requires `@opik.track` on the calling function. If the codebase isn't already instrumented:

1. Add `import opik` to the entrypoint file
2. Add `@opik.track(entrypoint=True, project_name="<project-name>")` to the main agent function
3. If it's a script (not a long-running server), add `opik.flush_tracker()` after the top-level call
4. You do NOT need to instrument every function — just the entrypoint is sufficient for agent config to work. If the user wants full tracing, they should run `/instrument`.

For TypeScript:
1. Import `{ track }` from `"opik"`
2. Wrap the entrypoint with `track({ projectName: "<project-name>" }, async (...) => { ... })`

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

- [ ] All tunable parameters (models, temperatures, prompts, token limits) are in the `Config` subclass (Python) or fallback object (TypeScript)
- [ ] String-based prompt fields use `opik.Prompt` / `Prompt`; multi-turn chat fields use `opik.ChatPrompt` / `ChatPrompt` — not plain strings
- [ ] `DEFAULT_CONFIG` instance has sensible defaults matching the original values
- [ ] `get_or_create_config()` / `getOrCreateConfig()` is called inside a tracked function, passing `fallback=DEFAULT_CONFIG`
- [ ] All call sites reference `cfg.*` instead of hardcoded values
- [ ] The entrypoint has `@opik.track` (Python) or `track()` wrapper (TypeScript)
- [ ] Script entrypoints call `opik.flush_tracker()` / `client.flush()` before exit
- [ ] No hardcoded API keys were introduced
- [ ] `project_name` is consistent across `@opik.track` and `get_or_create_config`
- [ ] `create_config` and `set_config_env` are NOT present in application code (these are ops-only)

## Anti-Patterns to Avoid

- **Skipping existing config classes**: An existing `@dataclass` or `BaseModel` with model/temperature/prompt fields MUST be migrated to `opik.Config` — not ignored.
- **Leaving hardcoded values**: Every tunable parameter in LLM calls should come from the config.
- **`get_or_create_config()` outside a tracked function**: This will raise an error at runtime. Always call it inside a `@opik.track`-decorated function (Python) or `track()` wrapper (TypeScript).
- **Calling `create_config` in application code**: Application code should only call `get_or_create_config`. `create_config` is for admin scripts or CI pipelines that explicitly push new config versions — not for agents.
- **Calling `set_config_env` in application code**: Environment tags are managed in the Opik UI or by ops tooling, not by the agent itself.
- **Inconsistent `project_name`**: Use the same project name in `@opik.track` and `get_or_create_config`.
- **Plain string prompts**: Use `opik.Prompt` / `Prompt` for string-based prompt fields and `opik.ChatPrompt` / `ChatPrompt` for multi-turn message templates — not plain strings.
- **Missing `fallback`**: Always pass `fallback=DEFAULT_CONFIG` to `get_or_create_config()` so the agent works even without a UI-published config and on backend errors.
- **Defaults on class fields**: Put defaults in the `DEFAULT_CONFIG` instance, not on the class field definitions.

## References

For detailed API signatures and advanced patterns, see:
- `../opik/references/tracing-python.md` — Python SDK reference
- `../opik/references/tracing-typescript.md` — TypeScript SDK reference
- `../opik/references/integrations.md` — All 40+ framework integrations
- `../opik/references/observability.md` — Core concepts (traces, spans, threads)

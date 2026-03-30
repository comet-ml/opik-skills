---
description: Add Opik observability to your code - automatically detects frameworks and adds the correct integration
argument-hint: [file or description of what to instrument]
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Skill
  - Bash
model: sonnet
---

# Add Opik Observability

Add tracing to the user's code so their LLM application is observable in Opik.

**User request:** $ARGUMENTS

## Step 1: Load the Skills

Use the Skill tool to load BOTH of these skills before doing anything else:

1. **`opik`** — Opik SDK reference: all integrations, tracing patterns, span types, code snippets
2. **`agent-ops`** — Agent architecture patterns, evaluation, what to trace and why

Load them both now. Do not proceed until both are loaded.

## Step 2: Discover Frameworks from Dependencies (Do This FIRST)

**Do NOT rely only on import statements.** Code may use dynamic imports (`__import__`, `importlib`), factory patterns, or lazy loading that makes frameworks invisible to import scanning.

Instead, start by reading dependency manifests to build a checklist of frameworks that MUST be instrumented:

1. **Read dependency files** — check `requirements.txt`, `pyproject.toml`, `setup.py`, `setup.cfg`, `Pipfile`, `package.json` (for TypeScript/Node)
2. **Build a framework checklist** — for each dependency that has an Opik integration (OpenAI, Anthropic, LangChain, CrewAI, LlamaIndex, etc.), add it to your checklist
3. **Note ALL languages** — if the project has both Python files AND TypeScript/JavaScript files (check for `package.json`, `tsconfig.json`, `*.ts`, `*.js`), you must instrument BOTH languages

This checklist is your source of truth. Every framework on it must be accounted for by the end.

## Step 3: Trace the Agent Flow

Now read the code to understand how it actually works. **Follow the execution flow**, don't just scan files in isolation:

1. **Find entry points** — look for `if __name__ == "__main__"`, CLI commands, HTTP handlers, exported functions. There may be MULTIPLE entry points.
2. **Trace the call graph** — from each entry point, follow function calls to understand the full execution path. Read every file that gets called.
3. **Find where each framework on your checklist is actually used** — it may be behind factories, registries, decorators, proxies, or dynamic imports. Search for:
   - The framework's package name in strings (e.g., `"openai"`, `"anthropic"`, `"crewai"` as arguments to `__import__()` or `importlib.import_module()`)
   - Class names from the framework (e.g., `OpenAI`, `Anthropic`, `ChatOpenAI`, `Agent`, `Crew`)
   - If you can't find where a dependency from the checklist is used, search the entire codebase for its package name as a string
4. **Identify existing tracing** — check if there's already tracing code. Verify it actually sends to Opik (not a homegrown stub or different tracing system). If it's fake or non-Opik, replace it.

## Step 4: Extract Configuration into an AgentConfig

After understanding the agent flow, extract hardcoded configuration values into a config module using `opik.AgentConfig`.

### What to Extract

Look for hardcoded values in the agent code that control behavior:
- **Model name**: `"gpt-4o"`, `"claude-3-sonnet"`, etc.
- **Temperature**: `temperature=0.7`
- **System prompt**: Any string passed as a system message
- **Max tokens**: `max_tokens=1024`
- **Top-p, top-k**: Sampling parameters
- **Any other tunable parameters** that affect agent behavior

### How to Extract (Python)

1. **Create a config file** (e.g., `agent_config.py`) using `opik.AgentConfig` as the base class:

```python
from typing import Annotated
import opik

class AgentConfig(opik.AgentConfig):
    model: Annotated[str, "LLM model to use"]
    temperature: Annotated[float, "Sampling temperature"]
    system_prompt: Annotated[str, "System prompt for the agent"]
    max_tokens: Annotated[int, "Maximum tokens in response"]
```

**Important rules for `opik.AgentConfig`:**
- Subclass `opik.AgentConfig` — do NOT use a plain `@dataclass`
- Use `Annotated[Type, "description"]` to add field descriptions
- Do NOT set default values on the class — pass values at instantiation
- All fields must have type annotations

2. **Publish the config and retrieve at runtime** using `get_agent_config()`:

```python
client = opik.Opik()

# Publish config to Opik for Blueprint management
config = AgentConfig(
    model="gpt-4o",
    temperature=0.7,
    system_prompt="You are a helpful assistant.",
    max_tokens=1024,
)
client.create_agent_config_version(config, project_name="my-agent")

# Retrieve at runtime — MUST be inside @opik.track
@opik.track(entrypoint=True, project_name="my-agent")
def run_agent(question: str) -> str:
    cfg = client.get_agent_config(
        fallback=AgentConfig(
            model="gpt-4o", temperature=0.7,
            system_prompt="You are a helpful assistant.",
            max_tokens=1024,
        ),
        project_name="my-agent",
        latest=True,  # or env="prod" or version="v1_abc"
    )
    response = openai_client.chat.completions.create(
        model=cfg.model,
        temperature=cfg.temperature,
        max_tokens=cfg.max_tokens,
        messages=[
            {"role": "system", "content": cfg.system_prompt},
            {"role": "user", "content": question},
        ],
    )
    return response.choices[0].message.content
```

3. **Do NOT extract**:
   - API keys or secrets (those stay in env vars)
   - Structural code logic (only runtime parameters)
   - Values that are truly constant and never need changing

### Edge Cases

- **Multiple agents**: Create separate config classes per agent, or a shared base
- **Framework-specific config** (LangChain, CrewAI): Extract parameters from framework constructors
- **Existing config patterns**: If the project already has a config file, integrate with it rather than creating a new one

## Step 4.5: Detect Conversational Agents and Wire thread_id

Check if the agent handles multi-turn conversations. Look for:
- **Message history lists** (`messages: list`, `conversation_history`, `chat_history`)
- **Session/conversation ID parameters** (`session_id`, `conversation_id`, `thread_id`)
- **Chat loop patterns** (while loops processing user messages)
- **Stateful turn handling** (appending to message history between calls)

### If Conversational Pattern Detected

Wire `thread_id` to group conversation turns:

1. **If the agent has a natural session identifier** (session_id, conversation_id parameter):
```python
@opik.track(entrypoint=True, project_name="chat-agent")
def handle_message(session_id: str, message: str) -> str:
    opik.update_current_trace(thread_id=session_id)
    # ... rest of the function
```

2. **If no natural session ID exists**, generate one at the session level:
```python
import uuid
thread_id = str(uuid.uuid4())

@opik.track(entrypoint=True, project_name="chat-agent")
def handle_message(message: str) -> str:
    opik.update_current_trace(thread_id=thread_id)
    # ... rest of the function
```

### If NOT a Conversational Agent

Skip this step. Single-shot agents do not need `thread_id`.

## Step 4.7: Design the Trace Structure

Before deciding what integration to use where, **map out what a single trace should look like** for one user request. A single request should produce exactly ONE trace with nested spans.

```
Trace: "agent_name" (general)
├── Span: "step_1" (tool) — e.g., search, retrieval
│   └── Span: "llm_call" (llm) — e.g., embedding, completion
├── Span: "step_2" (llm) — e.g., summarize
└── Span: "step_3" (llm) — e.g., synthesize final answer
```

For cross-process boundaries, propagate trace context (see `references/tracing-python.md` for distributed tracing).

## Step 5: Apply the Correct Integration

**Before writing any integration code, read the exact reference file for the language/framework you're about to instrument.** Do NOT guess import paths or API patterns from memory. Use the Read tool on the relevant reference:
- Python integrations -> read `references/integrations.md` from the `opik` skill
- Python tracing -> read `references/tracing-python.md` from the `opik` skill
- TypeScript -> read `references/tracing-typescript.md` from the `opik` skill

Key principles:

1. **Follow the trace tree** — Each node tells you what integration pattern to use
2. **Trace key functions** — Add `@opik.track` to functions you want visibility into
3. **Mark the entrypoint** — Add `entrypoint=True` to the main function's `@opik.track` decorator. Also add a docstring with `Args:` descriptions (required for Local Runner schema discovery):
   ```python
   @opik.track(entrypoint=True, project_name="my-agent")
   def run_agent(question: str, context: str = "") -> str:
       """Run the agent with a user question.

       Args:
           question: The user's question to answer.
           context: Optional additional context.
       """
   ```
4. **For TypeScript entrypoints**, use `track()` with explicit `params`:
   ```typescript
   const myAgent = track(
     {
       name: "my-agent",
       entrypoint: true,
       params: [{ name: "question", type: "string" }],
     },
     async (question: string) => { /* ... */ }
   );
   ```
   > TypeScript requires explicit `params` because compilation strips parameter names/types at runtime.
5. **Use framework integrations when available** — e.g., `track_openai()` instead of manual `@opik.track`
6. **Don't double-wrap** — If using an integration, don't also add decorators to the same calls
7. **Add flush for scripts** — `opik.flush_tracker()` for Python, `await client.flush()` for TypeScript
8. **Use correct span types** — `general`, `llm`, `tool`, `guardrail` (the ONLY valid types)
9. **Instrument ALL languages** — if the project has TypeScript files that make LLM calls, instrument them too
10. **Set a default project name via env var** — Use `OPIK_PROJECT_NAME` env var so traces don't end up in "Default Project"

## Step 6: Install Dependencies

**If you added Opik packages to dependency files, install them now.** Do not leave this for the user.

1. **Detect the project's package manager and use it:**
   - If `uv.lock` or `.python-version` exists -> `uv add opik`
   - If `poetry.lock` exists -> `poetry add opik`
   - If `Pipfile` exists -> `pipenv install opik`
   - If `requirements.txt` or `pyproject.toml` with no lock file -> `pip install opik`
   - If `package.json` (Node/TS) -> check for `yarn.lock` (yarn), `pnpm-lock.yaml` (pnpm), otherwise `npm install`
2. **Install for every language in the project**

## Step 7: Validate the Changes

1. **For Python** — run `python -c "from <entry_module> import <entry_function>"`
2. **For TypeScript** — run `npx tsc --noEmit` and `npx tsx --eval "import '<entry_file>'"`
3. **If validation fails** — fix the issue before reporting success

## Step 8: Verify and Report

After instrumenting:

1. **Verify one-trace-per-request** — trace through a single user request end-to-end
2. **Run the checklist** — compare your framework checklist against what you instrumented
3. **Verify Opik 2.0 features**:
   - **Config extraction**: Confirm an `AgentConfig` was created with `get_agent_config()` retrieval inside `@opik.track`
   - **Entrypoint**: Confirm exactly one function has `entrypoint=True` with a docstring (Python) or `params` (TypeScript)
   - **Thread ID** (if conversational): Confirm `thread_id` is wired
4. **Explain what was added and why**
5. **Show the key changes made**
6. **List any configuration the user still needs to set up**
7. **Next steps**: Suggest the user can now:
   - Run `opik connect --pair <CODE> python3 app.py` (or `npx tsx app.ts`) to pair with Opik UI
   - Create an Evaluation Suite with `/opik:create-eval-suite`
   - View traces and configuration in the Opik UI

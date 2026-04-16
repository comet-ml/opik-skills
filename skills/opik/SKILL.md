---
name: opik
description: Opik observability for LLM agents — Agent Configuration, Local Runner (opik connect), Test Suites, threads, integrations. Use for "configure my agent", "connect my agent", "evaluate my agent" or "integrate with Opik".
---

# Opik — Observability for LLM Agents

Integrating with Opik always means adding all three components unless the user explicitly asks for only one:

1. **Tracing** — instrument LLM calls with the appropriate integration or `@opik.track`
2. **Entrypoint** — mark the top-level function with `entrypoint=True` for Local Runner and UI integration
3. **Agent Configuration** — externalize all tunable parameters into `opik.Config`: model names, temperatures, top_p, max_tokens, all prompts and prompt templates, and any other runtime parameters the user may want to compare or optimize

## Setup

### Environment Config Decision Tree

**Before adding Opik config, inspect the project's existing config approach.** Follow this decision tree exactly:

1. **Check for existing `.env` / `.env.local` files and `dotenv` usage in code.**
   - If the project loads a `.env` file (via `python-dotenv`, `dotenv`, or framework auto-loading): **append** `OPIK_API_KEY` and `OPIK_WORKSPACE` to that same file. Do NOT create a separate config file.
   - If there is a `.env.example` or `.env.sample`: **also update it** with the new Opik vars (using placeholder values) so future developers know which vars are needed.

2. **If no `.env` file exists:**
   - Python: create or update `~/.opik.config` (INI format). This is the SDK's native config file.
   - TypeScript/JavaScript: create `.env` (or `.env.local` if the project uses Next.js or similar).

3. **Never introduce a second config mechanism.** If the project already uses `.env` for API keys, do NOT also create `~/.opik.config`. If it uses `~/.opik.config`, do NOT add Opik vars to `.env`.

4. **Never overwrite existing values.** If `OPIK_API_KEY` is already set in `.env`, leave it. Only add vars that are missing.

5. **Prefer setting `project_name` in code**, not in env files — one machine may log to many projects.

6. **If the user provides an API key and workspace in the prompt**, use those values directly. If they provide only an API key, ask for the workspace or default to `"default"` for local OSS.

### Config Formats

Python `~/.opik.config` (INI):

```ini
[opik]
api_key=your-api-key
url_override=https://www.comet.com/opik/api
workspace=your-workspace
```

Environment variables (append to existing `.env`):

```bash
# Opik
OPIK_API_KEY=your-api-key
OPIK_URL_OVERRIDE=https://www.comet.com/opik/api
OPIK_WORKSPACE=your-workspace
```

TypeScript uses `OPIK_WORKSPACE` as the env var and `workspaceName` in `new Opik({...})`.

### Standard Deployments

- Cloud: `https://www.comet.com/opik/api` — requires `api_key` + `workspace`
- Local OSS: `http://localhost:5173/api` — usually workspace `default`
- Self-hosted: use the deployment's custom URL, following the project's existing config style

### Interactive Config (optional)

```bash
opik configure
opik configure --use_local
npx opik-ts configure
npx opik-ts configure --use-local
```

Set the project name in code:

```python
@opik.track(project_name="my-project")
def run():
    ...
```

```typescript
const client = new Opik({ projectName: "my-project" });
```

## Python Instrumentation

```python
import opik

@opik.track(entrypoint=True, name="my-agent")
def agent(query: str) -> str:
    context = retrieve(query)
    return generate(query, context)

@opik.track(type="tool")
def retrieve(query: str) -> list:
    return search_db(query)

@opik.track(type="llm")
def generate(query: str, context: list) -> str:
    return llm_call(query, context)

result = agent("What is ML?")
opik.flush_tracker()  # required in scripts
```
Valid span types for manual instrumentation: `general`, `llm`, `tool`, `guardrail`.

**Framework integrations** — these capture tokens, model, and cost automatically:

```python
from opik.integrations.openai import track_openai        # OpenAI
from opik.integrations.anthropic import track_anthropic   # Anthropic
from opik.integrations.langchain import OpikTracer        # LangChain
from opik.integrations.crewai import track_crewai         # CrewAI
from opik.integrations.dspy import OpikCallback           # DSPy
from opik.integrations.adk import track_adk_agent_recursive  # Google ADK
```

**CRITICAL — LiteLLM `OpikLogger` inside `@opik.track`:**

If the codebase uses `litellm` AND you are adding `@opik.track` decorators, you MUST pass `current_span_data` via the metadata parameter on every `litellm.completion()` / `litellm.acompletion()` call. This tells the `OpikLogger` callback to nest under the active trace. Without it, `OpikLogger` creates **orphaned top-level traces** that are separate from your `@opik.track` hierarchy.

```python
from opik import track
from opik.opik_context import get_current_span_data
from litellm.integrations.opik.opik import OpikLogger
import litellm

litellm.callbacks = [OpikLogger()]

@track
def call_llm(messages, model="gpt-4o"):
    return litellm.completion(
        model=model,
        messages=messages,
        metadata={
            "opik": {
                "current_span_data": get_current_span_data(),
                "tags": ["litellm"],
            },
        },
    )

@track(entrypoint=True)
def agent(query: str) -> str:
    return call_llm([{"role": "user", "content": query}])
```

This pattern applies whenever you see `litellm.completion` or `litellm.acompletion` in existing code that you are instrumenting with `@opik.track`.

## TypeScript Instrumentation

```typescript
import { Opik } from "opik";

const client = new Opik({ projectName: "my-project" });

const trace = client.trace({
  name: "my-agent",
  input: { query: "What is ML?" },
});

const toolSpan = trace.span({
  name: "retrieve-context",
  type: "tool",
  input: { query: "What is ML?" },
});

// retrieval logic
toolSpan.end({ output: { documents: [] } });

const llmSpan = trace.span({
  name: "generate-response",
  type: "llm",
  input: { prompt: "What is ML?" },
});

// model call
llmSpan.end({ output: { response: "Machine learning is..." } });

trace.end({ output: { response: "Machine learning is..." } });
await client.flush();
```

Prefer the client-based path in TypeScript. Use `projectName` in code rather than machine-wide config when possible.

For framework-specific integrations such as Vercel AI SDK or LangChain.js, see `references/tracing-typescript.md`.

Always `await client.flush()` before exit.

Valid span types for manual instrumentation: `general`, `llm`, `tool`, `guardrail`.

## Threads (Conversations)

Group conversation turns via `thread_id`. Each turn = one trace; shared `thread_id` = one thread.

```python
@opik.track(entrypoint=True)
def handle_message(session_id: str, message: str) -> str:
    opik.update_current_trace(thread_id=session_id)
    return generate_response(session_id, message)
```

Thread metrics:

```python
from opik.evaluation import evaluate_threads
from opik.evaluation.metrics.conversation import (
    SessionCompletenessQuality, UserFrustrationMetric, ConversationalCoherenceMetric,
)

results = evaluate_threads(project_name="chat-agent", metrics=[
    SessionCompletenessQuality(), UserFrustrationMetric(), ConversationalCoherenceMetric(),
])
```

Use for chat agents, support bots, multi-step assistants. Skip for single-shot agents or batch processing.

**Pitfalls:** Missing `thread_id` → turns appear as unrelated traces. Shared `thread_id` across users → conversations get mixed.

## Agent Configuration

Externalize the parts of your agent you expect to tune over time into versioned, immutable config snapshots. This includes prompts, models, temperatures, token limits, and other runtime parameters you may want to compare, optimize, or roll out gradually.

**CRITICAL — Search for existing config classes first.** Before creating a new config, search the codebase for existing classes that hold tunable parameters (model names, temperatures, prompts, token limits, etc.). Look for names like `AgentConfig`, `Config`, `Settings`, `AgentSettings`, `ModelConfig`, or any `@dataclass`/Pydantic model with fields like `model`, `temperature`, `system_prompt`, `max_tokens`. **An existing config class is a migration target, not a reason to skip this step.** If found, convert it to inherit from `opik.Config`:

1. Replace the existing base (`@dataclass`, `BaseModel`, plain class) with `opik.Config`
2. Convert plain `str` prompt fields to `opik.Prompt`
3. Wire up `get_or_create_config()` inside the entrypoint
4. Update all call sites that reference the old config to use the new Opik-managed config

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
        project_name="my-agent",
        prompt="You are a helpful assistant for {{product}}.",
    ),
)

client = opik.Opik()

@opik.track(entrypoint=True, project_name="my-agent")
def run_agent(question: str) -> str:
    cfg = client.get_or_create_config(
        fallback=DEFAULT_CONFIG,
        project_name="my-agent",
        # optional: env="staging" | version="v1" | version="latest" (default: prod)
    )
    return llm_call(
        model=cfg.model,
        temperature=cfg.temperature,
        system_prompt=cfg.system_prompt.format(product="Opik"),
        question=question,
    )
```

- `get_or_create_config()` **must** be inside `@opik.track` — raises error otherwise
- On first call with no existing config, auto-creates from `fallback` and returns it
- On backend failure, returns `fallback` with `is_fallback=True` (never breaks the agent)
- Deploy to environment: `client.set_config_env(version="v1", env="prod")` — admin/ops only
- Prompt fields: use `opik.Prompt` for string-based templates, `opik.ChatPrompt` for multi-turn message templates; `project_name` is required on both and must match the `project_name` in `@opik.track` and `get_or_create_config`
- **Extract:** model, temperature, top_p, max_tokens, system prompt, tunable params
- **Don't extract:** API keys, structural logic, true constants

## Local Runner (opik connect)

Pair your local agent with the Opik browser UI. Get a pairing code from the UI, then:

```bash
opik connect --pair <CODE> python3 app.py        # Python
opik connect --pair <CODE> npx tsx app.ts         # TypeScript
```

Replace `python3 app.py` or `npx tsx app.ts` with the normal command you use to start your app locally.

Python: `@track(entrypoint=True)` + type-hinted parameters for schema discovery.
TypeScript: `track({ entrypoint: true, params: [{name, type}] }, fn)`.

After pairing: entrypoint registered as agent, UI shows input form, jobs from UI or Optimizer trigger runs.

| Issue | Fix |
|-------|-----|
| No entrypoint found | Add `entrypoint=True` (Python) or `entrypoint: true` (TS) |
| Invalid pair code | Codes expire — get a new one |
| Connection refused | Check Opik server (OSS) or API key (Cloud) |
| `get_or_create_config` fails saying some fields reference the wrong project | The `project_name` on one or more `opik.Prompt` / `opik.ChatPrompt` fields doesn't match the `project_name` passed to `get_or_create_config` — make them consistent |


## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Existing config class left unconverted (e.g., `@dataclass` with model/temperature/prompt fields) | Convert to `opik.Config` subclass — an existing config is a migration target, not a skip signal |
| Hardcoded config | Use `opik.Config` + `get_or_create_config()` |
| Missing entrypoint | Add `entrypoint=True` for Local Runner |
| No thread_id on conversational agent | Wire `thread_id` from session ID |
| `get_or_create_config()` outside `@track` | Must be inside decorated function |
| TS missing `params` | Add explicit `params` array |
| Missing `flush_tracker()` in scripts | Call before exit |

## References

| Topic | File |
|-------|------|
| Python SDK (decorators, async, distributed, config, entrypoint) | `references/tracing-python.md` |
| TypeScript SDK (client, decorators, entrypoint, params) | `references/tracing-typescript.md` |
| REST API | `references/tracing-rest-api.md` |
| All integrations | `references/integrations.md` |
| Core concepts (traces, spans, threads, metadata) | `references/observability.md` |
| Test Suites, `run_tests()`, 60+ built-in metrics, legacy `evaluate()` | `references/evaluation.md` |

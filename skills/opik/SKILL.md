---
name: opik
description: Opik observability for LLM agents — tracing (Python + TypeScript), Agent Configuration, Local Runner (opik connect), Evaluation Suites, threads, integrations. Use for "add tracing", "instrument my code", "configure my agent", "connect my agent", "evaluate my agent" or "integrate with Opik".
---

# Opik — Observability for LLM Agents

## Setup

Use the existing Opik config style in the target project if one already exists. Do not introduce a second config mechanism unless the user asks for it.

Default behavior for agents:

- If the user explicitly asks for environment variables, or the target is TypeScript/JavaScript, update `.env.local` if it exists, otherwise `.env`
- If the target is Python and there is no existing project-level config style, create or update `~/.opik.config`
- Prefer setting project names in code, not in shared machine config, because one machine may log to many projects

Standard deployments:

- Cloud: `https://www.comet.com/opik/api` and requires `api_key` plus `workspace`
- Local OSS: `http://localhost:5173/api` and usually uses workspace `default`
- Self-hosted: use the deployment's custom URL and credentials, following the same config style the project already uses

Python default config file:

```ini
[opik]
api_key=your-api-key
url_override=https://www.comet.com/opik/api
workspace=your-workspace
```

Local OSS example:

```ini
[opik]
url_override=http://localhost:5173/api
workspace=default
```

Environment variables:

```bash
export OPIK_API_KEY="your-api-key"
export OPIK_URL_OVERRIDE="https://www.comet.com/opik/api"
export OPIK_WORKSPACE="your-workspace"
# export OPIK_PROJECT_NAME="optional-project-env"
```

TypeScript uses `OPIK_WORKSPACE` as the environment variable and `workspaceName` in `new Opik({...})`.

Optional interactive/manual paths:

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

**Framework integrations** (prefer over manual `@opik.track` — capture tokens, model, cost):

```python
from opik.integrations.openai import track_openai        # OpenAI
from opik.integrations.anthropic import track_anthropic   # Anthropic
from opik.integrations.langchain import OpikTracer        # LangChain
from opik.integrations.crewai import track_crewai         # CrewAI
from opik.integrations.dspy import OpikCallback           # DSPy
from opik.integrations.adk import track_adk_agent_recursive  # Google ADK
```

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
    SessionCompletenessMetric, UserFrustrationMetric, ConversationalCoherenceMetric,
)

results = evaluate_threads(project_name="chat-agent", metrics=[
    SessionCompletenessMetric(), UserFrustrationMetric(), ConversationalCoherenceMetric(),
])
```

Use for chat agents, support bots, multi-step assistants. Skip for single-shot agents or batch processing.

**Pitfalls:** Missing `thread_id` → turns appear as unrelated traces. Shared `thread_id` across users → conversations get mixed.

## Agent Configuration

Externalize the parts of your agent you expect to tune over time into versioned, immutable config snapshots. This includes prompts, models, temperatures, token limits, and other runtime parameters you may want to compare, optimize, or roll out gradually.

```python
from typing import Annotated
import opik

class AgentConfig(opik.AgentConfig):
    model: Annotated[str, "LLM model"]             # NO defaults
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

client = opik.Opik()
client.create_agent_config_version(
    AgentConfig(
        model="gpt-4o",
        temperature=0.7,
        system_prompt=opik.Prompt(
            name="agent-system-prompt",
            prompt="You are a helpful assistant for {{product}}.",
        ),
    ),
    project_name="my-agent",
)
# Identical values → same version (dedup). Different values → new version.

@opik.track(entrypoint=True, project_name="my-agent")
def run_agent(question: str) -> str:
    cfg = client.get_agent_config(
        fallback=DEFAULT_AGENT_CONFIG,
        project_name="my-agent",
        # optional: latest=True | env="staging" | version="v1" (default: prod)
    )
    return llm_call(
        model=cfg.model,
        temperature=cfg.temperature,
        system_prompt=cfg.system_prompt.format(product="Opik"),
        question=question,
    )
```

- `get_agent_config()` **must** be inside `@opik.track` — raises error otherwise
- Deploy: `cfg.deploy_to("prod")` — tags a version with an environment
- Prompt fields: use `Prompt` (from `opik.api_objects.prompt.text.prompt`) / `ChatPrompt` (from `opik.api_objects.prompt.chat.chat_prompt`) typed config fields for managed prompts
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


## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Hardcoded config | Use `AgentConfig` + `get_agent_config()` |
| Missing entrypoint | Add `entrypoint=True` for Local Runner |
| No thread_id on conversational agent | Wire `thread_id` from session ID |
| `get_agent_config()` outside `@track` | Must be inside decorated function |
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
| Evaluation (suites, 41 built-in metrics, trajectory) | `references/evaluation.md` |

---
name: opik
description: Opik observability for LLM agents — tracing (Python + TypeScript), Agent Configuration (Blueprints), Local Runner (opik connect), Evaluation Suites, threads, integrations. Use for "add tracing", "instrument my code", "configure my agent", "connect my agent", "evaluate my agent".
---

# Opik — Observability for LLM Agents

## Setup

```bash
# Python
pip install opik && opik configure   # prompts for API key — https://www.comet.com/signup

# TypeScript
npm install opik
```

```bash
export OPIK_API_KEY="your-api-key"
export OPIK_URL_OVERRIDE="https://www.comet.com/opik/api"  # Cloud
export OPIK_PROJECT_NAME="my-project"
export OPIK_WORKSPACE="my-workspace"       # Cloud: required. OSS: "default"
# TypeScript uses OPIK_WORKSPACE_NAME (note _NAME suffix)
```

## Span Types

| Type | Use For |
|------|---------|
| `general` | Orchestration, entry points |
| `llm` | LLM API calls |
| `tool` | Tool execution, retrieval |
| `guardrail` | Safety/validation checks |

**Only** these four types are valid.

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

**Framework integrations** (prefer over manual `@opik.track` — capture tokens, model, cost):

```python
from opik.integrations.openai import track_openai        # OpenAI
from opik.integrations.anthropic import track_anthropic   # Anthropic
from opik.integrations.langchain import OpikTracer        # LangChain
from opik.integrations.crewai import track_crewai         # CrewAI
from opik.integrations.dspy import OpikCallback           # DSPy
from opik.integrations.adk import track_adk_agent_recursive  # Google ADK
```

Don't double-wrap — use integration OR `@opik.track`, not both.

## TypeScript Instrumentation

```typescript
import { track } from "opik";

const myAgent = track(
  { name: "my-agent", entrypoint: true, params: [{ name: "query", type: "string" }] },
  async (query: string) => {
    // agent logic
    return result;
  }
);
```

**`params` must be explicit** — TS compilation strips param names/types. If omitted, all assumed `string`.

**Framework integrations:**

```typescript
import { trackOpenAI } from "opik/openai";   // OpenAI
import { OpikTracer } from "opik/vercel";     // Vercel AI SDK
import { OpikTracer } from "opik";            // LangChain.js
```

Always `await client.flush()` before exit.

## Agent Configuration (Blueprints)

Externalize tunable parameters into versioned, immutable config snapshots.

```python
from typing import Annotated
import opik

class AgentConfig(opik.AgentConfig):
    model: Annotated[str, "LLM model"]             # NO defaults
    temperature: Annotated[float, "Sampling temperature"]

client = opik.Opik()
client.create_agent_config_version(
    AgentConfig(model="gpt-4o", temperature=0.7), project_name="my-agent",
)
# Identical values → same version (dedup). Different values → new version.

@opik.track(entrypoint=True, project_name="my-agent")
def run_agent(question: str) -> str:
    cfg = client.get_agent_config(
        fallback=AgentConfig(model="gpt-4o", temperature=0.7),
        project_name="my-agent",
        latest=True,   # OR env="prod" OR version="v1" — exactly one selector
    )
    return llm_call(model=cfg.model, temperature=cfg.temperature)
```

- `get_agent_config()` **must** be inside `@opik.track` — raises error otherwise
- Deploy: `cfg.deploy_to("prod")` — tags a version with an environment
- Prompt fields: use `Prompt` / `ChatPrompt` typed fields for managed prompts
- **Extract:** model, temperature, top_p, max_tokens, system prompt, tunable params
- **Don't extract:** API keys, structural logic, true constants

## Local Runner (opik connect)

Pair your local agent with the Opik browser UI. Get a pairing code from the UI, then:

```bash
opik connect --pair <CODE> python3 app.py        # Python
opik connect --pair <CODE> npx tsx app.ts         # TypeScript
```

Python: `@track(entrypoint=True)` + type-hinted parameters for schema discovery.
TypeScript: `track({ entrypoint: true, params: [{name, type}] }, fn)`.

After pairing: entrypoint registered as agent, UI shows input form, jobs from UI or Optimizer trigger runs.

| Issue | Fix |
|-------|-----|
| No entrypoint found | Add `entrypoint=True` (Python) or `entrypoint: true` (TS) |
| Invalid pair code | Codes expire — get a new one |
| Connection refused | Check Opik server (OSS) or API key (Cloud) |

## Evaluation Suites

```python
client = opik.Opik()
suite = client.get_or_create_evaluation_suite(
    name="my-suite",
    assertions=["Response is factually accurate", "Response is professional"],
    execution_policy={"runs_per_item": 3, "pass_threshold": 2},
)
suite.add_item(data={"input": "What is ML?"})
results = suite.run(
    task=lambda item: {"output": agent(item["input"])},
    model="gpt-4o",  # LLM judge
)
assert results.all_passed  # CI gate
```

Use `get_or_create_evaluation_suite()` — NOT `get_or_create_dataset()`. Suite-level assertions apply to all items; item-level assertions are additive.

## Threads (Conversations)

Group conversation turns via `thread_id`. Each turn = one trace; shared `thread_id` = one thread.

```python
@opik.track(entrypoint=True)
def handle_message(session_id: str, message: str) -> str:
    opik.update_current_trace(thread_id=session_id)
    return generate_response(session_id, message)
```

Thread metrics: `SessionCompletenessMetric`, `UserFrustrationMetric`, `ConversationalCoherenceMetric` via `evaluate_threads()`.

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
| Agent patterns (architecture, reliability, security) | `references/agent-patterns.md` |
| Evaluation (suites, 41 built-in metrics, trajectory) | `references/evaluation.md` |
| Production (dashboards, alerts, guardrails) | `references/production.md` |

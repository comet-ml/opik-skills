---
name: opik
description: Opik observability for LLM agents — Prompt Library, Local Runner (opik connect), Test Suites, threads, integrations. Use for "manage my prompts", "connect my agent", "evaluate my agent" or "integrate with Opik".
---

# Opik — Observability for LLM Agents

Integrating with Opik always means adding both components unless the user explicitly asks for only one:

1. **Tracing** — instrument LLM calls with the appropriate integration or `@opik.track`
2. **Entrypoint** — mark the top-level function with `entrypoint=True` for Local Runner and UI integration

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

## Prompt Library

Manage versioned prompts through the `opik.Opik` client. Use `create_prompt` / `get_prompt` for string-based prompts and `create_chat_prompt` / `get_chat_prompt` for multi-turn chat templates. Use `{{variable}}` syntax in prompt text for template variables rendered at call time via `.format()`.

**Storing model config alongside the prompt.** Model names, temperatures, and other parameters that you want to version together with the prompt text go in the `metadata` dict on the prompt. They are stored at the prompt version level, so when you fetch a prompt you get both the template and its associated config from `prompt.metadata`.

**CRITICAL — call `get_prompt` / `get_chat_prompt` inside a `@opik.track`-decorated function.** This is what links the fetched prompt version to the trace, making it visible in the Traces view in the Opik UI. Fetching at module level works but the prompt will not appear in traces.

**Python:**

```python
import opik

client = opik.Opik()

@opik.track(entrypoint=True, project_name="my-agent")
def run_agent(question: str) -> str:
    # Fetch inside @track so the prompt version is recorded in the trace
    prompt = client.get_prompt(name="agent-system-prompt")
    if prompt is None:
        prompt = client.create_prompt(
            name="agent-system-prompt",
            prompt="You are a helpful assistant for {{product}}.",
            metadata={"model": "gpt-4o", "temperature": 0.7, "max_tokens": 1024},
        )
    system_message = prompt.format(product="Opik")
    return llm_call(
        model=prompt.metadata["model"],
        temperature=prompt.metadata["temperature"],
        max_tokens=prompt.metadata["max_tokens"],
        system_prompt=system_message,
        question=question,
    )
```

For a multi-turn chat template:

```python
@opik.track(entrypoint=True, project_name="my-agent")
def run_agent(task: str) -> str:
    chat_prompt = client.get_chat_prompt(name="agent-chat-template")
    if chat_prompt is None:
        chat_prompt = client.create_chat_prompt(
            name="agent-chat-template",
            messages=[
                {"role": "system", "content": "You are a helpful assistant."},
                {"role": "user", "content": "Help me with {{task}}"},
            ],
            metadata={"model": "gpt-4o", "temperature": 0.7},
        )
    messages = chat_prompt.format(task=task)
    return llm_call(
        model=chat_prompt.metadata["model"],
        temperature=chat_prompt.metadata["temperature"],
        messages=messages,
    )
```

**TypeScript:**

```typescript
import { Opik, track } from "opik";

const client = new Opik({ projectName: "my-agent" });

const runAgent = track({ entrypoint: true, projectName: "my-agent" }, async (question: string) => {
    // Fetch inside track() so the prompt version is recorded in the trace
    let prompt = await client.getPrompt({ name: "agent-system-prompt" });
    if (prompt === null) {
        prompt = await client.createPrompt({
            name: "agent-system-prompt",
            prompt: "You are a helpful assistant for {{product}}.",
            metadata: { model: "gpt-4o", temperature: 0.7, maxTokens: 1024 },
        });
    }
    const systemMessage = prompt.format({ product: "Opik" });
    const { model, temperature, maxTokens } = prompt.metadata as { model: string; temperature: number; maxTokens: number };
    return llmCall({ model, temperature, maxTokens, systemMessage, question });
});
```

After the initial run the prompt is registered in the library and can be edited, versioned, and have its metadata updated from the Opik UI. `get_prompt` / `get_chat_prompt` always returns the latest published version, including its metadata.

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
| Using deprecated `opik.Prompt` / `opik.ChatPrompt` / `opik.Config` | Migrate to `client.get_prompt()` / `client.get_chat_prompt()` from the Prompt library |
| Storing model/temperature in a separate config object | Put them in `metadata` on the prompt — they version together with the template and are read via `prompt.metadata["model"]` etc. |
| Fetching prompt outside `@opik.track` | Prompt won't appear in traces — fetch inside the decorated function |
| Missing entrypoint | Add `entrypoint=True` for Local Runner |
| No thread_id on conversational agent | Wire `thread_id` from session ID |
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

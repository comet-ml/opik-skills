---
name: instrument-tracing
description: Add Opik tracing to an existing codebase. Detects language (Python/TypeScript), identifies LLM frameworks, adds appropriate decorators and integrations, marks entrypoints, and wires up environment config. Use for "instrument my code", "add opik tracing", "add observability", or "trace my agent".
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

# Instrument Tracing — Add Opik Tracing to a Codebase

You are instrumenting an existing codebase with Opik observability. Follow these steps precisely.

## Step 1 — Scope

If `$ARGUMENTS` is provided, scope your work to those files or directories. Otherwise, discover the project root and instrument the main application code.

## Step 2 — Detect Language & Frameworks

Scan the codebase to determine:

1. **Language**: Python (look for `*.py`, `pyproject.toml`, `requirements.txt`) or TypeScript (look for `*.ts`, `*.tsx`, `package.json`)
2. **LLM frameworks in use** — search imports for these patterns:

| Import pattern | Framework | Integration |
|---|---|---|
| `from openai` / `import OpenAI` | OpenAI | `track_openai` |
| `import anthropic` | Anthropic | `track_anthropic` |
| `from langchain` / `@langchain` | LangChain | `OpikTracer` callback |
| `from langgraph` | LangGraph | `OpikTracer` with `graph=` |
| `from crewai` | CrewAI | `track_crewai` |
| `import dspy` | DSPy | `OpikCallback` |
| `from google` … `genai` | Google Gemini | `track_genai` |
| `import boto3` … `bedrock` | AWS Bedrock | `track_bedrock` |
| `from llama_index` | LlamaIndex | `LlamaIndexCallbackHandler` |
| `import litellm` | LiteLLM | `OpikLogger` callback |
| `from pydantic_ai` | Pydantic AI | Logfire OTLP bridge |
| `from opik.integrations.adk` / `from google.adk` | Google ADK | `track_adk_agent_recursive` |
| `import ollama` | Ollama | `track_openai` with localhost base_url or manual `@opik.track` |
| `from agents import` / `from openai.agents` | OpenAI Agents SDK | `OpikTracingProcessor` |
| `from haystack` | Haystack | `OpikConnector` |
| `opik-openai` / `trackOpenAI` (TS) | OpenAI (TS) | `trackOpenAI` |
| `opik-vercel` / `OpikExporter` (TS) | Vercel AI SDK | `OpikExporter` |
| `opik-langchain` / `OpikCallbackHandler` (TS) | LangChain.js | `OpikCallbackHandler` |
| `opik-gemini` / `trackGemini` (TS) | Gemini (TS) | `trackGemini` |

3. **Existing Opik usage** — check if `opik` or `@opik.track` is already imported. If so, audit rather than re-instrument.

## Step 3 — Identify the Call Graph

Find:
- **Entrypoint**: the top-level function that kicks off the agent (e.g., `main`, `run`, `agent`, `handle_message`, a route handler, or whatever the user's main orchestration function is)
- **LLM call sites**: functions that call an LLM provider directly
- **Tool functions**: retrieval, search, API calls, or other tool-like operations
- **Existing config classes**: dataclasses, Pydantic models, or plain classes holding model names, temperatures, prompts, or other tunable parameters — note these for a future `/opik:instrument-agent-config` pass

## Step 4 — Add Framework Integrations

For each detected framework, add the appropriate integration at the module level. See the integration table above and `references/integrations.md` for the exact patterns.

**Python examples:**

```python
# OpenAI
from opik.integrations.openai import track_openai
client = track_openai(OpenAI())  # wrap existing client

# Anthropic
from opik.integrations.anthropic import track_anthropic
client = track_anthropic(anthropic.Anthropic())

# LangChain / LangGraph
from opik.integrations.langchain import OpikTracer
tracer = OpikTracer()
# pass config={"callbacks": [tracer]} to invoke()

# LiteLLM inside @opik.track — CRITICAL: pass span context
from opik.opik_context import get_current_span_data
# in every litellm.completion() call, add:
#   metadata={"opik": {"current_span_data": get_current_span_data()}}
```

**TypeScript examples:**

```typescript
// OpenAI
import { trackOpenAI } from "opik-openai";
const trackedClient = trackOpenAI(openai);

// Vercel AI SDK
import { OpikExporter } from "opik-vercel";
// set up NodeSDK with OpikExporter
```

## Step 5 — Add `@opik.track` Decorators (Python) or Client Tracing (TypeScript)

### Python

Add `import opik` at the top of each file you instrument.

| Function role | Decorator |
|---|---|
| Entrypoint (top-level agent) | `@opik.track(entrypoint=True, name="<agent-name>")` |
| LLM call | `@opik.track(type="llm")` |
| Tool / retrieval | `@opik.track(type="tool")` |
| Guardrail / validation | `@opik.track(type="guardrail")` |
| Other helper in the call chain | `@opik.track` |

- Place the decorator **above** any existing decorators (e.g., above `@app.route`)
- For async functions, `@opik.track` works the same way — no changes needed
- If the function is a **script entrypoint** (not a long-running server), add `opik.flush_tracker()` after the top-level call

### TypeScript

Use the client-based approach:

```typescript
import { Opik } from "opik";
const client = new Opik({ projectName: "<project-name>" });

// In the entrypoint function:
const trace = client.trace({ name: "<agent-name>", input: { ... } });
const span = trace.span({ name: "<operation>", type: "tool", input: { ... } });
// ... logic
span.end({ output: { ... } });
trace.end({ output: { ... } });
await client.flush();
```

For entrypoints that should be discoverable by `opik connect`:

```typescript
import { track } from "opik";

const myAgent = track(
  { name: "<agent-name>", entrypoint: true, params: [{ name: "query", type: "string" }] },
  async (query: string) => { /* ... */ }
);
```

## Step 6 — Conversational Agents: Add `thread_id`

If the agent handles multi-turn conversations (chat bots, support agents, multi-step assistants), wire `thread_id`:

```python
@opik.track(entrypoint=True)
def handle_message(session_id: str, message: str) -> str:
    opik.update_current_trace(thread_id=session_id)
    return generate_response(session_id, message)
```

Skip this for single-shot agents or batch processing.

## Step 7 — Environment Config

Follow the setup decision tree from the main opik skill:

1. If the project has `.env` / `.env.local` → append `OPIK_API_KEY`, `OPIK_WORKSPACE`, `OPIK_URL_OVERRIDE` (if missing)
2. If no `.env` exists → Python: create/update `~/.opik.config`; TypeScript: create `.env` or `.env.local`
3. Never introduce a second config mechanism
4. Never overwrite existing values
5. Update `.env.example` / `.env.sample` if one exists
6. Set `project_name` in code, not in env files

## Step 8 — Install Dependencies

Print the install command but do NOT run it automatically. Let the user decide.

**Python:**
```
pip install opik
```
Plus any integration packages if needed (most are included in `opik`).

**TypeScript:**
```
npm install opik
```
Plus framework-specific packages: `opik-openai`, `opik-vercel`, `opik-langchain`, `opik-gemini` as needed.

## Step 9 — Verify

After instrumentation, do a quick audit:

- [ ] Every LLM call site is traced (via integration wrapper or `@opik.track`)
- [ ] Exactly one function has `entrypoint=True`
- [ ] Script entrypoints call `opik.flush_tracker()` (Python) or `await client.flush()` (TypeScript)
- [ ] LiteLLM calls inside `@opik.track` pass `current_span_data` via metadata
- [ ] No hardcoded API keys were introduced
- [ ] Existing tests still import correctly (no circular imports introduced)

## Anti-Patterns to Avoid

- **Double-wrapping**: Don't add `@opik.track(type="llm")` to a function that already uses a framework integration (e.g., `track_openai`). The integration handles tracing.
- **Orphaned LiteLLM traces**: Always pass `current_span_data` when `OpikLogger` is used inside `@opik.track` code.
- **Missing entrypoint**: Without `entrypoint=True`, Local Runner (`opik connect`) won't discover the agent.
- **Missing flush**: Scripts that exit without flushing lose trace data.
- **Overwriting config**: Check before writing to `.env` or `~/.opik.config`.

## References

For detailed API signatures and advanced patterns, see:
- `references/tracing-python.md` — Python SDK reference
- `references/tracing-typescript.md` — TypeScript SDK reference
- `references/integrations.md` — All framework integrations
- `references/observability.md` — Core concepts (traces, spans, threads)

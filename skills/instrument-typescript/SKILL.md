---
name: instrument-typescript
description: Adding Opik observability to TypeScript/JS LLM apps — track() with entrypoint and explicit params for Local Runner, framework integrations.
---

# Instrument TypeScript Agents

## Basic Tracing

```typescript
import { Opik } from "opik";

const client = new Opik({ projectName: process.env.OPIK_PROJECT_NAME || "my-agent" });

const trace = client.trace({ name: "my-agent", input: { query } });
const span = trace.span({ name: "llm-call", type: "llm" });
span.end({ output: { response } });
trace.end({ output: { response } });
await client.flush();
```

## Entrypoint (Local Runner)

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

### Express Example

```typescript
import express from "express";
import { track } from "opik";

const summarize = track(
  { name: "summarize", entrypoint: true, params: [{ name: "message", type: "string" }] },
  async (message: string) => `Summary of: ${message}`
);

const app = express();
app.get("/summarize", async (req, res) => {
  res.send(await summarize(String(req.query.message ?? "")));
});
app.listen(3000);
```

Pair with: `opik connect --pair <CODE> npx tsx app.ts`

## Framework Integrations

```typescript
import { trackOpenAI } from "opik/openai";   // OpenAI
import { OpikTracer } from "opik/vercel";     // Vercel AI SDK
import { OpikTracer } from "opik";            // LangChain.js
```

## Pitfalls

- Always `await client.flush()` before exit
- Span types: `general`, `llm`, `tool`, `guardrail` only
- Missing `params` in `track()` means UI can't show typed input fields

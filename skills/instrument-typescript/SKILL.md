---
name: instrument-typescript
description: Step-by-step guide for adding Opik observability to TypeScript/JavaScript LLM applications. Covers the Opik client, track() function with entrypoint and explicit params for Local Runner, framework integrations, and tracing patterns.
---

# Instrument TypeScript Agents with Opik

Guide to making your TypeScript/JavaScript agent observable with Opik.

## Quick Start

```typescript
import { Opik } from "opik";

const client = new Opik({
  projectName: process.env.OPIK_PROJECT_NAME || "my-agent",
});

async function agent(query: string): Promise<string> {
  const trace = client.trace({
    name: "my-agent",
    input: { query },
  });

  const searchSpan = trace.span({ name: "search", type: "tool" });
  const context = await searchDB(query);
  searchSpan.end({ output: { context } });

  const llmSpan = trace.span({ name: "generate", type: "llm" });
  const response = await llmCall(query, context);
  llmSpan.end({ output: { response } });

  trace.end({ output: { response } });
  await client.flush();
  return response;
}
```

## Entrypoint Functions (Local Runner)

Mark functions as entrypoints so they can be triggered from the Opik UI via `opik connect`.

```typescript
import { track } from "opik";

const myAgent = track(
  {
    name: "my-agent",
    entrypoint: true,
    params: [{ name: "query", type: "string" }],
  },
  async (query: string) => {
    // agent logic here
    return `Response to: ${query}`;
  }
);
```

> **Important:** Due to TypeScript compilation stripping parameter names and types at runtime, you **must** explicitly declare `params` with `[{ name: "...", type: "..." }]`. If `params` is omitted, all parameters are assumed to be `string` type.

### Wiring into an Express Server

```typescript
import express from "express";
import { track } from "opik";

const summarize = track(
  {
    name: "summarize",
    entrypoint: true,
    params: [{ name: "message", type: "string" }],
  },
  async (message: string) => {
    // call your LLM here
    return `Summary of: ${message}`;
  }
);

const app = express();

app.get("/summarize", async (req, res) => {
  const result = await summarize(String(req.query.message ?? ""));
  res.send(result);
});

app.listen(3000, () => console.log("Listening on port 3000"));
```

Then pair with: `opik connect --pair <CODE> npx tsx summarise.ts`

### Multiple Parameters

```typescript
const research = track(
  {
    name: "research-agent",
    entrypoint: true,
    params: [
      { name: "topic", type: "string" },
      { name: "depth", type: "number" },
      { name: "include_sources", type: "boolean" },
    ],
  },
  async (topic: string, depth: number, includeSources: boolean) => {
    // research logic
    return result;
  }
);
```

## Framework Integrations

### OpenAI
```typescript
import { Opik } from "opik";
import { trackOpenAI } from "opik/openai";
import OpenAI from "openai";

const client = new Opik();
const openai = trackOpenAI(new OpenAI(), { client });
```

### Vercel AI SDK
```typescript
import { Opik } from "opik";
import { OpikTracer } from "opik/vercel";

const client = new Opik();
const tracer = new OpikTracer({ client });
```

## Thread ID for Conversations

```typescript
const trace = client.trace({
  name: "chat-turn",
  threadId: sessionId,  // Groups turns into a thread
  input: { message },
});
```

## Common Pitfalls

- **Missing flush**: Always `await client.flush()` before process exit
- **Project name**: Set via `OPIK_PROJECT_NAME` env var, not hardcoded
- **Span types**: Only `general`, `llm`, `tool`, `guardrail` are valid
- **Missing params in track()**: Without explicit `params`, the UI can't show typed input fields for the entrypoint
- **Entrypoint not found**: Ensure `entrypoint: true` is set in the `track()` options object

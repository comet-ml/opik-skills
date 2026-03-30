---
name: opik-connect
description: Guide for setting up Opik Connect (Local Runner) to pair your local agent with the Opik browser UI. Covers Python and TypeScript pairing, entrypoint registration, the opik connect command, and troubleshooting.
---

# Opik Connect (Local Runner)

Opik Connect lets you trigger your agent from the Opik browser UI while it runs on your local machine. The Local Runner registers your entrypoint functions, and the UI provides an input form derived from those entrypoints' parameters.

## Prerequisites

1. **opik CLI installed**: `pip install opik`
2. **Agent instrumented** with `entrypoint=True` on the main function(s)
3. **For Python**: Docstring with `Args:` on the entrypoint (for UI input form)
4. **For TypeScript**: Explicit `params` array in the `track()` options (for UI input form)

## Python Setup

Wrap your server endpoints with `@track(entrypoint=True)`. For a FastAPI app:

```python
# echo_app.py
from fastapi import FastAPI
import uvicorn
from opik import track

app = FastAPI()

@app.get("/echo")
@track(entrypoint=True)
async def echo(message: str = "") -> str:
    """Echo the message back.

    Args:
        message: The message to echo.
    """
    print(f"[LOG] {message} received")
    return f"{message}, world!"

uvicorn.run(app, host="0.0.0.0", port=3000)
```

The entrypoint name defaults to the function name. Customize it:

```python
@track(entrypoint=True, name="fantastic_agent")
async def echo(message: str = "") -> str:
```

### Python Pairing

1. Go to the Opik UI and get a pairing code
2. Run:

```bash
opik connect --pair <CODE> python3 echo_app.py
```

That's it. `echo` is registered as an agent. Its parameters (`message`) appear as the input form in the UI.

## TypeScript Setup

Similarly, designate entrypoints in your server app using the `track` function:

```typescript
// summarise.ts
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

> **Important:** Due to TypeScript compilation stripping parameter names and types at runtime, you **must** explicitly declare `params` with `[{ name: "...", type: "..." }]`. If `params` is omitted, all parameters are assumed to be `string` type.

### TypeScript Pairing

1. Go to the Opik UI and get a pairing code
2. Run:

```bash
opik connect --pair <CODE> npx tsx summarise.ts
```

## What Happens After Pairing

1. The runner registers the entrypoint function(s) with Opik
2. The Opik UI shows the agent with an input form (derived from the docstring `Args:` in Python or `params` in TypeScript)
3. User types input in the browser, clicks Run, and the agent executes locally
4. Full trace appears in Opik with spans, token usage, and cost
5. Any job created by the user in the UI or by the Optimizer triggers an agent run

## Features

| Feature | Description |
|---------|-------------|
| UI triggering | Type input in browser, execute locally |
| Trace replay | Click "Re-run" on any trace |
| Config iteration | Edit config in UI, re-run, compare |
| Parallel jobs | Runner handles concurrent executions |
| Optimizer integration | Optimizer creates jobs that trigger agent runs via the runner |

## Cloud vs OSS

### Cloud (API Key)

```bash
opik configure  # Set API key if not done
opik connect --pair <CODE> python3 app.py
```

### OSS (Self-Hosted)

```bash
# 1. Open Opik UI in browser
# 2. Click "Connect Agent" to get a pair code
# 3. Run:
opik connect --pair <CODE> python3 app.py
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "No entrypoint found" | Add `entrypoint=True` to `@track` (Python) or `track({ entrypoint: true }, ...)` (TypeScript) |
| "Connection refused" | Check Opik server is running (OSS) or API key is valid (Cloud) |
| "Invalid pair code" | Codes expire — get a new one from the UI |
| "Port in use" | Another runner may be active |
| Runner disconnects | Check network; runner auto-reconnects |
| Agent not showing in UI | Python: verify the entrypoint docstring has `Args:`. TypeScript: verify `params` array is provided |
| TS params not showing | Add explicit `params: [{ name: "...", type: "..." }]` to the `track()` options |

## Networking (OSS)

- Runner connects outbound to the Opik server (no inbound ports needed)
- Default: connects to `http://localhost:5173`
- Override: set `OPIK_BASE_URL` env var
- Heartbeat keeps connection alive during idle periods

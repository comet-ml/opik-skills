---
name: opik-connect
description: Opik Connect (Local Runner) — pair your local agent with the Opik browser UI for Python and TypeScript.
---

# Opik Connect (Local Runner)

Trigger your agent from the Opik browser UI while it runs locally.

## Python

Wrap server endpoints with `@track(entrypoint=True)`:

```python
from fastapi import FastAPI
from opik import track
import uvicorn

app = FastAPI()

@app.get("/echo")
@track(entrypoint=True)
async def echo(message: str = "") -> str:
    """Echo the message back.

    Args:
        message: The message to echo.
    """
    return f"{message}, world!"

uvicorn.run(app, host="0.0.0.0", port=3000)
```

Customize the agent name: `@track(entrypoint=True, name="fantastic_agent")`

## TypeScript

Use `track()` with explicit `params` (required — TS compilation strips param names/types):

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

If `params` omitted, all parameters assumed `string` type.

## Pairing

Get a pairing code from the Opik UI, then:

```bash
opik connect --pair <CODE> python3 echo_app.py      # Python
opik connect --pair <CODE> npx tsx summarise.ts      # TypeScript
```

After pairing: entrypoint registered as agent, UI shows input form from type hints (Python) or `params` (TypeScript), jobs from UI or Optimizer trigger runs.

## Troubleshooting

| Issue | Fix |
|-------|-----|
| No entrypoint found | Add `entrypoint=True` (Python) or `entrypoint: true` (TS) |
| Invalid pair code | Codes expire — get a new one |
| Connection refused | Check Opik server (OSS) or API key (Cloud) |

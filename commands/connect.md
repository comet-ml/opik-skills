---
description: Connect your agent to Opik for triggering from the browser UI via the Local Runner
argument-hint: [--pair CODE]
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
model: haiku
---

# Connect Agent to Opik (Local Runner)

Set up `opik connect` so the user's agent can be triggered from the Opik browser UI while running locally.

**User request:** $ARGUMENTS

## Step 1: Check Prerequisites

### 1a. Verify opik CLI is installed

Run `opik --version`. If not found:
- Check if `opik` is installed: `pip show opik` or `pip3 show opik`
- If not installed: `pip install opik`
- If installed but not on PATH: suggest `python -m opik --version`

### 1b. Verify there's an entrypoint function

Search the codebase for entrypoints in both Python and TypeScript:

**Python:**
```
grep -r "entrypoint=True" --include="*.py" .
```

**TypeScript:**
```
grep -r "entrypoint: true" --include="*.ts" --include="*.js" .
```

If no entrypoint found:
- Tell the user: "No entrypoint function found. Run `/opik:instrument` first to add an entrypoint to your main agent function."
- Stop here.

### 1c. Verify the entrypoint has schema information

**Python:** Check the entrypoint function has a docstring with `Args:` descriptions. The Local Runner uses this to build the input form in the UI. If missing, add it.

**TypeScript:** Check the `track()` call has a `params` array with `[{ name: "...", type: "..." }]`. If missing, add it — TypeScript compilation strips parameter names/types at runtime.

## Step 2: Detect Cloud vs OSS

Check for Opik configuration:

1. Check `OPIK_API_KEY` env var
2. Check `~/.opik.config` for `api_key` field
3. Check `OPIK_BASE_URL` or `url_override` in config

**If API key exists** -> Cloud mode
**If no API key but URL points to localhost** -> OSS mode
**If neither** -> Run `opik configure` first

## Step 3: Determine the App Startup Command

Identify the correct command to start the user's application:

**Python:**
- If using `uvicorn` (FastAPI): `python3 app.py` or the entry script
- If using Flask/Django: the appropriate run command
- If a script: `python3 script.py`

**TypeScript:**
- If using `tsx`: `npx tsx app.ts`
- If using `ts-node`: `npx ts-node app.ts`
- If compiled JS: `node app.js`

## Step 4: Connect

### Cloud Mode

```bash
opik connect --pair <CODE> <startup_command>
```

Example:
```bash
opik connect --pair ABC123 python3 app.py
```

### OSS Mode

1. Tell the user: "Open the Opik UI in your browser and look for the 'Connect Agent' button to get a pairing code."
2. Once they provide the code:

```bash
opik connect --pair <CODE> <startup_command>
```

Examples:
```bash
# Python
opik connect --pair ABC123 python3 echo_app.py

# TypeScript
opik connect --pair ABC123 npx tsx summarise.ts
```

## Step 5: Verify Connection

After connecting:
- Confirm the runner is connected and listening
- Tell the user they can now go to the Opik UI and trigger their agent from the browser
- The agent will execute locally on their machine, and traces will appear in Opik
- Any job created by the user in the UI or by the Optimizer will trigger an agent run

## Error Handling

| Error | Solution |
|-------|----------|
| "No entrypoint found" | Run `/opik:instrument` first |
| "Connection refused" | Check if Opik server is running (OSS) or API key is valid (Cloud) |
| "Invalid pair code" | Code expires — get a new one from the UI |
| "Port already in use" | Another runner may be active — check with `lsof -i :<port>` |
| "Authentication failed" | Run `opik configure` to set up credentials |
| TS params not in UI | Add explicit `params` array to the `track()` options |

## Notes

- The runner stays active as long as the terminal is open
- Multiple agents can be connected simultaneously
- Traces from UI-triggered runs appear in the same project as local runs
- Config changes made in the UI take effect on the next run (via Blueprints)
- The Optimizer creates jobs that are automatically dispatched to the runner

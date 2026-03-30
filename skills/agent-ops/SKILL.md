---
name: agent-ops
description: Agent lifecycle — architecture, configuration (Blueprints), Local Runner, evaluation, threads, production monitoring. Use for "evaluate my agent", "connect my agent", "configure my agent", "add guardrails".
---

# Agent Operations

## Agent Lifecycle

1. **Instrument** — `@opik.track` + `entrypoint=True` + `opik.AgentConfig`
2. **Configure** — Externalize config into `AgentConfig`. Opik manages Blueprints with env tags (`dev`/`prod`)
3. **Connect** — `opik connect --pair <CODE> <startup_cmd>` to pair with Opik UI
4. **Evaluate** — Evaluation Suites with assertions and execution policies
5. **Monitor** — Dashboards, online evaluation, alerts, guardrails
6. **Optimize** — MaskIDs for config A/B testing, promote winning Blueprints

## Config + Retrieval

```python
class AgentConfig(opik.AgentConfig):
    model: Annotated[str, "LLM model"]
    temperature: Annotated[float, "Sampling temperature"]

client = opik.Opik()
client.create_agent_config_version(
    AgentConfig(model="gpt-4o", temperature=0.7), project_name="my-agent",
)

@opik.track(entrypoint=True, project_name="my-agent")
def run_agent(question: str) -> str:
    cfg = client.get_agent_config(
        fallback=AgentConfig(model="gpt-4o", temperature=0.7),
        project_name="my-agent",
        latest=True,  # or env="prod" or version="v1"
    )
    return llm_call(model=cfg.model, temperature=cfg.temperature)
```

Promote: `cfg.deploy_to("prod")`. Retrieve prod: `env="prod"`.

## Local Runner

```bash
opik connect --pair ABC123 python3 app.py       # Python
opik connect --pair ABC123 npx tsx app.ts        # TypeScript
```

Python: `@track(entrypoint=True)` + docstring with `Args:`.
TypeScript: `track({ entrypoint: true, params: [{name, type}] }, fn)`.

## Evaluation Suites

```python
suite = client.get_or_create_evaluation_suite(
    name="my-suite",
    assertions=["Response is accurate", "Response is professional"],
    execution_policy={"runs_per_item": 3, "pass_threshold": 2},
)
results = suite.run(task=lambda item: {"output": agent(item["input"])}, model="gpt-4o")
assert results.all_passed
```

## Threads (Conversations)

```python
@opik.track(entrypoint=True)
def handle_message(session_id: str, message: str) -> str:
    opik.update_current_trace(thread_id=session_id)
    return generate_response(session_id, message)
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Hardcoded config | Use `AgentConfig` + `get_agent_config()` |
| Missing entrypoint | Add `entrypoint=True` for Local Runner |
| No thread_id on conversational agent | Wire `thread_id` from session ID |
| `get_agent_config()` outside `@track` | Must be inside decorated function |
| TS missing `params` | Add explicit `params` array |

## References

- `references/agent-patterns.md` — architecture, reliability, security
- `references/evaluation.md` — suites, 41 built-in metrics
- `references/production.md` — dashboards, alerts, guardrails

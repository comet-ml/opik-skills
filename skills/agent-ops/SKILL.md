---
name: agent-ops
description: This skill should be used when the user asks about agent architecture, evaluation, metrics, production monitoring, debugging agents, best practices for building reliable AI agents, agent configuration, Blueprints, Evaluation Suites, opik connect, Local Runner, thread tracking, or conversation metrics. Use for questions like "evaluate my agent", "set up production monitoring", "add guardrails", "detect hallucinations", "agent anti-patterns", "compare experiments", "create evaluation suite", "configure my agent", "connect my agent", "track conversations", "evaluate threads".
---

# Agent Operations: Build, Evaluate, and Monitor AI Agents

This skill covers the agent lifecycle beyond basic tracing: architecture patterns, configuration, evaluation, threads, and production monitoring. All examples use Opik for observability — for SDK details (tracing, integrations, span types), load the `opik` skill.

## The Agent Lifecycle

1. **Instrument** — Add `@opik.track` + `opik.AgentConfig` + `entrypoint=True` (see `opik` skill)
2. **Configure** — Externalize config into a dataclass. Opik manages Blueprints (immutable config snapshots) with environment tags (dev/prod)
3. **Connect** — Use `opik connect --pair <CODE> <startup_command>` to pair the Local Runner so the agent can be triggered from the Opik UI
4. **Evaluate** — Create Evaluation Suites with assertions and execution policies
5. **Monitor** — Track quality, cost, and reliability in production dashboards
6. **Optimize** — Use MaskIDs to test config variations, evaluate with suites, promote winning Blueprints

## Agent Configuration (Blueprints)

Opik's **Agent Configuration** externalizes your agent's tunable parameters into managed, version-controlled configs.

### Key Concepts

- **`opik.AgentConfig`** — Base class for config definitions. Subclass it with typed fields. No defaults.
- **Blueprint** — An immutable snapshot of a config version. Every config edit creates a new Blueprint.
- **Environment Tags** — Labels like `dev`, `staging`, `prod` that point to specific Blueprints.
- **MaskID** — A temporary override layer for A/B testing config variations (used by the Optimizer).
- **`entrypoint=True`** — Marks the main function so Opik can trigger the agent via the Local Runner.

### Config + Retrieval Pattern

```python
from typing import Annotated
import opik

class AgentConfig(opik.AgentConfig):
    model: Annotated[str, "LLM model to use"]
    temperature: Annotated[float, "Sampling temperature"]
    system_prompt: Annotated[str, "System prompt for the agent"]
    max_tokens: Annotated[int, "Maximum tokens in response"]

client = opik.Opik()

# Publish config
config = AgentConfig(model="gpt-4o", temperature=0.7,
                     system_prompt="You are a helpful assistant.", max_tokens=1024)
client.create_agent_config_version(config, project_name="my-agent")

# Retrieve at runtime — MUST be inside @opik.track
@opik.track(entrypoint=True, project_name="my-agent")
def run_agent(question: str) -> str:
    """Run the agent with a user question.

    Args:
        question: The user's question to answer.
    """
    cfg = client.get_agent_config(
        fallback=AgentConfig(model="gpt-4o", temperature=0.7,
                             system_prompt="You are a helpful assistant.", max_tokens=1024),
        project_name="my-agent",
        latest=True,       # OR env="prod" OR version="v1_abc"
    )
    response = openai_client.chat.completions.create(
        model=cfg.model, temperature=cfg.temperature, max_tokens=cfg.max_tokens,
        messages=[{"role": "system", "content": cfg.system_prompt},
                  {"role": "user", "content": question}],
    )
    return response.choices[0].message.content
```

### Environment Tags Workflow

1. Developer edits config -> new Blueprint created via `create_agent_config_version()`
2. Test with Evaluation Suite -> passes
3. Promote: `cfg.deploy_to("prod")` -> PROD tag moves to the new Blueprint
4. Production agent reads PROD Blueprint on next invocation via `env="prod"`

## Opik Connect (Local Runner)

The **Local Runner** lets you trigger your agent from the Opik browser UI while it runs on your local machine. Supports both Python and TypeScript.

### Setup Flow

1. Instrument with `entrypoint=True` (required)
2. Python: add a docstring with `Args:` to the entrypoint. TypeScript: add explicit `params` array.
3. Run `opik connect --pair <CODE> <startup_command>`
4. Agent appears in Opik UI -> type input -> click Run -> executes locally

### Examples

```bash
# Python
opik connect --pair ABC123 python3 echo_app.py

# TypeScript
opik connect --pair ABC123 npx tsx summarise.ts
```

### What the Runner Enables

- **UI-triggered execution** — Test your agent from the browser
- **Trace replay** — Click "Re-run" on any trace to replay with same input
- **Config iteration** — Edit config in UI, re-run, compare traces
- **Parallel jobs** — Runner handles concurrent executions
- **Optimizer integration** — Optimizer creates jobs dispatched to the runner

## Thread Tracking (Multi-Turn Conversations)

For conversational agents, group related traces into **threads** using `thread_id`.

### Instrumenting Conversational Agents

```python
import opik

@opik.track(entrypoint=True, project_name="chat-agent")
def handle_message(session_id: str, message: str) -> str:
    """Handle a chat message in a conversation session.

    Args:
        session_id: The conversation session identifier.
        message: The user's message.
    """
    opik.update_current_trace(thread_id=session_id)
    response = generate_response(session_id, message)
    return response
```

### Conversation Thread Metrics

```python
from opik.evaluation import evaluate_threads
from opik.evaluation.metrics.conversation import (
    SessionCompletenessMetric,
    UserFrustrationMetric,
    ConversationalCoherenceMetric,
)

results = evaluate_threads(
    project_name="chat-agent",
    metrics=[
        SessionCompletenessMetric(),
        UserFrustrationMetric(),
        ConversationalCoherenceMetric(),
    ],
)
```

## Evaluation Suites

**Evaluation Suites** provide structured testing with assertions and execution policies.

### Creating a Suite

```python
from opik import Opik

client = Opik()
suite = client.get_or_create_evaluation_suite(
    name="customer-support-suite",
    assertions=[
        "Response is factually accurate and not hallucinated",
        "Response is professional in tone",
    ],
    execution_policy={"runs_per_item": 3, "pass_threshold": 2},
)

suite.add_item(
    data={"input": "How do I reset my password?"},
    assertions=["Response mentions the password reset process"],
)

results = suite.run(
    task=lambda item: {"output": agent(item["input"])},
    model="gpt-4o",
)

assert results.all_passed  # CI gate
```

## Architecture Patterns

Trace every component with appropriate span types:

```python
import opik

@opik.track(entrypoint=True, name="research_agent")
def agent(query: str) -> str:
    plan = plan_action(query)
    results = execute_tool(plan)
    return generate_response(results)

@opik.track(type="tool")
def execute_tool(action: dict) -> str:
    return search_web(action["query"])

@opik.track(type="llm")
def generate_response(context: str) -> str:
    return llm_call(context)
```

### Built-in Agent Metrics

| Metric | What It Measures |
|--------|-----------------|
| `AgentTaskCompletion` | Did the agent fulfill its task? |
| `AgentToolCorrectness` | Were tools used correctly? |
| `TrajectoryAccuracy` | Did actions match expected sequence? |
| `AnswerRelevance` | Does the answer address the question? |
| `Hallucination` | Are there unsupported claims? |

## Common Anti-Patterns

| Category | Anti-Pattern |
|----------|-------------|
| Configuration | Hardcoded model/temperature/prompt instead of `opik.AgentConfig` + `get_agent_config()` |
| Entrypoint | Missing `entrypoint=True` — agent can't be triggered via Local Runner |
| Threads | Conversational agent without `thread_id` — turns appear as unrelated traces |
| Evaluation | Using old Datasets API instead of Evaluation Suites |
| Config retrieval | Calling `get_agent_config()` outside `@opik.track` — won't inject Blueprint metadata |
| TS params | Missing explicit `params` in TypeScript `track()` — UI can't show typed input fields |

## Detailed References

| Topic | Reference File |
|-------|----------------|
| Agent architecture, reliability, security patterns | `references/agent-patterns.md` |
| Evaluation Suites, datasets, experiments, all 41 metrics | `references/evaluation.md` |
| Production dashboards, alerts, guardrails, cost tracking, Blueprints | `references/production.md` |

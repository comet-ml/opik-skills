---
name: threads-conversations
description: Thread tracking for multi-turn conversations — thread_id, conversation metrics, evaluation.
---

# Threads & Conversations

Group conversation turns into threads via `thread_id`. Each turn = one trace; shared `thread_id` = one thread.

## Wiring thread_id

```python
@opik.track(entrypoint=True)
def handle_message(session_id: str, message: str) -> str:
    opik.update_current_trace(thread_id=session_id)
    return generate_response(session_id, message)
```

No natural session ID? Use `str(uuid.uuid4())` generated once per session.

## Thread Metrics

```python
from opik.evaluation import evaluate_threads
from opik.evaluation.metrics.conversation import (
    SessionCompletenessMetric, UserFrustrationMetric, ConversationalCoherenceMetric,
)

results = evaluate_threads(project_name="chat-agent", metrics=[
    SessionCompletenessMetric(), UserFrustrationMetric(), ConversationalCoherenceMetric(),
])
```

## When to Use

Use for chat agents, support bots, multi-step assistants. Skip for single-shot agents or batch processing.

## Pitfalls

- Missing `thread_id`: turns appear as unrelated traces
- `thread_id` per turn instead of per session: each turn becomes its own thread
- Shared `thread_id` across users: conversations get mixed

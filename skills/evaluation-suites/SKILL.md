---
name: evaluation-suites
description: Opik Evaluation Suites — assertions, execution policies, CI integration. Replaces old Datasets API.
---

# Evaluation Suites

Structured testing with assertions (plain strings for LLM judge) and execution policies.

```python
from opik import Opik

client = Opik()
suite = client.get_or_create_evaluation_suite(
    name="my-suite",
    assertions=["Response is factually accurate", "Response is professional"],
    execution_policy={"runs_per_item": 3, "pass_threshold": 2},
)

suite.add_item(data={"input": "What is ML?"})
suite.add_item(
    data={"input": "Should I take this medication?"},
    assertions=["Response advises consulting a doctor"],  # item-level, added to suite-level
)

results = suite.run(
    task=lambda item: {"output": agent(item["input"])},
    model="gpt-4o",  # LLM judge
)
assert results.all_passed  # CI gate
```

**Suite-level** assertions apply to all items. **Item-level** assertions are additive.
Use `get_or_create_evaluation_suite()` — NOT `get_or_create_dataset()`.
Suites appear under **"Evaluation Suites"** in the UI sidebar.

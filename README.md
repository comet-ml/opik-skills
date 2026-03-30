# Opik Claude Code Plugin

Log Claude Code sessions to [Opik](https://github.com/comet-ml/opik) for LLM observability, plus skills and agents for building observable AI applications.

## Features

- **Session Tracing**: Automatically log Claude Code sessions as Opik traces
- **Span Tracking**: Each tool call becomes a span within the trace
- **Subagent Support**: Nested agent calls are tracked with parent-child relationships
- **Skills**: Built-in knowledge for LLM observability, tracing, evaluation, agent configuration, and Local Runner

## Skills

| Skill | Description |
|-------|-------------|
| `opik` | Tracing (Python + TS), Agent Config (Blueprints), Local Runner, Evaluation Suites, threads, integrations |

## Installation

From within Claude Code:

```
/plugin marketplace add comet-ml/opik-skills
/plugin install opik
```

## Configuration

### Cloud
```bash
pip install opik
opik configure  # Prompts for API key — get one at https://www.comet.com/signup
```

### Self-Hosted (OSS)
```bash
pip install opik
opik configure  # Point to your Opik server URL (default: http://localhost:5173)
```

## Directory Structure

```
opik-skills/
├── skills/
│   └── opik/                      # Single unified skill
│       ├── SKILL.md               # Main guide (~200 lines)
│       └── references/
│           ├── tracing-python.md
│           ├── tracing-typescript.md
│           ├── tracing-rest-api.md
│           ├── integrations.md
│           ├── observability.md
│           ├── agent-patterns.md
│           ├── evaluation.md
│           └── production.md
├── README.md
└── LICENSE
```

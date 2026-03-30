# Opik Claude Code Plugin

Log Claude Code sessions to [Opik](https://github.com/comet-ml/opik) for LLM observability, plus skills and agents for building observable AI applications.

## Features

- **Session Tracing**: Automatically log Claude Code sessions as Opik traces
- **Span Tracking**: Each tool call becomes a span within the trace
- **Subagent Support**: Nested agent calls are tracked with parent-child relationships
- **Skills**: Built-in knowledge for LLM observability, tracing, evaluation, agent configuration, and Local Runner
- **Agents**: Code review agent for agent architecture best practices

## Skills

| Skill | Description |
|-------|-------------|
| `opik` | Opik SDK reference: tracing, integrations, span types, code snippets |
| `agent-ops` | Agent lifecycle: architecture, evaluation, monitoring, configuration |
| `agent-config` | Agent Configuration (Blueprints): versioned config, environments, masks |
| `opik-connect` | Local Runner: pair your agent with the Opik browser UI (Python + TypeScript) |
| `instrument-python` | Step-by-step Python instrumentation guide |
| `instrument-typescript` | Step-by-step TypeScript instrumentation guide |
| `evaluation-suites` | Evaluation Suites: assertions, execution policies, CI integration |
| `threads-conversations` | Thread tracking for multi-turn conversational agents |

## Commands

| Command | Description |
|---------|-------------|
| `/opik:instrument` | Auto-detect frameworks and add Opik observability to your code |
| `/opik:connect` | Connect your agent to Opik for UI triggering via Local Runner |
| `/opik:trace-claude-code` | Enable/disable Claude Code session tracing |

## Installation

From within Claude Code:

```
/plugin marketplace add comet-ml/opik-skills
/plugin install opik
```

## Configuration

```bash
pip install opik
opik configure
```

## Directory Structure

```
opik-skills/
├── .claude-plugin/
│   ├── plugin.json         # Plugin manifest
│   └── marketplace.json    # Marketplace definition
├── skills/
│   ├── opik/               # Main SDK reference + references/
│   ├── agent-ops/          # Agent lifecycle + references/
│   ├── agent-config/       # Blueprints guide
│   ├── opik-connect/       # Local Runner guide
│   ├── instrument-python/  # Python instrumentation
│   ├── instrument-typescript/ # TypeScript instrumentation
│   ├── evaluation-suites/  # Evaluation Suites
│   └── threads-conversations/ # Thread tracking
├── commands/
│   ├── instrument.md       # /opik:instrument command
│   ├── connect.md          # /opik:connect command
│   └── trace-claude-code.md # /opik:trace-claude-code command
├── agents/
│   └── agent-reviewer.md   # Agent code review agent
└── hooks/
    └── hooks.json          # Hook configuration
```

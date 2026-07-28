<h1 align="center" style="border-bottom: none">
  <div>
    <a href="https://www.comet.com/site/products/opik/?from=llm&utm_source=opik&utm_medium=github&utm_content=header_img&utm_campaign=opik">
      <picture>
        <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/comet-ml/opik/refs/heads/main/apps/opik-documentation/documentation/static/img/logo-dark-mode.svg">
        <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/comet-ml/opik/refs/heads/main/apps/opik-documentation/documentation/static/img/opik-logo.svg">
        <img alt="Comet Opik logo" src="https://raw.githubusercontent.com/comet-ml/opik/refs/heads/main/apps/opik-documentation/documentation/static/img/opik-logo.svg" width="200" />
      </picture>
    </a>
    <br />
    Opik Skills
  </div>
</h1>

<p align="center">
  Official Opik skills for coding agents — practical guidance for tracing, evaluating,<br/>
  configuring, and debugging LLM applications with
  <a href="https://github.com/comet-ml/opik">Opik</a>.
</p>

<div align="center">

[![License](https://img.shields.io/github/license/comet-ml/opik-skills)](./LICENSE)

</div>

> [!IMPORTANT]
> **The skills now live in [`comet-ml/opik-mcp`](https://github.com/comet-ml/opik-mcp/tree/main/src/opik_mcp/skills)** — a single source of truth, shared by the Opik MCP, the Claude Code plugin, and Ollie. **This repo is the front door**: an index of what exists and how to install it. There is nothing to keep in sync here.

[Opik](https://github.com/comet-ml/opik) is the open-source LLM observability and evaluation platform, built by [Comet](https://www.comet.com). These agent skills teach coding agents how to work with Opik in real codebases — and because a model doesn't know Opik cold the way it knows `git` or `aws`, the skill is what makes the CLI, SDK, or MCP usable for Opik at all.

## The skills

| Skill | What it covers |
| --- | --- |
| [`opik`](https://github.com/comet-ml/opik-mcp/tree/main/src/opik_mcp/skills/opik) | SDK reference — tracing, span types, framework integrations, threads, and the prompt library (Python, TypeScript, REST). |
| [`agent-ops`](https://github.com/comet-ml/opik-mcp/tree/main/src/opik_mcp/skills/agent-ops) | Agent architecture, evaluation, production monitoring, debugging, and reliability best practices. |
| [`opik-sdk`](https://github.com/comet-ml/opik-mcp/tree/main/src/opik_mcp/skills/opik-sdk) | Complete imperative Python SDK API reference — datasets, experiments, traces, spans, prompts, feedback scores, evaluations. |
| [`evaluation`](https://github.com/comet-ml/opik-mcp/tree/main/src/opik_mcp/skills/evaluation) | LLM evaluation workflows — writing judge prompts, evaluating RAG, synthetic data, error analysis, validating evaluators. |
| `instrument` | Task-shaped end-to-end routine — detect frameworks, add config, emit and **verify** a real trace. *(Being finished under [OPIK-7473](https://github.com/comet-ml/opik).)* |

## Install (without the MCP)

The skills are plain Markdown — drop them into your agent's skills directory.

**Shallow clone + copy** (works with any agent that reads skills from disk):

```bash
git clone --depth 1 https://github.com/comet-ml/opik-mcp.git
cp -r opik-mcp/src/opik_mcp/skills/* ~/.claude/skills/   # or your agent's skills directory
```

**Convenience export** (if you already use `uv` / have the package):

```bash
uvx opik-mcp skills install --dir ~/.claude/skills
```

## Using the skills through the Opik MCP

If your agent is connected to the [Opik MCP](https://github.com/comet-ml/opik-mcp), it fetches skills on demand — nothing to install:

```text
read_skill("opik")                 # overview + reference list
read_skill("opik/tracing-python")  # one reference file
```

## One-time Opik setup

Before the skills wire up instrumentation, authenticate Opik once in the environment where your agent works:

- Python: `opik configure`
- TypeScript: `npx opik-ts configure`

These save your Opik configuration locally (API key + connection details). For setup details, see the [Opik documentation](https://www.comet.com/docs/opik/).

## Example prompts

- "Add Opik tracing to this FastAPI app."
- "Instrument this TypeScript OpenAI app with Opik."
- "Create a Test Suite for this chatbot."
- "Add `thread_id` support so Opik groups each conversation correctly."

## Opik for Claude Code

This repo is part of a set of tools for observing Claude Code and other coding agents with [Opik](https://github.com/comet-ml/opik):

- [opik-mcp](https://github.com/comet-ml/opik-mcp): the Opik MCP server **and the home of the skills above**
- [opik-claude-code-plugin](https://github.com/comet-ml/opik-claude-code-plugin): log Claude Code sessions as Opik traces, with skills and agents included
- [ccsync](https://github.com/comet-ml/ccsync): export Claude Code conversation history to Opik
- [opik-skills](https://github.com/comet-ml/opik-skills): this front-door index **(this repo)**

## Learn more

- [Opik repository](https://github.com/comet-ml/opik)
- [Opik documentation](https://www.comet.com/docs/opik/)

## License

[Apache-2.0](./LICENSE)

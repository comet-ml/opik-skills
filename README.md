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
  Official Opik skill pack for coding agents. Install it with one command to give your agent<br/>
  practical guidance for tracing, evaluating, configuring, and debugging LLM applications with
  <a href="https://github.com/comet-ml/opik">Opik</a>.
</p>

<div align="center">

[![License](https://img.shields.io/github/license/comet-ml/opik-skills)](./LICENSE)

</div>

> **This repository is generated.** The skills are authored in
> [comet-ml/opik-mcp](https://github.com/comet-ml/opik-mcp) under `src/opik_mcp/skills/`
> and published here automatically. Edits made here are overwritten by the next sync —
> please open your pull request against `opik-mcp` instead.

[Opik](https://github.com/comet-ml/opik) is the open-source LLM observability and evaluation platform, built by [Comet](https://www.comet.com). These agent skills teach coding agents to instrument applications with Opik.

## Install

```bash
npx skills add comet-ml/opik-skills -g --all
```

This works across ~40 coding agents, including Claude Code, Cursor, Codex, and GitHub Copilot.

## Skills in this pack

| Skill | What it does |
| --- | --- |
| [`opik`](./skills/opik/SKILL.md) | Reference for the Opik SDK — tracing, span types, framework integrations, threads, and the prompt library (Python, TypeScript, REST). Use for "what span types exist", "how do I flush", "track_openai", "add OpikTracer", "version a prompt". To instrument a repo end to end, use the `opik-instrument` skill. |
| [`opik-diagnose`](./skills/opik-diagnose/SKILL.md) | Surface the Opik traces worth a developer's attention, ranked by signal — errors, failed tool calls, latency, regressions, and low online-eval scores — plus Diagnostics issues. Reads live/production traces via the SDK (search_traces and agent_insights) and works with no MCP; uses the MCP issue entity when connected. Returns a ranked shortlist, each item ready to hand to the explain skill. Use for "what is broken in production", "which traces need attention", "find failing or slow traces", "which tool calls are failing", "triage my agent". Not for offline experiment results (use evaluate or compare) and not for root-causing one trace (use explain). |
| [`opik-evaluate`](./skills/opik-evaluate/SKILL.md) | Build an LLM evaluation and run it against your app, returning an experiment with scores. Covers datasets, LLM judges, RAG evaluation, synthetic data, error analysis, and validating evaluators against human labels. Use when the user wants to measure or improve AI product quality, or asks about evals, judges, or evaluation metrics. |
| [`opik-explain`](./skills/opik-explain/SKILL.md) | Root-cause a specific Opik trace, or a pattern across traces, and return a grounded explanation. Uses the hosted Opik MCP when it is connected, and falls back to SDK scripting otherwise. Returns the root cause, the evidence spans as clickable Opik UI links, and one suggested next step. Use for "why did this trace fail", "explain this trace", "debug this trace", "why is my agent slow or wrong". Not for adding tracing to an app (use the instrument skill) or for changing code. |
| [`opik-instrument`](./skills/opik-instrument/SKILL.md) | Add Opik tracing to an existing app and verify a real trace lands. Installs the Opik package, detects the language and LLM framework, adds the minimum tracing, runs a safe representative path, confirms a trace in Opik, and returns the trace link. Use for "instrument my code", "add opik tracing", "add observability", "trace my agent". Not for building a new app from scratch, or a review-only pass with no code changes. |

## One-time Opik setup

Before using them, authenticate Opik once in the environment where your agent will work:

- Python: run `opik configure`
- TypeScript: run `npx opik-ts configure`

Those commands save your Opik configuration locally, including the API key and connection details the agent will use while wiring up instrumentation.

For setup details, see the [Opik documentation](https://www.comet.com/docs/opik/).

## Example prompts

Once installed, you can ask your agent things like:

- "Add Opik tracing to this FastAPI app."
- "Instrument this TypeScript OpenAI app with Opik."
- "Help me connect this local agent to Opik with `opik connect`."
- "Create a Test Suite for this chatbot."
- "Add `thread_id` support so Opik groups each conversation correctly."

## Repository structure

```text
opik-skills/
├── skills/
│   ├── opik/
│   ├── opik-diagnose/
│   ├── opik-evaluate/
│   ├── opik-explain/
│   └── opik-instrument/
├── README.md
├── index.json
└── LICENSE
```

## Opik for Claude Code

This repo is part of a set of tools for observing Claude Code and other coding agents with [Opik](https://github.com/comet-ml/opik):

- [opik-claude-code-plugin](https://github.com/comet-ml/opik-claude-code-plugin): log Claude Code sessions as Opik traces, with skills and agents included
- [ccsync](https://github.com/comet-ml/ccsync): export Claude Code conversation history to Opik
- [cost-intelligence-proxy](https://github.com/comet-ml/cost-intelligence-proxy): meter Claude Code token spend and cost per call
- [opik-skills](https://github.com/comet-ml/opik-skills): agent skills for instrumenting your code with Opik **(this repo)**

## Learn more

- [Opik repository](https://github.com/comet-ml/opik)
- [Opik documentation](https://www.comet.com/docs/opik/)

## License

[Apache-2.0](./LICENSE)

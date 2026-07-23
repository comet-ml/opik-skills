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

[Opik](https://github.com/comet-ml/opik) is the open-source LLM observability and evaluation platform, built by [Comet](https://www.comet.com). These agent skills teach coding agents to instrument applications with Opik.

This repo contains a number of skills that teach AI coding agents how to work with Opik in real codebases.

After install, your agent can help with tasks like:

- adding Opik tracing to Python or TypeScript apps
- wiring up framework integrations instead of manual instrumentation
- configuring and using `AgentConfig` blueprints
- connecting a local agent to the Opik UI with `opik connect`
- creating and running Test Suites
- tracking multi-turn conversations with `thread_id`

## Install

```bash
npx skills add comet-ml/opik-skills
```

This installs the `opik` skill into your local skills environment.

## What the `opik` skill covers

| Area | Guidance included |
| --- | --- |
| Tracing | Python decorators, TypeScript client tracing, REST API tracing, span types |
| Integrations | OpenAI, Anthropic, LangChain, CrewAI, DSPy, Google ADK, Vercel AI SDK, and more |
| Prompt Library | `client.create_prompt` / `client.get_prompt` (and chat variants), versioning, `metadata` for model/temperature |
| Local Runner | `opik connect`, pairing flow, entrypoint requirements, troubleshooting |
| Evaluation | Test Suites, `run_tests()`, assertions, execution policies, CI gating |
| Conversations | `thread_id`, conversation metrics, common pitfalls |
| Observability | trace boundaries, metadata, feedback scores, distributed tracing |

## One-time Opik setup

These skills are designed to help your coding agent fully integrate Opik into your project, including tracing, evaluations, the Prompt Library, threads, and `opik connect`.

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
- "Move these hardcoded model settings into `AgentConfig`."
- "Create a Test Suite for this chatbot."
- "Add `thread_id` support so Opik groups each conversation correctly."

## Repository structure

```text
opik-skills/
├── skills/
│   └── opik/
│       ├── SKILL.md
│       └── references/
│           ├── evaluation.md
│           ├── integrations.md
│           ├── observability.md
│           ├── tracing-python.md
│           ├── tracing-rest-api.md
│           └── tracing-typescript.md
├── README.md
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

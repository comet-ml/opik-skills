# Source of truth for Opik coding-agent skills

`comet-ml/opik-skills` is the **canonical source** for the skills and reference
content that coding agents use to work with Opik. Everywhere else this content
ships — the Claude Code plugin, Ollie, the Opik MCP's on-demand skill surface,
and the TypeScript `configure` rules-stubs — is a **consumer**: it vendors the
shared set from here at a pinned ref and is guarded by a drift check. Consumers
must not fork it.

Tracked in [OPIK-7471](https://comet-ml.atlassian.net/browse/OPIK-7471).

## Why here

- **Public and host-neutral.** Any coding agent can install it
  (`npx skills add comet-ml/opik-skills`), and public consumers can depend on it
  — which an internal service (Ollie) cannot be.
- **Plain markdown.** The lowest-common-denominator artifact every consumer can
  vendor; the skills CLI and plugin packaging read real files.
- **Already a distribution channel**, so "source of truth" and "a real channel"
  are the same repo.

The canonical content is **seeded from Ollie's skills**, which carried the
freshest content and the provenance discipline below.

## Shared vs. host-specific

**Shared** (canonical here; vendored by consumers) — see
[`skills/shared-skills.yaml`](./skills/shared-skills.yaml). Migration to this
repo is in progress (per-skill PRs under OPIK-7471).

**Host-specific** (stay in their home repo; never vendored):

- **Ollie** — `navigation`, `local-runners`, the `instrumentation-*` phase skills.
- **Plugin** — `commands/`, the reviewer agent, hooks, the session logger.

## Provenance convention

Every `SKILL.md` and reference `.md` that describes Opik SDK/backend behavior
carries frontmatter:

```yaml
last_updated: "YYYY-MM-DD"
source_commit: "<opik repo commit/tag the content was verified against>"
```

When a change reflects new Opik behavior (not just a wording fix), bump
`source_commit` to the Opik commit you verified against and set `last_updated`
to that day. This keeps future sync checks honest.

## Consumers & sync

| Consumer | Repo | Mechanism |
|---|---|---|
| Claude Code plugin | `comet-ml/opik-claude-code-plugin` | `git subtree` pull of the shared set at a pinned ref |
| Ollie | `comet-ml/ollie-assist` | `make sync-skills` vendor script |
| TS `configure` rules | `comet-ml/opik` (`sdks/typescript/.../configure`) | vendor the shared reference into the rules stubs |
| Opik MCP | `comet-ml/opik-mcp` | bundles the shared set, surfaced via `read_skill` (OPIK-7472) |

Each consumer runs a **drift check** in CI that fails if its vendored copy
diverges from the pinned canonical ref without a version bump.

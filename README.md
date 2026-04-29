# claude-reachy-mini

[![ci](https://github.com/nolte/claude-reachy-mini/actions/workflows/ci.yml/badge.svg)](https://github.com/nolte/claude-reachy-mini/actions/workflows/ci.yml)

Claude Code plugin that bundles skills, agents, and specifications for efficient development with the [Reachy Mini](https://www.pollen-robotics.com/reachy-mini/) robot — Pollen Robotics' / Hugging Face's open-source desktop companion.

## What you get

- **Skills** — focused, on-demand knowledge and workflow primitives Claude Code pulls in when relevant (SDK usage, behavior scaffolding, Home Assistant bridge, …)
- **Agents** — larger, autonomous helpers for tasks that span multiple steps (e.g. deploying and live-testing a behavior on the device)
- **Specifications** — bilingual source-of-truth documents under `spec/` that govern every skill and agent

## First concrete use case

A Reachy Mini that dances to music, controlled bidirectionally through Home Assistant (HA triggers motions, Reachy calls HA services).

## Quickstart

Install the plugin into Claude Code via its marketplace mechanism, or develop locally:

```bash
claude --plugin-dir .
```

Local automation runs through `Taskfile.yml`:

```bash
task lint     # pre-commit checks
task test     # placeholder until runtime tests exist
task docs     # build the MkDocs site
```

## Documentation

Full documentation lives under `docs/` and is published at <https://nolte.github.io/claude-reachy-mini>.

## License

[MIT](LICENSE)

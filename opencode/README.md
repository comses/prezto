# opencode

Local OpenCode workstation configuration and the OMF fork of the
oh-my-opencode-slim plugin (OmO Slim).

## Contents

| Path | Purpose |
|---|---|
| `opencode.jsonc` | Global OpenCode configuration: provider, models, plugin path, permissions |
| `oh-my-opencode-slim.json` | OmO Slim plugin configuration and agent presets (`asu-rto`, `openai`, `omf-asu-rto`, `opencode-go`) |
| `oh-my-opencode-slim/` | OmO Slim plugin source checkout; built with Bun into `dist/` |
| `tui.json` | TUI plugin reference |
| `skills/` | Local skill directories (clonedeps, codemap, deepwork, reflect, simplify, verification-planning, worktrees) |
| `.oh-my-opencode-slim/` | Generated plugin runtime artifacts (`skills-manifest.json`) |
| `package.json` | OpenCode plugin SDK dependency |
| `node_modules/` | Installed dependencies (gitignored) |

## Runbooks

- [`runbooks/omo-slim-upgrade-runbook.md`](runbooks/omo-slim-upgrade-runbook.md) — repeatable procedure for upgrading the local OpenCode CLI and rebuilding the OmO Slim plugin. Covers preconditions, safety rules, the upgrade procedure, verification gates, troubleshooting, and rollback. Written for both human operators and automated agents.

## Active preset

The active preset is `asu-rto` (set in `oh-my-opencode-slim.json`). The `observer` agent is disabled.

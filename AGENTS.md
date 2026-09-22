# Agent Guide

This is a personalized Prezto checkout with local OpenCode tooling under
`opencode/`. Preserve unrelated workstation customizations and keep changes
small.

## Start here

- For OpenCode work, read `opencode/README.md` and the relevant file first.
- For upgrades, follow `opencode/runbooks/omo-slim-upgrade-runbook.md` exactly;
  its safety, verification, and rollback gates are authoritative.
- For plugin source changes, also follow
  `opencode/oh-my-opencode-slim/AGENTS.md`. That directory is a Git submodule;
  do not mix parent-repository and submodule commits.

## OpenCode moving parts

- `opencode/opencode.jsonc`: strict global OpenCode configuration. Preserve its
  schema declaration and existing settings; restart OpenCode after changes.
- `opencode/oh-my-opencode-slim.json`: OmO Slim presets and agent mappings. Do
  not regenerate it during routine maintenance. `asu-rto` is active and
  `observer` is intentionally disabled unless the task says otherwise.
- `opencode/oh-my-opencode-slim/`: Bun/TypeScript plugin source. Build output in
  `dist/` must be regenerated before runtime validation.
- `opencode/skills/*/SKILL.md`: local agent workflows. Keep skill metadata,
  trigger wording, implementation, and nearby documentation consistent.
- `opencode/.oh-my-opencode-slim/`, `node_modules/`, `dist/`, and
  `.slim/clonedeps/repos/` are generated, installed, or read-only artifacts;
  do not hand-edit them.

## Working rules

- Inspect `git status` before editing. Never discard unrelated changes.
- Treat every `*/external` path listed in `.gitmodules` as a submodule boundary.
- Do not use destructive Git operations or expose credentials and private
  configuration in output.
- When asked to commit, use Conventional Commits: `type(scope): summary`.
  Prefer common types such as `feat`, `fix`, `docs`, `refactor`, `test`, and
  `chore`; keep the summary concise and imperative.
- Validate proportionately: `zsh -n` for edited shell files; `opencode debug
  config` for OpenCode loading; the runbook's local `doctor` and agent checks
  for OmO Slim configuration; and, inside the plugin submodule, `bun run
  check:ci`, `bun run typecheck`, and `bun test` for source changes.
- A successful `doctor` does not prove the plugin loaded. Runtime agent
  inspection is a separate required gate.

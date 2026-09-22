# OpenCode and oh-my-opencode-slim upgrade runbook

Use this runbook to upgrade the local OpenCode CLI and the locally built
Open Modeling Foundation fork of oh-my-opencode-slim (OmO Slim). It is written
for both human operators and automated agents.

This procedure changes workstation tooling only. It does not authorize or
perform changes to staging, production, OpenStack, or other remote systems.

## Local installation layout

OpenCode reads its global configuration from:

```text
~/.config/opencode/opencode.jsonc
```

That configuration loads OmO Slim from the local path:

```text
~/.config/opencode/oh-my-opencode-slim
```

On the current workstation, that path resolves to:

```text
~/.zprezto/opencode/oh-my-opencode-slim
```

The plugin is loaded from generated files under `dist/`. Updating its Git
checkout without rebuilding `dist/` leaves OpenCode running old code or unable
to load the plugin.

The user plugin configuration is:

```text
~/.config/opencode/oh-my-opencode-slim.json
```

The expected active preset is `asu-rto`.

## Safety rules

- Close normal OpenCode sessions before upgrading when practical. An automated
  agent may perform the upgrade from its active OpenCode session, but that
  session will keep the old CLI and plugin code in memory and must not be used
  as post-restart verification.
- Do not use `git reset --hard`, `git clean`, force updates, or automatic merge
  conflict resolution.
- Do not overwrite or regenerate `oh-my-opencode-slim.json` during a routine
  upgrade.
- Stop if the OmO Slim checkout has uncommitted changes. Preserve and review
  them rather than discarding them.
- Stop if the Git update is not a fast-forward.
- Do not treat a successful `doctor` result as proof that the plugin loaded.
  Runtime agent inspection is a required, separate verification gate.
- Do not expose provider credentials, tokens, or private configuration in logs
  or reports.

## Preconditions

Run these checks in a normal local shell:

```sh
opencode --version
readlink -f ~/.config/opencode/oh-my-opencode-slim
git -C ~/.config/opencode/oh-my-opencode-slim status --short --branch
git -C ~/.config/opencode/oh-my-opencode-slim remote -v
```

Expected results:

- `opencode --version` prints an installed version.
- The plugin path resolves to the intended OMF checkout.
- The plugin worktree has no uncommitted files.
- `origin` points to the approved OMF repository.

Record the current versions and commit for rollback:

```sh
opencode --version
git -C ~/.config/opencode/oh-my-opencode-slim rev-parse HEAD
node -p "require(process.env.HOME + '/.config/opencode/oh-my-opencode-slim/package.json').version"
```

Select a Bun executable. Prefer the checkout-local executable when present:

```sh
test -x ~/.config/opencode/oh-my-opencode-slim/node_modules/.bin/bun
~/.config/opencode/oh-my-opencode-slim/node_modules/.bin/bun --version
```

If that executable is absent, use an approved global Bun installation. Prefer
the `packageManager` version in the plugin's `package.json`; a newer Bun is
acceptable when `bun install --frozen-lockfile` and the complete build pass and
the checkout remains clean. Stop rather than substituting another package
manager or accepting lockfile changes.

## Upgrade procedure

### 1. Update the OmO Slim source

Fetch the approved remote branch and inspect the incoming commits:

```sh
git -C ~/.config/opencode/oh-my-opencode-slim fetch origin master
git -C ~/.config/opencode/oh-my-opencode-slim log --oneline HEAD..origin/master
```

Fast-forward to the fetched commit:

```sh
git -C ~/.config/opencode/oh-my-opencode-slim merge --ff-only origin/master
```

This command works whether the local checkout is attached to `master` or uses
a detached HEAD. If it reports that a fast-forward is impossible, stop and
investigate the checkout state.

### 2. Install dependencies and rebuild OmO Slim

Using the checkout-local Bun executable:

```sh
~/.config/opencode/oh-my-opencode-slim/node_modules/.bin/bun \
  --cwd ~/.config/opencode/oh-my-opencode-slim install --frozen-lockfile
~/.config/opencode/oh-my-opencode-slim/node_modules/.bin/bun \
  --cwd ~/.config/opencode/oh-my-opencode-slim run build
```

If using an approved global `bun`, run the equivalent commands from the plugin
checkout:

```sh
cd ~/.config/opencode/oh-my-opencode-slim
bun install --frozen-lockfile
bun run build
```

The build must produce at least:

```text
dist/index.js
dist/server/index.js
dist/tui2.js
dist/cli/index.js
```

Verify them:

```sh
test -f ~/.config/opencode/oh-my-opencode-slim/dist/index.js
test -f ~/.config/opencode/oh-my-opencode-slim/dist/server/index.js
test -f ~/.config/opencode/oh-my-opencode-slim/dist/tui2.js
test -f ~/.config/opencode/oh-my-opencode-slim/dist/cli/index.js
```

Any nonzero result is a failed build. Do not continue to OpenCode verification.

### 3. Upgrade OpenCode

OpenCode's `autoupdate` setting is notification-only, so perform the upgrade
explicitly:

```sh
opencode upgrade
opencode --version
```

The CLI also accepts `opencode upgrade <target>` when an exact version is
required.

### 4. Validate OmO Slim configuration

Run the CLI built from the local checkout, not an independently downloaded npm
version. For a global configuration check, run it from the home directory so
the result does not depend on an unrelated project's local OpenCode config:

```sh
cd ~
node ~/.config/opencode/oh-my-opencode-slim/dist/cli/index.js doctor
```

Expected result:

```text
[user] <home>/.config/opencode/oh-my-opencode-slim.json <success>
[project] No config file found
[preset] asu-rto <success>
```

The CLI currently renders `<success>` as a check mark.

If a specific project's overrides also need validation, repeat `doctor` from
that project as a separate check. The global upgrade itself does not depend on
any application repository.

This checks configuration syntax and preset existence only. Continue to the
runtime checks even when it passes.

### 5. Verify plugin loading and effective agents

First verify that the plugin registered its primary agent:

```sh
opencode debug agent orchestrator
```

If OpenCode reports `Agent orchestrator not found`, the plugin did not load.
Stop and use the troubleshooting section below.

Inspect every enabled specialist:

```sh
opencode debug agent orchestrator
opencode debug agent oracle
opencode debug agent librarian
opencode debug agent explorer
opencode debug agent designer
opencode debug agent fixer
```

Each command must return an agent object with the expected effective model:

| Agent | Expected model |
|---|---|
| `orchestrator` | `asu-rto/glm-5-2` |
| `oracle` | `openai/gpt-5.6-sol` |
| `librarian` | `asu-rto/qwen3-coder-30b-a3b-instruct` |
| `explorer` | `asu-rto/north-mini-code` |
| `designer` | `asu-rto/kimi-k2-7-code` |
| `fixer` | `asu-rto/qwen38-27b` |

`oracle` intentionally uses OpenAI in the `asu-rto` preset. It is not evidence
that the OpenAI preset was selected.

Verify that Observer remains disabled:

```sh
opencode debug agent observer
```

Expected result: `Agent observer not found`.

### 6. Confirm checkout hygiene

```sh
git -C ~/.config/opencode/oh-my-opencode-slim status --short
```

Expected result: no output. `node_modules/` and `dist/` are ignored build
artifacts and should not appear as tracked changes.

Start a new OpenCode process only after all verification gates pass.

## Troubleshooting

### Doctor passes but agents are missing

This means configuration is valid but does not prove that OpenCode loaded the
plugin. Check the resolved plugin path:

```sh
opencode debug config
```

The `plugin` list should include a file URL for the configured local checkout,
typically:

```text
file:///home/<user>/.config/opencode/oh-my-opencode-slim
```

The displayed URL may instead contain the resolved symlink target. Compare it
with the path reported by the preflight `readlink -f` command.

Then verify the entry point and rebuild:

```sh
test -f ~/.config/opencode/oh-my-opencode-slim/dist/index.js
bun --cwd ~/.config/opencode/oh-my-opencode-slim run build
```

If preflight selected a checkout-local Bun, substitute that executable for
`bun` here and in the rollback commands.

Repeat `opencode debug agent orchestrator` after the build.

### Agents use the wrong preset

Check for an environment override:

```sh
printenv OH_MY_OPENCODE_SLIM_PRESET
```

No output is expected for the normal configuration. Then inspect:

```text
~/.config/opencode/oh-my-opencode-slim.json
```

Confirm that its top-level `preset` is `asu-rto`. Do not edit or regenerate the
file merely because `oracle` uses OpenAI; that assignment is intentional.

After changing a preset intentionally, close and restart OpenCode before
checking effective agents.

### Bun is unavailable

Read the required version without modifying the checkout:

```sh
node -p "require(process.env.HOME + '/.config/opencode/oh-my-opencode-slim/package.json').packageManager"
```

Install Bun through the workstation's approved tool-management method, then
repeat dependency installation and the build. Prefer the declared version, but
a newer Bun is acceptable under the verification rules above. Do not use npm
inside the checkout as a replacement for the locked Bun workflow.

### Build or runtime verification fails after upgrading

1. Stop and retain the command output without credentials.
2. Record the new OpenCode version and OmO Slim commit.
3. Confirm the plugin checkout remains clean.
4. Re-run the build once after `bun install --frozen-lockfile`.
5. If it still fails, use the rollback procedure or investigate compatibility
   before starting normal work.

## Rollback

Use the versions and commit recorded during preflight. Ensure the plugin
checkout is clean, then select the previous commit and rebuild it:

```sh
git -C ~/.config/opencode/oh-my-opencode-slim switch --detach <previous-omo-commit>
bun --cwd ~/.config/opencode/oh-my-opencode-slim install --frozen-lockfile
bun --cwd ~/.config/opencode/oh-my-opencode-slim run build
```

If OpenCode itself must be rolled back:

```sh
opencode upgrade <previous-opencode-version>
```

Repeat the complete configuration and runtime verification sequence after any
rollback.

## Completion criteria

The maintenance is complete only when all of the following are true:

- OpenCode reports the intended upgraded version.
- OmO Slim is at the intended OMF commit.
- The OmO Slim build succeeds and all required `dist/` entry points exist.
- The locally built doctor reports valid configuration and the `asu-rto` preset.
- `opencode debug agent orchestrator` proves that the plugin loaded.
- Every enabled agent resolves to the expected model mapping above.
- Observer remains disabled.
- The OmO Slim Git checkout is clean.
- A fresh OpenCode process is started after the upgrade; the upgrading session
  is not reused for normal work.

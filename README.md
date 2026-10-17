# @l1nxuan-du/dsh-preset-jackpot

English | [中文](README.zh.md)

## Summary

**Jackpot Mode** is a [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) agent preset that tunes DeepSeek Flash for software engineering work. It installs as an ordinary DSH bundle: one `@deepseek-ai/dsh-agent-preset` declaration under the preset id `jackpot`, carrying a development-focused persona and the shipped `standard` tool set.

Author: Linxuan Du. License: MIT.

## Install

From any environment with `dsh` available — a DSH checkout (`pnpm dsh …`) or an installed CLI — add the package to a profile:

```sh
dsh plugin --profile web add @l1nxuan-du/dsh-preset-jackpot
```

The command installs the package and selects it into the profile's `dsh.profile.bundles`, so one command is enough. One package serves the published DSH lines `0.2.0-rc.2`, `0.2.1-alpha.2`, and `0.2.1-rc.2` (`next`), so the requirement is a DSH release at or above `0.2.0-rc.2`.

The target profile must mount `@deepseek-ai/dsh-agent-preset-registry`, which declares presets; the shipped `web` profile does. A profile without it installs the package but has nowhere to register the declaration.

Restart a running Host afterwards: profile bundle patches are read at startup. Verify the install landed by dumping the composed tree, which should list `preset-jackpot`:

```sh
dsh --profile web --dump-config | grep -A 3 preset-jackpot
```

### Install without npm

The package is a plain DSH bundle, so a checkout works the same way:

```sh
dsh plugin --profile web add /path/to/dsh-preset-jackpot
```

Git addresses, tarball URLs, and any absolute path are accepted in the same position.

## Use

Create a new session and select **Jackpot Mode** in the preset picker. Sessions keep the composition they started with, so an existing session does not pick the preset up; start a new one.

`description` reads: 祝你好运！ / Good Lucky!

## What the preset declares

| Field | Value |
|---|---|
| Preset id | `jackpot` |
| Display name | Jackpot Mode |
| Roster order | 11 |
| `persona.prefix` | `new-system-prompt.dsh.md` (84 lines), verbatim: the latest Codex base instructions merged with `system-prompt-A.md` |
| `persona.complete` | `false` |
| `persona.includeRuntimeContext` | `false` |
| Plugin rows | The `standard` composition of both published DSH lines, intersected |

Two consequences follow from the persona configuration:

- The persona registers into the preset's scope as the `deployment:persona-prefix` section. A scoped section shadows the global section of the same name, so this text **replaces** the deployment persona rather than appending to it. Because `complete` is `false`, the harness identity, each tool's own guidance, the harness-source path, and the web-surface section still render.
- `includeRuntimeContext: false` drops the optional runtime-context snapshots — the sandbox policy and approval policy — while the `working-directory:current` context stays, because it is registered as required.

## Model Experience

The preset changes what the model reads at the start of every request, not what it can do:

- **Prompt sections.** One persona section replaces the deployment persona. Token cost is the 84-line prompt, and it is constant per session.
- **Runtime context.** Each turn's snapshot carries the working directory only. Sandbox and approval policy snapshots are suppressed, so the model is not told which permission mode it runs under; a denied tool call is its only signal.
- **Tools.** Unchanged from `standard`: the same schemas in the same order, so the tool catalog is stable across presets that share this composition.
- **KV cache.** The preset mounts once per process and a live agent's composition never changes mid-session, so the section text stays byte-identical across turns and remains eligible for provider-side cache reuse. New sessions pick up a changed declaration; running ones do not.

#### KV Cache effect

A live agent keeps the composition it started with, so the assembled prompt is stable for the session's lifetime. Editing this package affects only subsequently created agents.

## Known Limitations and Deferred Work

- **The tool rows are a copy, not an import.** DSH presets declare their child plugins inline; there is no mechanism to inherit another preset's list. The rows here are the **intersection** of the tool compositions shipped with DSH `0.2.0-rc.2`, `0.2.1-alpha.2`, and `0.2.1-rc.2`, configuration and isolation realms included, so one package mounts on each line. A host release that adds, removes, reconfigures, or relocates a tool leaves this package behind until its rows are updated to match. Re-check `packages/bundle/web-app/presets/standard.patch.yml` in the DSH checkout on each upgrade.
- **Two `standard` rows are deliberately omitted.**
  - `tool-schedule`: no `@deepseek-ai/dsh-tool-schedule` release exists for the `0.2.0-rc.2` line, so the row would fail to import there and reject the whole preset mount. The four reminder tools are therefore absent on every line; on `0.2.1` you can add the row back next to `tool-jobs`.
  - `time-context`: on `0.2.0-rc.2` that row belongs to the opt-in `@deepseek-ai/dsh-experimental-schedule-bundle`, which mounts it on the **host** plane. A preset declaring the same row concurrently would register the `timeContext` projection twice, so the preset leaves it to the host. On `0.2.1-alpha.x`, where presets own the row, the trade-off costs the preset its current-time context.
- **The row set is cross-version, so per-version wording is not.** The `plan-mode` guidance text matches `0.2.1-alpha.2`; `tool-subagent` carries `backgroundMode: continuable` (the `0.2.0-rc.2` value — `0.2.1-alpha.2` no longer reads that key, and its schema preserves unknown keys rather than rejecting them) and no `toolFilter`. `0.2.1-alpha.2` set a `toolFilter` deny list for the schedule tools, but from `0.2.1-alpha.3` on `tool-subagent` throws on that key, which made sessions on this preset fail with `agent-preset/invalid`; the deny list was moot here anyway, because the preset ships no schedule tools.
- **The preset rows must keep up with their plugins.** Two rows carry inline guidance text — `plan-mode`, and, indirectly through tool schemas, the tool rows. A DSH release that changes those contracts can make this declaration stale or invalid; a failed activation stays visible on the roster with its diagnostic rather than failing silently.
- **The bundle patch is read at startup.** Installing into a running Host needs a restart before the preset appears; the plugin manager reports `restart-required` for a bundle whose patch content changed.
- **No automated tests.** The package ships declarations only; verifying it means installing it into a profile and starting a session on the preset.

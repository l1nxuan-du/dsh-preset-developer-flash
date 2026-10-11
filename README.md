# @l1nxuan-du/dsh-preset-developer-flash

English | [中文](README.zh.md)

## Summary

**Developer (Flash) Mode** is a [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) agent preset that tunes DeepSeek Flash for software engineering work. It installs as an ordinary DSH bundle: one `@deepseek-ai/dsh-agent-preset` declaration under the preset id `developer-flash`, carrying a development-focused persona and the shipped `standard` tool set.

Author: Linxuan Du. License: MIT.

## Install

From any environment with `dsh` available — a DSH checkout (`pnpm dsh …`) or an installed CLI — add the package to a profile:

```sh
dsh plugin --profile web add @l1nxuan-du/dsh-preset-developer-flash
```

The command installs the package and selects it into the profile's `dsh.profile.bundles`, so one command is enough. Requires a DSH release at or above `0.2.1-alpha.2`, the version this package was built against.

The target profile must mount `@deepseek-ai/dsh-agent-preset-registry`, which declares presets; the shipped `web` profile does. A profile without it installs the package but has nowhere to register the declaration.

Restart a running Host afterwards: profile bundle patches are read at startup. Verify the install landed by dumping the composed tree, which should list `preset-developer-flash`:

```sh
dsh --profile web --dump-config | grep -A 3 preset-developer-flash
```

### Install without npm

The package is a plain DSH bundle, so a checkout works the same way:

```sh
dsh plugin --profile web add /path/to/dsh-preset-developer-flash
```

Git addresses, tarball URLs, and any absolute path are accepted in the same position.

## Use

Create a new session and select **Developer (Flash) Mode** in the preset picker. Sessions keep the composition they started with, so an existing session does not pick the preset up; start a new one.

`description` reads: 开发者（Flash）模式 — 专注于开发的系统提示词，比起 Developer 模式放弃了挽救 DeepSeek Flash 模型的审美的想法。

## What the preset declares

| Field | Value |
|---|---|
| Preset id | `developer-flash` |
| Display name | Developer (Flash) Mode |
| Roster order | 11 |
| `persona.prefix` | The 76-line collaboration brief, verbatim |
| `persona.complete` | `false` |
| `persona.includeRuntimeContext` | `false` |
| Plugin rows | The shipped `standard` composition, unchanged |

Two consequences follow from the persona configuration:

- The persona registers into the preset's scope as the `deployment:persona-prefix` section. A scoped section shadows the global section of the same name, so this text **replaces** the deployment persona rather than appending to it. Because `complete` is `false`, the harness identity, each tool's own guidance, the harness-source path, and the web-surface section still render.
- `includeRuntimeContext: false` drops the optional runtime-context snapshots — the sandbox policy and approval policy — while the `working-directory:current` context stays, because it is registered as required.

## Model Experience

The preset changes what the model reads at the start of every request, not what it can do:

- **Prompt sections.** One persona section replaces the deployment persona. Token cost is the 76-line brief, and it is constant per session.
- **Runtime context.** Each turn's snapshot carries the working directory only. Sandbox and approval policy snapshots are suppressed, so the model is not told which permission mode it runs under; a denied tool call is its only signal.
- **Tools.** Unchanged from `standard`: the same schemas in the same order, so the tool catalog is stable across presets that share this composition.
- **KV cache.** The preset mounts once per process and a live agent's composition never changes mid-session, so the section text stays byte-identical across turns and remains eligible for provider-side cache reuse. New sessions pick up a changed declaration; running ones do not.

#### KV Cache effect

A live agent keeps the composition it started with, so the assembled prompt is stable for the session's lifetime. Editing this package affects only subsequently created agents.

## Known Limitations and Deferred Work

- **The tool rows are a copy, not an import.** DSH presets declare their child plugins inline; there is no mechanism to inherit another preset's list. The rows here replicate the tool composition shipped with DSH 0.2.1-alpha.2, configuration and isolation realms included. A host release that adds, removes, reconfigures, or relocates a tool leaves this package behind until its rows are updated to match. Re-check `packages/bundle/web-app/presets/standard.patch.yml` in the DSH checkout on each upgrade.
- **The preset rows must keep up with their plugins.** Two rows carry inline guidance text — `plan-mode`, and, indirectly through tool schemas, the tool rows. A DSH release that changes those contracts can make this declaration stale or invalid; a failed activation stays visible on the roster with its diagnostic rather than failing silently.
- **The bundle patch is read at startup.** Installing into a running Host needs a restart before the preset appears; the plugin manager reports `restart-required` for a bundle whose patch content changed.
- **No automated tests.** The package ships declarations only; verifying it means installing it into a profile and starting a session on the preset.

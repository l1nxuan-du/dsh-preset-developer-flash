# @l1nxuan-du/dsh-preset-developer-flash

[English](README.md) | 中文

## 简介

**开发者（Flash）模式**是一个面向 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 的 Agent 预设，用于让 DeepSeek Flash 更好地承担软件工程任务。它以标准 DSH bundle 的形式安装：在预设 id `developer-flash` 下声明一个 `@deepseek-ai/dsh-agent-preset`，携带一份专注开发的 persona，以及 DSH 内置的 `standard` 工具集。

作者：杜凌轩（Linxuan Du）。许可证：MIT。

## 安装

在任何可用 `dsh` 的环境中（DSH 检出目录用 `pnpm dsh …`，或已安装的 CLI），把包装进某个 profile：

```sh
dsh plugin --profile web add @l1nxuan-du/dsh-preset-developer-flash
```

这一条命令即完成安装，并把该 bundle 选入 profile 的 `dsh.profile.bundles`。同一个包同时服务 DSH 的两条公开版本线 —— `0.2.0-rc.2`（`latest`）与 `0.2.1-alpha.2`（`alpha`）—— 因此要求 DSH `0.2.0-rc.2` 或更高版本。

目标 profile 必须挂载 `@deepseek-ai/dsh-agent-preset-registry`（它负责声明预设），随包发布的 `web` profile 已自带；缺少它的精简 profile 能装上包，但没有地方注册声明。

运行中的 Host 需要重启：profile 的 bundle patch 在启动时读取。可以 dump 组合后的配置树来确认安装生效，输出里应出现 `preset-developer-flash`：

```sh
dsh --profile web --dump-config | grep -A 3 preset-developer-flash
```

### 不从 npm 安装

本包就是一个普通 DSH bundle，用本地检出同样可以：

```sh
dsh plugin --profile web add /path/to/dsh-preset-developer-flash
```

Git 地址、tarball URL 与任何绝对路径都可以放在同一位置。

## 使用

新建会话，在预设选择器中选择 **Developer (Flash) Mode**。会话会保留启动时的组合，因此已存在的会话不会自动切换，需要新建。

`description` 为：开发者（Flash）模式 — 专注于开发的系统提示词，比起 Developer 模式放弃了挽救 DeepSeek Flash 模型的审美的想法。

## 预设声明内容

| 字段 | 值 |
|---|---|
| 预设 id | `developer-flash` |
| 显示名 | Developer (Flash) Mode |
| 名册顺序 | 11 |
| `persona.prefix` | 76 行协作提示词，逐字原文 |
| `persona.complete` | `false` |
| `persona.includeRuntimeContext` | `false` |
| 插件行 | 两条公开版本线各自 `standard` 组合的交集 |

persona 的配置带来两个后果：

- persona 以 `deployment:persona-prefix` 段注册进预设作用域。作用域段会按名字遮蔽全局同名段，因此这段文字是**替换**部署层 persona，而不是追加在它后面。由于 `complete` 为 `false`，harness 身份、各工具自带的使用指引、harness 源码路径以及 Web 界面段都仍会渲染。
- `includeRuntimeContext: false` 会丢弃可选的运行时上下文快照 —— 沙箱策略与审批策略 —— 而 `working-directory:current` 因为注册为 required 会保留。

## 模型体验

该预设改变的是模型每次请求开头读到的东西，而不是它能做什么：

- **提示词段。** 一段 persona 替换掉部署层 persona。token 成本即这份 76 行提示词，且在会话内恒定。
- **运行时上下文。** 每轮快照只带工作目录；沙箱与审批策略快照被抑制，因此模型不知道自己处在哪种权限模式下，工具被拒是它唯一的信号。
- **工具。** 与 `standard` 完全一致：同样的 schema、同样的顺序；共享该组合的预设之间工具目录保持稳定。
- **KV Cache。** 预设每个进程只挂载一次，运行中的 Agent 组合在会话内不会变化，因此段文本逐字节稳定，可继续被提供方缓存复用。新建会话才会读到改动后的声明，运行中的不会。

#### KV Cache 影响

运行中的 Agent 保留启动时的组合，因此整个会话生命周期内装配出的提示词是稳定的。修改本包只影响此后新建的 Agent。

## 已知限制与待办

- **工具行是拷贝而非引用。** DSH 预设以内联方式声明子插件，没有继承另一个预设列表的机制。这里的行是 DSH `0.2.0-rc.2` 与 `0.2.1-alpha.2` 随包发布的工具组合的**交集**，含配置与隔离 realm，因此同一个包在两条版本线上都能挂载。宿主版本若新增、删除、改配或迁移工具，本包就会落后，直到同步这些行。每次升级请对照 DSH 检出中的 `packages/bundle/web-app/presets/standard.patch.yml` 复核。
- **有两行 `standard` 行被有意省略。**
  - `tool-schedule`：`0.2.0-rc.2` 版本线根本没有 `@deepseek-ai/dsh-tool-schedule` 发布版，该行在那里会导入失败并导致整个预设挂载被拒。因此四条提醒工具在两条线上都缺席；在 `0.2.1-alpha.x` 上可以把它加回 `tool-jobs` 旁边。
  - `time-context`：在 `0.2.0-rc.2` 上这一行属于可选的 `@deepseek-ai/dsh-experimental-schedule-bundle`，由它挂在**宿主**层。预设若同时声明同一行，会把 `timeContext` 这个 projection 注册两次，因此预设把它留给宿主。在 `0.2.1-alpha.x`（预设自己拥有该行）上，这个取舍意味着预设失去了当前时间上下文。
- **行集是跨版本的，按版本定制的文案则不是。** `plan-mode` 的指引文案取 `0.2.1-alpha.2`；`tool-subagent` 同时带上 `backgroundMode: continuable`（`0.2.0-rc.2` 的取值 —— `0.2.1-alpha.2` 已不再读取该键，其 schema 会保留未知键而不是报错）与 `0.2.1-alpha.2` 的 `toolFilter` 拒绝列表。
- **预设行需要跟随其插件演进。** 有两处行内携带指引文本 —— `plan-mode`，以及通过工具 schema 间接携带的工具行。DSH 版本若改变这些约定，本声明可能过期或失效；激活失败会带诊断信息留在名册上，而不会静默失败。
- **bundle patch 在启动时读取。** 装进运行中的 Host 需要重启后预设才会出现；patch 内容发生变化的 bundle，插件管理器会报告 `restart-required`。
- **没有自动化测试。** 本包只发布声明；验证方式就是装进 profile 并新建会话试用。

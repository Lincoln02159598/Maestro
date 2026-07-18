# PRD.md - 产品需求文档(基于实际实现)

> 本文档描述的是**当前源码实际实现的产品**,不是愿景或路线图。README/docs 的愿景与代码现实的差异在第 4、6 节单独列出。

## 分析快照

- 分支:`docs/multi-agent-review-loop-analysis`
- HEAD:`9affd8cfb27c8cfe39817ae92879445dc0632b40`
- 工作区状态:任务前 clean;本文档为本次任务新增
- 子模块状态:无
- 分析范围:`README.md`、`CLAUDE.md`、`src/shared/agentIds.ts`、`src/shared/agentMetadata.ts`、`src/main/agents/`、`src/main/cue/`、`src/main/group-chat/`、`src/shared/pianola/`、`src/cli/`、Encore 特性门控点
- 未覆盖范围:实际用户使用数据、外部网站文案

## 证据分类

- Evidence:源码、配置、入口直接证明
- Inference:由多处证据推导
- Unknown:文档声明但源码未验证、或需运行时确认

## 核心结论

[Evidence] Maestro 是一个**本地优先的桌面应用**,用于**同时管理多个 AI 编码助手**(Claude Code、Codex、OpenCode、Factory Droid、Copilot-CLI、Gemini-CLI、Qwen3-Coder 等),提供键盘优先的单窗口多 agent 工作区,并叠加事件驱动自动化(Cue)、多 agent 协作(Group Chat)、任务 DAG 编排(Pianola)与可扩展插件系统。
证据:`package.json:4-9`(`description: "Maestro hones fractured attention into focused intent."`)、`src/shared/agentIds.ts:16-28`、`src/main/index.ts`(主进程编排各子系统)。

---

## 1. 项目定位与目标用户

- [Evidence] 定位:面向开发者的「多 AI 编码助手统一驾驶舱」,强调键盘优先、本地数据、多 agent 并行。
  证据:`CLAUDE.md`「Project Overview」、`README.md`(顶部定位)。
- [Inference] 目标用户:同时使用多个 AI coding CLI(Claude Code、Codex 等)、希望在单一界面内切换/对比/编排它们的高级开发者与团队。
  推导依据:产品围绕多 agent + 自动化 + 群聊编排构建,非面向终端新手。

## 2. 解决的问题与适用场景

- 问题:多个 AI coding CLI 各自独立终端,上下文割裂、难以并行与编排。
- 适用场景:
  - 并行运行多个 agent 处理不同任务;
  - 一键向多个 agent 派发相同 prompt;
  - 自动化(文件变更/定时/GitHub 事件触发 agent);
  - 多 agent 复核与测试流水线(Cue/Pianola);
  - 远程(SSH、web-desktop、手机)控制本地 agent。

## 3. 产品边界

- 本地优先:数据存储于 userData(electron-store + SQLite),无强制云账户。
- [Evidence] 无用户登录/账户认证流程。
  证据:全仓未发现 auth/login 账户系统;Sentry 与可选 cloudflared 隧道为唯一对外网络面。
- 外部依赖:各 agent CLI 二进制(claude/codex/opencode 等)须在 PATH 或通过自定义路径配置;可选 `gh`(Cue GitHub 轮询)、`cloudflared`(隧道)。

## 4. 功能实现状态矩阵

| 功能 | 文档/README 声明 | 源码状态 | 运行时入口 | 测试证据 | 最终判断 |
| --- | --- | --- | --- | --- | --- |
| 多 agent 管理 | 声明(Active) | `src/shared/agentIds.ts` 11 个 ID;创建/删除/分组实现 | 主窗口 Left Bar + IPC `sessions`/`groups` | 大量 renderer/main 测试 | **已完整实现** |
| Claude Code 集成 | Active | definitions + parser + storage + maestro-p 包装 | 主流程 | cue/process 测试 | **已完整实现** |
| Codex / OpenCode / Factory Droid | Active/Beta | definitions + parsers + storage | 主流程 | 各 parser 测试 | **已实现** |
| Copilot-CLI | Beta(BETA_AGENTS) | definition + parser | 主流程 | 测试存在 | **已实现(Beta)** |
| Gemini-CLI / Qwen3-Coder / Hermes / Pi / OMP | CLAUDE.md 未列;`agentMetadata` 标 Beta | ID 已注册、可创建 agent | 可选 agent 类型 | 见各 capability | **已注册但文档未推广** |
| Cue 自动化引擎 | 文档有 | `src/main/cue/` 全套;`cueEngine.start('system-boot')` 门控于 `maestroCue` | Encore 开关 | `src/__tests__/main/cue/`(66 文件) | **已完整实现(门控)** |
| Group Chat(主持人+参与者) | 文档有 | `src/main/group-chat/` 全套 | 交互式入口 | 11+ 文件测试 | **已实现(仅交互式)** |
| Pianola DAG 编排 | 文档未在主表 | `src/shared/pianola/` + CLI `pianola orchestrate` | Encore `pianola` | orchestrator 测试 | **已实现(门控,部分 stub)** |
| 插件系统 | CLAUDE.md 有专章 | `src/main/plugins/` + `src/shared/plugins/` 真实沙箱/签名/账本 | Encore `plugins` | 31+22 测试文件 | **已完整实现(门控)** |
| Plugin SDK(`@maestro/plugin-sdk`) | 插件文档 | `packages/plugin-sdk/` 独立发布 | npm 包 | drift 测试 | **已实现** |
| Web-desktop(浏览器/PWA) | 文档有 | `src/web-desktop/`(4 文件)+ `src/web/public/` | `dev:web-desktop` | 2 测试 | **已实现** |
| SSH 远程执行 | 文档有 | `wrapSpawnWithSsh` + runners | spawn 路径 | 测试 | **已实现** |
| Auto Run / Playbooks | 文档有 | `src/renderer/hooks/batch/` + `src/cli/services/batch-processor.ts` | UI + CLI | 大量测试 | **已实现** |
| Goal-Driven Auto Run | CLI 文档 | `src/shared/goalDriven/` + CLI `goal-run` | CLI + hook | 测试 | **已实现** |
| 自动更新 | electron-builder 配置 | `publish: github` + window-manager auto-updater | 启动 | release.yml | **已实现** |
| cloudflared 隧道 | 设置项 | `src/main/tunnel-manager.ts` | 设置开关 | 测试存在 | **已实现(可选)** |
| 实验性:Cadenza / Movement | AppShell 中 Encore 门控渲染 | `CadenzaLayer`/`MovementOverlay` | Encore | - | **已实现(门控,实验性)** |
| `requestMerge`(Pianola) | - | stub,始终返回未合并 | - | orchestrator 注释 | **仅占位** |

## 5. 已实现 / 部分实现 / 仅占位

- **已完整实现**:多 agent 管理、Cue、Group Chat(交互)、插件系统、Plugin SDK、web-desktop、SSH、Auto Run/Playbooks、Goal-Run、自动更新。
- **部分实现**:Pianola(DAG 引擎完整,但 `requestMerge` 为 stub,fix prompt 泛化);Copilot-CLI 等为 Beta。
- **仅占位**:Pianola 的实际 git 合并路径。

## 6. README/docs 愿景 vs 源码现实

[Evidence] CLAUDE.md「Supported Agents」表仅列 `claude-code / codex / opencode / factory-droid / copilot-cli(Beta) / terminal(Internal)` 为 Active,但 `src/shared/agentIds.ts:16-28` 注册了 **11 个** ID(另含 `gemini-cli / qwen3-coder / hermes / pi / omp`)。后者在 `src/shared/agentMetadata.ts` 的 `BETA_AGENTS` 中标记为 Beta。
判断:文档落后于代码 —— 5 个 agent 已在代码可选用,但未进入主文档表。属「文档声明 vs 源码」的偏差(代码更全)。

[Inference] Cue 在 `docs/agent-guides/CUE-PIPELINE.md` 标注「Branch note: Cue source lives on the rc branch」,但本次审计在 `main`(实际为本任务分支 off main)已验证 `src/main/cue/cue-engine.ts` 存在且被 `index.ts:2723` 调用。判断:Cue 已合并到当前代码基线,文档的「仅在 rc」说明已过时。

## 7. 非目标 / 当前限制

- 非目标:不提供云端账户/同步(本地优先)。
- 限制:
  - Group Chat 无无人值守/事件触发入口(仅交互式)。
  - Cue 无原生「循环 N 次」控制流(需展开写)。
  - Pianola 无实际合并执行路径。
  - 测试矩阵缺 macOS(CI 仅 ubuntu+windows)。

## 8. 数据与隐私边界

[Evidence] 数据落盘于 `app.getPath('userData')`:`maestro-*.json`(electron-store)、`stats.db`、cue.db、`history/<sessionId>.json`、`session-images/`。
证据:`src/main/stores/instances.ts`、`src/main/stats/stats-db.ts:85`、`src/main/history-manager.ts`。

[Evidence] 对外网络面:Sentry(错误上报,受 `crashReportingEnabled` 门控)、cloudflared 隧道(可选)、Cue/插件的网络出口(受插件 egress guard 与 capability 控制)、`gh` CLI(Cue GitHub 轮询)。
证据:`src/renderer/main.tsx:24-63`、`src/main/tunnel-manager.ts`、`src/main/plugins/plugin-host-handlers.ts`、`src/main/cue/cue-github-poller.ts`。

## 9. 产品风险

- [Inference] agent CLI 二进制版本/参数漂移风险:Maestro 强依赖各 CLI 的 batch 模式参数(如 `claude --print --output-format stream-json`),上游 CLI 升级可能破坏解析。
- [Inference] node-pty beta 依赖带来的退出稳定性风险(已用 SIGKILL 绕过)。
- [Evidence] 插件 `agents:dispatch` / `process:spawn` 为高风险 act 动词,虽经 allowlist scope 与账本保护,仍是最大权限面。
  证据:`src/shared/plugins/permissions.ts:199-214`。

---

## 已确认事实

- 产品为本地优先多 AI 编码助手管理器,核心子系统(多 agent、Cue、Group Chat、Pianola、插件、web-desktop、SSH、Auto Run)均已实现。
- 11 个 agent ID 已注册(文档仅推广 6 个)。
- 无账户/登录系统。

## 合理推断

- 目标用户为同时使用多个 AI CLI 的高级开发者。
- 文档落后于代码(agent 推广数、Cue 分支说明)。

## Unknown 与待验证事项

- Cadenza / Movement 等 Encore 实验特性的产品意图(代码门控存在,但无对应 PRD 级文档说明其面向场景)。
- 5 个未推广 agent(gemini-cli 等)的成熟度是否达到 Beta 可用(需实际运行各 CLI 验证,本任务不执行)。

## 批判性评估

- 功能矩阵高度完整,但「文档声明 vs 源码」存在两处落后(agent 推广、Cue 分支)。
- 多个高价值子系统(Cue、Pianola)受 Encore 门控,默认关闭 —— 对新用户而言「声明的能力」与「开箱默认体验」存在落差。

## 建设性改善建议

[Recommendation] 同步 CLAUDE.md/README 的 agent 表与 `agentIds.ts` 现状(增列 gemini-cli/qwen3-coder/hermes/pi/omp 并标注 Beta)。
优先级:中;实施难度:低;依据:`src/shared/agentIds.ts:16-28` vs `CLAUDE.md` Supported Agents 表的已验证偏差。

[Recommendation] 为 Pianola 的 `requestMerge` 补齐实际合并执行路径或显式标注「不可用」,避免 CLI `orchestrate` 给出误导性「编排完成但未合并」。
优先级:中;实施难度:中;依据:`src/cli/commands/pianola-orchestrate.ts:705-741` stub 与注释。

## 主要证据索引

- `README.md`、`CLAUDE.md`(Project Overview / Supported Agents)
- `src/shared/agentIds.ts:16-28`
- `src/shared/agentMetadata.ts:17-148`(`BETA_AGENTS`)
- `src/main/index.ts:2719-2768`(Encore 门控启动)
- `src/main/cue/cue-engine.ts`、`src/main/group-chat/`、`src/shared/pianola/`
- `src/main/plugins/`(插件系统)
- `src/main/stores/instances.ts`、`src/main/stats/stats-db.ts:85`
- `src/cli/commands/pianola-orchestrate.ts:705-741`

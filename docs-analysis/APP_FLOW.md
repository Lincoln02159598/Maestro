# APP_FLOW.md - 用户应用流程

> 以用户视角描述真实应用流程,并与源码调用链对应。不为完整性虚构流程。

## 分析快照

- 分支:`docs/multi-agent-review-loop-analysis`
- HEAD:`9affd8cfb27c8cfe39817ae92879445dc0632b40`
- 工作区状态:任务前 clean;本文档为本次任务新增
- 子模块状态:无
- 分析范围:`src/main/index.ts`(启动)、`src/main/app-lifecycle/`(退出)、`src/renderer/components/Wizard/`、`src/renderer/hooks/agent/`、`src/renderer/components/MainPanel/MainPanelContent.tsx`、`src/main/process-manager/`、`src/main/storage/`
- 未覆盖范围:实际 GUI 交互的动态回放(未运行应用)

## 证据分类

- Evidence:源码直接证明的流程
- Inference:由调用链推导
- Unknown:需运行应用才能确认的视觉/时序细节

## 核心结论

[Evidence] 启动流程为 Electron 标准 `app.whenReady` → 依次初始化 stores/prompts/history/stats/IPC/process listeners → 条件启动 Cue/Pianola/插件 → 创建主窗口 → 渲染 React 应用。退出走自定义 `quitHandler`,带确认对话框、5s 安全超时与 SIGKILL hard-exit。
证据:`src/main/index.ts:888-2940`(whenReady 链)、`src/main/app-lifecycle/quit-handler.ts:201-319`。

[Evidence] **当前源码未发现用户登录或账户认证流程。** 应用为本地优先,无登录/注册/云同步步骤。
证据:全仓无 auth/login/signup 模块;渲染入口 `src/renderer/main.tsx:109-135` 直接挂载 `<MaestroConsole/>`,前置仅有可选 Sentry 初始化。

---

## 1. 安装与首次启动

- [Evidence] 桌面端通过 electron-builder 打包为 mac(dmg/zip)/win/linux 安装包;CLI `maestro-cli`、`maestro-p` 随包发布(`package.json build.extraResources`)。
- 首次启动:
  - 触发 `app.whenReady` 链(`src/main/index.ts:888`)。
  - `initializeStores()` 建立 electron-store 实例(`:355`)。
  - 一次性 `runSettingsMigrations`(`:377-387`)。
  - `initializePrompts()` 失败则 fatal exit(`:1102-1116`)。
  - 创建主窗口 `restoreWindows()`(`:2858`)。
- 首次运行向导(Wizard):`src/renderer/components/Wizard/`(MaestroWizard、WizardContext),引导配置首个 agent。
  证据:`src/renderer/components/Wizard/MaestroWizard.tsx`、`CLAUDE-WIZARD.md`。

## 2. 主界面进入与核心任务流

[Inference] 主界面由 `AppShell.tsx` 空间布局:Left Bar(SessionList)、MainPanel、RightPanel。
推导依据:`src/renderer/components/AppShell.tsx:206-274`。

核心场景:**向 agent 发送 prompt 并获得响应**

```
前置条件:已创建至少一个 agent(会话),agent CLI 二进制可用
→ 用户操作:在 MainPanel 输入框输入 prompt 并回车
→ 系统响应:
   渲染层 useAgentExecution → window.maestro.process.spawn(config)(src/renderer/services/process.ts)
   主进程 ProcessManager.spawn()(ProcessManager.ts:85-175)→ 选 PtySpawner/ChildProcessSpawner
   → 子进程(claude/codex/...)启动,stdin 投递 prompt
→ 数据变化:
   process-listeners(data/usage/session-id/error/exit)把输出转成渲染事件(safeSend)
   → useAgentListeners 订阅 → sessionStore 写入 aiTabs 日志
→ 异常分支:
   error-patterns 匹配 → agentError 事件 → AgentErrorBanner 显示
   进程崩溃 → exit-listener → 状态转 idle/red
→ 完成条件:进程退出(exitCode)→ 会话状态转 idle(绿)
```
证据:`src/main/process-manager/ProcessManager.ts:85-175`、`src/main/process-listeners/index.ts:32-60`、`src/renderer/hooks/agent/useAgentListeners.ts`、`src/renderer/components/MainPanel/MainPanelContent.tsx:631-1030`。

## 3. 视图/模式切换

[Evidence] 无路由器。MainPanel 的中心视图由 `MainPanelContent.tsx:631-1030` 按 `Session.inputMode` / `activeTabId` / `activeFileTabId` / `activeBrowserTabId` 等字段分派:AI 终端、命令终端(keep-alive overlay)、文件预览、向导、平铺标签组、浏览器标签。Log Viewer 与 Group Chat 在 `App.tsx:3035-3540` 作为 MainPanel 的整体替换。
证据:`src/renderer/components/MainPanel/MainPanelContent.tsx`、`src/renderer/App.tsx:3035-3540`。

## 4. 创建/删除 agent(会话)

- 创建:通过 UI 或 CLI `maestro-cli create-agent`(走 WebSocket `create_session`)→ `src/renderer/hooks/session/useSessionCrud.ts:243-245` 构造 `Session`(cwd/fullPath/projectRoot 同一工作目录)→ `window.maestro.sessions.*` 持久化到 `maestro-sessions.json`。
- 删除:UI 或 CLI `remove-agent`(WebSocket `delete_session`)。

## 5. 数据保存与持久化

[Evidence] 持久化分三层:
- electron-store:`maestro-sessions.json`(会话/标签)、`maestro-settings.json`、`maestro-groups.json` 等。
- SQLite:`stats.db`(用量)、cue.db(Cue 事件)。
- 历史:`history/<sessionId>.json`(每会话 ≤5000 条,原子写)。
证据:`src/main/stores/instances.ts`、`src/main/stats/stats-db.ts`、`src/main/history-manager.ts`。

[Evidence] 数据导入/导出:存在 playbook 导入/导出(带 assets 的 ZIP,`src/main/ipc/handlers/playbooks.ts`)、marketplace 导入(`src/main/ipc/handlers/marketplace.ts`)、theme 导入/导出(CLI `theme import/export`)。
证据:`docs/agent-guides/CLI-PLAYBOOKS.md`、`src/cli/index.ts`(theme 命令组)。

## 6. 自动化与多 agent 编排(用户视角)

- **Auto Run**:在 RightPanel 的 Auto Run tab 编辑 markdown 任务清单,一键批量向 agent 派发(`src/renderer/hooks/batch/`)。
- **Cue**:在 `.maestro/cue.yaml` 定义事件订阅,通过 Cue Modal 监控(`src/renderer/components/CueModal/`),需开启 `maestroCue` Encore。
- **Group Chat**:创建群聊(主持人 + 多 agent 参与者),用户发首条消息驱动。
- **Pianola**(CLI):`maestro-cli pianola orchestrate <plan>` 跑任务 DAG。

## 7. 错误提示与异常恢复

- agent 错误:`AgentErrorBanner`(`MainPanel/AgentErrorBanner.tsx`)。
- 会话恢复:`useAutoResumeCoordinator`、`useSessionRecovery`(renderer hooks);Cue 队列持久化与崩溃恢复(`src/main/cue/cue-recovery-service.ts`)。
- 窗口崩溃:`render-process-gone` → `processManager.killAll()`(`src/main/index.ts:817-819`)防 PTY FD 泄漏。

## 8. 设置修改

[Evidence] 设置 UI 为 12 个 tab 的 SettingsModal(`src/renderer/components/Settings/SettingsModal.tsx:64` TAB_ITEMS)。每个 setter 即时写穿 `window.maestro.settings.set`(`src/renderer/stores/settingsStore.ts:896-931`),主进程 electron-store 持久化。

## 9. 多窗口切换

[Evidence] 多窗口为真实能力:`WindowRegistry`(`src/main/window-registry.ts`)为真相源,`safeSend` 广播到所有窗口(`src/main/index.ts:577`)。窗口状态 `maestro-window-state.json` 按设备存储、不同步。

## 10. 退出、升级、再启动

- 退出:`before-quit` → 若有未确认事项则 `preventDefault` + 发 `app:requestQuitConfirmation` + 5s 超时 → 确认后 `performCleanup()`(顺序:保存窗口状态 → 停 watcher → flush 镜像 → 停 Cue/插件 → `processManager.killAll` → flush 遥测 → `closeStatsDB`)→ `FORCE_EXIT_GRACE_MS` 后 SIGKILL 自杀。
  证据:`src/main/app-lifecycle/quit-handler.ts:231-318,347-456`。
- 升级:electron-builder `publish: github`,`window-manager` 初始化 auto-updater;更新会触发退出(release.yml)。
- 再启动:队列持久化在 `cue_event_queue` 表,启动时 `restoreAll()` 重放(`cue-engine.ts:596`)。

## 10a. 登录 / 账户(明确声明)

[Evidence] 当前源码未发现用户登录或账户认证流程。无注册、无登录、无云同步账户。Sentry 上报受设置门控,cloudflared 隧道为可选。

---

## 已确认事实

- 启动/退出流程清晰,退出带确认 + hard-exit 绕过 node-pty 死锁。
- 无登录/账户系统。
- 核心任务流(spawn agent → listeners → store → 视图)端到端可追踪。

## 合理推断

- 视图分派完全基于 Session 字段,无路由;多窗口通过 WindowRegistry + 广播。

## Unknown 与待验证事项

- 首次运行向导的实际步骤顺序与跳过逻辑(未运行动态验证)。
- 多窗口下各窗口独立会话上下文的视觉一致性(需运行)。

## 批判性评估

- 退出流程依赖 SIGKILL-to-self 是稳健性妥协,已在代码注释充分说明。
- 「无登录」是明确产品决策,文档应一致表述,避免误以为存在云账户能力。

## 建设性改善建议

[Recommendation] 在 README 显式声明「本地优先、无账户、无云同步」,与源码一致,避免用户误期。
优先级:低;实施难度:低;依据:本节已验证无账户系统。

## 主要证据索引

- `src/main/index.ts:888-2940`(启动)、`:817-819`(崩溃)、`:2858`(建窗)
- `src/main/app-lifecycle/quit-handler.ts:201-319,347-456`
- `src/main/process-manager/ProcessManager.ts:85-175`
- `src/main/process-listeners/index.ts:32-60`
- `src/renderer/components/MainPanel/MainPanelContent.tsx:631-1030`
- `src/renderer/App.tsx:3035-3540`
- `src/renderer/components/Settings/SettingsModal.tsx:64`
- `src/renderer/hooks/session/useSessionCrud.ts:243-245`
- `src/main/stores/instances.ts`

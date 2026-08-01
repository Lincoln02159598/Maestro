# BACKEND_Development.md - 后端(主进程)开发指南

> 在 Maestro 语境下「后端」= Electron 主进程(`src/main/`)。无独立服务器后端;WebServer 是主进程内嵌服务。

## 分析快照

- 分支:`docs/multi-agent-review-loop-analysis`
- HEAD:`9affd8cfb27c8cfe39817ae92879445dc0632b40`
- 工作区状态:任务前 clean;本文档为本次任务新增
- 子模块状态:无
- 分析范围:`src/main/`(index、ipc/handlers、process-manager、stores、stats、cue、group-chat、plugins、agents、parsers、storage、web-server、app-lifecycle、utils)
- 未覆盖范围:`__tests__/main` 仅抽样;未运行主进程

## 证据分类

- Evidence:源码直接证明
- Inference:多证据推导
- Unknown:需运行时确认

## 核心结论

[Evidence] 主进程是单一 Node.js 进程(`dist/main/index.js`),通过 `ipcMain.handle`(全仓 ~511 处,53 个 handler 模块)向渲染进程暴露能力,通过内嵌 Fastify `WebServer` 向浏览器/CLI 暴露能力,通过 `ProcessManager` spawn 外部 agent CLI 子进程。
证据:`src/main/index.ts:2681`(`setupIpcHandlers`)、`src/main/preload/index.ts:84-267`、`src/main/web-server/WebServer.ts:157`、`src/main/process-manager/ProcessManager.ts:85-175`。

---

## 1. 入口与初始化

- 入口:`src/main/index.ts`(~3500 行)。模块级:注册自定义协议(`app://`、`maestro-image://`,`:301-321`)、`initializeStores()`(`:355`)、Sentry(`:409-463`)、单例声明(`:488-503`)。
- `app.whenReady` 链(`:888-2940`):prompts → history → statsDB → IPC handlers → process listeners → agentRun 捕获 → 条件启动 Cue/Pianola/插件 → 建窗 → 全局热键 → watcher。详见 `SOURCE_ARCHITECTURE.md` 启动时序图。

## 2. IPC 与 command 注册

- 注册:`setupIpcHandlers()`(`index.ts:3015-3444`)内联展开 53 个 `registerXxxHandlers({...})`。
- 通道命名:`namespace:method` 字面量(git/process/cue/plugins/groupChat/...),无中央通道表。
- 统一封装:`createHandler/createDataHandler`(`src/main/utils/ipcHandler.ts:96-301`),返回 `{success, data?, error?}`。
- Preload 桥:`contextBridge.exposeInMainWorld('maestro', {...})`,~50 命名空间,每命名空间一 `createXxxApi()` 工厂。

### 典型 handler 示例:process:spawn

```
入口:ipc/handlers/process.ts → handleProcessSpawn( ipc/handlers/process/handle-spawn.ts )
→ 参数:ProcessConfig(sessionId, toolType, cwd, command, args, prompt, sshRemoteId, ...)
→ 验证:cwd 展开 ~、同 sessionId 旧进程 kill
→ 返回:{success, data: {pid?}}
→ 调用服务:ProcessManager.spawn
→ 数据访问:无直接 DB(进程映射在内存)
→ 事务边界:无(进程级)
→ 错误类型:spawn 失败 → {success:false, error}
→ 调用方:渲染 services/process.ts、WebServer executeCommand
→ Evidence:ipc/handlers/process.ts:124-138、process/handle-spawn.ts、ProcessManager.ts:85-175
```

## 3. 数据访问层 / 数据库 / Schema / 迁移

### electron-store(9 实例,`stores/instances.ts:81-179`)

- `maestro-bootstrap.json`(同步路径真相源)、`maestro-settings.json`、`maestro-sessions.json`、`maestro-groups.json`(syncPath)
- `maestro-agent-configs.json`、`maestro-agent-capabilities.json`(productionDataPath,dev/prod 共享)
- `maestro-window-state.json`(默认 userData,按设备)
- `maestro-claude-session-origins.json`、`maestro-agent-session-origins.json`(syncPath)
- 迁移:`stores/migrations/`(一次性幂等,各带 marker):multi-window-state、playbooks-folder、api-mode-default、adaptive-mode-default。

### SQLite(better-sqlite3,WAL)

- `stats.db`(`stats/stats-db.ts:79`):迁移 `stats/migrations.ts`;CRUD 模块 query-events/auto-run/session-lifecycle/aggregations/data-management/image-annotations/shortcut-usage/multi-window-usage;单例 `stats/singleton.ts`。
- `cue.db`(`cue/cue-db.ts:9-120`):表 `cue_events / cue_heartbeat / cue_github_seen`,加列迁移 `:92-99`。

### 会话存储

- `storage/index.ts:36-42`:`initializeSessionStorages()` 注册 5 实现(Claude/OpenCode/Codex/FactoryDroid/Copilot),均继承 `BaseSessionStorage`。

### 历史

- `history-manager.ts`:迁移自 legacy `maestro-history.json` → `history/<sessionId>.json`,原子写,每会话 ≤5000 条。

## 4. 事务 / 并发 / 后台任务

- [Evidence] 无关系型事务(electron-store 单键写;SQLite 同步单连接)。并发靠:ProcessManager 按 sessionId 串行(kill 旧进程)、Cue `max_concurrent`(默认 1)+ 持久化队列 + 烧坏即溢出、插件 RPC 限流(1MB/32 in-flight/200 rps)。
- 后台任务:Cue heartbeat(30s)、GitHub 轮询、`UsageRefreshScheduler`、插件 `BackgroundSupervisor`(指数退避重启)、`cliWatcher`、`settingsWatcher`、history 目录监听。

## 5. 鉴权 / 权限 / 安全边界

- 进程内 IPC:无鉴权(渲染为可信方,contextBridge 隔离)。
- WebServer:UUID token 路径前缀(`$TOKEN/`),可选持久化;绑定 0.0.0.0 依赖 token。
- 插件:capability + scope + broker 默认拒绝 + sealed 授权账本 + ed25519 签名(`permissions.ts`、`permission-broker.ts`、`authorization-ledger.ts`)。
- SSH:`wrapSpawnWithSsh`(`utils/ssh-spawn-wrapper.ts`)。
- 错误吞噬:`createHandler` 统一上报,非静默;插件失败隔离。

## 6. 外部服务 / 文件系统

- agent CLI(claude/codex/...)、`gh`(Cue GitHub)、cloudflared(隧道)、Sentry、models.dev(Copilot 模型拉取)。
- 文件系统:userData 下各 JSON/SQLite/history/session-images;`maestro-image://` 协议提供图片。

## 7. 扩展方式

- 新增 agent:`src/shared/agentIds.ts` + `src/main/agents/definitions.ts` + `capabilities.ts` + parser + storage(`AGENT_SUPPORT.md`)。
- 新增 IPC handler:`src/main/ipc/handlers/` + preload 命名空间 + `global.d.ts`。
- 新增 Cue 事件/模板变量:`src/shared/cue/contracts.ts` + 触发源 + template-context-builder(`CLAUDE-CUE.md` 改动配方表)。

## 8. 测试 / 调试入口 / 已知限制

- 测试:`src/__tests__/main/`(349+ 文件),覆盖 cue/group-chat/plugins/process-manager/ipc。
- 调试:`maestro-cli doctor`、`debug` IPC 命名空间、Sentry。
- 限制:退出需 SIGKILL 绕过 node-pty 死锁;`setupIpcHandlers` 内联展开;Pianola `requestMerge` stub。

---

## 已确认事实

- 主进程通过 IPC(511 处)+ WebServer + ProcessManager 三面暴露能力。
- 持久化分 electron-store(9)+ SQLite(2 库)+ history 文件。
- 无关系型事务,并发靠 sessionId 串行 + 队列 + 限流。

## 合理推断

- 「无中央通道表 + 内联注册」是可维护性折中,换取每个 handler 的丰富依赖注入。

## Unknown 与待验证事项

- electron-store 多实例并发写在崩溃时的原子性(electron-store 内部 atomic rename,未运行压力测试)。
- WebServer token 在持久化模式下的轮换策略细节。

## 批判性评估

- `index.ts` 3500 行集中编排,新成员定位成本高。
- SQLite 用同步 better-sqlite3 单连接,主进程长查询可能阻塞 IPC(未发现长查询,但需警惕)。

## 建设性改善建议

[Recommendation] 为 IPC 通道建立中央常量表(由 handler 模块反向注册),减少字面量漂移与 preload/handler 不一致风险。
优先级:中;实施难度:中;依据:`preload/index.ts` 与 `ipc/handlers/*` 的字面量通道名各自维护。

## 主要证据索引

- `src/main/index.ts:888-2940,2681,3015-3444`
- `src/main/utils/ipcHandler.ts:96-301`
- `src/main/preload/index.ts:84-267`
- `src/main/stores/instances.ts:81-179`
- `src/main/stats/stats-db.ts:79-85`、`src/main/cue/cue-db.ts:64-122`
- `src/main/storage/index.ts:36-42`
- `src/main/process-manager/ProcessManager.ts:85-175`
- `src/main/ipc/handlers/process.ts:124-138`
- `src/main/plugins/permission-broker.ts:72-121`、`authorization-ledger.ts`
- `src/main/app-lifecycle/quit-handler.ts:58-86`

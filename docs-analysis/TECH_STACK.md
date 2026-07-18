# TECH_STACK.md - 技术栈与依赖

## 分析快照

- 分支:`docs/multi-agent-review-loop-analysis`
- HEAD:`9affd8cfb27c8cfe39817ae92879445dc0632b40`
- 工作区状态:任务开始前 clean;本文档为本次任务新增,位于 `./docs-analysis/`
- 子模块状态:无子模块
- 分析范围:`package.json`、构建脚本、`src/main`、`src/renderer`、`src/cli`、`packages/plugin-sdk`、CI 配置
- 未覆盖范围:`package-lock.json` 逐项(仅抽样关键依赖);未实际执行安装或构建

## 证据分类

- Evidence:由 `package.json`、源码 import、构建脚本直接证明
- Inference:由多个 import 与调用点推导
- Unknown:锁文件中声明但源码未直接引用、需运行时验证的依赖

## 核心结论

[Evidence] Maestro 是一个 **Electron + React + TypeScript** 桌面应用,辅以独立 CLI(`maestro-cli`)、TUI 包装器(`maestro-p`)和浏览器/PWA 构建(web-desktop)。单一 `package.json`,非 workspace monorepo;另有 `packages/plugin-sdk` 作为独立发布的 npm 包。
证据:`package.json:1-10`(`name: maestro`、`main: dist/main/index.js`、`bin: maestro-cli / maestro-p`)、`packages/plugin-sdk/package.json:1-28`。

---

## 1. 编程语言与用途

| 语言 | 用途 | Evidence |
| --- | --- | --- |
| TypeScript | 主进程、渲染进程、CLI、shared、preload 全栈 | `src/**/*.ts(x)`、`tsconfig.*.json` |
| JavaScript(构建/脚本) | 构建、打包、代码生成脚本 | `scripts/*.mjs`(26 个) |
| Markdown | 系统提示词、用户文档 | `src/prompts/*.md`(93)、`docs/` |

[Evidence] 无 Rust/Go/Python 进参与运行时(仅 `scripts/check-python.mjs` 作 preinstall 探测,非运行时依赖)。
证据:`package.json:33`(preinstall)。

## 2. 框架与平台

| 类别 | 技术 | 版本 | Evidence |
| --- | --- | --- | --- |
| 桌面端 | Electron | ^41.5.0 | `package.json` deps |
| 前端框架 | React | ^18.2.0 | `src/renderer/main.tsx`、`App.tsx` |
| 后端/主进程 | Electron main + Node.js(Electron 内置) | - | `src/main/index.ts` |
| CLI | Commander.js | ^14.0.2 | `src/cli/index.ts` |
| 浏览器构建 | Vite + WebSocket 桥 | vite ^8.0.11 | `src/web-desktop/*`、`vite.config.web-desktop.mts` |
| 状态管理 | Zustand | ^5.0.11 | `src/renderer/stores/*`(33 个 store) |
| 终端 UI | @xterm/xterm + addon-fit/search/unicode11 | - | `src/renderer/components/XTerminal.tsx` |
| 流程图编辑 | React Flow(reactflow) | ^11.11.4 | `src/renderer/components/CuePipelineEditor/` |
| HTTP/WS 服务 | Fastify + @fastify/websocket、cors、rate-limit、static | fastify ^4.25.2 | `src/main/web-server/WebServer.ts` |
| Markdown | remark/rehype + react-markdown + Shiki | shiki ^4.0.2 | `src/renderer/components/Markdown/` |

## 3. 数据库与数据访问

[Evidence] 使用 **better-sqlite3(^12.11.1)** 同步 SQLite(WAL 模式),无 ORM。两个独立库:
- 统计库 `stats.db`:`src/main/stats/stats-db.ts`,带迁移系统 `src/main/stats/migrations.ts`。
- Cue 库(cue.db):`src/main/cue/cue-db.ts:9-120`,表 `cue_events / cue_heartbeat / cue_github_seen`。
证据:`src/main/stats/stats-db.ts:79-85`、`src/main/cue/cue-db.ts:64-122`。

[Evidence] 其余持久化使用 **electron-store**(JSON 文件),实例集中在 `src/main/stores/instances.ts:81-179`。
证据:`src/main/stores/instances.ts`、`src/main/stores/index.ts:40-74`。

## 4. 关键运行时依赖(逐项)

### node-pty 1.2.0-beta.12
- 实际用途:为交互式 agent(TUI 模式 Claude、terminal)提供伪终端。
- 初始化位置:`src/main/process-manager/spawners/PtySpawner.ts:21`。
- 调用位置:`ProcessManager.spawn()` 路由(`src/main/process-manager/ProcessManager.ts:177-211` `shouldUsePty`)。
- 关键运行路径:是。agent 交互式会话的核心。
- 是否可选:否。
- Evidence:`package.json`、`src/main/process-manager/ProcessManager.ts`。`asarUnpack` 显式包含 node-pty(`package.json build.asarUnpack`)。

[Inference] node-pty 是退出时 hard-exit(SIGKILL to self)的根因 —— 其与 fsevents 的 TSFN 死锁(MAESTRO-3B)。
推导依据:`src/main/app-lifecycle/quit-handler.ts:58-86` 的注释与 `process.kill(pid, 'SIGKILL')`;`vitest.config.mts:42-43` 因原生插件多线程 SEGFAULT 而强制 `pool: 'forks'`。

### better-sqlite3 ^12.11.1
- 用途:统计与 Cue 的同步持久化。同上 §3。
- 关键路径:是(统计、Cue 事件、GitHub seen)。
- Evidence:`src/main/stats/stats-db.ts`、`src/main/cue/cue-db.ts`。

### fastify ^4.25.2
- 用途:内置 Web/WebSocket 服务,服务 web-desktop、PWA、`maestro-cli` 远程桥。
- 初始化:`src/main/web-server/WebServer.ts:157`。
- 调用:`ensureCliServer()`(`src/main/ipc/handlers/web.ts:99`)在启动时创建。
- 关键路径:web-desktop 与 CLI 远程驱动场景下是;纯桌面本地使用非必需(按需启动)。
- Evidence:`src/main/web-server/WebServer.ts`、`src/main/web-server/web-server-factory.ts`。

### commander ^14.0.2
- 用途:`maestro-cli` 与 `maestro-p` 的命令行解析。
- Evidence:`src/cli/index.ts`、`src/maestro-p/index.ts`。

### zustand ^5.0.11
- 用途:渲染进程全部状态。无 Redux/MobX。
- Evidence:`src/renderer/stores/*.ts`(33 文件),`useStoreWithEqualityFn` 来自 `zustand/traditional`(`src/renderer/App.tsx:523`)。

## 5. 构建工具与包管理

| 类别 | 技术 | Evidence |
| --- | --- | --- |
| 包管理器 | npm(`package-lock.json`),CI 用 `npm ci` | `.github/workflows/ci.yml` |
| 主进程构建 | tsc(`tsconfig.main.json`)+ `scripts/gen-build-info.mjs` | `package.json build:main` |
| 渲染/web-desktop 构建 | Vite ^8.0.11 | `vite.config.mts`、`vite.config.web-desktop.mts` |
| 预加载打包 | esbuild(`scripts/build-preload.mjs`,多入口) | `scripts/build-preload.mjs:50-69` |
| CLI 打包 | esbuild(`scripts/build-cli.mjs`) | `package.json build:cli` |
| 打包发布 | electron-builder(mac/win/linux) | `package.json build`、`package` |
| 代码生成 | `gen-cli-reference.mjs` 等 | `package.json scripts` |

[Evidence] 本地部分脚本使用 `bun run`(如 `validate:push`),但安装与 CI 统一为 npm。
证据:`package.json:46-53`。

## 6. 测试与 CI 技术栈

| 类别 | 技术 | 版本 | Evidence |
| --- | --- | --- | --- |
| 单元/集成测试 | Vitest | ^4.1.5 | `vitest.config.mts` 等 4 个配置 |
| E2E(桌面) | Playwright(electron) | ^1.59.1 | `playwright.config.ts`、`e2e/` |
| 类型检查 | tsc(多 tsconfig) | - | `package.json lint` |
| Lint | ESLint(flat config) | - | `eslint.config.mjs`、`lint:eslint` |
| 格式化 | Prettier | - | `format*` 脚本 |
| Git hooks | Husky 9 + lint-staged | ^9.1.7 | `.husky/`、`package.json:428-435` |
| 覆盖率 | @vitest/coverage-v8 | - | `vitest.config.mts:76`,无阈值 |

详见 `测试与CI.md`。

## 7. 可观测性、安全与外部集成

| 类别 | 技术 | Evidence |
| --- | --- | --- |
| 错误监控 | @sentry/electron(main/renderer/preload) | `src/main/index.ts:409-463`、`src/renderer/main.tsx:24-63` |
| 隧道 | cloudflared(可选,外部二进制) | `src/main/tunnel-manager.ts` |
| 签名 | ed25519(插件签名) | `src/shared/plugins/signing.ts`、`src/main/plugins/plugin-signature.ts` |
| OS 密钥环 | @napi-rs/keyring(授权账本加密锚) | `package.json build.asarUnpack` |
| MCP | `mcp serve`(stdio MCP 服务,暴露插件工具) | `src/cli/index.ts`(mcp 命令组) |

## 8. Feature Flag(Encore Features)

[Evidence] 通过 `encoreFeatures` 设置项门控的长生命周期子系统,设置时构造、运行时自门控、可在不重启情况下开关:
`maestroCue`、`pianola`、`plugins`、`usageStats`、`coworking`、`directorNotes`、`documentGraph`、`leaderboard`、`opencodeServer`、`symphony`。
证据:`src/shared/settingsMetadata.ts`(encore 键集合)、`src/main/index.ts:2719-2768`(Cue/Pianola/Plugin 启动门控)。

## 9. 依赖有效性检查

[Inference] `package.json` 中大量依赖确有源码引用(react/electron/zustand/fastify/commander/node-pty/better-sqlite3 等),未见明显无效依赖。但锁文件级逐项审计未做。
推导依据:对运行时关键依赖均找到初始化与调用点。

[Unknown] `package.json` 中开发依赖全集(尤其一次性脚本依赖)是否全部仍被使用,未逐项核对;需运行未使用依赖检测(如 depcheck)才能确认,本任务禁止安装/构建,故标记 Unknown。

---

## 已确认事实

- 技术栈核心为 Electron 41 + React 18 + TypeScript + Zustand 5 + Vite 8,数据库 better-sqlite3,CLI Commander,Web 服务 Fastify。
- 无 monorepo workspace;`packages/plugin-sdk` 为独立发布包(vendored 自 `src/shared/plugins`)。
- 10 个 Encore 特性门控子系统。
- 11 个 agent ID(`src/shared/agentIds.ts`)。

## 合理推断

- node-pty 是退出 hard-exit 与 Vitest 强制 forks pool 的共同根因。
- 测试与构建脚本混用 bun/npm,但发布链路统一 npm。

## Unknown 与待验证事项

- 开发依赖全集的有效性(depcheck 未执行)。
- cloudflared、@napi-rs/keyring 等平台相关依赖在所有 CI 矩阵腿的实际可用性(本任务不运行 CI)。

## 批判性评估

- node-pty 1.2.0-beta.12 为 **beta 版**,是退出死锁风险点,代码已以 SIGKILL-to-self 显式绕过(`quit-handler.ts:58-86`),属于已知技术债而非隐患。
- 主进程构建仍用 tsc(非打包器),配合 electron-builder 的 `files: ["dist/**/*"]`,构建产物体积与启动时间可能偏大(未测量)。

## 建设性改善建议

[Recommendation] 评估将主进程构建由 tsc 切换为 esbuild bundle(与 preload/CLI 一致),以减小产物体积与冷启动时间。
优先级:低;实施难度:中;依据:preload/CLI 已用 esbuild 打包,主进程是唯一例外。

## 主要证据索引

- `package.json:1-120`
- `src/shared/agentIds.ts:16-28`
- `src/shared/settingsMetadata.ts`(encore 键)
- `src/main/stores/instances.ts:81-179`
- `src/main/stats/stats-db.ts:79-85`
- `src/main/cue/cue-db.ts:64-122`
- `src/main/web-server/WebServer.ts:157`
- `src/main/process-manager/ProcessManager.ts:177-211`
- `src/main/app-lifecycle/quit-handler.ts:58-86`
- `vitest.config.mts:42-43,76`
- `packages/plugin-sdk/package.json:1-28`

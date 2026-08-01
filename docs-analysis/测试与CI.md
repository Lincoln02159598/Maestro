# 测试与CI.md

> 严格区分:测试文件存在 ≠ 命令配置 ≠ CI 执行 ≠ 仅手动。

## 分析快照

- 分支:`docs/multi-agent-review-loop-analysis`
- HEAD:`9affd8cfb27c8cfe39817ae92879445dc0632b40`
- 工作区状态:任务前 clean;本文档为本次任务新增
- 子模块状态:无
- 分析范围:`vitest.*.config.*`、`playwright.config.ts`、`src/__tests__/`、`e2e/`、`.github/workflows/`、`.husky/`、`package.json` 脚本
- 未覆盖范围:未实际执行测试/CI(本任务禁止);测试内容仅抽样

## 证据分类

- Evidence:配置/脚本/CI yaml 直接证明
- Inference:多证据推导
- Unknown:需运行才能确认(覆盖质量、实际通过率)

## 核心结论

[Evidence] 测试体系以 **Vitest** 为核心(jsdom/node 双 project 分摊 DOM 成本),辅以 **Playwright**(Electron E2E,7 specs)与专项配置(integration/performance/e2e)。CI 仅 `.github/workflows/ci.yml`(38 行)在 **ubuntu + windows** 矩阵跑 `test` 与 `lint/format`,**不跑** e2e/integration/performance/coverage,且**无 macOS 测试腿**。pre-push hook(`validate:push`)是 CI 外唯一的完整门禁。
证据:`vitest.config.mts:51-87`、`playwright.config.ts:16-31`、`.github/workflows/ci.yml:3-37`、`.husky/pre-push`、`package.json:46-61`。

---

## 1. 测试框架与配置

| 配置                                   | 框架                 | 目标          | 关键点                                                                                                         | Evidence |
| -------------------------------------- | -------------------- | ------------- | -------------------------------------------------------------------------------------------------------------- | -------- |
| `vitest.config.mts`                    | Vitest               | 单元/集成(主) | jsdom+node 双 project;`pool:forks` `maxWorkers:4`(避免原生插件多线程 SEGFAULT);超时 10s;coverage v8 **无阈值** | `:42-87` |
| `vitest.integration.config.ts`         | Vitest               | 真实 agent 流 | jsdom;3min 超时;`singleFork:true` 串行;`bail:1`                                                                | `:18-33` |
| `vitest.e2e.config.ts`                 | Vitest               | WS/真服流     | node;30s 超时                                                                                                  | `:1-31`  |
| `vitest.performance.config.mts`        | Vitest               | 性能/压力     | jsdom;30s;React plugin                                                                                         | `:1-27`  |
| `playwright.config.ts`                 | Playwright(electron) | 桌面 E2E      | `workers:1` 串行;`fullyParallel:false`;CI 2 次重试;trace on retry                                              | `:16-50` |
| `packages/plugin-sdk/vitest.config.ts` | Vitest               | SDK           | 独立;含 typecheck(test-d)                                                                                      | `:1-23`  |

[Evidence] **Vitest 用 `pool:forks` 而非 threads**:threads 快约 19% 但在 node-pty/better-sqlite3 多 worker 下 SEGFAULT。
证据:`vitest.config.mts:42-43` 注释。

## 2. 测试结构(`src/__tests__/`,1404+ 文件)

| 子目录         | 文件数(约) | 用途                                               |
| -------------- | ---------- | -------------------------------------------------- |
| `main/`        | 349+       | 主进程(cue/group-chat/plugins/process-manager/ipc) |
| `renderer/`    | 394+       | 渲染组件(@testing-library/react)                   |
| `shared/`      | 100+       | `src/shared` 纯逻辑                                |
| `cli/`         | 56         | CLI 命令+服务                                      |
| `integration/` | 9          | 真实 agent(group-chat/provider/symphony)           |
| `e2e/`         | 2          | Vitest e2E(CopilotProcessManager/WebServerSync)    |
| `performance/` | 5          | AutoRun/ThinkingStream 压力                        |
| `maestro-p/`   | 8          | maestro-p                                          |
| `web/`         | 26         | PWA hooks/components                               |
| `web-desktop/` | 2          | 浏览器 shim                                        |

代表文件:Cue 66 文件(`cue-engine.test.ts`、`cue-concurrency.test.ts`、`cue-security.test.ts`、`cue-yaml-roundtrip.test.ts`...);group-chat 11;plugins 53(31 main+22 shared);CLI agent-spawner/batch-processor/maestro-client/goal-runner。

[Evidence] 全局 setup `src/__tests__/setup.ts`(776 行):lucide-react Proxy mock、`window.maestro.*` IPC mock(~30 命名空间)、DOM polyfill(ResizeObserver/IntersectionObserver/matchMedia)。helpers 8 文件(mockSession/mockTab/resetStores/listenerLeakAssertions...)。
证据:`src/__tests__/setup.ts:25-776`、`src/__tests__/helpers/`。

## 3. Playwright E2E(`e2e/`)

7 specs + 3 fixtures:

- `autorun-setup/editing/batch/sessions.spec.ts`(Auto Run 向导/编辑/批量/会话)
- `browser-tab.spec.ts`、`bionify-reading-mode.spec.ts`
- `plugins.spec.ts`(1124 行,最重:trust gate、broker 矩阵、FC3 调度、事件投递、sealed 授权持久化、panel 渲染宿主、贡献键绑定、扩展市场)
- fixtures:`electron-app.ts`(746 行,隔离 `MAESTRO_DATA_DIR`、GPU off、`MAESTRO_E2E_TEST`)、`plugin-harness.ts`、`plugin-signing.ts`
- 需预构建(`build:main && build:renderer`)。
  证据:`e2e/*.spec.ts`、`e2e/fixtures/electron-app.ts:74-85`。

## 4. CI 工作流

### `ci.yml`(38 行,唯一测试门禁)

- 触发:PR→main/rc/\*-RC;push→main/rc。
- Job `lint-and-format`(ubuntu):`npm ci` → `prettier --check .` → `eslint src/` → `npm run lint`(三 tsconfig 类型检查)。
- Job `test`:矩阵 `os:[ubuntu-latest, windows-latest]`,`fail-fast:false`,`npm ci` → `npm run test`(`vitest run`,8GB heap)。
- **不包含**:e2e、integration、performance、coverage、plugin-sdk、maestro-p、web-desktop;**无 macOS 测试腿**。

### `release.yml`(673 行)

- 触发:semver tag `v*` / date tag `20*` / dispatch。
- 矩阵:macos-15(固定,非 latest,因 macOS 26 签名 bug)、ubuntu-latest(linux x64)、ubuntu-24.04-arm(arm64)、windows-latest。
- Job:build(每 OS 打包+原生模块架构校验)→ release(GitHub Release + Discord)→ sync-docs(同步 `docs/releases.md`)。
- 防护:每架构独立 npm 缓存(防 ARM/x64 prebuild 污染,#116)、原生模块架构校验脚本。

### `plugin-sdk-publish.yml`(40 行)

- `plugin-sdk-v*` tag / dispatch;bun install+build+vitest+`npm publish`。

### `stale.yml`(96 行)

- 每日 cron stale bot;60 天 stale + 14 天 close;豁免标签。

证据:`.github/workflows/*.yml`。

## 5. package.json 脚本 vs CI

| 脚本               | 命令                      | CI 执行?                          |
| ------------------ | ------------------------- | --------------------------------- |
| `test`             | `vitest run`(8GB)         | **是**(ubuntu+windows)            |
| `lint`             | 三 tsconfig tsc           | **是**(lint job)                  |
| `lint:eslint`      | `eslint src/`             | **是**(CI 直跑 `npx eslint src/`) |
| format             | `prettier --check .`      | **是**(CI 直跑)                   |
| `test:coverage`    | `vitest run --coverage`   | 否(手动;无阈值)                   |
| `test:e2e`         | build + playwright        | 否(手动,需构建)                   |
| `test:integration` | vitest integration config | 否(手动,需真实 agent)             |
| `test:performance` | vitest performance config | 否(手动)                          |
| `validate:push`    | format+lint+eslint+test   | 否(由 pre-push hook 触发)         |

## 6. 覆盖率

[Evidence] provider `v8`(`@vitest/coverage-v8`),reporter text/json/html,目录 `./coverage`,include `src/**`,exclude tests/`.d.ts`/`preload.ts`。**无 thresholds** —— 覆盖率仅信息性,CI 不收集。
证据:`vitest.config.mts:75-87`;全仓 grep 无 `thresholds`。

## 7. Husky / Git hooks

[Evidence] Husky 9,`prepare`→`setup-git-hooks.mjs` 设 `core.hooksPath .husky`。

- `pre-commit` / `pre-merge-commit`:仅 `lint-staged`(prettier+eslint fix)。
- `pre-push`:删除-only 推送跳过;否则 `npm run validate:push`(format+lint+eslint+test)—— **CI 外唯一完整门禁**。
- `lint-staged`(`package.json:428-435`):prettier --write;JS/TS 额外 eslint --fix。
  证据:`.husky/pre-push`、`package.json:52,428-435`。

## 8. EXIST vs CONFIGURED vs CI-EXECUTED 矩阵

| 能力            | 文件存在         | 命令配置           | CI 执行                       |
| --------------- | ---------------- | ------------------ | ----------------------------- |
| 单元/集成       | ~1400+           | `test`             | **是**(ubuntu+windows)        |
| 真实 agent 集成 | 9                | `test:integration` | 否                            |
| Vitest e2E      | 2                | vitest.e2e.config  | 否                            |
| Playwright e2E  | 7+3              | `test:e2e`         | 否                            |
| 性能            | 5                | `test:performance` | 否                            |
| plugin-sdk      | 3                | 自身 vitest        | 仅 plugin-sdk tag             |
| 覆盖率          | v8 配置          | `test:coverage`    | 否(无阈值)                    |
| macOS 测试腿    | 多测试 mac-aware | n/a                | **否**(矩阵仅 ubuntu+windows) |
| 类型检查        | 全 tsconfig      | `lint`             | 是(ubuntu)                    |
| 格式/Lint       | 全仓/src         | CI 直跑            | 是(ubuntu)                    |

## 9. 测试覆盖矩阵(抽样,按模块)

| 模块           | 单元        | 集成               | E2E                         | CI 执行       | 主要缺口                                          |
| -------------- | ----------- | ------------------ | --------------------------- | ------------- | ------------------------------------------------- |
| Cue            | 66 文件(强) | engine-integration | -                           | 是            | 长链/5000 字截断的端到端(见 review-workflow 文档) |
| Group Chat     | 11          | integration        | -                           | 是            | 无 headless 自动化(设计如此)                      |
| Plugins        | 53(强)      | -                  | plugins.spec.ts(深)         | 单元是;E2E 否 | E2E 不在 CI                                       |
| ProcessManager | 有          | -                  | -                           | 是            | 跨平台 PTY 行为(macOS 缺 CI 腿)                   |
| CLI            | 56          | -                  | -                           | 是            | headless 真实 agent                               |
| Renderer       | 394         | -                  | autorun/browser/bionify E2E | 单元是        | E2E 不在 CI                                       |

---

## 已确认事实

- Vitest(jsdom/node 双 project)+ Playwright(7 specs)+ 4 专项配置;~1400+ 测试文件。
- CI 仅 ci.yml 跑 test+lint/format 于 ubuntu+windows;无 e2e/integration/performance/coverage;无 macOS 测试腿。
- pre-push 是 CI 外唯一完整门禁;coverage 无阈值。
- release.yml 有跨平台打包 + 原生模块架构校验(含 ARM64)。

## 合理推断

- 双 project + forks pool 是为绕开原生插件 SEGFAULT 的有意权衡。
- e2e/integration 不进 CI 是因依赖真实 agent/构建产物与时间成本(bail:1、3min 超时佐证)。

## Unknown 与待验证事项

- 实际通过率与覆盖率数值(未运行)。
- macOS 测试腿缺失是否导致平台特定 bug 漏网(需历史 issue 数据)。
- 各测试是否真正覆盖主要失败路径(仅抽样,未逐断言审计)。

## 批判性评估

- CI 门禁偏窄(仅单元+静态检查),e2e/plugins 深度测试不在 CI —— 质量依赖 pre-push 本地执行,存在「未 push 即合」的绕过面。
- 无 macOS 测试腿与无覆盖率阈值是两大可观测性缺口(CLAUDE.md 明确要求两腿皆绿才可合并,但 CI 实际仅两腿且无 mac 测试腿,存在文档与 CI 的轻微张力)。
- 「测试文件多」不等于「覆盖好」:1404 文件中含大量 renderer 组件测试与 PWA 遗留测试,质量需逐模块看断言。

## 建设性改善建议

[Recommendation] 在 CI 增加 macOS 测试腿(或至少 lint 腿),与本机平台矩阵对齐,满足 CLAUDE.md「两腿皆绿」的实际意图。
优先级:高;实施难度:中(成本:macOS 分钟数);依据:`ci.yml:23-37` 矩阵 vs `CLAUDE.md` Validate Before Push 的已验证偏差。

[Recommendation] 为核心 e2e(`plugins.spec.ts` 关键信任路径)在 CI 增加 nightly 或手动 dispatch 的 e2E job,避免深度测试长期不在任何自动化路径上。
优先级:中;实施难度:中;依据:`e2e/plugins.spec.ts`(1124 行)与 ci.yml 不含 e2e 的已验证现状。

[Recommendation] 设定最低覆盖率阈值(渐进,先信息性后强制),建立回归基线。
优先级:低;实施难度:低;依据:`vitest.config.mts:75-87` 无 thresholds。

## 主要证据索引

- `vitest.config.mts:42-87`、`vitest.integration.config.ts:18-33`、`vitest.e2e.config.ts:1-31`、`vitest.performance.config.mts:1-27`
- `playwright.config.ts:16-50`
- `packages/plugin-sdk/vitest.config.ts:1-23`
- `src/__tests__/setup.ts:25-776`、`src/__tests__/helpers/`
- `e2e/*.spec.ts`、`e2e/fixtures/electron-app.ts:74-85`
- `.github/workflows/ci.yml:3-37`、`release.yml`、`plugin-sdk-publish.yml`、`stale.yml`
- `.husky/pre-push`、`.husky/pre-commit`
- `package.json:46-61,428-435`

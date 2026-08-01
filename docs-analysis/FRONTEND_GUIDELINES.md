# FRONTEND_GUIDELINES.md - 前端开发指南

> 区分:① 源码中明确存在并执行的规范;② 可由重复模式推断的约定;③ 仓库无证据的规范;④ 推荐未来采用的规范。

## 分析快照

- 分支:`docs/multi-agent-review-loop-analysis`
- HEAD:`9affd8cfb27c8cfe39817ae92879445dc0632b40`
- 工作区状态:任务前 clean;本文档为本次任务新增
- 子模块状态:无
- 分析范围:`src/renderer/`(App.tsx、main.tsx、stores、hooks、components、services、contexts)、`src/shared/themes.ts`、`tailwind.config.mjs`、`src/renderer/index.css`
- 未覆盖范围:组件逐个审计(135+ 组件仅抽样)

## 证据分类

- Evidence:源码直接证明的规范
- Inference:由重复模式推断
- Unknown:仓库无证据
- Recommendation:建议(非现状)

## 核心结论

[Evidence] 前端为 React 18 单页应用,无路由器。状态管理统一用 **Zustand(33 store)**,无 Redux/MobX/CSS-in-JS。样式为 **Tailwind + 内联 `style={theme.colors.*}`**,无统一 Design Token。入口链 `index.html → main.tsx → App.tsx(~3600 行)→ AppShell`。
证据:`src/renderer/main.tsx:109-135`、`src/renderer/stores/`(33 文件)、`tailwind.config.mjs`、`src/shared/themes.ts:398`。

---

## 1. 前端入口 / 页面 / 路由 / 布局

- 入口:`src/renderer/main.tsx`(`ReactDOM root`,两种挂载:Cadenza HUD `?cadenzaHud` 与正常),挂载 `<ErrorBoundary><LayerStackProvider><WizardProvider><MaestroConsole/></...>`。
- [Evidence] **无路由器**。单页单窗口,视图由 `Session` 字段分派(`MainPanelContent.tsx:631-1030`)。
- 布局:`AppShell.tsx`(277 行)空间组合 —— 标题栏 / SessionList(Left)/ MainPanel(中)/ RightPanel(右)/ 叠加层(Toast、CenterFlash、ThoughtStream、PermissionPrompt、Cadenza、Movement)。

## 2. 组件层级与目录组织(① 现存规范)

- `components/`:135+ 顶层 `.tsx` + 子目录(MainPanel/、Settings/、Markdown/、Wizard/、InlineWizard/、CueModal/、CuePipelineEditor/、SessionList/、GroupChat\*)。
- Modal 统一分区:`AppModals/`(8 文件按关注点切分)+ `AppStandaloneModals`、`AppOverlays`。
- [Evidence] Modal 注册走 `modalStore` + `modalPriorities.ts`(z-index/escape 优先级表),消费走 `useModalLayer()`(替代手写 `registerLayer`)。
  证据:`src/renderer/stores/modalStore.ts`、`src/renderer/hooks/ui/useModalLayer.ts`、`CLAUDE.md` 规范。
- [Evidence] 大量规范在 `CLAUDE.md` 以「Commonly-reimplemented functions」强制(如 `useEventListener`、`useFocusAfterRender`、`useDebouncedValue`、`<Markdown preset>`),并有 `docs/agent-guides/` 去重跟踪。
  证据:`CLAUDE.md`「Before Writing New Code」表 + `DEDUP-TRACKER.md`。

## 3. 状态管理(① 现存规范)

- 每个关注点一个 Zustand store,文件即 store,带 header 注释(「replaces old context」「`useXStore.getState()` 可在 React 外用」)。
- 选择器细粒度订阅:`selectActiveSession` 等。
- [Evidence] **等值选择器是性能关键**:`sessionEquality.ts` 提供 `sidebarSessionEquality/gitPollSessionEquality/projectRootSessionEquality`,经 `useStoreWithEqualityFn` 使用,避免流式日志/token 更新每 200ms 重渲染整壳。
  证据:`src/renderer/App.tsx:523`、`src/renderer/stores/sessionStore.ts:1-13` 注释。
- 持久化:settings 启动批量 `loadAllSettings`(`settingsStore.ts:2300`),setter 即时写穿 `window.maestro.settings.set`;session 经 `window.maestro.sessions.*` 持久化;多数 store 为内存态。

## 4. 数据获取 / IPC 封装(① 现存规范)

- IPC 契约:`window.maestro.*`(单命名空间,~50 子命名空间),类型在 `global.d.ts`(~3960 行,`MaestroAPI` 起 line 191)。
- 服务封装:`src/renderer/services/`(17 文件),经 `ipcWrapper.ts` 的 `createIpcMethod`(两种错误模式:`defaultValue` 吞掉+Sentry 或 `rethrow`),带 `ipcCache` TTL。
- 事件订阅:`on*` 方法返回 unsubscribe,集中在 `useAgentListeners`(`hooks/agent/`)。

## 5. 样式系统 / Design Token / 主题(① + ③)

- [Evidence] Tailwind 配置极简(`tailwind.config.mjs` 12 行),**无 Tailwind 主题 token**;颜色全部经内联 `style={theme.colors.*}`。
- 主题真相源:`src/shared/themes.ts`(877 行,`THEMES` 表 + 每 theme ANSI 16 色),`renderer/constants/themes.ts` 为纯 re-export。`useResolvedTheme` 解析(含插件贡献主题,dracula 兜底)。
- CSS 变量:`useThemeStyles` 运行时注入 `--accent-color/--highlight-color/--scrollbar-*`。
- [Evidence] **当前仓库未发现统一的 Design Token 体系**(无 spacing/radius/font-scale token 表)。仅有 `colorblindPalettes.ts`(可访问色板)。
  证据:`tailwind.config.mjs`、`src/shared/themes.ts`。

## 6. 响应式 / 移动端 / 可访问性 / i18n

- 移动适配:`isCoarsePointer()`/`useLongPress`/触摸原语(`src/renderer/utils/touch.ts`);AppShell 有移动抽屉/边缘滑动;`html[data-runtime='web-desktop']` 选择器门控手机规则。
- [Evidence] 可访问性:Wizard 有 `ScreenReaderAnnouncement`、aria、键盘导航;E2E 含 accessibility 断言(`e2e/autorun-setup.spec.ts`)。
- [Unknown] **未发现国际化(i18n)框架** —— UI 文案为硬编码英文(部分用户提示词为中文)。无 i18next/react-intl 依赖证据。

## 7. 图标 / 静态资源

- 图标:lucide-react(测试中以 Proxy 自动 mock,`setup.ts:25-77`)。
- 图片:`maestro-image://` 协议 + `session-images/`;SVG 导出统一 `saveSvgToProject()`。

## 8. 终端

- `XTerminal.tsx`(xterm + addon-fit/search/unicode11,自定义按键路由 `XtermKeyAction`)、`TerminalView.tsx`(React 管理器,keep-alive overlay 以 `visibility:hidden` 保留 WebGL canvas)。

## 9. Markdown 渲染(① 现存规范, exemplary)

- 统一 `<Markdown preset="chat|document|wizard-bubble|release-notes">`(`Markdown.tsx`),管线 `preprocessMarkdown → buildMarkdownPlugins(remark/rehype)→ ReactMarkdown + 每 preset 组件映射`。HAST 级 `rehype-sanitize`。旧 `MarkdownRenderer` 为薄包装。

## 10. 前端测试 / 构建

- 测试:Vitest jsdom 项目 + `@testing-library/react`;`src/__tests__/setup.ts`(776 行)全局 mock(lucide、`window.maestro.*`、DOM polyfill)。
- 构建:Vite(`vite.config.mts`)。详见 `测试与CI.md`、`TECH_STACK.md`。

## 11. 桌面 / 浏览器 / 移动差异

- 桌面渲染进程与 web-desktop 共用同一 React 树;web-desktop 经 `electron-shim.ts` 把 `electron` 别名为 WebSocket 桥实现,`window.maestro` 由同一 preload 工厂填充。
- mobile:web-desktop 的 PWA + touch hooks;`src/web/` 遗留移动 PWA(死代码)。

## 12. 前端安全边界

- CSP(`index.html:17-19`,含 `wasm-unsafe-eval` for Shiki)。
- Markdown HTML 经 `rehype-sanitize`;`allowRawHtml` 受控。
- IPC 经 contextBridge 隔离,无 Node 直暴露。

## 13. UI 技术债务

- `App.tsx` ~3600 行 god-component(正以 hook/store 抽取渐进拆分,见 Tier 迁移注释)。
- `src/web/` 遗留死代码与 `web-desktop` 并存。
- 33 store 的跨 store 协调风险(如 session 删除须同时清理 contextTimeline/fileExplorer,`sessionStore.ts:195-213`)。

---

## 已确认事实

- Zustand 33 store + Tailwind 内联主题色 + 无路由 + 统一 Markdown 渲染。
- 等值选择器是流式更新的性能护栏。
- 无统一 Design Token、无 i18n 框架。

## 合理推断

- 「文件即 store + header 注释」是团队强约束(CLAUDE.md 去重表佐证一致性文化)。

## Unknown 与待验证事项

- 是否存在未文档化的 design token 约定(仅看代码未发现)。
- 移动端实际手势体验(需运行 web-desktop)。

## 批判性评估

- 缺 Design Token 与 i18n 是两大空缺;主题色走内联 style 在大型组件树中维护成本高。
- `App.tsx` god-component 是最大可维护性风险。

## 建设性改善建议

[Recommendation] 引入 Design Token 层(spacing/radius/font-scale/color),经 CSS 变量 + Tailwind theme 扩展,逐步替换散落的 magic value。
优先级:中;实施难度:中;依据:`tailwind.config.mjs` 无 theme token、`themes.ts` 仅颜色的已验证现状。

[Recommendation] 引入 i18n 框架(如 i18next)抽取硬编码文案,为多语言用户与提示词本地化奠基。
优先级:低;实施难度:高;依据:全仓无 i18n 依赖、UI 文案硬编码。

## 主要证据索引

- `src/renderer/main.tsx:109-135`、`src/renderer/App.tsx`(整文件)、`:523`
- `src/renderer/components/AppShell.tsx:206-274`
- `src/renderer/components/MainPanel/MainPanelContent.tsx:631-1030`
- `src/renderer/stores/*.ts`(33 文件)、`sessionStore.ts:1-13,195-213`、`settingsStore.ts:723,2300`、`modalStore.ts:1-15`
- `src/renderer/services/ipcWrapper.ts:91-186`
- `src/renderer/global.d.ts:191`
- `tailwind.config.mjs`、`src/shared/themes.ts:398`、`src/renderer/hooks/ui/useThemeStyles.ts`
- `src/renderer/components/Markdown/Markdown.tsx`
- `CLAUDE.md`(Before Writing New Code 表)、`docs/agent-guides/DEDUP-TRACKER.md`

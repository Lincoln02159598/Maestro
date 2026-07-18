# 基于 Maestro 的多 Agent 协作复核工作流:可行性分析与实现方案

> 分析对象:能否在 Maestro 中实现一条「多 agent 协作工作流」,例如
> **Claude Code(cc)写代码 -> Codex 复核 -> cc 修改 -> ... 循环 3 轮 -> OpenCode 跑自动化测试**。
>
> 结论先行:**今天就能实现,几乎不需要二次开发。** Maestro 已内置三套可支撑该场景的机制,
> 本文档给出可行性评估、方案对比,以及一份可直接部署的 Cue 配置(`agent.completed` 链式流水线)。
>
> 分析日期:2026-07。基于当前 `main` 分支源码与 `docs/agent-guides/` 文档核实。

---

## 1. 需求拆解

目标工作流:

```
implement(cc) ─► review-1(codex) ─► revise-1(cc)
             ─► review-2(codex) ─► revise-2(cc)
             ─► review-3(codex) ─► revise-3(cc)
             ─► run-tests(opencode)
```

关键特征:

1. **多 agent 协作**:不同步骤由不同 agent 类型执行(cc / codex / opencode)。
2. **顺序链路 + 循环**:A 完成后触发 B,B 完成后触发 A,固定轮数。
3. **上下文交接**:B 需要读到 A 的产物(A 改的代码)与意见(A 的总结)。
4. **无人值守**:能自动跑完,而不是每跳都需要人点一下。
5. **末端动作**:由第三个 agent 执行自动化测试。

> 注:需求原文中的 "openclaw" 对应 Maestro 已支持的 **opencode**。

---

## 2. 现有能力盘点(三条可行路径)

通过对 `src/main/cue/`、`src/main/group-chat/`、`src/shared/pianola/`、`src/cli/` 的源码与文档核实,Maestro 已经具备以下能力,按「开箱即用程度」从高到低排列。

### 2.1 路径 A:Cue 自动化引擎(最贴合,推荐)

Cue 是事件驱动自动化引擎,核心设计就是**「一个 agent 完成后自动触发下一个 agent,并把上一个 agent 的输出注入下一个的 prompt」**,与需求几乎一一对应。

| 需求点 | Cue 对应能力 | 源码位置 |
| --- | --- | --- |
| A 完成后触发 B | `agent.completed` 事件 + `source_session` 字段 | `src/main/cue/cue-completion-service.ts` |
| 把 A 的输出传给 B | `{{CUE_SOURCE_OUTPUT}}` 模板变量(下游 prompt 自动注入上游 stdout) | `src/shared/templateVariables.ts` |
| A->B->A->B...->C 多跳链 | 链式传播,深度上限 `MAX_CHAIN_DEPTH = 10` | `src/main/cue/cue-engine.ts` |
| 不同 agent 类型参与 | 每条订阅可指定任意 `agent_id` | `src/shared/cue/contracts.ts` |
| 自动 / 无人值守触发 | `cli.trigger`、`github.pull_request`、`file.changed`、定时器等 10 种触发源 | `src/cli/commands/cue-trigger.ts` |
| 可视化编排 | `CuePipelineEditor`(React Flow 拖拽画布,双向同步 YAML) | `src/renderer/components/CuePipelineEditor/` |
| 声明式配置 | `.maestro/cue.yaml` | `CUE_CONFIG_PATH` |

**结论**:7 条订阅串成一条链即可完整实现该工作流,0 行源码改动。

### 2.2 路径 B:Pianola 多 agent 任务 DAG 编排器

代码库还内置了一个更重的编排器 **Pianola**(`src/shared/pianola/`、`src/cli/commands/pianola-orchestrate.ts`),是一个完整的任务依赖图(DAG)引擎,其 reactive fix loop 里硬编码了 `MAX_FIX_ATTEMPTS = 3` —— 与需求的「循环 3 次」高度吻合。

- **优点**:DAG 依赖、并发控制、失败级联、`needs_review` 状态机,工程化程度最高。
- **缺点**:
  - 需开启 Encore 特性开关(`encoreFeatures.pianola` + `autopilot`)。
  - **必须桌面 app 在运行时**才能用(走 WebSocket dispatch)。
  - 默认 fix prompt 是泛化的("处理遗留问题"),不会把 codex 的具体复核原文喂回 cc,需要定制 `dispatchFix`。
  - `requestMerge` 路径目前是 stub(不实际合并)。
- **工作量**:配置 + 定制 fix prompt,约 1-2 天。不如 Cue 切合「纯自动化流水线」的诉求。

### 2.3 路径 C:`maestro-cli send` + shell 脚本(最轻量)

CLI 已有 `maestro-cli send <agent-id> <message> [--session <id>]`,直接 spawn agent 子进程、返回 JSON(`response` / `sessionId` / `usage`),无需桌面 app。所有 agent 都已声明 headless batch 模式参数:

| Agent | headless 调用 |
| --- | --- |
| Claude Code | `claude --print --verbose --output-format stream-json --dangerously-skip-permissions` |
| Codex | `codex exec --json --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check` |
| OpenCode | `opencode run --format json` |

一个 bash 循环就能串起来(send 给 cc -> 取 `.response` -> 拼进 codex 的 review prompt -> 取复核 -> 拼进 cc 的 revise prompt -> 循环 3 次 -> send 给 opencode 跑测试)。0 源码改动,代价是编排/重试/失败处理/历史记录都得自己写。

### 2.4 不推荐的路径

- **Group Chat**:确实支持多 agent 协作(主持人 + 参与者 hub-and-spoke),但**只能交互式驱动** —— 必须有用户发首条消息、主持人靠 `@mention` 路由,没有 headless / 事件触发入口,无法无人值守跑 3 轮。适合「人盯着、随时插话」的场景。
- **Send-to-Agent / Director's Notes**:都是手动一次性操作或只读元视图,不适用于自动循环。

### 2.5 总体判断

| 问题 | 回答 |
| --- | --- |
| 今天能否实现(含变通)? | **能。** Cue 引擎开箱即用,是最贴合的方案。 |
| 需要多大二次开发? | **几乎为零** —— 核心是写 `.maestro/cue.yaml` + prompt,不改源码。 |
| 若要更精细能力(可配置循环次数、跨轮记忆、超长复核交接) | 约半天到 2 天的源码微调,详见第 5 节。 |

**这不是「能不能做」的问题,而是「用哪条现成路径做」的问题。推荐用 Cue。**

---

## 3. 选定方案:Cue 链式流水线

### 3.1 设计要点

1. **代码交接走文件系统,意见交接走模板变量**。
   cc / codex / opencode 三个 agent 指向**同一个项目目录**,cc 改的文件 codex 直接能读;codex 的复核文本通过 `{{CUE_SOURCE_OUTPUT}}` 注入 cc 的 prompt。这是 Cue 的惯用法,也绕开了「把整份代码塞进 prompt」的体积问题。

2. **每条链都带 `source_sub`**。
   cc 同时承担 `implement` 和 `revise-1/2/3`,它每次完成都发出 `agent.completed`。靠 `source_sub` 精确匹配上一跳(如 `review-2` 只认 `revise-1` 触发的完成),否则会自激 / 串扰。这是 loop 能正确跑起来的核心。

3. **`settings.owner_agent_id` + 每条订阅显式 `agent_id`**。
   三个 agent 共享同一项目根目录时,不 pin owner 会导致非 owner 的订阅被静默抑制(Cue 头号 gotcha,见 `CLAUDE-CUE.md` Top gotchas #1)。owner 是「调度者」,`agent_id` 是「执行者」。

4. **5000 字截断的应对(生产版 v2 的核心改进)**。
   `{{CUE_SOURCE_OUTPUT}}` 在 `SOURCE_OUTPUT_MAX_CHARS = 5000`(`src/main/cue/cue-output-filter.ts`)处截断。复核意见可能很长,因此 v2 让 codex 把完整复核写到 `.maestro/reviews/review-N.md`,stdout 只回传短摘要 + "详见该文件";cc 读文件拿完整意见。三轮复核都留盘,事后可回溯。

### 3.2 关键参数与约束

| 参数 / 约束 | 值 | 含义 |
| --- | --- | --- |
| `MAX_CHAIN_DEPTH` | 10 | 链最大深度;本流水线最深 7,安全。 |
| `SOURCE_OUTPUT_MAX_CHARS` | 5000 | 注入下游 prompt 的单源输出截断长度。 |
| `max_concurrent` | 1(默认) | 每会话并发,线性链串行即可,且防同一触发重叠。 |
| `timeout_minutes` | 30(默认) | 单次运行超时。 |
| batch 模式 | 每跳全新 context | 不保留对话记忆(跨轮记忆见第 5 节)。 |

### 3.3 链路深度核算

| 跳 | 订阅名 | 执行者 | 上游 | 深度 |
| --- | --- | --- | --- | --- |
| 0 | `implement` | cc | `cli.trigger`(根) | 0 |
| 1 | `review-1` | codex | implement | 1 |
| 2 | `revise-1` | cc | review-1 | 2 |
| 3 | `review-2` | codex | revise-1 | 3 |
| 4 | `revise-2` | cc | review-2 | 4 |
| 5 | `review-3` | codex | revise-2 | 5 |
| 6 | `revise-3` | cc | review-3 | 6 |
| 7 | `run-tests` | opencode | revise-3 | 7 |

---

## 4. 部署与使用

### 4.1 配置文件

完整可部署配置见同目录或仓库根的 `cue-review-loop.example.yaml`(生产版 v2,文件化复核交接)。核心结构:

```yaml
settings:
  timeout_minutes: 30
  max_concurrent: 1
  queue_size: 64
  owner_agent_id: AGENT_CC        # 三个 agent 共享 projectRoot 时必须 pin

subscriptions:
  - name: implement               # 根触发,cc 实现
    event: cli.trigger
    agent_id: AGENT_CC
    prompt: |
      ... {{CUE_CLI_PROMPT}} ...

  - name: review-1                # codex 复核,完整意见写 .maestro/reviews/review-1.md
    event: agent.completed
    source_session: AGENT_CC
    source_sub: implement
    agent_id: AGENT_CODEX
    prompt: |
      ... {{CUE_SOURCE_OUTPUT}} ... 写文件 review-1.md,stdout 只回短摘要 ...

  - name: revise-1                # cc 读 review-1.md 后修改
    event: agent.completed
    source_session: AGENT_CODEX
    source_sub: review-1
    agent_id: AGENT_CC
    prompt: |
      ... {{CUE_SOURCE_OUTPUT}} ... 读 .maestro/reviews/review-1.md ...

  # ... review-2 / revise-2 / review-3 / revise-3 同模式 ...

  - name: run-tests               # opencode 跑自动化测试
    event: agent.completed
    source_session: AGENT_CC
    source_sub: revise-3
    agent_id: AGENT_OPEN
    prompt: |
      ... {{CUE_SOURCE_OUTPUT}} ... 运行测试并报告 PASS/FAIL ...
```

### 4.2 使用步骤

1. 把三个占位符 `AGENT_CC` / `AGENT_CODEX` / `AGENT_OPEN` 换成真实值(`maestro-cli list agents` 查看 ID,推荐 UUID)。
2. 复制到共享项目根目录:`cp cue-review-loop.example.yaml <项目>/.maestro/cue.yaml`。
3. 确认 cc / codex / opencode 三个 agent 的 cwd 都指向该共享项目根目录。
4. 触发(需桌面 app 在运行,trigger 走 WebSocket):
   ```bash
   maestro-cli cue trigger implement --prompt "实现用户登录的 JWT 校验中间件"
   ```
   也可把 `implement` 的 `event` 改成 `github.pull_request`,让流水线在每个新 PR 上自动跑。

### 4.3 v2 模式的已知脆弱点

codex 必须可靠地完成「写文件 + 输出短摘要」两件事。若某个 agent 偶尔只输出意见、不写文件,cc 读不到文件会卡住。这是文件化交接唯一的脆弱点,需在真实项目里跑一两遍校准 prompt。对有文件写能力的 agent(cc / codex)这是常规操作,但 prompt 里务必把两步指令写得很显式。

---

## 5. 后续可选增强(涉及源码改动,需先定方向)

### 5.1 跨轮记忆(约半天)

**问题**:Cue 每跳以 batch 模式起全新 context,codex 在第 2、3 轮不「记得」自己上一轮提过什么。

**改法**:在 Cue spawn 链路里透传 `--resume <session_id>`(各 agent 的 resume 参数已在 `src/main/agents/definitions.ts` 定义,如 Claude `:204`、Codex `:273`)。改动点在 `src/main/cue/cue-spawn-builder.ts` 附近,需顺带处理「哪一跳该 resume、哪一跳该开新 context」(例如:同一 agent 在不同轮之间 resume,但 implement 与 revise 之间不该 resume)。

### 5.2 可配置循环次数

把「3 轮」从写死的 6 条订阅变成参数化。两条路线:

- **轻量(纯配置,不动引擎)**:在链里插一个 `action: command`(`mode: shell`)节点做计数器,达到 N 次后让条件分支走向 `run-tests`,否则回环。但 Cue 没有原生 if/else,得用 shell 退出码 + 多条订阅拼出来,可读性一般。
- **重量(改用 Pianola,约 1-2 天)**:Pianola 内置 `MAX_FIX_ATTEMPTS = 3` 的 reactive fix loop,天然是「复核不通过就回修、最多 N 次」。但要开 Encore 开关、依赖桌面 app 运行,且默认 fix prompt 泛化需要定制。

### 5.3 触发源扩展

把入口 `implement` 的 `event` 从 `cli.trigger` 换成:
- `github.pull_request` —— 每个 PR 自动跑复核流水线;
- `file.changed` —— 指定路径(如 `src/**`)变更时自动跑;
- `task.pending` —— 当某 markdown 出现未勾选任务时跑。

均为纯配置改动。

---

## 6. 关键源码索引(便于核实)

| 关注点 | 文件 |
| --- | --- |
| 订阅 schema(字段全集) | `src/shared/cue/contracts.ts`(`CueSubscription`) |
| `agent.completed` 链式分发 | `src/main/cue/cue-completion-service.ts` |
| 模板变量(`CUE_*`) | `src/shared/templateVariables.ts` |
| 单跳执行(prompt 替换、spawn) | `src/main/cue/cue-executor.ts` |
| spawn 参数构建(batch 模式、SSH) | `src/main/cue/cue-spawn-builder.ts` |
| 输出过滤与 5000 字截断 | `src/main/cue/cue-output-filter.ts` |
| 模板上下文构建(按事件类型) | `src/main/cue/cue-template-context-builder.ts` |
| 引擎协调 + 深度上限 | `src/main/cue/cue-engine.ts` |
| 架构 why / gotchas | `CLAUDE-CUE.md` |
| 模块级参考 + YAML 示例 | `docs/agent-guides/CUE-PIPELINE.md` |
| 用户向 Cue 完整规格 | `src/prompts/_maestro-cue.md` |
| Pianola DAG 引擎 | `src/shared/pianola/pianola-orchestrator.ts`、`pianola-tasks.ts` |
| Pianola CLI 入口 | `src/cli/commands/pianola-orchestrate.ts` |
| headless agent spawn | `src/cli/services/agent-spawner.ts` |
| 各 agent headless 参数 | `src/main/agents/definitions.ts` |
| Group Chat 架构 | `docs/agent-guides/GROUP-CHAT.md` |

---

## 7. 结论

- 该多 agent 协作复核工作流**今天即可实现**,无需二次开发,推荐采用 **Cue 链式流水线**(路径 A)。
- 已产出可部署配置 `cue-review-loop.example.yaml`(生产版 v2,文件化复核交接,规避 5000 字截断)。
- 三条可选增强(跨轮记忆、可配置循环次数、触发源扩展)按需推进,其中触发源扩展为纯配置;跨轮记忆与可配置循环次数涉及源码微调,建议在真实项目验证基础流水线后再决定是否投入。

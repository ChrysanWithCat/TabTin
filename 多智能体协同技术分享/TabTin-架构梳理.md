# TabTin 架构梳理

> 基于源码阅读整理，目的是给内部技术分享提供选材。不是官方架构文档，是我作为外部读者从仓库里读出来的结构。
> 仓库：`E:\GitHub\TabTin` ｜ 版本：Public Preview ｜ 协议：AGPL-3.0

---

## 0. 一句话定位

TabTin 不是一个 AI 聊天壳，也不是一个 coding agent，它是一个**把"人 + Agent"当成一等公民来设计的协作平台**：云端管组织/会话/权限，本地跑 Daemon 提供真实执行环境，多端客户端只是壳。它要解决的核心问题是——**一个人用 Agent 做完的活，怎么变成下一个同事的起点**。

---

## 1. 部署形态：三件套

```
┌─────────────────────────────────────────────────────────────┐
│  云端（SaaS / Community Server，Docker Compose）            │
│  Django + Celery + PostgreSQL + Redis + Centrifugo + OSS     │
└───────────────▲─────────────────────────────────────────────┘
                │ WebSocket 长连接（异步协议在 contracts/asyncapi）
                │
┌───────────────┴─────────────────────────────────────────────┐
│  本地 Daemon（Node 常驻进程，跑在用户电脑上）                  │
│  终端 PTY / 文件系统 / 浏览器 / 表格内核 / Shadow Git        │
│  同时对外暴露 MCP Server，可被其他工具调用                     │
└───────────────▲─────────────────────────────────────────────┘
                │ IPC / 本地 HTTP
┌───────────────┴─────────────────────────────────────────────┐
│  客户端                                                      │
│  Electron（主端）/ iOS / Android / Web / AdminDash           │
└──────────────────────────────────────────────────────────────┘
```

关键认知：
- **Agent 真正跑在本地 Daemon 里**，不是云端。云端只做调度、记忆、协作同步、计费。
- 移动端只是"遥控器"——README 明确说移动端不独立执行任务，必须有电脑在线。
- Community Server 可以 `docker compose up` 在本机跑整套后端，数据卷在本地。

---

## 2. 技术栈

| 层 | 技术 |
|---|---|
| 前端 / 桌面 | React 19、TypeScript、Electron、Tailwind、pnpm@9 monorepo |
| 后端 | Python Django + DRF、Celery（异步任务）、PostgreSQL、Redis、Centrifugo（实时推送） |
| 本地执行 | Node.js Daemon、Go 写的 CLI（`tabtin-cli-go`）、Python runtime |
| 协作 | CRDT（`collab-core`，类似 Yjs 的 provider/awareness 模型） |
| 实时通道 | WebSocket Gateway（AsyncAPI 契约） |
| 表格内核 | `table-kernel` + `table-kernel-pglite`（本地用 PGlite/SQLite） |
| 终端 | PTY（`pty-core` / `terminal-core`） |
| 浏览器 | 自有 `browser-core`（不是套 Puppeteer，有 RecordingSession/ResourceTracker） |
| 包规模 | **8 个 app + 90 个 package**；后端 Django 有 **4788 个 .py 文件、36 个业务 app** |

---

## 3. 核心概念模型

来自 `docs/architecture/product-concepts.md`，也是整个系统的领域语言：

| 概念 | 是什么 | 关键边界 |
|---|---|---|
| **Organization** | 组织/租户 | 不同 Organization 默认隔离 |
| **Workspace** | 成员的**私有执行现场** | 每个 Workspace 只有一个执行根；含文件、终端、Skill、Checkpoint |
| **Agent** | AI 参与身份 | 拥有角色/规则/模型/Skill/记忆/执行偏好；**表示"谁参与"，不表示"在哪执行"** |
| **App** | 工作应用 | 文档/表格/终端/浏览器/消息/集成；人和 Agent 操作同一份 App 结果 |
| **Device** | 实际执行环境 | 提供终端/文件/浏览器/设备控制；必须遵守 Organization 与 Workspace 权限 |

这个五件套是理解整个架构的钥匙。两个最容易被忽略的设计：

1. **Agent ≠ Device**：同一个 Agent 可以在不同 Device 上跑。你手机上发起任务，在桌面 Daemon 的 Workspace 里执行。这在现在的 CLI agent（Codex/Claude Code）里是没有的概念。
2. **Workspace 是私有的**：交接任务 ≠ 把你的整个目录共享给同事。双方本地环境独立，交接只带"可共享上下文 + 引用材料"，权限继续生效。

---

## 4. 分层架构：一个请求怎么走完

```
┌──────────────────────────────────────────────────────┐
│ 客户端 UI（Electron / Web / iOS / Android）           │
│  ├─ 聊天界面  ├─ 文档/表格/PPT 编辑器  ├─ Agent 面板 │
└───────────────▲──────────────────────────────────────┘
                │ 共享事件流（CRDT 同步 + Agent 事件）
┌───────────────┴──────────────────────────────────────┐
│ Agent Host（packages/agent-host，182 文件）           │
│ 宿主粘合层：会话状态、HITL 审批、delivery outbox、     │
│  hooks（记忆/规则/上下文/LSP）、TabTin 专属 tools       │
└───────────────▲──────────────────────────────────────┘
                │ 纯引擎接口（GatewayPort / EngineHooks）
┌───────────────┴──────────────────────────────────────┐
│ Agent Runtime（packages/agent-runtime，569 文件）     │
│ 纯 ReAct 引擎：LLM 循环、工具编排、上下文压缩、         │
│ 防失控 guard rails、subagent、会话持久化               │
└───────────────▲──────────────────────────────────────┘
                │ 所有 LLM 调用走 Django LLM Proxy
┌───────────────┴──────────────────────────────────────┐
│ Django 后端（apps/tabtin_django，36 个业务 app）       │
│ 认证/计费/组织/Agent 配置/会话/调度/RAG/集成           │
└───────────────▲──────────────────────────────────────┘
                │
┌───────────────┴──────────────────────────────────────┐
│ 本地 Daemon（apps/tabtin-daemon，122 文件）            │
│ 终端 PTY / 文件 / 浏览器 / 表格内核 / Shadow Git       │
│ 同时作为 MCP Server 对外提供能力                       │
└──────────────────────────────────────────────────────┘
```

---

## 5. Agent Runtime 内部：真正的肉在这

这是整个项目最值得读的部分。从 `src/` 目录结构能看出它是一个**生产级 agent 引擎**，不是 demo：

### 5.1 主循环（`engine/core/`）
- `loop.ts`：ReAct 主循环
- `model-stream.ts`：LLM 流式
- `llm-request-builder.ts` / `llm-call-snapshot.ts`：每次 LLM 调用的快照（用于审计/回放）
- `abort.ts` / `retry-state.ts`：中断与重试

### 5.2 工具系统（`engine/tooling/`）
- `tool-orchestration.ts`：工具编排入口
- `tool-schema-validator.ts` / `tool-schema-helpers.ts`：JSON Schema 校验
- `tool-output-sanitizer.ts` / `tool-output-summary.ts`：工具输出消毒与截断
- `tool-error.ts` / `tool-error-code.ts`：错误分类
- `dynamic-tool-manager.ts`：动态工具生命周期
- `skill-slash.ts`：Skill 斜杠命令

### 5.3 防失控 Guard Rails（`engine/guards/`）
这是大家用 coding agent 都遇到过的痛点，它都有对应机制：
- `tool-repetition-tracker.ts`：同一个工具重复调用监控
- `text-repetition-detector.ts`：文本重复检测
- `iteration-budget.ts`：迭代预算上限
- `message-size-budget.ts`：单条消息大小预算
- `budget-tracker.ts`：token 预算
- `context-overflow-recovery.ts`：上下文溢出恢复

### 5.4 上下文压缩（`compact/`）
不是简单"摘要一下"，是**分层压缩**：
- `compaction-orchestrator.ts`：压缩编排器
- `layered-prune.ts`：分层裁剪
- `time-based-microcompact.ts`：基于时间的微压缩
- `incremental-prompt.ts` / `incremental-user.ts`：增量 prompt
- `subagent-summary.ts`：子代理摘要
- `pressure-router.ts`：压力路由（什么时候该压缩）
- `checkpoint.ts`：压缩前打检查点

### 5.5 子代理（`subagent/`）
- `agent-tool.ts`：把"调子代理"做成一个工具
- `fork-query.ts`：从当前会话分叉出去
- `parent-midflight-injector.ts`：父 agent 可以中途给正在跑的子代理插话
- `completion-envelope.ts`：完成信封
- `model-catalog.ts`：子代理可选模型目录

注意：这是**星型子代理**（父调子），不是 PPT 里讲的"网状多智能体协同"。TabTin 自己也做 subagent，但做的是更克制的那一种。

### 5.6 HITL 人机审批（`permissions/`）
- `local-permission-handler.ts`：本地审批入口
- `hitl-persist.ts`：**审批状态持久化**
- `pending-approvals-restorer.ts`：未决审批恢复
- `approval-key.ts`：审批 memo 的 key（同类操作不重复问）
- `memo-store.ts` / `memo-sync-client.ts`：审批记忆
- `os-error-blacklist.ts`：OS 错误黑名单

### 5.7 会话持久化（`session/`）
- `message-block-storage.ts`：消息块存储
- `event-storage.ts`：事件存储
- `snapshot-storage.ts`：快照存储
- `persistent-queue.ts` / `persistent-queue-file.ts`：持久队列
- `fork-local-session.ts` / `fork-tool-id-remap.ts`：会话分叉
- `subagent-index.ts` / `subagent-manager.ts`：子代理索引

---

## 6. Agent Host：把引擎接到真实世界

`packages/agent-host` 是 Runtime 纯引擎和具体宿主（Electron/Daemon）之间的粘合层：

- **state/**：session / turn / hitl / lease / owner / catalog / device-identity / delivery / attribution / conversation 全套 store
- **delivery/**：事件中继、outbox 模式、retry queue、delta coalesce、LLM snapshot HTTP ledger、message delivery outbox
- **conversation/**：run coordinator、run queue、abort、identity
- **hooks/**：memory / rules / context / relevant-recall / worktree-routing / LSP diagnostic / agent-profile
- **tools/**：TabTin 专属工具——document-tools、data-tools、attachment-tools、tabcode-adapter、file-edit-patch
- **policy/**：tool-risk-policy、judge memo、agent-modes tool gate
- **tracker/**：host scheduler（定时任务在本地跑）

---

## 7. 安全与权限：最工程化的部分

### 7.1 单一判决函数 `security-policy/judge.ts`

所有工具调用在 `tool-orchestration.ts` 入口过一次 `judge()`，5 步流程：

1. **Step 0：Agent Mode Tool Guard**——ask/plan/study 三种受限模式下，哪些工具能调
2. **Step 1：红线分级**
   - `catastrophic`（`rm -rf /` 等）：三档一律 deny
   - `risk`（sudo 等）：always_ask deny、auto 转 ask、full_access 放行
   - `sensitive_out_deny`：同上
3. **Step 2.5：sensitive_in_ask**——仅 always_ask 触发
4. **Step 3：审批模式**——`always_ask` / `auto`（替我批）/ `full_access`
5. 其他：object_read 直接 allow，object/object_write ask，MCP 一律 ask

硬编码红线（`hardline-v3.ts`）：
- `rm -rf /` 类灾难命令
- `sudo`
- PowerShell 混淆命令（opaque command）
- Windows 删除命令
- 敏感路径

### 7.2 safe-fs

`packages/safe-fs` 是 `node:fs` 的薄包装：
- 所有本地文件操作走这一层
- macOS TCC / Windows ACL / 杀软拦截 / 云盘占位 统一归一成 `OSAccessError`
- **Windows 上杀软劫持 read 会卡死——8 秒超时兜底**，归为 `OS_AV_BLOCKED`
- 不重试、不 fallback，重试由 runtime 做

### 7.3 审批 memo

用户批过一次"允许访问这个路径"，同类操作下次不再问。key 由 `build-pattern-key.ts` 生成。

---

## 8. 检查点（Checkpoint）：Shadow Git

`packages/checkpoint-core` 是整个项目最工程化的实现之一：

- 在用户项目目录**外面**维护一个隐藏 Git 仓库
- 通过 `core.worktree` 指向用户项目目录
- **不污染用户自己的 `.git`**
- Agent 每做完一轮自动 commit，commit 类型分 9 种：
  - `agent_turn_done` / `safety_before_restore` / `safety_before_replace`
  - `tabcode_replace` / `error_compensation` / `tabdata_auto_anchor`
  - `pre_approval` / `manual` / `system_recovery`
- 处理的脏活：
  - 嵌套 `.git` 目录自动重命名为 `_disabled`，避免 git 混乱
  - `index.lock` 残留超 30 秒自动清理
  - 每 20 次 commit 自动 GC
  - 磁盘超 500MB 告警
  - 崩溃恢复 manifest（重启后自动恢复 disabled 的 git dir）
  - 支持 restore / rewind（`moveHead` 两种语义）
  - `writeTree()` 轻量快照不产生 commit，用于 run 开始前打基线

---

## 9. 协作：CRDT + 实时通道

- **`packages/collab-core`**：CRDT 协作层
  - `provider.ts`：协作 provider
  - `presence-protocol.ts`：在线状态
  - `commands.ts` / `controlEvents.ts`：操作命令
  - `subdocuments.ts`：子文档
  - `sharding.ts`：分片
  - `version/`：版本历史（previewApi、useVersionHistory）
  - `useOfflineReplay.ts`：离线回放
- **`apps/collab-live`**：协作服务端
- **`packages/ws-gateway-client`**：前端 WebSocket 客户端
- **`contracts/asyncapi/ws-gateway.yaml`**：WebSocket 异步契约
- **Centrifugo**：云端实时消息中间件

---

## 10. 工作应用（App 层）

| App | 后端 Django app | 前端 package | 说明 |
|---|---|---|---|
| 文档 | `tabdoc` | `tabdoc-ui` / `doc-editor` / `doc-renderer` / `tabdoc-host-runtime` | 在线文档，Agent 可直接编辑 |
| 表格 | `tabdata` | `table-core` / `table-kernel` / `table-engine` / `table-engine-canvas` / `smartsheet` | 多维表格，本地用 PGlite |
| PPT | `tabslide` | `tabslide` | 演示文稿 |
| 网站 | `tabsite` | `tabsite-core` / `tabsite-templates` | Agent 生成网站 |
| 备忘录 | `tabmemo` | — | |
| 代码 | `tabcode` | — | coding 工具 |
| 终端 | — | `terminal-core` / `pty-core` / `lsp-runtime` | PTY + LSP |
| 浏览器 | — | `browser-core` / `browser-capabilities` / `crawlspace-core` / `anti-detect` | 自有浏览器自动化 |
| 会话 | `chat` / `tabchat` | `tabtin-chat-client` | 聊天 |
| 定时任务 | `tracker` | `agent-host/tracker` | cron/trigger 在本地 Daemon 跑 |
| Agent 实例 | `tins` | — | Tin = Agent 实例 |

---

## 11. 后端业务域（Django 36 个 app）

按职责分组：

- **用户与权限**：`users/auth`（JWT、session、device flow、API key、admin RBAC）、`users/wallet`（钱包/充值/幂等）、`users/membership`（订阅/配额/tier）
- **Agent**：`agent` / `agent_memory` / `user_portrait`（用户画像蒸馏）
- **协作**：`collab` / `channel_gateway`
- **工作应用**：`tabdoc` / `tabdata` / `tabslide` / `tabsite` / `tabmemo` / `tins` / `tabcode` / `tabchat` / `tabtinspace`
- **能力**：`skills` / `rag` / `fts`（全文检索）/ `capabilities` / `extensions`
- **集成**：`integrations_github` / `integrations_feishu`
- **安全**：`credential_vault` / `client_errors`
- **平台**：`analytics` / `diagnostics` / `i18n` / `platform_config` / `maintenance` / `login_relay` / `updater` / `common`

---

## 12. 数据契约与 Codegen

`contracts/` 目录是跨端契约：
- `openapi/tabtin-api.yaml`：REST API
- `asyncapi/ws-gateway.yaml`：WebSocket
- `models/shared-models.yaml`：跨端共享 JSON Schema（Draft 2020-12）
- `packages/wire-codegen`：从契约生成 TS/Python 类型

这是一个**契约先行**的多端协作模式——后端改 API 必须先改 YAML，前端 codegen 出类型。

---

## 13. 值得讲的工程亮点（分享选材池）

按"对内部技术同事有冲击感"排序：

### 第一梯队（强烈建议讲）

1. **Shadow Git 检查点**——不污染用户 .git，用 `core.worktree` 挂影子仓库，9 种 checkpoint 类型，处理嵌套 git dir / stale lock / GC / 崩溃恢复。这是"工作可交接"原则在工程上的具体落地。

2. **HITL 审批状态持久化 + crash resume**——Agent 要删文件前停下等人点头，这个状态写盘，app 崩溃重启还停在那儿。对应"责任有人承担"。

3. **单一风险判决函数 judge()**——三档审批模式（always_ask/auto/full_access）+ 红线分级（catastrophic/risk/sensitive）+ 硬编码危险命令检测 + 审批 memo。所有工具调用必经此门。

### 第二梯队（讲者备注备料，被问再展开）

4. **分层上下文压缩**——layered-prune + time-based microcompact + subagent summary + pressure router，不是简单摘要。

5. **Guard rails**——tool repetition tracker、text repetition detector、iteration budget、context overflow recovery。大家用 coding agent 都遇到过死循环。

6. **safe-fs 对 Windows 杀软的 8 秒超时**——这种脏活细节最能体现工程成熟度。

7. **Agent ≠ Device 的解耦**——身份和执行环境分离，手机发起、桌面执行。

8. **契约先行 + wire-codegen**——OpenAPI + AsyncAPI + 共享 JSON Schema，多端类型自动生成。

### 第三梯队（体量证据，一句话带过）

9. **8 app + 90 package + 4788 个 Python 文件**——不是 demo。

10. **CRDT 协作**——人和 Agent 同时编辑同一份文档。

11. **Daemon 同时是 MCP Server**——本地能力可以被其他 MCP 客户端调用。

---

## 14. 它没有做什么（也是一种态度）

从源码和 product-concepts.md 能看出的克制：

- **不做共享整个本地目录的远程文件系统**——Workspace 私有，交接只带引用
- **Agent 不自动获得全部权限**——必经 judge()
- **交接不绕过已有资源权限**——材料权限继续生效
- **没有做网状多智能体协同**——它的 subagent 是星型的，父调子，父子之间也有 HITL
- **重要判断和责任确认由人做**——Agent 执行，人验收

这几条和 PPT 第 8 页"两盆冷水"形成呼应：它不是不知道多智能体，而是被"65% 集体幻觉"这个问题提醒之后，选择先把人和单 agent 的可靠性做扎实。

---

## 15. 对分享的建议

现 PPT 13 页节奏刚好。如果要加源码细节，建议：

- **P10（架构亮点）**把"全栈开源"那条换成 Shadow Git，这是从源码挖出来的真东西
- **P9（四个机制）**讲"执行记录可追溯"时口头带一句：追溯靠影子 Git 实现
- 讲者备注里补：HITL 崩溃恢复、judge() 判决、safe-fs 杀软超时、分层压缩——被问到再展开
- 不建议加页，20-30 分钟已经满了

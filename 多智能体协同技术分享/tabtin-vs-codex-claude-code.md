# TabTin vs Codex / Claude Code：亮点对比

> 配套阅读：`codex-vs-claude-code-internals.md`。那篇讲这两个工具的内核差异；这篇讲 TabTin 站在什么位置、它多做了什么、又诚实承认什么不如。
> 整理时间：2026-09-12

---

## 0. 一句话

Codex 和 Claude Code 解决的是**"一个人怎么跟 AI 把活干完"**。
TabTin 解决的是**"一个人干完的活，怎么变成下一个同事的起点"**。

它不是 coding agent 的替代品，是站在这条链右边那片空白上——协作与交接层。

---

## 1. 根本差异：设计对象不同

| 维度 | Codex | Claude Code | TabTin |
|---|---|---|---|
| 设计假设 | 一个开发者在终端前 | 一个开发者在终端前 | 一个团队，人和 Agent 一起干活 |
| 主要场景 | coding | coding | 调研 / 文档 / 表格 / PPT / 浏览器采集 / 代码 |
| 工作产物 | 代码 diff | 代码 diff | 在线文档 / 表格 / PPT，人和 Agent 同一份 |
| 上下文归属 | 你的本地会话 | 你的本地会话 | 组织级，可交接 |
| 多用户 | 无 | 无 | 一等公民 |
| 交付形态 | CLI + IDE 插件 + Web | CLI + IDE 插件 + Web | 桌面 + iOS + Android + Web |

---

## 2. TabTin 多出来的能力（那两个完全没有）

### 2.1 工作交接——这是它的立身之本

- Codex / Claude Code：会话历史在你本地，换人就断了。同事接手只能靠你口头讲"我当时怎么想的"。
- TabTin：任务续接时冻结对话上下文 + 引用材料，接手人在**自己的 Agent 和 Workspace** 里继续做。双方本地目录独立，但上下文可追溯，权限继续生效。
- 这是产品理念里最核心的一句：**"一个人做完的活，成为下一个人的起点。"**

### 2.2 多人实时协作

- 那两个是单用户工具。
- TabTin 有 CRDT 协作层（`collab-core`）：人和 Agent 同时编辑同一份文档 / 表格，有 presence（在线状态）、版本历史、离线回放。
- 这意味着"两个人 + 一个 Agent 同时改同一份文档"是一等公民，不是外挂。

### 2.3 Agent 身份体系

- 那两个就是一个 assistant，每次靠 prompt 临时塑造。
- TabTin 里 Organization 下可以配多个 Agent 角色，每个有自己的规则、模型、Skill、记忆。
- 验证过的调研方法不是依赖某个人的 prompt，是固化成可复用的 Agent 角色。

### 2.4 工作应用层不只是终端

- 那两个：代码 + shell + 文件。
- TabTin：文档、多维表格、PPT、网站、浏览器采集，Agent 直接操作在线协作对象，不用你在聊天框和办公软件之间搬运产物。

### 2.5 团队治理

- 那两个：没有。
- TabTin：组织 / 成员 / 权限 / 模型 / 用量 / 执行记录管理后台；有凭证保险库（credential_vault）、用量统计、诊断、维护开关。

### 2.6 多端发起

- 那两个：CLI + IDE 插件 + Web。
- TabTin：桌面 + iOS + Android + Web。手机上能发起任务，桌面 Daemon 在线时实际执行。

---

## 3. 同类能力里 TabTin 做得更深的地方

### 3.1 检查点（Checkpoint）

| | Codex | Claude Code | TabTin |
|---|---|---|---|
| 机制 | worktree 隔离（为并行任务） | `/rewind` checkpoint | Shadow Git（影子仓库） |
| 覆盖范围 | 文件系统隔离 | **只追踪编辑工具，Bash 副作用撤不回来** | 9 种 checkpoint 类型，覆盖 agent turn、审批前、错误补偿、手动、崩溃恢复 |
| 对用户仓库的影响 | worktree 是真实 git 目录 | 自己存的 undo 数据 | 用 `core.worktree` 挂影子仓库，**不污染你自己的 .git** |
| 崩溃恢复 | 无 | 无（30 天保留） | 有 manifest，重启后自动恢复嵌套 git dir、清理 stale lock |
| 设计目的 | 并行任务隔离 | 自己回滚 | **为交接给别人设计** |

工程细节：处理嵌套 `.git` 目录自动重命名、`index.lock` 残留超 30 秒自动清理、每 20 次 commit 自动 GC、磁盘超 500MB 告警。

### 3.2 人机审批（HITL）

- Codex：靠 Seatbelt / Landlock **内核级沙箱**，不是审批——是"根本不让你跑"。
- Claude Code：`PermissionRequest` hooks，但**审批状态不持久化，崩溃就丢**。
- TabTin：单一 `judge()` 判决函数：
  - 三档审批模式：`always_ask` / `auto`（替我批）/ `full_access`
  - 红线分级：`catastrophic` / `risk` / `sensitive`
  - 硬编码危险命令检测：`rm -rf /`、`sudo`、PowerShell 混淆命令、Windows 删除命令、敏感路径
  - **审批状态写盘，app 崩溃重启还停在那儿等你批**
  - 审批 memo：同类操作批过一次下次不再问

### 3.3 防失控 Guard Rails

- 那两个主要靠人盯着和 hooks。
- TabTin 有专门的运行时保护（`agent-runtime/guards/`）：
  - `tool-repetition-tracker`：同一个工具重复调用
  - `text-repetition-detector`：文本重复
  - `iteration-budget`：迭代预算上限
  - `message-size-budget`：单条消息大小
  - `context-overflow-recovery`：上下文溢出恢复

### 3.4 上下文压缩

- Codex：auto-compaction（headless 模式还有静默失败的已知问题）。
- Claude Code：`PreCompact` / `PostCompact` hooks，需要自己 hook 进来重新注入关键约束。
- TabTin：分层压缩（`compact/`）——`layered-prune` + `time-based-microcompact` + `subagent-summary` + `pressure-router`，压缩前自动打 checkpoint。

### 3.5 子代理（Subagent）

| | Codex | Claude Code | TabTin |
|---|---|---|---|
| 拓扑 | worktree 并行 | Agent Teams（网状，工具链六成） | **星型**：父调子 |
| 父子交互 | 独立 worktree，事后合 diff | 网状消息 | fork query、parent midflight injector（父可中途插话）、父子之间也走 HITL |
| 取舍 | 隔离优先 | 互联优先 | **可靠性优先**——先把单 agent 做扎实 |

这和"65% 集体幻觉"那条冷水呼应：它不是不知道多智能体，是选择先把可靠性做扎实。

---

## 4. 诚实承认：TabTin 不如那两个的地方

讲的时候不要回避，技术同事一眼就看穿：

1. **coding 深度不如**：LSP、test runner、git 集成、仓库熟悉度它们更成熟。TabTin 的 tabcode 是较新的 app。
2. **沙箱硬度不如 Codex**：Codex 有 Seatbelt / Landlock **内核级**沙箱，TabTin 是应用层 `judge()`，不是 OS 级强制。
3. **生态不如**：MCP servers、hooks、社区插件、教程资源没那么丰富。
4. **模型选择受后端控制**：TabTin 走 LLM proxy，模型由组织配置；那两个更开放。
5. **启动开销**：完整平台比 CLI 重，不适合"打开就改一行代码"的场景。
6. **成熟度**：Public Preview，不同组件成熟度有差异（README 自己说的）。

---

## 5. 讲的时候怎么用这个对比

**不要做成"TabTin 全面吊打 Codex/CC"**——那不诚实，技术同事也不信。

正确的讲法是：

> Codex 和 Claude Code 把"一个人跟 AI 干活"这条链磨到了极致，但它们的设计假设从头到尾就是单用户。
> TabTin 不是来替代它们的，它站在这条链的**右边那片空白**上——一个人干完了，怎么让下一个人接得住。

这个对比正好强化"七层技术栈右边那层基本是空白"那张图：Codex/CC 把左边做满了，TabTin 填的是右边那一层。

---

## 6. 三句话带走

1. **Codex/CC 是工具，TabTin 是场所**——它们给你一把更好的锤子，它给你一个大家共用的工作台。
2. **那两个的安全是"硬隔离"（沙箱）或"人盯着"（hooks），TabTin 的安全是"状态持久化 + 责任可追溯"**——审批不会因为崩溃消失，工作不会因为换人丢失。
3. **它的 subagent 是星型不是网状**——在大家都在吹"多智能体自治"的时候，它选择先把单 agent 的可靠性做扎实，这是一个值得讲的工程克制。

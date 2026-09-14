# TabTin 团队协作真实流程（源码梳理）

> 本文不是产品宣传稿，而是对着 `apps/tabtin_django` 源码读出来的真实数据模型与状态机。
> 分享时凡是提到"协作流程"，都能在本文里指到具体的表、字段和状态。
> 代码位置：`apps/tabtin_django/apps/tabtinspace/models.py`、`apps/tabchat/handoff/models.py`、`apps/agent/models.py`。

---

## 0. 先把两个基础概念掰开（这是全系统的地基）

很多人第一次看会把"Agent"和"工作目录"混在一起。源码里它俩是**完全解耦**的：

| 概念 | 模型 | 回答的问题 | 关键字段 |
|---|---|---|---|
| **Agent** | `agent.Agent` | "**谁**在参与"（纯 AI 身份） | `name`、`goal`、`custom_rules`、`preferred_model_id`、`owner_user` |
| **Workspace** | `tabtinspace.Workspace` | "**在哪台设备的哪个目录**跑" | `device`、`working_dir`、`trust_status`、`approval_grant`、`execution_limits` |

源码原话：

> "Agent 领域模型——纯 AI 身份，与 Workspace / Space 解耦。只含人格/规则/配置；设备与工作目录属于 Workspace。"
> "不再挂默认执行 Agent。身份与现场解耦；运行时组合在 `ChatSession.agent_id + workspace_id`。"

也就是说：**会话 = 谁干（Agent）× 在哪干（Workspace）**，这两个是运行时才拼起来的，不是预设绑定。同一个 Agent 可以在不同电脑的不同目录上跑；同一份目录也可以挂不同 Agent。

Workspace 自带的治理字段（这些是"怎么撑住"的硬约束）：

- `trust_status`：`trusted` / `untrusted`，目录要先被信任
- `approval_grant`：`always_ask` / `auto` / `full_access` 三档审批（这是挂在现场上的，不是全局开关）
- `execution_limits`：`max_iterations_per_run`、`max_credits_per_run`——**每次 run 的迭代和钱都有上限**
- 唯一性约束：`(organization, created_by, device, normalized_working_dir)`——目录身份按"组织+人+设备+路径"隔离

---

## 1. 从建团队到交付出资产：一条任务的完整生命周期

### 阶段一：建组织、建项目

```
Organization（组织）
   └── OrganizationMember（成员身份）
Project（团队协作房间）
   ├── ProjectMembership（谁以什么角色在这个项目里）
   └── ProjectMemberWorkspace（每个成员在本项目里绑自己的私有现场）
```

**Project** 本身：

- 状态机：`active / paused / completed / archived / trashed`
- 可见性：`private`（仅创建者）/ `shared`（已共享）

**ProjectMembership**（成员关系）：

- 角色五档：`owner / admin / editor / viewer / participant`
- 状态：`pending`（待接受）→ `active`（已生效）——**邀请发出去，对方自己决定接不接**，不能硬塞
- 每个成员还能带 `responsibility`（职责描述）

**关键设计决策（源码注释原话）：**

> "Project 不代表共享本地目录；文件/代码维度仍是 Workspace 的单根契约。"
> "Project 不持有默认执行绑定；每个成员执行落到 ProjectMemberWorkspace 指向的私有 Workspace。"

**这意味着：建了项目 ≠ 大家共用一个文件夹。每个人在这个项目里干活，用的还是自己电脑上、自己那份受信任的目录。** 这是和传统"团队共享盘"最不一样的地方。

### 阶段二：派任务、接任务

任务本体是 `ProjectTask`，它有**两套独立的状态**，很多人会混：

**第一套：接不接（assignment）**

| 值 | 含义 |
|---|---|
| `pending` | 待确认 |
| `accepted` | 已接受 |
| `rejected` | 已拒绝 |

创建人 `created_by` 指定 `responsible_user`（责任人），但**接不接由责任人本人决定**。

**第二套：干到哪了（work）**

| 值 | 含义 |
|---|---|
| `todo` | 待执行 |
| `in_progress` | 执行中 |
| `in_review` | 待验收 |
| `blocked` | 受阻 |
| `done` | 已完成 |
| `cancelled` | 已取消 |

**接任务这一刻才做两件事**（不是建任务时定死）：

1. 选 `selected_agent`——用哪个 AI 身份去干
2. 确认 `project_member_workspace`——在哪份私有现场跑，记 `workspace_confirmed_at`

> 翻译成人话：派活的人只说"这件事你负责"；"用哪个 AI、在你哪台电脑的哪个目录上干"，是你接的时候自己拍板的。

### 阶段三：跑任务

一次执行是一条 `ProjectTaskRun`：

- **快照不可改**：`responsible_user`、`agent`、`workspace`、`device`、`chat_session` 全部在创建时快照下来，源码原话"执行绑定在创建后不可改写"，另有 `binding_snapshot` JSON 兜底
- 状态机：`preparing → pending → running → completed / failed / cancelled`
- **同一任务同时只能有一个 active run**（数据库唯一约束兜底，不会两个进程同时跑同一件事）
- 失败有脱敏原因 `safe_failure_reason`

**过程留痕是独立的一张 `ProjectTaskEvent`（不可变时间线）**：谁（actor）、在什么时间、做了什么 `event_type`、带了什么 `payload`。这张表只追加不修改，所以事后能完整还原"这一步是谁、什么时候、怎么动的"。

**重要：Agent 跑出来的候选产物放在 `result_items`，在验收之前只有责任人自己看得见。**

### 阶段四：验收、沉淀成团队资产

责任人把任务推到 `in_review`，然后验收：

- 从 `result_items` 里挑哪些算交付物
- 验收通过时，被挑中的产物才被"**发布**"成 `ProjectTaskDeliverable`
- 每个 Deliverable 挂到一条 **`ContextItem`**——这就是 Project 的**资产区**

源码原话：

> "责任人验收时明确发布到 Project 资产区的交付物。"

**关键：产物不是自动进团队资产的，是责任人验收后手动/明确发布才进的。** 验收前是责任人私有的草稿，验收发布后才变成团队可见资产。

### 资产区：东西分三级挂（`ContextItem`）

一条上下文资产，只挂三种宿主之一（互斥约束）：

| 宿主 | 挂哪 | 举例 |
|---|---|---|
| `workspace` | 个人现场 | 你自己的文档、你跑出来的中间产物 |
| `project` | 项目（团队资产区） | 任务发布的交付物、团队文档 |
| `organization` | 组织 | 不挂具体项目的组织云盘文件 |

资产本身是一棵知识库树（`parent` 自引用 + `collection` 文件夹），带全文搜索（PG `tsvector` + GIN 索引）、置顶、归档、回收站。**团队资产区是真的能被整个项目成员检索、复用的，不是聊天记录里的一条消息。**

---

## 2. 资产怎么被带走、任务怎么被接手：Handoff 模块

这是"沉淀之后还能流动"的后半段，也是最容易被忽略的部分。实现在 `apps/tabchat/handoff/`，四张表：

### 2.1 交接包本体 `HandoffPackage`

发起人可以是**人，也可以是 Agent**（数据库 XOR 约束：两者二选一，Agent 是一等发起方，不是附属品）。

包里固定四个区块：

| 区块 | 字段 | 内容 |
|---|---|---|
| 工作目标 | `goal` | 一句话说清要干嘛 |
| 当前进展 | `progress_json` | 进展要点数组 |
| 下一步 | `next_steps_json` | 带勾选状态的 checklist |
| 风险/待确认 | `risks_json` | 每条带 `high_risk` 标记 |

两个权限档位（`scope`）：

- `view_only`：只能看
- `continuable`：可以继续接着做

包本身状态机：`draft → sent → superseded / revoked`，带版本号（可以发"补充版"取代旧版）。

### 2.2 接收人逐人状态机 `HandoffRecipient`

一个包可以发给多个人，每个人有自己的状态：

```
sent 已发送
  → viewed 已查看
      → acknowledged 已了解
      → taking_over 由我继续 ── 接手后开新会话 linked_session_id，关联 tracker 任务
      → delegated_to_agent 交给自己的 Agent
      → rejected 已拒绝
```

也就是说：交接不是"发个摘要就完了"，接收人可以**只看一眼**、**表示了解**、**真的接手自己干**、**转手交给自己的 AI**，或者**拒绝**——每一档都是独立的权限和动作。

### 2.3 带过去的材料怎么不越权 `HandoffReference`

这是"资产区东西怎么共享出去"的核心设计。交接包能带五种材料：`im_message / document / table / attachment / chat_session`。但分两种处理：

| 材料类型 | 处理方式 | 为什么 |
|---|---|---|
| **回源型**（文档、表格、IM 消息） | 包里只存快照摘要；接收人看正文时**回源读取 + 按他自己的权限实时校验**；没权限返回结构化 `access_denied`（**不静默消失**） | 资产的所有权和修改权还在原处，谁能看由他自己的权限决定 |
| **快照型**（Agent 会话） | 那是发起人个人资产，对方回源看不了；创建时用发起人权限读一遍，**冻结成一份清洗版快照** `frozen_snapshot_json`，之后看快照、不回源、不逐条鉴权 | 个人会话没法授权，就冻结一份脱敏快照带走 |

**设计要点：分享 ≠ 开放目录。** 能带材料，但权限边界实时生效；不能回源的才冻结快照。

### 2.4 全程审计 `HandoffEvent`（append-only）

事件类型：`created / sent / viewed / acknowledged / taken_over / delegated / supplemented / revoked / rejected`。

任何一个交接包都能回答三个问题：**谁、什么时候发起的；谁看过、谁接手了；后来谁补充过、谁撤回了。**

---

## 3. 把整条链串起来（分享时可以照着这张图讲）

```
① 建组织 Organization
      ↓ 拉成员 OrganizationMember
② 建项目 Project + ProjectMembership（邀请 pending → 本人接受）
      ↓ 每人绑自己的私有 Workspace（Project 不是共享目录）
③ 派任务 ProjectTask：指定 responsible_user
      ↓ assignment：pending 待确认 → 本人 accepted / rejected
④ 接任务：当场选 selected_agent + project_member_workspace
      ↓
⑤ 跑 ProjectTaskRun：快照 agent/workspace/device，同时只跑一个
      ↓ ProjectTaskEvent 不可变时间线逐事件落库
⑥ 验收：result_items 验收前仅责任人可见
      ↓ 责任人明确"发布"
⑦ 交付物进 Project 资产区 ContextItem（团队可检索、可复用）
      ↓ 资产可被新交接包引用带走（回源实时鉴权 / 冻结清洗快照）
⑧ 任务没做完 → 发 HandoffPackage（目标 + 进展 + 下一步 + 风险 + 引用材料）
      ↓ 接收人 查看 → 了解 → 由我继续 / 交给 Agent / 拒绝
⑨ 接手开新会话接着干，HandoffEvent 全程 append-only 可追溯
```

---

## 4. 这份流程里最值得在分享时点出来的几个设计决策

1. **身份和现场解耦**：Agent（谁）和 Workspace（哪台机哪个目录）运行时才拼，所以"手机发起、桌面执行""同一个 AI 在不同项目里跑"都自然成立。
2. **Project 不是共享文件夹**：团队协作靠的是"任务 + 资产区 + 交接包"，不是把所有人塞进同一个目录。
3. **两套状态机**：先决定"接不接"（assignment），再记录"干到哪"（work）——接活是人的动作，进度是状态。
4. **执行即快照**：Run 一创建就把人、AI、现场、设备钉死，过程不可变时间线，同时只跑一个，可重跑（`rerun_of`）。
5. **验收前私有、验收后发布**：中间产物不污染团队资产，沉淀是责任人主动行为。
6. **分享不越权**：带材料分"回源实时鉴权"和"冻结清洗快照"两种，无权不静默消失。
7. **交接是多档状态机，不是一个开关**：看 / 了解 / 由我继续 / 交给自己的 AI / 拒绝，各是各的权限。
8. **Agent 是一等公民**：它能发起交接包，也能当接收人；不只是"人操作的工具"。

---

*梳理时间：2026-09-15 · 基于 develop 分支源码 · 表名/字段以仓库实际代码为准*

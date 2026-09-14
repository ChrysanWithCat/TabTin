# TabTin 如何进行团队协作：从 Project 到 Agent Team

> 这份文档作为技术分享的补充材料，**不改动 PPT 和原演讲稿**。  
> 重点回答：TabTin 的团队协作不只是“分享 AI 聊天记录”，还包括 Project 协作面、任务派发、成员私有执行现场、消息到 Agent 任务、子 Agent 团队模式、权限审批与结果验收。

---

## 1. 核心判断：TabTin 做的是“人 + Agent 的团队工作流”

TabTin 的团队协作可以分成四层：

```text
团队协作层：Organization / Project / Team Space / Channel / Message
任务派发层：Project Task / 负责人 / 接受或拒绝 / 状态流转 / 结果验收
Agent 团队层：父 Agent / 子 Agent / 角色派发 / 后台并行 / 汇总回收
执行治理层：Workspace / Device / HITL 审批 / 预算 / 权限 / 记录
```

所以它不是把“多人聊天 + AI 回复”拼在一起，而是把一次 AI 工作变成团队可接续、可检查、可分派、可验收的流程。

可以用一句话概括：

**TabTin 把团队成员、Agent、任务、执行环境和交付物放进同一套组织化工作流里；人负责目标、分工和验收，Agent 负责执行、整理和推进，系统负责上下文、权限、状态和证据链。**

---

## 2. 协作对象模型：不是只有 Chat，而是一组一等协作对象

TabTin 的基础模型来自仓库里的产品概念文档：

```text
Organization
├── Workspace：成员的私有执行现场
├── Agent：AI 参与身份
├── App：工作应用与能力
└── Device：实际执行环境
```

这几个对象的分离，是团队协作能成立的基础。

| 对象 | 在协作中的作用 | 关键边界 |
| --- | --- | --- |
| Organization | 团队、成员、权限、模型、资源和治理边界 | 不同组织默认隔离 |
| Project / Team Space | 团队围绕某个项目共同讨论、分派任务、沉淀交付物的共享协作面 | 共享的是项目协作面，不是所有人的本地目录 |
| Workspace | 成员自己的执行现场，包含工作根、文件、终端、Skill、Checkpoint 等 | 每个人在自己的 Workspace 里执行 |
| Agent | AI 参与身份，有角色、规则、模型、Skill、记忆和执行偏好 | Agent 表示“谁参与”，不表示“在哪里执行” |
| App | 文档、表格、PPT、消息、浏览器、终端等工作对象 | 人和 Agent 可以围绕同一份产物协作 |
| Device | 真正跑任务的设备环境，比如桌面端、Daemon、远程无头机 | 必须遵守组织、成员、Workspace 权限 |

这个模型决定了 TabTin 的团队协作不是“把一个人的机器开放给团队”，而是：

- 共享项目、任务、消息、上下文和交付物；
- 执行仍落在成员自己的 Workspace 和 Device；
- Agent 通过组织内的身份、规则、工具和权限参与工作；
- 结果进入文档、表格、PPT、任务交付物、消息线程等团队对象。

---

## 3. Project / Team Space：共享协作面，不是共享执行目录

仓库里的 `project.ts` 直接把 Project 定义为“团队协作房间，后端 Space(type=team_space)”。注释里还有一句关键设计：

```text
Project 是共享协作面，执行落到成员各自的伴生 Workspace（my_workspace）。
```

这句话非常重要。它说明 TabTin 的团队协作是分层的：

```text
Project / Team Space
  ├── 团队成员、频道、消息、任务、评论、交付物、事件
  └── 每个成员自己的 companion Workspace
        ├── working_dir
        ├── agent_id / execution_agent_id
        ├── bound_device_id / control_device_id
        └── device_status
```

也就是说：

- Project 是大家共同看的项目协作面；
- 成员进入 Project 后，会有自己的 `my_workspace` 作为执行现场；
- 一个任务被派给某个成员后，不是让他远程操作发起人的目录，而是在他的伴生 Workspace 里执行；
- 这样既能协作，又不会把本地环境、临时文件、密钥和无关上下文全部暴露给团队。

这比普通“群聊里叫 AI”更深：TabTin 把“讨论在哪里发生”和“任务在哪里执行”拆开了。

---

## 4. 团队空间里的频道协作：不只是私聊，而是项目上下文

在 Electron 端，Team Space 的频道会按 Project 分组进入私信侧栏。默认频道包括：

- `#general`：项目通用讨论；
- `#agent-updates`：Agent 更新、执行进展或自动事件更适合沉淀在这里。

这意味着团队不是只在一堆孤立 DM 里沟通，而是在 Project 维度聚合：

```text
Project A
├── #general
├── #agent-updates
├── Task 1
├── Task 2
└── Deliverables / Events / Conversations
```

团队成员可以在频道中讨论需求、贴材料、@同事或 Agent，也可以把某条消息进一步转成 Agent 任务。这一点连接了“人类讨论”和“Agent 执行”。

---

## 5. 任务派发：TabTin 有 Project Task 的完整生命周期

你提到的“任务派发”确实是当前文档需要补深的部分。TabTin 的 Project Task 不是普通待办事项，而是带有负责人、执行配置、运行会话、交付物和验收状态的工作单元。

从类型和 API 看，一个 Project Task 至少包含：

- `title` / `description` / `priority`：任务目标和优先级；
- `created_by`：谁创建；
- `responsible_user`：派给谁；
- `assignment_status`：指派状态，`pending / accepted / rejected`；
- `work_status`：工作状态，`todo / in_progress / in_review / blocked / done / cancelled`；
- `selected_agent`：选哪个 Agent 执行；
- `project_workspace`：绑定哪个项目伴生 Workspace；
- `workspace_confirmed` / `execution_ready`：执行现场是否确认、是否可运行；
- `latest_run` / `latest_completed_run`：最近执行与最近成功执行；
- `conversations`：任务下的准备会话和执行会话；
- `deliverables`：交付物；
- `events`：任务事件；
- `result_summary`：结果摘要；
- `version`：任务版本。

### 5.1 任务派发流程

可以把流程讲成下面这条链路：

```text
创建任务
  ↓
指定负责人 responsible_user
  ↓
负责人接受 / 拒绝 assignment
  ↓
负责人选择 Agent + Workspace
  ↓
系统准备执行会话 prepare run
  ↓
启动任务 run，可带 message 和 attachments
  ↓
Agent 在负责人执行现场推进任务
  ↓
过程中产生 comments / events / conversations / result_items
  ↓
负责人或团队验收结果 accept result
  ↓
沉淀 deliverables，任务进入 done / in_review 等状态
```

对应到前端 API，能看到这些操作：

| 操作 | 含义 |
| --- | --- |
| `createTask(projectId, payload)` | 在 Project 内创建任务，并指定负责人 |
| `respondTaskAssignment(projectId, taskId, accept)` | 负责人接受或拒绝任务指派 |
| `configureTaskExecution(projectId, taskId, { agent_id, workspace_id })` | 确认由哪个 Agent、哪个 Workspace 执行 |
| `prepareTaskRun(projectId, taskId)` | 准备执行会话 |
| `startTaskRun(projectId, taskId, { message, attachments })` | 启动一次任务执行 |
| `addTaskComment(projectId, taskId, content)` | 在任务上补充讨论或说明 |
| `cancelTask(projectId, taskId)` | 取消任务 |
| `acceptTaskResult(projectId, taskId, payload)` | 验收任务结果并形成交付物 |
| `setTaskResultVisibility(projectId, taskId, visibility)` | 控制结果可见性 |

### 5.2 任务派发和普通 ToDo 的区别

普通 ToDo 只回答“谁要做什么”。TabTin 的任务派发还回答：

- 这个任务是从哪个 Project 来的？
- 负责人是否接受？
- 用哪个 Agent 做？
- 在哪个 Workspace / Device 上执行？
- 是否已经准备好执行环境？
- 每次执行对应哪个会话？
- 执行过程中发生了什么事件？
- 结果有哪些可检查的交付物？
- 谁验收了结果？

因此它更接近“AI 原生项目管理任务”，而不是传统任务列表。

---

## 6. 从团队消息直接派生 Agent 任务

TabTin 还有一层很关键的能力：团队讨论里的消息可以直接变成 Agent 任务。

在 `TeamSpaceCreateTaskDialog.tsx` 里，界面标题是“询问 Agent”，说明文案是：

```text
源消息和回复线程会自动带入，你也可以补充更明确的问题、边界或产出要求。
```

这表示：

1. 团队成员在 Project 频道里讨论；
2. 某条消息或某个回复线程里已经有上下文；
3. 用户可以基于这条消息创建 Agent 任务；
4. 系统自动把源消息和回复线程作为默认上下文；
5. 用户再补充边界、问题、输出格式或限制；
6. 创建出的 Agent 会话保留 `source_message_ids`，能回溯它来自哪些讨论。

接口形态也能说明这点：

```text
POST /conversations/{conversationId}/messages/{messageId}/agent-task
body: { agent_id, additional_context }
```

返回结果中包含：

- `session_id`：生成的 Agent 会话；
- `thread_id`：会话线程；
- `space_id` / `organization_id`：所属项目空间和组织；
- `default_prompt`：从消息线程生成的默认任务提示；
- `source_message_ids`：来源消息集合。

这解决的是一个真实协作痛点：团队讨论经常卡在“谁把刚才讨论整理成任务给 AI 做”。TabTin 的设计是让消息线程本身成为任务入口，而不是重新复制粘贴上下文。

---

## 7. Agent 子团队 / Group Dispatch：父 Agent 可以派发多个子 Agent

更深层的一点是：TabTin 不只是“团队成员之间派任务”，Agent 内部也支持“子 Agent 团队模式”。

在 `packages/agent-runtime/src/subagent/agent-tool.ts` 里，`agent` 工具用于启动一个子 Agent 独立处理任务。这个工具的字段已经能看出它不是简单函数调用，而是一个任务派发协议。

### 7.1 子 Agent 派发的核心字段

| 字段 | 作用 |
| --- | --- |
| `prompt` | 给子 Agent 的自包含任务：目标、背景、输入、交付格式、验收标准 |
| `description` | 这个子任务的 3-5 个词短标签 |
| `role` | 子 Agent 的角色名或身份，比如“数据整理员”“技术审查员”；Group 派发时务必填写 |
| `template_id` | 可选，关联子 Agent 配置模板，由宿主展开为模型、工具等参数 |
| `model` | 可指定子 Agent 使用的模型；缺省跟随父 Agent；不可用时运行时确定性降级并说明 |
| `readonly` | 只读模式，写操作由运行时硬拦，适合研究、审查、规划、安全检查 |
| `background` | 后台运行，父 Agent 立刻拿到子 Agent ID，继续做别的事 |
| `wait_agent_ids` | 对后台子 Agent 做 fan-in 汇总，等待一组子 Agent 到终态 |
| `check_agent_id` | 只读查询某个子 Agent 的状态、步数、最近工具 |
| `message_agent_id` | 给运行中的子 Agent 投递中途指引 |
| `resume_agent_id` | 续跑或追问已结束的子 Agent，保留它自己的工作上下文 |
| `interrupt` | 中断运行中或排队中的子 Agent，并用新指令重定向后续工作 |
| `report_schema` | 可要求 `findings` 结构化结果，先输出发现、证据、置信度 JSON |
| `fork_context` | 是否把父对话历史完整转交给子 Agent；默认 false |

这套字段意味着：父 Agent 可以像团队负责人一样进行任务拆解、角色分工、并行推进和结果汇总。

### 7.2 role 和 prompt 是分离的

源码注释里强调了一点：

```text
role 是“谁来做”，prompt 是“做什么”。
```

这对团队模式很重要。比如同样是“调研竞品”，可以派给不同角色：

```text
role: 数据采集员
prompt: 收集 A/B/C 三个竞品的价格、版本、更新日期，并列出来源链接。

role: 技术可行性审查员
prompt: 评估这些竞品功能在我们当前架构中的实现难度和风险。

role: 文档整理员
prompt: 把调研结果整理成给管理层看的 1 页摘要。
```

如果把角色混在 prompt 里，UI 和协作视图很难展示“当前团队里谁在做什么”。TabTin 把 role 单独拿出来，说明它的子 Agent 不是匿名工具调用，而是可以被协作界面识别和呈现的工作成员。

---

## 8. 子代理团队模式的关键机制

### 8.1 默认上下文隔离：避免父 Agent 原文污染子 Agent

`fork-query.ts` 里有一个很关键的默认值：

```text
DEFAULT_INHERIT_MODE = 'none'
```

含义是：默认情况下，子 Agent 不继承父对话历史，只接收父 Agent 派给它的任务 prompt。

这和很多人直觉相反，但非常合理。因为如果把父会话全文塞给子 Agent，可能出现几个问题：

- 子 Agent 被父会话里“讨论怎么派任务”的原文带偏；
- 弱模型分不清哪些是自己的任务，哪些是父 Agent 的规划；
- 父上下文里可能有无关信息、隐私信息或高噪声内容；
- 多层嵌套时上下文膨胀，成本和不确定性都会增加。

所以 TabTin 默认采用“任务隔离”：

```text
父 Agent 负责拆任务
子 Agent 只看被派给自己的自包含任务
需要完整继承时，才显式设置 fork_context:true
```

这是团队模式里的“最小必要上下文”原则。

### 8.2 Worker 纪律：子 Agent 是被派发的工作单元

`fork-query.ts` 还描述了子 Agent 的工作纪律：

- 子 Agent 是 fork 出来的子进程；
- 子 Agent 面向父 Agent 做一次性汇报；
- 子 Agent 不应该再继续派发子 Agent；
- 子 Agent 在自己的任务范围内静默执行；
- 回报格式不在系统里硬编码，而由父 Agent 的 task 决定。

这说明 TabTin 的子 Agent 团队不是完全自治的网状组织，而是更像“任务负责人 + 多个专业执行者”的 mission / group 模式。

### 8.3 后台并行：fan-out / fan-in

`background: true` 让父 Agent 可以把多个长任务先派出去：

```text
父 Agent
├── 子 Agent 1：数据收集（background）
├── 子 Agent 2：技术审查（background）
├── 子 Agent 3：风险分析（background）
└── 父 Agent 自己继续整理框架或处理其它工作
```

每个后台子 Agent 会返回一个 ID。之后父 Agent 用 `wait_agent_ids` 一次性等待这些子 Agent 到终态，再汇总结果：

```text
fan-out：派出去并行跑
fan-in：等待全部完成后汇总
```

这就是“子代理团队模式”的核心执行形态。

### 8.4 状态查询、插话和中断重定向

TabTin 不是派出去就不可控。运行中还有几种控制方式：

- `check_agent_id`：查询某个子 Agent 当前是排队、运行、完成、失败还是取消，并查看步数和最近工具；
- `message_agent_id`：给运行中的子 Agent 投递中途指引，下一轮生效；
- `interrupt + resume_agent_id`：中断运行中或排队中的子 Agent，等它真正停下后，用新 prompt 重定向继续；
- `resume_agent_id`：对已结束子 Agent 续跑或追问，保留它自己的上下文，避免重新派一个导致前面工作丢失。

这相当于团队管理里的：

```text
看进度 → 补充要求 → 改方向 → 续接已有工作
```

### 8.5 只读子 Agent：把审查和执行分开

`readonly: true` 让子 Agent 进入只读问答模式，写操作由运行时硬拦。

这对团队协作很重要，因为很多子任务本质上不应该有写权限：

- 安全审查；
- 资料核验；
- 竞品调研；
- 方案评审；
- 测试风险分析；
- 代码 Review；
- 成本估算。

这样父 Agent 可以派一个“审查员”去看证据，但不让它改文件；再派另一个“执行员”去真正落地修改。角色、权限和产出可以分开治理。

### 8.6 结构化发现：不是只交一句“完成了”

`report_schema: 'findings'` 要求子 Agent 先输出发现、证据、置信度 JSON。这适合团队需要可检查证据链的场景。

比如安全审查子 Agent 不应该只说“没发现问题”，而应该报告：

```text
发现了什么
证据在哪里
置信度是多少
还没确认什么
需要谁进一步判断
```

这和 TabTin 的产品原则一致：结果可检查，过程可追溯。

### 8.7 模型和模板：不同角色可以用不同能力

子 Agent 可以通过 `model` 选择模型，也可以通过 `template_id` 关联某个子 Agent 配置模板。运行时会从可用模型清单里解析，命不中时确定性降级并说明。

这意味着团队模式不是所有子 Agent 都用同一种配置，而是可以按角色分配能力：

| 角色 | 可能配置 |
| --- | --- |
| 数据采集员 | 便宜、速度快、只读、擅长检索整理 |
| 架构审查员 | 推理强、上下文长、只读 |
| 文档撰写员 | 写作风格模板、术语表、文档 Skill |
| 代码执行员 | 有 Workspace 写权限、终端权限、审批流程 |
| 测试分析员 | 测试模板、覆盖率规则、缺陷分类规则 |

这才是“团队”的味道：不是复制多个同质 Agent，而是让不同角色承担不同职责。

---

## 9. 子 Agent 调度与预算：不是无限并发，而是受控团队执行

团队模式如果没有调度，很容易变成“无限开 Agent，成本和状态都爆炸”。TabTin 在运行时做了调度和预算控制。

`BudgetTracker` 里可以看到几个关键设计：

- 子 Agent 提交时先进入调度器；
- 有空位则 `active` 直接运行；
- active 满了则进入 `queued`；
- queue 满了则拒绝，返回 `queue_full`；
- token / credits 预算在父子 Agent 树中共享；
- 预算耗尽时可以清理队列；
- 并发槽位按子 Agent 嵌套深度分池，避免父子互等造成死锁；
- 默认队列上限有兜底设计，避免“派任务总是丢”。

这说明 TabTin 的子代理团队模式有工程上的“组织纪律”：

```text
能派发，但要排队；
能并行，但要限流；
能续跑，但要登记；
能中断，但要等存储 settle；
能花费，但要纳入预算树。
```

---

## 10. SubagentManager：让子 Agent 成为会话内可管理对象

`SubagentManager` 是 session 维度的子 Agent 运行登记中心。它解决的问题是：子 Agent 尤其是后台子 Agent 不能只是内存里的一次函数调用，否则会出现这些问题：

- 父 turn 结束了，后台子 Agent 怎么继续？
- runtime 重建后，后台子 Agent 怎么拿到新的 live 依赖？
- 用户要取消某个子 Agent，怎么只取消这个 session 的，而不误伤别的 session？
- 子 Agent 完成后，如何通知父 Agent？
- 排队中的子 Agent 怎么显示状态、怎么取消？
- 中断后如何等它真正 settle，避免两个 run 同时写同一个 `messages.jsonl`？

SubagentManager 做的事情包括：

- `registerRun`：登记子 Agent run；
- `cancel` / `dispose`：取消或清理本 session 的子 Agent；
- `getStatus` / `list`：查询子 Agent 状态；
- `rebindLiveDeps`：runtime 重建后重新绑定 emitter、HITL、预算、风险策略、workspace root 等活体依赖；
- `spawnBackground`：让后台子 Agent 脱离父 turn 生命周期；
- `notifyCompleted`：子 Agent 到终态后，把完成事件投进 NotificationQueue，跨 turn 唤醒父 Agent；
- `reportProgress`：回填 stepCount 和 latestTool，供 UI 或状态查询展示；
- `waitUntilSettled`：中断重定向时等待旧 run 完全停稳。

这意味着 TabTin 对子 Agent 的处理接近“团队成员运行态管理”，而不是“调用一个工具函数”。

---

## 11. 实时协作：人和 Agent 面向同一份工作对象

TabTin 还有实时协作层。`docs/development/local-startup.md` 里提到 `apps/collab-live`：

```text
Collab Live — Hocuspocus / Y.js WebSocket Server
负责文档、表格、演示文稿的多人实时协同编辑。
```

这说明 TabTin 的团队协作不只在任务列表和聊天窗口里，还落到具体工作对象：

- 文档；
- 表格；
- 演示文稿；
- 消息；
- Agent 会话；
- 任务交付物。

因此它可以形成这样的协作方式：

```text
人提出目标 → Agent 生成内容 → 人实时编辑 → Agent 再整理 → 团队评论 → 任务验收 → 结果沉淀
```

和普通 AI 聊天工具相比，关键差异是：TabTin 试图让 Agent 进入团队正在协作的对象，而不是让人把 AI 回复复制到另一个正式文档里。

---

## 12. 交接与续接：不是转发结果，而是转移可继续工作的上下文

TabTin 当前提供几种会话共享与交接方式：

| 协作方式 | 含义 |
| --- | --- |
| 实时查看 | 跟进任务进展，不能操作发送方现场 |
| 实时协作 | 可以在任务里发言驱动 Agent；执行、审批与费用仍在任务所有者这边 |
| 查看并抄走 / Fork | 可实时查看，并复制成自己的任务继续推进 |
| 交给同事继续 | 冻结上下文，创建对方自己的任务 |

这里面最重要的是：

```text
看、说、复制、接手，是不同权限。
```

“交给同事继续”并不是把 A 的整个电脑目录开放给 B，而是：

1. 冻结可共享的必要会话上下文；
2. 记录任务中引用的文档、表格、云端文件、本地文件；
3. 接收人选择自己的 Agent 和 Workspace；
4. 创建一个独立续接任务；
5. 资源能否访问仍受原权限约束。

这正好和 Project Task 的派发机制互补：

- Project Task 解决“团队内如何分配和验收工作”；
- 会话续接解决“一个正在进行的 Agent 工作如何交给别人继续”；
- 交接包解决“把目标、进展、下一步、风险和引用材料一次性发给团队”。

---

## 13. 一条完整团队协作链路示例：竞品调研到方案输出

假设团队要做一个“竞品调研 + 技术可行性方案”。TabTin 中可以这样运转：

### 13.1 PM 创建 Project

PM 创建一个 Project，例如：

```text
Project：AI 协作产品竞品调研
频道：#general / #agent-updates
成员：PM、研发、测试、设计、运营
```

系统为成员提供项目协作面；每个成员进入后有自己的伴生 Workspace。

### 13.2 在频道里讨论并形成 Agent 任务

团队在 `#general` 里讨论：

```text
我们需要比较 TabTin、Claude Code、Codex、Cursor、Manus 的团队协作能力，
重点看任务派发、多人接手、子 Agent、权限审批和交付物沉淀。
```

PM 直接从这条消息创建 Agent 任务：

- 源消息和回复线程自动带入；
- PM 补充输出格式：要对比表、结论摘要、证据链接；
- 选择一个“竞品调研 Agent”。

### 13.3 Project Task 派给负责人

PM 创建 Project Task：

```text
任务：输出竞品协作能力对比文档
负责人：产品同事 A
优先级：high
```

产品同事 A 接受任务，并配置：

```text
Agent：竞品调研 Agent
Workspace：A 的项目伴生 Workspace
Device：A 当前在线的桌面或 Daemon
```

### 13.4 父 Agent 启动子 Agent 团队

竞品调研 Agent 作为父 Agent，把任务拆成多个子 Agent：

```text
子 Agent 1
role: 数据采集员
readonly: true
background: true
prompt: 收集各竞品公开资料，列出来源、时间、功能点。

子 Agent 2
role: 协作机制分析员
readonly: true
background: true
report_schema: findings
prompt: 对比各产品在多人协作、任务接手、权限控制上的机制，给证据和置信度。

子 Agent 3
role: 技术架构审查员
readonly: true
background: true
report_schema: findings
prompt: 从工程角度判断 TabTin 的 Project、Workspace、SubagentManager、BudgetTracker 设计有什么差异。

子 Agent 4
role: 文档撰写员
background: false 或等待前三者后 resume
prompt: 根据调研结论整理成面向技术分享的文档。
```

父 Agent 一边派发后台任务，一边先搭文档结构。等后台子 Agent 完成后，用 `wait_agent_ids` 汇总结果。

### 13.5 中途控制

如果发现方向偏了，负责人或父 Agent 可以：

- 用 `check_agent_id` 看某个子 Agent 当前状态；
- 用 `message_agent_id` 给“数据采集员”补一句“优先找官方文档，不要用二手博客”；
- 用 `interrupt + resume_agent_id` 中断一个方向错误的子 Agent，并重定向任务；
- 对已完成但不够详细的子 Agent 用 `resume_agent_id` 追问。

### 13.6 结果进入团队协作对象

最终结果不是停留在某个 AI 聊天窗口里，而是进入：

- Project Task 的 `result_summary`；
- 文档或表格；
- `deliverables`；
- 任务事件和评论；
- 必要时同步到 `#agent-updates`。

团队成员可以评论、补充、验收或要求返工。负责人验收后，任务状态进入完成或待评审。

---

## 14. 为什么这不是普通的“Agent Teams”宣传概念

很多产品讲 Agent Teams，容易变成“几个 Agent 在一起聊天”。TabTin 从仓库线索看更务实，它更像一个受控的 star / mission 模式：

```text
人类负责人 / 父 Agent
      ↓ 派发任务
多个角色化子 Agent
      ↓ 一次性或结构化汇报
父 Agent 汇总
      ↓
Project Task / 文档 / 表格 / 交付物 / 团队验收
```

它的特点是：

- 子 Agent 有角色 `role`，不是匿名并发；
- 子 Agent 默认不继承父上下文，降低污染；
- 可以后台并行，也可以等待汇总；
- 可以查状态、插话、中断、续跑；
- 可以只读审查，也可以交给有权限的执行 Agent；
- 有预算、队列、并发和深度保护；
- 结果回到父 Agent 和 Project Task，而不是任由 Agent 网状扩散。

边界也要讲清楚：当前线索不表明它是“完全网状自治 Agent 社会”。更准确的说法是：

**TabTin 支持由人或父 Agent 进行目标拆解与角色化派发的子代理团队模式；它强调可控、可查、可汇总，而不是无限自治。**

---

## 15. 和普通 AI 工具的差异

| 维度 | 普通 AI 聊天 / 单人 Agent 工具 | TabTin |
| --- | --- | --- |
| 协作入口 | 个人聊天窗口 | Organization / Project / Team Space / Channel / Task |
| 任务派发 | 人手工复制需求给别人 | Project Task 指定负责人，支持接受/拒绝、状态、评论、验收 |
| 执行现场 | 通常绑定当前用户本机或会话 | 负责人选择自己的 Agent + Workspace + Device |
| 消息上下文 | 讨论和 AI 任务割裂 | 源消息和回复线程可直接生成 Agent 任务 |
| 多 Agent | 常见是多个聊天并行 | 父 Agent 通过 agent 工具进行角色化子 Agent 派发 |
| 子 Agent 上下文 | 经常直接共享大上下文 | 默认不继承父对话，只给自包含任务，必要时显式 fork_context |
| 子 Agent 控制 | 通常只能等结果 | 支持后台、等待、状态查询、中途指令、中断重定向、续跑 |
| 权限 | 粗粒度，容易过宽 | Workspace、Device、HITL、readonly、组织权限分层 |
| 成本控制 | 容易失控 | BudgetTracker 统一管理父子预算、并发、队列和深度 |
| 交付物 | AI 回复需要复制粘贴 | 结果进入文档、表格、任务 deliverables、events、comments |
| 交接 | 发截图、摘要或最终文件 | 冻结上下文、引用材料、独立续接任务、交接包 |
| 责任 | AI 输出后责任模糊 | 人负责接收、配置执行、审批和验收 |

---

## 16. 技术分享里可以这样讲

可以把 TabTin 的团队协作讲成三句话：

> 第一层，TabTin 把团队协作放在 Project / Team Space 里，项目有频道、消息、任务、评论、事件和交付物；但执行不是共享一个目录，而是落到每个成员自己的伴生 Workspace。  
> 第二层，TabTin 有真正的任务派发：任务可以指定负责人，负责人接受或拒绝，选择 Agent 和 Workspace，启动执行，最后沉淀结果并验收。  
> 第三层，Agent 自身也能组成受控的子代理团队：父 Agent 按 role 派发子 Agent，后台并行，等待汇总，必要时查询、插话、中断、续跑，并通过 readonly、预算和队列控制风险。

如果需要更短：

**TabTin 的团队协作，不是把 AI 聊天分享出去，而是把 AI 工作组织成“项目协作面 + 任务派发 + 子 Agent 团队 + 可治理执行现场”的闭环。**

---

## 17. 需要注意的边界

为了避免讲过头，建议明确这些边界：

- TabTin 不是共享所有成员本地目录的远程文件系统；
- Project 是共享协作面，执行仍落到成员自己的 Workspace；
- Agent 不会自动获得全部权限，执行要遵守组织、成员、Workspace 和 Device 权限；
- 任务交接不会绕过已有资源权限；
- 子 Agent 团队更准确地说是 star / mission / group 派发模式，不是完全网状自治 Agent 社会；
- 当前项目仍处于 Public Preview，具体体验和功能开关要以实际版本为准。

这些边界不是缺点，反而是 TabTin 的协作设计重点：**它不是把所有东西都共享给 AI 或同事，而是在共享、执行、交接、审批、验收之间建立清晰边界。**

---

## 18. 本文依据的仓库线索

- `docs/architecture/product-concepts.md`：产品理念、Organization / Workspace / Agent / App / Device 模型、任务续接、交接包和非目标。
- `docs/development/local-startup.md`：Collab Live 实时协作服务，基于 Hocuspocus / Y.js，负责文档、表格、演示文稿的多人实时协同编辑；后端域包括组织、空间、Agent、任务、实时协作、IM 等。
- `apps/tabtin-electron/src/renderer/src/types/project.ts`：Project / ProjectCompanionWorkspace / ProjectTask / ProjectTaskRun 类型，体现 Project 是 team_space，执行落到成员各自伴生 Workspace，任务含负责人、状态、执行配置、会话、交付物、事件和验收字段。
- `apps/tabtin-electron/src/renderer/src/services/projectApi.ts`：Project Task 的创建、邀请、接受任务、配置执行、准备/启动运行、取消、评论、验收、结果可见性等 API。
- `apps/tabtin-electron/src/renderer/src/lib/groupConversationsForInbox.ts`、`TeamSpaceChannelGroup.tsx`：Project 频道在私信侧栏按 Space 分组，默认频道包括 `#general` 和 `#agent-updates`。
- `apps/tabtin-electron/src/renderer/src/components/tabchat/TeamSpaceCreateTaskDialog.tsx`、`services/tabchatApi.ts`：团队消息和回复线程可以自动带入，创建 Agent 任务，并保留 `source_message_ids`。
- `apps/tabtin-electron/src/renderer/src/components/tabchat/sessionSharePresentation.tsx`：实时查看、查看并抄走 / Fork、实时协作、交给同事继续等共享档位。
- `apps/tabtin-electron/src/renderer/src/components/tabchat/AgentWorkspacePickerDialog.tsx`：群聊中为 Agent 选择个人 Workspace 执行现场，进一步印证“共享协作面”和“个人执行现场”的分离。
- `packages/agent-runtime/src/subagent/agent-tool.ts`：子 Agent 派发工具，包含 role、prompt、template、model、readonly、background、wait、check、message、resume、interrupt、report_schema、fork_context 等字段。
- `packages/agent-runtime/src/subagent/fork-query.ts`：子 Agent 默认不继承父对话历史，采用任务隔离；包含 worker 纪律和 resume 上下文说明。
- `packages/agent-runtime/src/session/subagent-manager.ts`：session 维度的子 Agent 运行登记、后台运行、完成通知、状态查询、取消、中断 settle、live 依赖重绑定。
- `packages/agent-runtime/src/engine/guards/budget-tracker.ts`：子 Agent 调度、active / queued / rejected、队列上限、预算共享、按嵌套深度分池避免死锁。
- `多智能体协同技术分享/tabtin-vs-codex-claude-code.md`、`多智能体协同技术分享/TabTin-架构梳理.md`：已有分享材料中对协作、交接、CRDT、Agent 身份体系、Shadow Git 和治理能力的整理。

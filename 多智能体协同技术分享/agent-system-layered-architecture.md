# 从"裸调用 API"到 Codex / Claude Code：分层系统架构拆解

> 上一份文档是"两边各有什么坑"的横向对比，这一份换个角度：**纵向地、一层一层地把系统搭出来**——从最原始的一次 HTTP 请求开始，每加一层能力，就说明"为什么需要这层""这层解决了上一层的什么缺陷""Codex 和 Claude Code 在这一层分别做了什么选择"。看完你应该能回答："如果我自己要搭一个 agentic CLI，我需要按什么顺序造轮子？"

---

## 全局分层地图

```
Layer 7  编排层        多任务并行 / 子代理 / 云端 fan-out
Layer 6  状态持久层     checkpoint / resume / worktree 隔离
Layer 5  协议扩展层     MCP（工具协议标准化）/ Hooks（治理钩子）
Layer 4  执行沙箱层     权限模型 + 操作系统级隔离
Layer 3  记忆上下文层   系统指令注入 + 项目记忆文件 + 压缩策略
Layer 2  Agent 循环层   tool_use ⇄ tool_result 的机械循环
Layer 1  裸 API 层     一次 messages 请求，无状态，无工具
```

**核心论点**：Layer 1 到 Layer 2 是"从聊天机器人变成能干活的东西"的质变，后面每一层都是在解决"Agent 循环失控"这一个母问题的不同侧面——上下文会漂、工具会闯祸、长任务会断、多任务会打架。Codex 和 Claude Code 是这个分层模型的两个不同实现，差异全部来自第 4 层往上的设计取舍。

---

## Layer 1：裸 API 调用——没有"agent"这回事

最原始的形态就是一次 HTTP 请求：

```bash
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "帮我看看这段代码有什么问题"}]
  }'
```

这一层的本质限制：**模型只能"说"，不能"做"**。它没有文件系统访问能力，不知道你的代码库长什么样，返回一段文本就结束了——这就是为什么早期"AI 编程助手"都是复制粘贴代码块的体验。Codex（OpenAI Responses API）和 Claude（Anthropic Messages API）在这一层的接口形状不同，但本质都一样：**一个请求，一个响应，无副作用**。

---

## Layer 2：Agent 循环——把"回答问题"变成"执行任务"

这是从 API 到"agent"的第一次、也是最关键的一次跃迁。机制上非常简单，就是给模型一组 `tools` 定义，让它在文本回复之外可以吐出一个结构化的"我要调用这个工具"请求：

```
1. 客户端发送 messages + tools 定义
2. 模型返回一个 tool_use / function_call 块（而不是纯文本）
3. 客户端（不是模型！）在本地实际执行这个工具调用
4. 客户端把执行结果作为 tool_result 追加进 messages，再发一次请求
5. 重复 2-4，直到模型不再请求工具调用，返回最终文本
```

**关键认知**：模型从头到尾没有"执行"任何东西——它只是生成了一段结构化文本（"请帮我跑 `npm test`"），**真正执行 Bash、真正写文件的是宿主进程（Claude Code / Codex CLI 本身）**。所以"Agent"这个词描述的其实是"模型 + 宿主循环"这个整体，模型单独拿出来只是个更擅长吐结构化输出的 API。这也是为什么"提示注入"（prompt injection）能成为攻击面——模型没有能力区分"这是用户真正想让我执行的命令"还是"这是我从某个文件里读到的、伪装成指令的文本"，拦截判断的责任被甩给了宿主循环。

**这一层暴露的问题**（催生了后面所有层）：
- 模型不知道项目背景 → 需要 Layer 3（记忆/上下文注入）
- 模型可能被诱导执行危险命令，或者单纯犯错 → 需要 Layer 4（沙箱/权限）
- 内置工具集有限，扩展工具需要改宿主代码 → 需要 Layer 5（MCP 协议）
- 长任务中途出错想撤销，没有天然的"事务"概念 → 需要 Layer 6（状态持久化）
- 单个循环干不完大任务，想并行拆分 → 需要 Layer 7（编排）

---

## Layer 3：记忆与上下文层——解决"模型每次都是失忆状态"

Layer 2 的循环本身不持久化任何东西：每次冷启动，模型对项目一无所知。这一层要解决两个子问题：**（a）怎么把项目知识灌进去，（b）怎么应对长任务把 context window 撑爆**。

### (a) 系统指令注入：两种完全不同的哲学

- **Codex：替换式**。`model_instructions_file` 字段直接**替换**官方内置的 `base_instructions`（默认是 Codex 团队写的行为规范），AGENTS.md 是在这个基础上**追加**的项目指令。也就是说 Codex 把"基础人格/安全对齐层"和"项目知识层"做成了两个独立、其中一个可被整体替换的模块。
- **Claude Code：只追加式**。系统提示词（Anthropic 内置，不可替换）之上，CLAUDE.md 是纯粹的上下文追加，不存在"替换基础指令"这个开关。这是两边安全模型的一个本质分歧：Codex 承认"高级用户可以整体换掉基础指令"（企业场景下用来定制合规话术），Claude Code 不开放这个口子。

### (b) 长任务的上下文压缩

Agent 循环每转一圈都在往 messages 数组里追加 tool_result，线性增长，必然撞到 context window 上限。两边都做了"compaction"（自动摘要旧对话，替换成一段精简总结），但触发和补救机制不同：

- Claude Code：`PreCompact`/`PostCompact` 两个 hook 点，可以在压缩前阻断、压缩后重新注入关键约束（对应上一份文档 4.1 的漂移问题）。
- Codex：`model_auto_compact_token_limit` 配置项理论上应该在接近上限时自动压缩，但 headless `exec` 模式下有社区报告显示**压缩根本不触发**，直接在撞顶时崩溃——说明这一层的实现成熟度两边并不对等。

**这一层教会我们的系统设计原则**：光把"项目文档"塞进 system prompt 是不够的，你还需要一个"压缩/摘要子系统"，否则 Agent 循环开长了必然自爆。这是几乎所有自建 agent 系统第一次上生产会踩的坑。

---

## Layer 4：执行沙箱层——解决"工具调用可能是灾难性的"

Layer 2 的循环里，"客户端执行工具调用"这一步如果不加约束，等于让一个可能产生幻觉的模型拿到了你机器的完整权限。这一层是两边差异最大、也是上一份文档重点讲的地方，这里换个角度讲**它在系统里处于什么位置**：它插在"模型请求工具调用"和"客户端真正执行"这两步之间，是一个**审批/拦截中间件**。

### Codex 的实现：矩阵式配置，双轴正交

```
approval_policy × sandbox_mode = 实际行为
```

- `approval_policy`: `untrusted | on-request | never | granular`（`granular` 可以细分到 `sandbox_approval` / `rules` / `mcp_elicitations` / `request_permissions` / `skill_approval` 五个独立开关，`on-failure` 已废弃）
- `sandbox_mode`: `read-only | workspace-write | danger-full-access`
- 再叠加 `[permissions.<profile>]` 里的 `read`/`write`/`deny`/`network_allow` 路径级白名单，可以定义多套 profile 并用 `extends` 继承

这是一个**声明式配置矩阵**，审批策略（要不要问人）和沙箱策略（能碰什么）是两个正交的轴，组合出四种以上典型工作模式（比如 `never` + `read-only` 用来跑只读代码审查任务，完全不需要人盯着）。

### Claude Code 的实现：单一状态机 + 运行时中间件

`permission_mode` 是一个单值状态：`default | plan | acceptEdits | auto | dontAsk | bypassPermissions`，不是矩阵而是一条状态轴，配合 `PreToolUse`/`PermissionRequest` hook 在运行时对每次工具调用做实时裁决——本质是把"审批策略"做成了**可编程的中间件**（你可以写脚本动态决定，而不只是选一个预设枚举），代价是治理逻辑分散在各处 hook 脚本里，不像 Codex 那样在一个 config.toml 矩阵里能看全。

**系统设计取舍对比**：Codex 用"配置矩阵"换来了可预测性和易审计性（一眼看出这个环境能干什么），Claude Code 用"运行时中间件"换来了灵活性（可以根据工具调用的具体内容动态决策），但灵活性的代价是行为不再是纯声明式的，你需要真的跑一遍才知道某条命令会不会被拦。

---

## Layer 5：协议扩展层——解决"内置工具集不够用"

Layer 4 管的是"已有工具能不能用"，这一层解决"工具从哪来"。裸 Agent 循环里工具是宿主硬编码的（Bash、Read、Edit...），要扩展就得改宿主源码。**MCP（Model Context Protocol）** 把"工具从哪来"标准化成一个进程间协议：

```
宿主进程 (Codex / Claude Code)
    │  JSON-RPC over stdio / HTTP
    ▼
MCP Server（独立进程，任何语言实现）
    │  暴露 tools/list、tools/call 等标准方法
    ▼
外部系统（数据库、内部 API、第三方 SaaS...）
```

这一层的关键系统价值：**把"扩展 Agent 能力"从"改宿主代码"降级成"起一个符合协议的独立进程"**——这也是为什么 MCP server 生态能在两个完全不同的宿主（Codex、Claude Code）之间通用：协议在这一层已经把宿主实现细节屏蔽掉了。

同一层还有一个正交的东西：**Hooks**。MCP 解决"加什么工具"，Hooks 解决"工具调用前后要不要做点什么"（校验、格式化、审计、通知）——是绑定在 Layer 2 循环的固定切入点上的"面向切面"能力，两边都在这个位置放了插件机制，只是密度不同（Claude Code 30+ 事件 vs Codex 的 hooks 目前更薄，`[features] hooks = true` 才是新加的能力）。

---

## Layer 6：状态持久层——解决"长任务需要事务语义"

Agent 循环本身是无状态的请求-响应对，但一个真实编码任务动辄改几十个文件、跑几十次工具调用——**中途出错怎么办**？这一层给循环加上"检查点"概念：

- **Claude Code：checkpoint（对话内 undo）**。每个用户 prompt 自动打一个快照，`/rewind` 可以把"文件改动"和"对话历史"分别或一起回滚——本质是给 Layer 2 循环的每一轮包了一层"影子文件系统"。局限前面讲过：只覆盖 Claude 自己编辑工具的改动，Bash 的副作用管不到，这是因为快照机制挂在 Edit/Write 工具的调用点上，而不是挂在文件系统本身（不是真正的 COW 快照或者 git commit）。
- **Codex：worktree（文件系统级隔离，而非时间点回滚）**。不是"记录下来好回滚"，而是"从一开始就物理隔离"——每个任务在独立 git worktree + 独立分支里跑，天然不会跟主目录/其他任务冲突，"回滚"直接等价于"扔掉这个 worktree"。

**系统设计上这是两种不同的事务模型**：Claude Code 类似数据库的 **MVCC/日志式回滚**（在同一份数据上记录变更历史，可以回到任意历史点），Codex 类似 **写时复制的分支隔离**（每个事务在自己的副本上跑，成功了再合并，失败了直接丢弃整个副本）。选型逻辑对应回第 3 节讲的范式区别：交互式场景更适合"能回到任意历史点"，批量异步场景更适合"独立副本，失败即丢弃"。

---

## Layer 7：编排层——解决"单个 Agent 循环干不完大任务"

到 Layer 6 为止，系统单位还是"一个 Agent 循环处理一个任务"。真实工程场景经常需要**多个 Agent 循环协同**：

- **子代理（subagent）**：在同一个宿主进程内，派生一个"迷你 Agent 循环"去处理子任务，完成后把结果汇总回父循环（Claude Code 的 `SubagentStart`/`SubagentStop`，Codex `[features] multi_agent`）。本质是 Layer 2 的递归——子代理内部还是同一套 tool_use ⇄ tool_result 循环，只是有独立的上下文窗口和独立的权限继承规则。
- **并行 fan-out**：多个独立进程各自跑一个完整 Agent 循环，靠 Layer 6 的 worktree 隔离保证互不干扰，靠编排层（人工或第三方 MCP swarm 封装）做批量派发和结果收集。
- **云端异步**：Agent 循环整个搬到远程环境执行，本地只保留一个"任务句柄"，本质是把 Layer 4 的沙箱边界从"进程级"升级成"整台远程机器级"。

**这一层没有对错，只有场景匹配**：子代理适合"任务需要共享大量上下文，只是某个子步骤想隔离一下"；并行 fan-out 适合"任务之间完全独立，拆开跑更快"；云端异步适合"任务耗时长、不需要实时盯着、本地资源不够"。

---

## 从头到尾串起来看：一个请求是怎么变成"AI 帮你重构了一个微服务"的

把 7 层串成一条时间线，看一次真实的"重构服务 X"任务是怎么被逐层处理的：

```
你输入: "重构 payments 服务，把浮点金额计算换成 Money 值对象"
   │
   ▼ [Layer 3] 系统提示词 + AGENTS.md/CLAUDE.md 被拼进 messages
     （模型现在知道"这个目录禁止改动 __generated__""必须用 Money 类型"）
   │
   ▼ [Layer 2] 模型吐出第一个 tool_use: 读取 payments 目录结构
   │
   ▼ [Layer 4] 审批层判定：read-only 操作，直接放行，不问人
   │
   ▼ [Layer 5] 如果需要读内部工单系统确认需求细节，走 MCP 工具调用
   │
   ▼ [Layer 2] 循环继续：模型请求 Edit 工具改某个文件
   │
   ▼ [Layer 4] 审批层判定：写操作，命中 workspace-write 白名单，
     Claude Code 侧同时触发 PreToolUse hook 做二次校验
   │
   ▼ [Layer 6] 改动前打一个 checkpoint（Claude Code）
     或者这整个任务从一开始就在独立 worktree 里跑（Codex）
   │
   ▼ [Layer 2] 循环重复几十轮，改文件、跑测试、看报错、再改
   │
   ▼ [Layer 3] context 快撑满，触发 compaction，摘要旧的排错过程
   │
   ▼ [Layer 7] 如果任务被拆成"payments 服务"+"billing 服务"两个独立子任务，
     两个 Agent 循环并行跑在两个 worktree 里，最后人工合并 diff
   │
   ▼ 最终输出：一组 diff / commit，等待你 review
```

**系统层面的结论**：你今天用的 Codex / Claude Code，从裸 API 的角度看，就是在一个"tool_use ⇄ tool_result"的机械循环外面，逐层裹上了"记忆注入""执行审批""协议扩展""状态回滚""多任务编排"这五层治理机制。**这五层解决的都是同一个母问题——一个会犯错、没有记忆、没有权限边界的文本生成器，怎么被安全、持续、可协作地用来做真实世界的破坏性操作（改文件、跑命令）**。理解了这一点，你在自己的场景里做技术选型时，问的问题就不再是"Codex 好还是 Claude Code 好"，而是"我的场景在这 7 层里，哪几层的成熟度和设计哲学最匹配我的风险偏好"。

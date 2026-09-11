# 多代理协同（Multi-Agent Collaboration）深挖笔记

> 本文系统梳理「多代理协同（Multi-Agent Collaboration）」的技术本质：它与子代理的架构差异、真正解决的底层问题、认知层面的价值机制，以及必须正视的反面证据与工程选型判断标准。

## 两种协调拓扑：子代理 vs 多代理协同

这两个概念不是同一维度的"版本升级"关系，而是**两种完全不同的协调拓扑**：子代理是"星型/层级"结构（一切通过主线程），多代理协同（以 Claude Code 的 Agent Teams 为代表）是"网状/对等"结构（队友之间可以直接对话）。

### 架构本质区别

Claude Code 官方文档用一句话点破了两者的核心分野：

> Unlike subagents, which run within a single session and can only report back to the main agent, you can also interact with individual teammates directly without going through the lead.

| 维度 | 子代理（Subagents） | 多代理协同（Agent Teams） |
| --- | --- | --- |
| 通信拓扑 | 单向：只能向主线程汇报，子代理之间零通信 | 双向网状：teammate 之间可以直接互发消息 |
| 任务分配 | 主线程手动/按需派发，一次一个任务，做完就回收 | 共享任务看板，teammate 可以"自领任务"（self-claim） |
| 文件写入并发控制 | 无内置机制，得靠 worktree 隔离规避冲突 | 有文件锁（file locking），一个 teammate 写文件时会加锁，阻止别的 teammate 静默覆盖 |
| 依赖关系 | 不存在"任务间依赖"概念，子代理互相不知道彼此存在 | lead 在拆解任务时编码依赖关系，协调层强制执行，不满足前置条件的任务不会启动 |
| 上下文关系 | 完全隔离，摘要（summary）回传 | 同样是 "each running in their own isolated context window"，但通信信道从"只能汇报"变成了"可以互相消息 + 共享任务列表" |

### 子代理的优劣

**优势**：

- **token 成本低、行为可预测**：单向汇报模式意味着"跑起来就是一次性的、边界清晰的委托"，评测表格明确标注 Token cost: Lower（对比 Agent Teams 的 Clearly higher）。
- **上下文纯净度最高**：完全没有 teammate 间的横向通信，主线程收到的永远是一份干净摘要，不会被"两个子代理互相拉扯讨论"的噪音污染。
- **心智负担小**：失败模式简单——子代理挂了就是挂了，不存在"这个任务卡住是因为在等另一个 teammate 的消息"这种分布式协调 bug。

**劣势**：

- **无法处理"需要讨论"的任务**。如果任务本身需要多个视角互相校验、互相提出异议（例如"安全审查发现的问题需要立刻通知正在做性能优化的另一个 agent，让它别把这个不安全的写法也用到别处"），子代理架构做不到——它天生是"编排型"，不是"协作型"。
- **主线程成为单点瓶颈**：所有信息都要先回到主线程、再由主线程决定怎么分发给下一个子代理，协调开销全压在主线程的推理能力和 context 预算上。

### 多代理协同（Agent Teams）的优劣

**优势**：

- **能做真正的横向协作**，官方例子很典型：an agent team with a security reviewer can flag a finding to the performance reviewer mid-run without stalling the whole team——这是子代理架构结构性做不到的能力。
- **内置并发写保护**：file locking 直接解决了"多个 agent 同时改同一文件"这种子代理时代只能靠 worktree 硬隔离绕开的问题，协作粒度可以下沉到"同一个文件的不同函数"这种更细的层级。
- **依赖调度自动化**：不需要主线程手动等待、手动判断"B 任务能不能开始"，协调层自己按依赖图调度。

**劣势**：

- **token 成本显著更高**——每个 teammate 都是一整个独立会话，横向通信本身也消耗 token（each teammate is a separate session）。
- **仍是实验性功能，成熟度不足**。官方 GitHub issue 显示，截至目前连文档里承诺的任务看板工具（`TaskCreate`/`TaskList`/`TaskUpdate`）都还没在运行时实现全：These tools do not exist at runtime — not for the main session, not for spawned agents...and not as deferred tools via ToolSearch，workaround 是 Agents can coordinate through SendMessage and prompt-based instructions, but without the task tools there is no shared task board, no dependency gating。也就是说"共享任务看板 + 依赖门控"这套卖点，在当前版本里**部分是文档画的饼，还没完全对齐实现**。另一份第三方溯源文档也将其形容为 "\~60% complete - Infrastructure exists, core implementation missing"。
- **调试复杂度陡增**：分布式协调天然多一类 bug——消息路由错、死锁式的"互相等对方任务完成"、mailbox 消息顺序问题，这些在子代理架构里根本不存在。

### Codex 侧：目前只有"子代理"这一层

这是最值得单独强调的一点：**Codex 目前完全对应"子代理"这一档，还没有对标 Claude Code Agent Teams 的"多代理协同"能力**。

GitHub feature request 明确指出：Current subagent workflows are useful for parallel work, but coordination is mostly parent-mediated: subagents run in separate agent threads and the parent agent waits for results, then consolidates them——用户明确是在**请求官方新增**一个共享协调层（`shared_scope`）。

Codex 的多个子代理之间没有 SendMessage、没有共享任务看板、没有文件锁——它们唯一的"协同"发生在父线程收集完所有结果之后，由父线程做整合。这是两边产品成熟度上一个实打实的差距点，不是设计哲学不同，而是 **Codex 这一层能力还没做出来**。

## 多代理协同解决的三类结构性问题

如果只看功能对比，容易停留在"能不能让 agent 互相发消息"这个表面。往下钻一层，多代理协同真正解决的是三类结构性问题。

### 问题一：单个 context 装不下"互不污染的独立推理链"

这是最容易被忽略的一点。哪怕父 agent 的 context window 再大，它本质上是**一条**推理链——所有信息进入这个 context 之后，都会互相影响后续生成的每一个 token。你不可能在同一个 context 里让"假设 A 成立"和"假设 A 不成立"两条思路真正独立地展开而不互相渗透——先写的内容天然会锚定后写的内容。

这正是子代理架构（hierarchical/star topology）结构性做不到的事：子代理各自独立，**但它们的独立性只在"执行"层面**，一旦结果汇总回父 agent 的单一 context，所有独立性就在综合这一步被压扁成一条线性推理。多代理协同的价值在于——**综合过程本身也是分布式的**，不需要先塞进一个单一 context 才能互相比对。

### 问题二：层级汇报是有损漏斗，点对点通信是保真传递

子代理向父线程汇报，天然要经过"压缩成摘要"这一步——子代理中间的推理、读的文件、犯的错都留在自己 context 里，只有一句话回传。这意味着：**如果 B 子代理正需要 A 子代理发现的某个极其具体的细节（不是结论，而是"哪一行代码、哪个边界条件"），这个细节大概率在 A 向父线程压缩成摘要的过程中已经被丢掉了**，父线程再转述给 B 时又经过一次转述损耗——信息传两次就衰减两次。

peer-to-peer 消息把这个"必须经过中枢转述"的漏斗直接短路掉，让"发现问题的那个 agent"和"需要这个信息的那个 agent"点对点对话，信息保真度只损耗一次而不是两次。Claude Code 官方的例子——安全审查 agent 中途直接把发现告诉性能优化 agent——本质就是在避免"等父线程汇总、父线程再压缩转述"这个额外的有损环节。

### 问题三：并发状态一致性——分布式系统的老问题换个载体

多个 agent 同时改同一批文件，本质和多线程/多进程并发写共享资源是**同一个问题**——需要锁（mutex）、需要判断谁先谁后、需要防止"静默覆盖"。子代理架构里没有这个机制，只能靠"物理上不共享同一份文件"（worktree 隔离）来回避问题，代价是隔离粒度粗、协作深度浅。Agent Teams 的 file locking 相当于把传统分布式系统里的加锁语义直接搬进了多 agent 编排层——这不是什么全新发明，而是把已经解决过一次的问题重新在新的执行体（agent）上解决了一遍。

## 认知本质：对标"减少相关误差"，而非"让模型更聪明"

这是最值得展开的一点，涉及一个专门的研究方向——**multi-agent debate**。MIT/Google Brain 的一篇奠基性论文提出的框架是：多个模型实例各自独立生成初始答案，然后互相阅读、互相批评，再更新自己的答案，反复几轮。

效果相当显著：

- 在算术任务上，debate 达到 **81.8%** 准确率，对比单模型的 **67.0%** 和单模型自我反思的 **72.1%**；
- 在跨模型 debate（ChatGPT + Bard）场景下，20 道 GSM8K 题目解出 **17 道**，而单个模型各自只能解出 **11–14 道**。

这个跨模型结果尤其能说明问题：**异质的推理路径互相纠错的效果，明显好于同一个模型自己反思自己**。

背后的机制解释了"多代理协同"在认知层面到底解决了什么：它不是靠"人多力量大"堆算力，而是靠**制造出多条独立采样的推理链，然后让分歧本身变成信号**——如果多条独立链条收敛到同一个答案，这种收敛比单链自信输出更值得信任；如果没收敛，分歧点本身就精确定位了"哪里存在不确定性"。这跟统计学里"减少相关误差（correlated error）才能真正提升集成效果"是同一个道理——单个 agent 自己反思自己，产生的仍然是高度相关的错误（同一个模型的同一类盲点很难通过反思自己发现），而多个独立视角（不同 agent、不同起点、不同分工）更有机会覆盖到彼此的盲区。

## 反面证据：多代理协同不是免费的

如果只讲到这里就停，会显得像在无脑吹多代理架构。深入一层看，有两个关键的反面证据必须放进来。

### 第一，"等预算"对比下，单 agent 不一定输

3 个 agent 辩论 2 轮，大约消耗单次调用 6 倍的推理算力，而原始论文并没有做"给单 agent 同样多的算力预算"这个对照实验。后续有研究专门补上了这个对照——在多跳推理任务上，给单 agent 和多 agent 系统同样的 token 预算后，发现"单 agent 系统可以匹配甚至超过多 agent 系统"的表现。

这是一个非常关键的反驳：**如果你本来就打算多花几倍 token，直接把这些 token 让单个 agent 做更长的自洽采样（self-consistency）或更深的推理，效果不一定比拆成多个 agent 差**。多代理协同买到的不是"计算量本身的效率"，而是"独立视角"这个更结构化的东西——如果这个"独立性"没有真正被制造出来（比如所有 agent 都是同一个模型、同一套 prompt），那多花的算力就白花了。

### 第二，同质 agent 容易出现"集体幻觉"而非互相纠错

有研究对 100 个 debate 失败案例做了人工分析，发现 **65%** 的失败是"多个 agent 互相强化了同一个错误答案"，而不是互相纠正。论文原文自己也承认，agent 有时会"自信地断言自己的答案是对的"，哪怕这个答案是错的、且是几个 agent 共同收敛到的错误答案——原因很直接：**当所有 agent 共享同一套训练分布，它们大概率也共享同一类盲点，这时候"辩论"反而会放大错误而不是纠正它**，因为每个 agent 看到"别人也这么想"会把这当成一种确认信号。

相关的还有一种叫"错误从众"（Incorrect Conformity）的失败模式——本来推理正确的一方，读了同伴错误的回应之后反而放弃了自己正确的判断，这正好是 debate 框架设计初衷的反面。

## 上下文隔离机制：默认隔离 + 显式例外通道

直接结论：**默认都不共享**，都是"隔离的独立上下文窗口 + 摘要回传"模式，但两边都留了一个"例外通道"可以显式继承上下文。

### Claude Code：官方文档明确写死"默认隔离"

官方文档原话：

> Subagents are specialized AI assistants that handle specific types of tasks. Use one when a side task would flood your main conversation with search results, logs, or file contents you won't reference again: the subagent does that work in its own context and returns only the summary.

在 "Manage subagent context" 一节里说得更绝对：

> Each subagent starts with a fresh, isolated context window. It doesn't see your conversation history, the skills you've already invoked, or the files Claude has already read. Claude composes a delegation message that summarizes the task, and the subagent works from there.

三个佐证细节：

- **CLAUDE.md 是"加载一份新的"，不是"共享同一份内存"**：子代理启动时会重新加载一遍完整的 CLAUDE.md 层级，但这是独立发生的加载动作，不是引用父会话已经展开好的那份 context。
- **压缩互不影响**：子代理的 transcript 存在独立文件里，when the main conversation compacts, subagent transcripts are unaffected. They're stored in separate files；反过来子代理自己触发压缩也不会影响父会话，这从侧面证明两边的 context 是两套独立状态机。
- **返回信道极窄**：只有子代理的最终一条消息会回传给父会话，中间的工具调用、读的文件、推理过程全部留在子代理自己的 context 里，永远不注入父会话——only the subagent's final message returns to the parent. Every intermediate tool call, every file the subagent read, every line of test output it processed — all of it stays inside the subagent's context and is never injected into yours。

**唯一的例外是 fork（**`/subtask`** 或 Claude 主动请求 fork 类型）**：普通子代理是 "Fresh context with the prompt you pass"，fork 则是 "Full conversation history"，并且明确指出 A fork inherits everything the main session has at the moment it spawns. Any other subagent starts fresh from its definition。也就是说"共享上下文"在 Claude Code 里是一个**需要显式触发的特殊模式**，而不是子代理的默认行为。

### Codex：同样默认隔离，且有官方未实现的 feature request 作反证

Codex 官方文档把子代理机制的设计动机讲得很直白——就是为了解决 "context pollution" 和 "context rot"：

> Even with large context windows, models have limits. If you flood the main chat...with noisy intermediate output such as exploration notes, test logs, stack traces, and command output, the session can become less reliable over time.
>
> Subagent workflows help by moving noisy work off the main thread: Keep the main agent focused on requirements, decisions, and final outputs. Run specialized subagents in parallel for exploration, tests, or log analysis. Return summaries from subagents instead of raw intermediate output.

代价也写得很明白——因为每个子代理是完全独立起的一次模型调用，不是复用父会话已经攒好的 context：Because each subagent does its own model and tool work, subagent workflows consume more tokens than comparable single-agent runs。如果子代理真的共享父会话的 context，不会存在"更耗 token"这个说法。

**最关键的反证来自一份 Codex 的官方 GitHub feature request**，用户明确抱怨"现在没有共享协调层"，要求官方加一个：

> Current subagent workflows are useful for parallel work, but coordination is mostly parent-mediated: subagents run in separate agent threads and the parent agent waits for results, then consolidates them.

这条 issue 想要新增的是可选的 `shared_scope`（共享消息总线），恰好反向证明了**当前 Codex 子代理之间、子代理与父代理之间没有共享 context，只有"结果消费"关系**——这是一个功能请求还没被合并的状态，不是已有能力。

Codex 侧同样有"可以显式继承上下文"的口子，但目前是命令式而非声明式的：另一份 issue 提到 spawn\_agent exposes fork\_context, but custom subagent frontmatter/config cannot express the same capability——说明 Codex 底层 API 有 `fork_context` 参数可以让子代理拿到调用方的 thread context，但这只在临时的 spawn 调用里可用，没有像 Claude Code 的 `isolation`/`fork` 那样被固化成一个正式的、可配置的声明式字段。

### 一个容易混淆的点："共享工作区" ≠ "共享上下文"

第三方对 Codex 多代理编排的实测总结提到：

> Subagents run inside one session and share the workspace while working on separate threads. Git worktrees give each agent its own branch and directory, so parallel feature work does not collide.

这里的 "share the workspace" 说的是文件系统/git 仓库这个物理层面，子代理仍然是 "separate threads"——各自独立的对话线程和上下文窗口，只是可能落在同一个目录或者各自 worktree 里。这个"文件系统共享 vs 上下文共享"的区分，在 Claude Code 那边也完全一致：`isolation: worktree` 控制的是文件系统隔离，和 context 是否共享是两条完全正交的轴。

### 一句话总结

| 维度 | 默认行为 | 例外机制 | 关键证据 |
| --- | --- | --- | --- |
| **Claude Code** | 子代理独立、全新 context window，只回传最终一条消息 | `fork`（`/subtask`），显式继承完整对话历史 | 官方文档原话 "fresh, isolated context window...doesn't see your conversation history" |
| **Codex** | 子代理各自独立 thread、独立模型调用，父代理收集摘要 | `spawn_agent` 的 `fork_context` 参数（imperative，未声明式暴露） | 官方文档强调 "more tokens" 因为每个子代理独立跑；GitHub issue 明确说明当前无共享协调层 |

两边的设计动机也一致——都是把"防止长 context 里塞满探索噪音、导致主线索质量下降"当成子代理存在的**第一理由**，而不是并行加速。如果真共享 context，这个核心卖点就不成立了，这也是为什么"默认不共享"几乎是这类架构的必然选择，而不是两家各自恰好做出的选型。

## 工程实践：怎么选

多代理协同真正值得投入的场景，需要同时满足两个条件：

1. **任务本身存在"多条本该独立、但会互相影响执行过程"的推理/工作路径**——不是简单的并行拆分，而是拆分之后彼此的中间发现会改变对方该怎么做。这是子代理架构结构性做不到、需要点对点通信/共享状态才能覆盖的场景。
2. **参与协同的 agent 要有真正的异质性**——不同模型、不同 system prompt、不同的信息来源（比如一个 agent 专门去查外部文档、另一个纯靠代码推理），而不是"同一个模型开三个实例互相点头"。前面引用的跨模型 debate 结果和"异质协作 debate 显著优于同质协作"的结论，都在指向同一件事：**多代理协同买的是独立视角带来的交叉验证能力，不是 agent 数量本身**；如果做不到真正异质，大概率只是在用几倍的 token 制造一场自我肯定的表演。

给一个直接可操作的判断标准：

- **任务可以干净拆成互不依赖的子块、只要最终结果** → 子代理。成本低、可预测、Codex/Claude Code 都成熟可用。
- **任务需要"边做边对齐"、子任务之间存在互相影响或需要实时校验对方结论** → 目前只有 Claude Code 的 Agent Teams 能做（还得接受它是实验性功能、工具链没完全跟上文档承诺这个现实）；Codex 这条路线暂时得自己在编排层手搓（比如轮询各子代理输出、拿到中间结果后手动喂给另一个子代理）。
- **涉及并发写同一批文件** → 子代理架构必须靠 worktree 硬隔离规避冲突；Agent Teams 有 file locking 原生支持，细粒度协作的天花板更高，但要接受它还在实验阶段可能不稳定这个代价。
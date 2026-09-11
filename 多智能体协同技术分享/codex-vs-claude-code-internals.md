# Codex vs Claude Code：底层机制深度拆解

> 面向对象：已经熟练使用 CLI / AGENTS.md / CLAUDE.md 的重度用户。本文档不讲"怎么装""怎么写第一条 prompt"，只讲两者内核实现细节、边界条件、以及会让人半夜爬起来 debug 的坑。所有配置项和行为均来自官方文档 / 源码 PR，标注了版本敏感点。

---

## 1. Codex 的底层机制

### 1.1 沙箱不是"权限提示"，是内核级 syscall 拦截

Codex 的沙箱不是应用层的路径白名单，而是操作系统原生的强制访问控制：

- **macOS**：使用 **Seatbelt**（`sandbox_exec`，`.sbpl` profile）。项目里能看到专门拆分出来的 `codex-sandboxing` crate，网络策略是一个独立的 `seatbelt_network_policy.sbpl` 文件，在 `[sandbox_workspace_write]` 未显式开启 `network_access = true` 时，**在 OS 层面无条件下发拒绝规则**——不是 Codex 应用层判断"要不要发请求"，而是内核直接把 socket syscall 拦下来。这也是为什么很多人发现"config.toml 里设了 `network_access = true` 但 curl 还是超时"——Seatbelt 的规则生成逻辑曾经不读这个字段，必须用 `--sandbox danger-full-access` 整体禁用沙箱才能绕过。
- **Linux**：使用 **Landlock**（内核 5.13+ 的 LSM）+ seccomp 过滤 syscall。同样是内核态强制，不依赖 ptrace 或 LD_PRELOAD 这类可被绕过的用户态钩子。

**可复现实验**：

```bash
# 观察 Seatbelt 网络拒绝是内核级而非应用级
codex --sandbox workspace-write -c sandbox_workspace_write.network_access=false \
  exec "curl -m 3 https://example.com"
# 现象：不是"Codex 拒绝执行"的提示，而是 connect() 本身超时/ECONNREFUSED
# 说明请求已经离开了 Codex 进程边界，卡在内核的 Seatbelt profile 上
```

工程含义：**Codex 的沙箱逃逸面是 OS 级的**，比 Claude Code 纯用户态的权限询问机制更硬，但也意味着调试"为什么这条命令跑不通"时，你要去看 sbpl / landlock 规则而不是应用日志。

### 1.2 AGENTS.md 的固化规则：一次性构建，不是热加载

容易被忽略的三点：

1. **只加载一次**：Codex 在**启动时**构建一次指令链（TUI 下通常等于每次 `codex` 进程生命周期一次），运行期间修改 AGENTS.md **不会**触发热重载。这与 Claude Code 的 `InstructionsLoaded` 事件（支持 `nested_traversal`、`path_glob_match` 等触发方式，可以在会话中懒加载子目录规则）形成鲜明对比。
2. **精确覆盖优先级**：
   ```
   ~/.codex/AGENTS.override.md  (存在则用，否则退回 AGENTS.md，只取一个，不叠加)
   → <git-root>/AGENTS.md ... 逐级向下直到 cwd 每一层 AGENTS.override.md > AGENTS.md
   ```
   每一层最多生效一个文件，`.override.md` 命中则**替换**而非追加同级的 `AGENTS.md`。
3. **硬截断**：`project_doc_max_bytes` 默认 **32 KiB**，从根向叶拼接，超限直接截断，空文件跳过。这意味着如果你的 monorepo 根目录 AGENTS.md 写得很啰嗦，子目录里更关键的规则可能根本没进 context——**没有任何报错提示你哪部分被砍了**。

**可复现实验**：

```bash
codex --ask-for-approval never "Summarize the current instructions."
# 对比根目录 vs 子目录运行结果，验证覆盖优先级
codex --cd services/payments --ask-for-approval never \
  "Show which instruction files are active."
```

排查线索：`~/.codex/log/codex-tui.log` 或 `session-*.jsonl` 能审计到底加载了哪些文件——出问题先看这个，而不是猜。

### 1.3 子代理的 Git Worktree 隔离：三种执行模式的本质差异

Codex app（不仅是 CLI）支持三种运行模式，很多人只知道 Cloud，不知道本地也有 worktree 隔离：

| 模式 | 运行位置 | 修改对象 | 典型坑 |
|---|---|---|---|
| Local | 本机 | 主 checkout | 并行任务互相污染 `git status` |
| Worktree | 本机 | 独立目录 + 独立分支 | 依赖 `.worktreeinclude` 声明哪些文件需要复制到新 worktree（如 `.env`、`node_modules` 软链），漏配会导致 worktree 里跑不起来 |
| Cloud | 远程沙箱 | 远程环境 | 与本地文件系统完全隔离，无法访问本机未提交的改动 |

**关键机制**：Worktree 模式下，每条并行任务线都拿到独立的目录 + 独立分支，diff 互不干扰，可以分别 review / merge / 丢弃。第三方 MCP 封装（如 `codex-mcp-swarm`）把这个能力暴露成 `worktree: true` 参数，本质是在每次任务派发前跑一次 `git worktree add`，任务结束后可选择保留 worktree 供检查（`codex_cancel` 保留 worktree 便于事后分析失败原因）。

工程含义：如果你在写编排层去并行调度多个 Codex 子任务，**永远走 worktree 隔离**，不要在同一个 checkout 里跑多个 Codex 进程——否则会遇到两个进程同时写同一个文件、`git add` 互相踩踏的经典并发 bug。

### 1.4 工具输出静默截断：head+tail 拼接，而不是简单 cutoff

这是最容易被忽视、也最容易导致"模型看起来在瞎编"的机制。Codex 的 exec 输出截断策略：

- **双重限制**：字节数（默认约 10KB）+ 行数（256 行）同时生效，谁先触发谁说了算。
- **截断方式**：当按字节截断时，**不是保留头部丢尾部**，而是头尾各留一半（比如各 5KB），中间挖空，标注 `[... omitted N lines ...]`。这是从"只留头导致 cargo/构建报错信息在尾部被吃掉"这个真实 bug 修复而来的。
- **危险边界**：unified exec 的 `HeadTailBuffer` 上限是 **1 MiB**，超过这个量级采集端本身就会在中间丢数据——如果调用方没有显式设置 `max_output_tokens`，某些 code-mode 路径会尝试返回"raw_output"，但这个 raw 值本身可能已经是被采集器掐过的头尾拼接，**没有任何 `...truncated...` 标记**，脚本或模型会把这段拼接当成完整数据处理。

**可复现实验**：

```bash
codex exec "for i in $(seq 1 50000); do echo line_$i; done"
# 观察返回内容里 line_1..line_N 和 line_(50000-M)..line_50000 之间是否有明显跳号
# 中间的行号是否连续，能直接验证 head+tail 截断而非顺序截断
```

工程含义：**长日志/长测试输出永远重定向到文件再让 Codex 读文件的头/尾/grep 结果**，不要指望一次 exec 拿到完整输出——尤其是 CI 日志、`cargo test` 这类"错误信息在最后几行"的场景，虽然新策略已经保底保留尾部，但中间的堆栈上下文依然是黑洞。

---

## 2. Claude Code 的底层机制

### 2.1 本地 Agent 循环：8 个阶段，而不是"发消息—等回复"

Claude Code 的官方 hooks 文档给出了明确的生命周期图：一次会话是

```
Setup(可选) → SessionStart
 → 每轮: UserPromptSubmit → UserPromptExpansion(斜杠命令展开)
    → 嵌套 agentic loop: PreToolUse → PermissionRequest → PostToolUse
       → PostToolUseFailure → PostToolBatch → SubagentStart/Stop → TaskCreated/Completed
 → Stop / StopFailure
 → TeammateIdle → PreCompact → PostCompact → SessionEnd
```

事件按频率分三档：**每会话一次**（SessionStart/SessionEnd）、**每轮一次**（UserPromptSubmit/Stop/StopFailure）、**每次工具调用**（PreToolUse/PostToolUse，但 `EndConversation` 调用会跳过这两个）。

这套循环目前暴露了 **30+ 个 hook 事件**（`PermissionDenied`、`WorktreeCreate`、`FileChanged`、`Elicitation`/`ElicitationResult`、`PreModelSwitch`/`PostModelSwitch` 等都是后期陆续加的），远超大多数教程只讲的 `PreToolUse`/`PostToolUse`。

### 2.2 Hooks 钩子：exit code 语义远比"2 就是拒绝"复杂

重度用户最容易踩的三个细节：

**(1) exit code 2 并非所有事件通用**。`PermissionRequest`、`StopFailure`、`Notification` 等事件里 exit 2 **被忽略**；`WorktreeCreate` 反过来是"任意非零退出码都失败"；`PreToolUse`/`Stop`/`SubagentStop`/`PreCompact`/`PreModelSwitch` 等才是"exit 2 = 真正阻断"。写通用 hook 脚本时不能假设 exit 2 是万能刹车。

**(2) JSON 输出优先于 exit code，但 exit 2 的阻断效果无法被 JSON 覆盖**。也就是说即便你在 stdout 打印 `permissionDecision: "allow"`，只要进程 exit 2，该阻断的事件依然阻断——JSON 只能提供理由文案，不能翻案。

**(3) 判定 stdout 是 JSON 还是纯文本的规则很微妙**：以 `{` 开头 `}` 结尾才按 JSON 解析；多行输出中如果每行都能独立解析成 JSON 但没有一行设置了合法字段，则整体按纯文本处理；只要有一行设置了字段，整体判定为"解析失败"（非阻断错误，事件继续执行，但会在 transcript 里出现 `<hook name> hook error`）。这意味着如果你的 hook 脚本的 shell profile 在启动时打了一行欢迎语到 stdout，会直接污染 JSON 判定，导致 hook 静默失效。

**(4) 超时不等于阻断**：`PreToolUse` 上一个跑超时的 `command`/`http`/`mcp_tool` hook **不会**阻断工具调用（走正常权限流程），但 Agent SDK 的 callback hook 超时**会**阻断——两条链路行为不一致，混用时要小心。

**可复现实验**（用官方文档给的经典例子验证匹配链路）：

```jsonc
// .claude/settings.json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "if": "Bash(rm *)",
        "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh"
      }]
    }]
  }
}
```
```bash
#!/bin/bash
# .claude/hooks/block-rm.sh
COMMAND=$(jq -r '.tool_input.command')
if echo "$COMMAND" | grep -q 'rm -rf'; then
  jq -n '{hookSpecificOutput:{hookEventName:"PreToolUse",permissionDecision:"deny",permissionDecisionReason:"blocked"}}'
else
  exit 0
fi
```
验证点：故意让 Claude 执行 `echo $(rm -rf /tmp/x)`（命令替换里藏 `rm`），观察 hook 依然会触发——因为 `if` 字段对 Bash 的匹配会深入检查 `$()` 和反引号内的子命令，这是很多人写 `if` 规则时会漏掉的攻击面。

### 2.3 MCP 返回大小限制：25000 token 硬顶，10000 就开始警告

环境变量层面的限制经常被忽略：

| 变量 | 作用 | 默认值 |
|---|---|---|
| `MAX_MCP_OUTPUT_TOKENS` | 单次 MCP 工具响应最大 token 数 | 25,000（超过 10,000 就会警告） |
| `BASH_MAX_OUTPUT_LENGTH` | Bash 工具单次输出最大字符数 | 约 30,000 字符，**中间截断**（保留头尾） |
| `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` | 单次文件读取最大 token | 文件内容超过 25,000 token 直接报错拒绝返回，而不是截断 |

注意 `Read` 工具和 `Bash` 工具的失败模式不同：**Bash 是静默中间截断**（你拿到的是不完整但"看起来完整"的输出），**Read 工具是直接报错**（`File content (47382 tokens) exceeds maximum allowed tokens (25000)`）。如果你的 MCP server 返回一个大 JSON blob，踩中的是 `MAX_MCP_OUTPUT_TOKENS`，同样是硬截断，且默认没有 UI 提示你数据被砍了多少——除非你主动去读调试日志。

**工程含义**：给 MCP server 设计返回值时，服务端自己要做分页/摘要，不要指望 Claude Code 客户端帮你优雅截断——它只会粗暴地砍掉超限部分。

### 2.4 Checkpoint 会话回滚：只回滚"Claude 编辑工具"做的改动

`/rewind`（或双击 `Esc Esc`）的关键限制，很多人不知道：

- **每次用户 prompt 自动创建一个 checkpoint**，可以选择恢复 **Both / Code only / Conversation only** 三种粒度。
- **checkpoint 只追踪 Claude 自己的文件编辑工具**做出的改动（Edit/Write 等），**不追踪 Bash 工具执行的副作用**——如果 Claude 用 `Bash` 跑了 `rm -rf` 或者数据库迁移脚本，rewind 完全无法撤销这部分。这是官方文档反复强调的"one important thing it does not track"。
- 持久化 30 天（`cleanupPeriodDays` 可配置），跨 session resume 依然可用，但**目前没有官方支持的 headless/`-p` 模式恢复接口**（社区一直在提 issue 要求暴露 `--checkpoint <uuid>` 之类的非交互恢复方式，截至目前仍未合并）。

**工程含义**：checkpoint 是"文件编辑的本地 undo"，**不能替代 git commit**，尤其在你允许 Claude 用 Bash 做有副作用操作（装包、跑迁移、调用外部 API）的场景下，务必额外用 git 或者显式快照兜底。

### 2.5 CLAUDE.md 加载优先级：拼接而非覆盖，local 在同级最后生效

和 Codex 的"同级只取一个文件"不同，Claude Code 的模型是**全部拼接进 context，靠位置而非删除来实现优先级**：

```
企业策略 (macOS: /Library/Application Support/ClaudeCode/CLAUDE.md
          Linux: /etc/claude-code/CLAUDE.md)
  ↓（最先加载，最广泛，但注意：越靠前的文件在长 context 里反而越容易被"稀释"）
用户内存 ~/.claude/CLAUDE.md
  ↓
项目内存 ./CLAUDE.md 或 ./.claude/CLAUDE.md（团队共享，可提交到 git）
  ↓
项目内存(本地) ./CLAUDE.local.md（已弃用，但仍兼容；同级追加在 CLAUDE.md 之后，代表"个人覆盖"）
```

三个容易被忽略的细节：

1. **递归但不是全量预加载**：从 cwd 向上递归到（但不包括）文件系统根目录，一路读到的 CLAUDE.md 全部在启动时加载；但 **cwd 子树下嵌套的 CLAUDE.md 不会在启动时加载**，只有当 Claude 实际读取该子目录下的文件时才懒加载进来（对应 `InstructionsLoaded` hook 的 `nested_traversal` / `path_glob_match` 触发原因）。
2. **`CLAUDE.local.md` 已被官方标记为 deprecated**，推荐用 `@import` 语法替代（`@~/.claude/my-personal-rules.md`），原因是 import 在多 git worktree 场景下表现更好——`.local.md` 是路径耦合的，worktree 一多就会出现"每个 worktree 都要单独维护一份本地文件"的问题，而 `@import` 可以指向用户 home 目录下的共享路径。
3. **导入深度上限为 5 层**，超过会静默停止展开，这个和 Codex 的 32KB 字节硬截断是两种完全不同的"防止指令膨胀失控"策略：Codex 是按大小砍，Claude Code 是按导入链路深度砍。

**可复现实验**：

```bash
# 会话内直接确认到底加载了哪些内存文件，而不是靠猜测
claude
> /memory
```

---

## 3. 两者 Agent 范式的本质区别

这是很多人只看到"两个都是终端里敲命令的 AI coding agent"，但底层假设完全不同：

| 维度 | Codex | Claude Code |
|---|---|---|
| 默认执行模型 | 偏向**异步/云端沙箱**：Cloud 模式直接脱离本机，Local/Worktree 也强调"派发后不打断"，适合"扔任务走人" | 偏向**本地交互式代理**：设计假设是你就坐在终端前，随时用 `Esc Esc`、`/rewind`、Plan Mode 打断和纠偏 |
| 隔离粒度 | 进程级 + 内核级沙箱（Seatbelt/Landlock）+ worktree 级文件系统隔离，隔离是**默认前提** | 权限系统 + hooks 是**可插拔的护栏**，默认更信任交互式用户实时把关，隔离（worktree/sandbox）是可选能力而非默认心智模型 |
| 指令持久化哲学 | AGENTS.md 一次性构建、按目录严格覆盖（同级只保留一份），偏"配置即代码"，改了要重启生效 | CLAUDE.md 全部拼接 + auto memory 自我学习（Claude 自己写的 `MEMORY.md`，限 200 行/25KB），偏"持续累积的活文档"，支持会话内热更新（`ConfigChange`/`FileChanged` 事件） |
| 纠错机制 | 更依赖"重新派发一个干净的 worktree 任务"，因为异步/云端模式下没有实时对话可以打断 | 更依赖"当场打断 + rewind 回滚"，因为交互式循环本身就是为实时纠偏设计的 |
| 输出治理 | 静默 head+tail 截断，默认更粗暴——因为异步场景下没人盯着实时输出去做二次确认 | 分层限制（Bash 30K 字符中间截断 / MCP 25K token 硬顶 / Read 直接拒绝），且有 `PostToolUse`/`PostToolBatch` hook 可以在截断前介入做二次处理 |

一句话总结：**Codex 的默认假设是"我派发完就走开"，所以一切安全边界都做成不可逾越的硬隔离；Claude Code 的默认假设是"我人还在这盯着"，所以更多做成可打断、可回滚、可热更新的软边界。** 这直接决定了下面第 5 节的编排策略。

---

## 4. 隐性坑清单

### 4.1 长会话上下文漂移（Context Drift）

- Claude Code：`PreCompact`/`PostCompact` 事件说明官方也承认压缩会丢信息——用 `PostCompact` hook 注入"压缩后重新提醒关键约束"是标准做法（社区常见模式：`post-compaction hook` 重新注入项目背景、角色设定、关键规则）。**如果你没配这个 hook，每次自动压缩后 Claude 对早期上下文的把握会明显退化**，尤其是"不要改这个文件""这个函数是故意这样写的"这类隐性约束最先丢失。
- Codex：已知问题是 **headless `codex exec` 模式下 auto-compaction 可能根本不触发**——即便配置了 `model_context_window`/`model_auto_compact_token_limit`，社区反馈会话在触顶前不压缩，直接崩溃报 "ran out of room"，重试还会进入死循环（因为每次重试都在原会话基础上累加）。**排查线索**：看 session 文件里有没有出现 `compact` 事件；如果 `turn_context` 后紧跟 `task_complete` 且 `last_agent_message: null`，大概率是压缩静默失败。

### 4.2 工具返回超大输出丢失

前面 1.4 / 2.3 已经详细讲了机制，这里补充一个跨工具的通用排查方法：**任何时候模型的回答和你本地实际文件内容对不上，先怀疑截断，而不是怀疑"模型幻觉"**。具体做法：

```bash
# Claude Code：主动看 debug log 确认 Bash 输出是否被截断
claude --debug 2>&1 | grep -i "truncat"
# Codex：观察 raw_output 头尾行号是否连续
```

### 4.3 模型幻觉执行命令（Hallucinated Command Execution）

两个具体的高发场景：

1. **命令替换/变量展开导致的权限逃逸**：Claude Code 的 `if` 匹配表已经明确指出——当命令是 `$TOOL git push` 这种"工具本身无法判断变量展开成什么"的形式时，**Claude Code 会选择直接放行 hook（也就是说无法判定时倾向于跑）**。这是一个明确的设计取舍：`if` 过滤是 best-effort，**真正的强制拦截要用权限系统（permission rules），而不是指望 hook 的 `if` 字段能扛住所有绕过姿势**。
2. **PermissionDenied 的 retry 语义**：Claude Code 在 auto 模式下拒绝一次工具调用后，如果 hook 返回 `hookSpecificOutput.retry: true`，模型会认为"可以再试一次"——但如果拒绝是"no-verdict"（没有分类器判定结果的拒绝），**这个 retry 标记会被忽略**。如果你的编排逻辑依赖"拒绝后自动重试"，要确认触发的是哪种拒绝路径，否则会出现模型卡死在"以为能重试但其实不能"的状态。

### 4.4 网络策略配置生效延迟/不一致（Codex 特有）

如 1.1 所述，`sandbox_workspace_write.network_access = true` 在部分 Codex 版本的 macOS Seatbelt 实现里**不生效**（Linux Landlock 路径是正常读取这个配置的），必须用 `--sandbox danger-full-access` 整体降级沙箱。**如果你的 CI/自动化脚本依赖"仅开网络、其他限制不变"这种精细化沙箱策略，先在目标平台上实测，不要直接信任 config.toml 字段名的字面含义。**

---

## 5. 混合编排方案：Codex 批量异步 + Claude Code 本地深度重构

结合第 3 节的范式差异，给出一个可落地的分工模型：

```
                    ┌─────────────────────────────┐
                    │   任务分诊 (人工 or 编排层)   │
                    └───────────────┬─────────────┘
                                    │
            ┌───────────────────────┴───────────────────────┐
            ▼                                                ▼
   批量 / 并行 / 低风险任务                          单点 / 深度 / 高风险任务
   （翻译、样板代码、多语言同步、             （核心模块重构、跨文件依赖梳理、
    独立 bugfix、CI 修复）                       需要人工实时把关的架构改动）
            │                                                │
            ▼                                                ▼
   Codex Cloud / Worktree 模式                        Claude Code 本地交互式会话
   - 每个任务一个独立 worktree                         - Plan Mode 先出方案，人工 review
   - danger-full-access 仅在隔离环境内开启              - hooks 做质量门禁
     （因为已经有 worktree/容器边界兜底）                  (PostToolUse 跑 lint/type-check)
   - 用 codex-mcp-swarm 之类的封装做                    - 用 /rewind 做即时纠错
     batch wait，一次拿回所有 diff                       - CLAUDE.md 固化架构约束，
   - 产出是"可审查的 diff 集合"，                          auto-memory 记录本次重构的
     由人工或下游 Claude Code 会话做二次审查                 踩坑经验
```

**具体编排建议**：

1. **用 Codex 做"扇出"（fan-out）**：一个需求拆成 N 个互不依赖的子任务（比如给 20 个微服务同步升级同一个依赖版本），每个子任务分配独立 worktree，走 `codex exec` 或第三方 MCP swarm 封装并行跑，`codex_wait` 批量收集结果。这类任务的特点是**单次任务的错误代价低、容易独立验证**（跑一下测试就知道对不对），异步/云端沙箱的隔离性价比最高。
2. **用 Claude Code 做"扇入"（fan-in）+ 精修**：Codex 批量产出的 diff 汇总后，用 Claude Code 本地会话做**跨文件一致性检查**——这一步天然需要交互式判断（"这个模式在 A 服务里改成这样，在 B 服务里是不是应该保持原样，因为它的调用方语义不同"），Plan Mode + hooks 质量门禁比异步沙箱更适合这种需要持续人工介入的场景。
3. **CLAUDE.md 和 AGENTS.md 不要各写各的**：把共享的硬约束（禁止改动的文件、必须用的测试命令、代码风格）同时写进两边，避免团队里"Codex 那边这样但 Claude Code 那边又是另一套规则"的割裂——见第 6 节的联动配置示例。
4. **把 Codex 的静默截断问题交给 Claude Code 兜底**：如果 Codex 批量任务产出的日志/diff 体积很大，让 Claude Code 侧的 hook（`PostToolUse` 读取 Codex 产出文件时）主动做分片摘要，而不是指望 Codex exec 一次性把完整日志吐给编排层。

---

## 6. 高级配置样例

### 6.1 CLAUDE.md（项目级，含分层规则 + hooks 联动）

```markdown
# CLAUDE.md

## 架构约束（禁止 AI 自行决定的部分）
- 数据库迁移必须通过 `scripts/migrate.sh`，禁止直接执行 DDL
- `src/legacy/` 目录下代码只做最小修复，不做重构（历史包袱，牵一发动全身）
- API 响应结构变更必须先更新 `docs/api-contract.md` 再改代码

## 常用命令
\`\`\`bash
bun run test          # 单测
bun run test:e2e      # 端到端，耗时较长，非必要不主动跑
bun run lint --fix
\`\`\`

## 个人偏好导入（多 worktree 场景下比 CLAUDE.local.md 更稳）
@~/.claude/personal-conventions.md

## 团队规则细分（按需懒加载，减小启动 context 占用）
See @.claude/rules/testing.md for test conventions.
See @.claude/rules/security.md for security review checklist.
```

配套 `.claude/settings.json`：压缩后自动补充关键约束，避免长会话漂移；lint 作为编辑后强制门禁。

```jsonc
{
  "hooks": {
    "PostCompact": [{
      "hooks": [{
        "type": "command",
        "command": "cat ${CLAUDE_PROJECT_DIR}/.claude/hooks/reinforce-context.sh"
      }]
    }],
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/lint-check.sh"
      }]
    }],
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "if": "Bash(rm *)",
        "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh"
      }]
    }]
  },
  "disableAllHooks": false
}
```

`reinforce-context.sh` 示例（对应 4.1 的上下文漂移兜底）：

```bash
#!/bin/bash
# 压缩后重新注入关键约束，走 additionalContext，而非直接打印文本
jq -n '{
  hookSpecificOutput: {
    hookEventName: "PostCompact",
    additionalContext: "提醒：src/legacy/ 只做最小修复；数据库变更必须走 scripts/migrate.sh；API 契约变更先改 docs/api-contract.md。"
  }
}'
```

### 6.2 AGENTS.md（Codex，含 override 分层示例）

```markdown
# AGENTS.md  (仓库根目录)

## 测试
- 提交前必须跑 `pnpm test --filter=<changed-package>`，不要跑全量测试（monorepo 全量测试超时）
- 禁止修改 `**/__generated__/**` 下的文件，这些是 codegen 产物

## 提交规范
- commit message 遵循 Conventional Commits
- 不要自动 `git push`，只允许本地 commit，push 由人工确认
```

子目录覆盖（`services/payments/AGENTS.override.md`，只在该目录及以下生效，完全替换掉根目录规则里与本目录冲突的部分）：

```markdown
# services/payments/AGENTS.override.md

## 本目录专属（覆盖根目录同级规则）
- 任何改动必须先跑 `make test-payments`，忽略无关模块的失败
- 涉及金额计算的代码，禁止使用浮点数，必须走 `Money` 值对象
- 本目录的沙箱网络访问务必确认 --sandbox 参数，Seatbelt 在部分版本下
  不读取 config.toml 的 network_access 字段（见正文 4.4）
```

`~/.codex/config.toml` 关键片段（worktree + 沙箱联动）：

```toml
[sandbox_workspace_write]
network_access = false      # 默认关网络，需要网络的子任务显式用 --sandbox danger-full-access

[features]
child_agents_md = true      # 让 Codex 主动把 AGENTS.md 的层级/优先级说明附加进 system 指令，
                             # 即使当前目录没有 AGENTS.md 也会告知模型这套机制的存在

project_doc_max_bytes = 65536  # 默认 32KiB 不够用时可以调大，但要清楚这是硬截断上限
```

---

## 附：可复现实验速查表

| 验证目标 | 命令/操作 |
|---|---|
| Codex Seatbelt 网络拦截是内核级 | `codex exec "curl -m 3 https://example.com"`（不设 network_access） |
| Codex AGENTS.md 覆盖优先级 | `codex --cd <subdir> --ask-for-approval never "Show which instruction files are active."` |
| Codex exec 输出 head+tail 截断 | `codex exec "for i in $(seq 1 50000); do echo line_$i; done"`，检查行号跳变 |
| Claude Code hook 匹配链路 | 部署 `block-rm.sh`，用 `echo $(rm -rf /tmp/x)` 测试命令替换是否被识别 |
| Claude Code 内存加载审计 | 会话内输入 `/memory` |
| Claude Code MCP 输出截断 | 让 MCP server 返回 >25000 token 的 JSON，观察是否有截断提示 |
| Claude Code checkpoint 边界 | 让 Claude 用 Bash 执行一个文件删除，再 `/rewind`，验证 Bash 副作用是否真的被撤销（预期：不会） |

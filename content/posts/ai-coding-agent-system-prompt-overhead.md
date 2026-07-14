# 你的 AI 编程助手在烧 Token，而你浑然不知 —— 四大 Agent 系统提示开销深度对比

> 同一行 bug，Claude Code 用 23000 个 token 修好，Gemini 烧了 118500 个。差距不在模型聪明程度，而在"系统提示"这个你看不见的黑洞。

---

## 一、一次 Bug 修复，揭开冰山一角

2026 年初，开发者 Lars de Ridder 做了一个实验：给四个主流 AI 编程助手喂完全相同的 Express.js bug——`res.send(null)` 返回了字符串 "null" 而不是空 body——然后观察它们各自消耗了多少 token。

结果令人震惊：

| Agent | Token 消耗 | 耗时 | 工具调用次数 |
|-------|-----------|------|-------------|
| Claude Opus 4.6 | ~23,000 | 47 秒 | 6 次 |
| Codex CLI (GPT-5.5) | ~29,000–47,000 | 34 秒 | — |
| Claude Sonnet 4.5 | ~42,000–44,000 | — | — |
| Gemini 2.5 Pro | 最高 118,500 | — | — |

最省的和最费的差了整整 5 倍，修的是同一个 bug。

**而更扎心的发现是：Claude Opus 消耗的 23K token 中，70% 被工具定义（tool definitions）吃掉，真正花在理解代码和解决问题上的不到三分之一。**

这就是"系统提示开销"（system prompt overhead）——每次你和 AI 编程助手对话时，在你说第一句话之前，context window 已经被悄悄填满了很大一块。

---

## 二、Context Window 在装什么？

在深入对比之前，先理解一个基本事实：每次你给 AI 编程助手发一条消息，它会把**整个对话历史**重新发送给模型。这包括：

- **System Prompt（系统提示）**：告诉模型它是谁、有什么能力、行为规范
- **Tool Definitions（工具定义）**：每一个可调用的工具（读文件、执行命令、搜索代码……）的 JSON Schema
- **Memory/Rules 文件**：CLAUDE.md / AGENTS.md / MEMORY.md 等持久化指令
- **对话历史**：你之前的所有消息和模型的回复
- **工具调用结果**：每次读文件、跑命令的输出，全部留在 context 里

**为什么重要？** 因为这决定了你的 token 账单。API 模式下，每一条消息都要为 context 中的所有 token 付费。一个 20K token 的 system prompt，聊 10 轮就是至少 200K 的输入开销——而且这些 token 不是花在你的任务上，是花在"告诉模型怎么当助手"上。

---

## 三、四大 Agent 系统开销逐一看

### 1. Claude Code（Anthropic）

**启动开销：20,000–30,000 token**

在你输入第一个字符之前，Claude Code 已经加载了：

- 系统 prompt（工具使用规范、安全策略、行为准则）
- 所有工具定义（Bash、Read、Write、Edit、Grep、WebFetch……）
- CLAUDE.md（项目级指令文件）
- Auto Memory（自动记忆）
- Skills（技能描述）
- MCP 工具（如果配置了 MCP Server，每个 MCP 工具的 schema 都会注入）

官方文档的交互式演示也证实：session 启动时 context 已预填充了大量内容。`/context` 命令可以实时查看各项占比。

**应对策略：**

- **Subagents（子代理）**：把大量文件读取委托给子代理，子代理有自己的独立 context window，父代理只收到摘要。这是 Claude Code 最核心的 context 管理手段。
- **Auto-compaction**：context 使用率达到 ~85% 时自动触发，将对话历史压缩为摘要。压缩后系统级内容（CLAUDE.md、unscoped rules、auto memory）会从磁盘重新注入。
- **`/compact` / `/clear`**：手动触发压缩或清空对话。
- **1M token 扩展**：Opus 4.6+ / Sonnet 4.6+ 支持 100 万 token context，天花板大幅抬高，但 context rot（注意力衰减）仍会发生。

**为什么重要？** Claude Code 的 70% 工具开销说明：它的系统提示非常"重"。但子代理机制巧妙地绕过了这个问题——把脏活累活扔到独立 context 里，主对话保持清爽。

---

### 2. Codex CLI（OpenAI）

**启动开销：系统 prompt + 工具定义 + AGENTS.md（上限 32 KiB）**

Codex 的数字有些"欺骗性"：

| 数字 | 含义 | Token 数 |
|------|------|---------|
| 广告标的 | GPT-5.5 API context window | 1,050,000 |
| Codex 实际限制 | 输入 272K + 输出 128K | 400,000 |
| 真正可用 | 272K × 0.95（减去 5% 缓冲区） | ~258,400 |

而且 context window 超过 272K 输入 token 后，计费翻倍（2x 输入价格，1.5x 输出价格）。

Codex 的 AGENTS.md 类似于 CLAUDE.md，从项目根目录向叶子目录逐层加载，但有一个硬上限：所有 AGENTS.md 累计超过 32 KiB（约 8K–10K token）后直接截断。这是一个粗放但有效的"防手滑"设计。

**应对策略：**

- `/status` 查看当前 token 使用量
- `/compact` 压缩对话历史
- `/clear` / `/new` 清空或新建会话
- Auto-compaction 在接近限制时自动触发（阈值可配置）

**为什么重要？** Codex 的"名义 105 万→实际 25.8 万"的巨大落差揭示了行业通病——厂商宣传的最大 context window 和你能用的 context window 是两回事。系统开销 + 输出预留 = 你的真实可用空间远小于广告数字。

---

### 3. OpenCode

**启动开销：~68,000 token（GitHub 官方 issue 承认）**

OpenCode 是四个 Agent 中系统提示最"臃肿"的一个。GitHub issue #26661 标题直截了当：

> "Reduce initial system prompt token overhead (~68k tokens before first user message)"

这 68K 怎么来的？

- 100+ 个 Skill 的完整描述全部加载到 system prompt
- 所有工具定义的 JSON Schema（tool descriptions 每轮消耗 3000–4000 token）
- 所有子代理的 system prompt 在 task tool 初始化时一次性注入（issue #7269）

有社区用户做了实验：关闭 Skills 后，system prompt 从 30K 骤降到 4K–5K，减少了 83% 的开销。

**为什么重要？** OpenCode 的极端案例说明了一个规律：**工具生态越丰富，隐形税收越高。** 每个 Skill、每个 Subagent、每个 MCP 工具，都在你不注意的时候往 context 里塞了几百到几千个 token。这些"为了你好"的能力，最终变成了你账单上的固定成本。

---

### 4. Cursor（基于 IDE 的 Agent）

Cursor 的架构和前三者不同——它是 IDE 集成的，不是纯 terminal agent。它的 context 管理策略更像"按需加载"：

- 不会预加载所有工具定义
- 根据当前打开的文件、光标位置动态构建 context
- 使用 RAG（检索增强生成）从代码库中检索相关片段

**代价是：** Cursor 在 benchmark 中完成同一任务消耗了 188K token，是 Claude Code 的 5.5 倍（来源：tech-insider.org 的对比测试）。这不是系统提示开销，而是因为它的 context 构建策略在每一步都"撒网更广"。

**为什么重要？** Cursor 证明了：没有 system prompt 包袱 ≠ 更省 token。Context 构建策略的差异可以比 system prompt 本身产生更大的开销差异。

---

## 四、隐形开销排行

把四种 Agent 的系统级开销放在一起看：

| Agent | 启动 System 开销 | Context 上限（实际） | 核心问题 |
|-------|-----------------|---------------------|---------|
| **Claude Code** | 20K–30K token | 200K → 1M（Opus 4.6+） | 工具定义占 70%，但有子代理可绕过 |
| **Codex CLI** | 中等（系统+工具+AGENTS.md） | ~258K 可用（广告 105 万） | 宣传与实际差距巨大，高端越贵 |
| **OpenCode** | **~68K token** | 模型依赖 | 技能和工具定义完全失控 |
| **Cursor** | 低（按需加载） | IDE 集成，非固定 window | 检索策略消耗可能更大 |

---

## 五、这不仅仅是省钱的问题

系统提示开销的代价远不止 API 账单。三个更深层的影响：

### 1. Context Rot（上下文衰减）

Google DeepMind 和 Chroma Research 的研究都指向同一个结论：**context 越大，模型对其中任何特定信息的注意力越弱。** Chroma 测试了 18 个前沿模型，发现 300 token 的精简 prompt 取得的效果甚至超过 113,000 token 的"全量" prompt。

当你用了 68K 的 system prompt（OpenCode），模型真正能集中注意力处理的"你的代码"可能只有剩下空间的 50% 效率。

### 2. Compaction 陷阱

所有 Agent 都有 auto-compaction（自动压缩），但它是有损的。压缩后，你 3 小时前随口说的"这个端点用旧格式"可能被摘要成"有一些格式注意事项"——然后就丢了。Claude Code 的官方文档诚实列出了哪些内容会在压缩后存活、哪些会消失。

### 3. 第一 Token 延迟

68K token 的 system prompt 意味着模型在处理你的第一个字之前，必须读完一部中篇小说的量。这就是 OpenCode 用户抱怨的 "first time to token is very long"——尤其是本地跑大模型时。

---

## 六、作为开发者，你能做什么？

不管用哪个 Agent，这些策略是通用的：

**1. 精简 Memory 文件**

CLAUDE.md / AGENTS.md 不要写成百科全书。只放每条对话都需要的核心信息。一个长篇架构文档比一个 5 行的"当前 sprint 目标"更费 token，但价值可能更低。

**2. 禁用不用的工具和技能**

每关掉一个不需要的 MCP Server，每禁用一个用不上的 Skill，都等于往 context 里腾出了几百到几千 token。OpenCode 用户从 30K 降到 5K 的案例就是最好的证明。

**3. 善用 Subagent / Task 委托**

如果你在用 Claude Code 或支持子代理的工具，把大文件读取、全库搜索等"脏活"委托出去。子代理用完即焚，垃圾不带回主对话。

**4. 主动 Compact/Clear**

不要等到自动压缩触发。完成一个大任务后，手动 `/compact` 或 `/clear` 再开始下一个。HackerNoon 作者 Oleg Efimov 的建议是：**复杂多文件工作在 context 使用率达到 80% 前完成，剩下 20% 只做简单的单文件修改。**

**5. 用工具看清真相**

Context Lens（`npm install -g context-lens`）是一个开源的本地代理，能截获你 Agent 的 API 调用，可视化展示 context 组成。不需要改代码，不需要 SDK 插桩——跑起来就知道你的 token 去了哪里。

---

## 七、写在最后

"AI 编程一年省了多少时间"已经讨论得够多了。但很少有人问另一面：**你为了省这些时间，多烧了多少 token？**

系统提示开销是 AI 编程 Agent 的"隐形税收"——你不需要主动操作它，但每个 session、每一轮对话、每一次工具调用都在扣。Claude Code 扣了 70%，OpenCode 扣了 68K，Codex 广告说给你 105 万实际只给了 25.8 万。

好消息是，所有这些 Agent 都在快速迭代。Claude Code 的子代理机制、Codex 的可配置压缩阈值、OpenCode 社区发起的"瘦身"运动——方向都是一致的：**让 Agent 把更多 token 花在你的代码上，而不是花在"如何当 Agent"上。**

下一次打开终端，先敲个 `/context` 或 `/status`，看看你的第一个问题还没问出去，context 已经装了多少东西。那个数字可能会让你重新思考"这张显卡到底在给谁打工"。

---

*数据来源：Context Lens 实验数据（2026.2）、Claude Code 官方文档、OpenAI Codex 官方文档、OpenCode GitHub Issues（#26661, #7269, #11995）、Chroma Research "Context Rot" 研究报告（2025.7）、HackerNoon "The Context Window Tax"（2026.5）、getunblocked.com "Codex Context Window"（2026.7）*

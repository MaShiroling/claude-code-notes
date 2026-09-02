# Claude Code 多 Agent(SubAgent)面试手册

> 整理自《Claude Code 多 Agent 图解:SubAgent 实现机制怎么做?》(公众号 @小林 coding)
> 面试高频:Multi-Agent 架构、agent 间通信、并发调度。

---

## 〇、一句话总纲

Claude Code 的多 Agent 不是「一个主 agent 嵌几个 subagent」那么朴素，而是三套机制——**常规 Subagent（父子型）、Fork Subagent（缓存优化的父子分身）、Coordinator 模式（主从型）**，在隔离、通信、并发、成本每个维度都有精致设计。

---

## 一、为什么要 Multi-Agent（单 agent 的三个麻烦)

任务例:「调研 React 18 新特性 → 在项目里实现 useTransition 例子 → 代码评审」。
1. **上下文爆炸**:三个阶段的内容全塞一个上下文，token 爆炸;
2. **职责混乱**：一个 agent 又当研究员又当程序员又当评审员，容易跑偏;
3. **没法并发**:一次只能做一件事。

Multi-Agent 核心思想：老板带团队——拆任务给职责清晰的专家，老板只看方向、收结果、做决策。

### 工业界三种形态

| 形态 | 说明 | Claude Code 对应 |
|---|---|---|
| 父子型 | 主 agent 派 subagent 搞定子问题拿回结果 | 常规 Subagent(Task/Agent 工具） |
| 平级协作型 | 职责对等、共享状态/消息协作（工程难落地） | — |
| 主从型（Coordinator-Worker) | 协调者不干活，只派 worker、收结果、合成 | Coordinator 模式 |

Fork Subagent = 父子型的缓存优化版。

**Subagent 的本质**:「主 agent 通过 Agent 工具派出去的另一个独立 agent 实例」——有自己的工具池、上下文、生命周期。例如 Explore 子 agent 只带只读工具池去做调研。

---

## 二、隔离机制（最关键的一环）

### 维度一：工具隔离——按 agent 身份走三道准入门

源码:`src/tools/AgentTool/agentToolUtils.ts` 的 `filterToolsForAgent`。

1. **所有 subagent 通用黑名单**:①能派新 subagent 的工具（防递归嵌套）;②能主动问用户的工具（对话权归主 agent);③能切规划模式的工具；④能停止其他任务的工具;
2. **自定义 agent 加严黑名单**：用户自己写的 agent 没经过官方审核，多防一道;
3. **后台异步 agent 走白名单**：默认不准用，明确列出才能用（比黑名单更保险）。

（注:MCP 工具全放行。)原则:**不要假设所有 agent 都能用所有工具，按 agent 类型做细粒度权限控制。**

### 维度二：上下文隔离——按字段粒度决策（全篇最精髓)

两个极端都不行：完全共享（子读了文件污染父的 readFileState 视图）;完全新建（子收不到 Ctrl+C 中止信号，与世界脱节）。
答案:**不按整体决策，按字段决策**——每项状态单独判断该克隆、共享、屏蔽还是新建。源码:`createSubagentContext`(src/utils/forkedAgent.ts)。

| 决策 | 做法 | 原因 |
|---|---|---|
| ① 读文件缓存 | **克隆一份**给子 | 子读了 file.ts 到 200 行，父不能误以为自己读过 |
| ② 写全局状态 | **直接关闭**(setAppState 设为空操作） | 防异步 subagent 和主线程抢同一 UI 状态 |
| ③ 注册后台任务 | **例外保留通路**(setAppStateForTasks) | 否则子起的后台进程成没人回收的孤儿进程 |
| ④ 身份追踪 | 独立 agentId + **深度 +1** | 嵌套超阈值（如 5 层）报警/强制停止，防失控 |

> 所谓上下文隔离，不是一刀切的「全隔离」或「不隔离」，而是按每个状态的语义单独决策。

---

## 三、父子 Agent 怎么通信（分水岭：开没开 agent-teams 模式)

**面试边界（讲清这个比笼统说「双向」更显读过源码）**:

- **默认形态**:subagent ≈ 一次「重型工具调用」。父派出去、只能等，结果作为 tool_result 交回。消息基本只有**子→父单向通知**，父没法中途插话;
- **团队（agent-teams）模式**（实验特性，需显式开启）：补齐**父→子**通道，完整双向消息驱动。

### 默认形态的两个补丁

1. **auto-background**:subagent 跑超 **2 分钟**(120_000ms 常量 + feature 门控）自动转后台，父先干别的；本质是「同步工具调用自动降级为异步通知」;
2. **完成通知 = XML 伪装成用户消息**:task-notification XML(task-id/output-file/status/summary/result/usage）拼成字符串塞进父的对话历史。

**为什么用 XML + 伪装用户消息？** ①LLM 对 XML 天然友好（Claude 训练强调 XML);②纯文本可直接进对话历史，不走工具结果结构;③天然复用 agentic loop，父不需要额外状态机「等通知」。**「把系统事件伪装成对话」是非常值得学的一招。**

### 团队模式:消息驱动（信箱 + 字条)

- 每个 subagent 一份档案（`LocalAgentTaskState`):agentId、status(pending/running/completed/failed/killed)、result、progress、isBackgrounded、**pendingMessages（信箱）**;
- **父→子**：父调 **SendMessage 工具**（仅 agent-teams 开启才启用）往子的信箱追加消息，扔完就走，不等;
- **子→父**：子每轮工具调用结束后瞄一眼信箱，把新字条作为「用户消息」注入自己的对话历史进下一轮;
- **唤醒机制**：子已完成/被停，父发 SendMessage 会**自动唤醒**——从磁盘 transcript 恢复完整对话历史 + 拼新消息重新跑。subagent 完成了也不是「死了」;
- 子→父方向复用默认形态的 task-notification。落到底就两个关键字:**异步 + 消息**，没有函数调用、没有锁。

并发不是团队模式专利：默认形态配 auto-background 也能并发，Coordinator 模式把并发拉到极致。

---

## 四、Fork Subagent：省钱省延迟的隐藏大招

**背景**:Claude Code 的 system prompt **上万 token**;prompt 缓存命中条件是**字节级完全相同**（差一个字符、工具顺序不同都失效），命中后价格只要 **10%**、延迟大降。

**思路**:Fork 一个与父 agent 前缀字节级相同的分身，复用父的 prompt cache。源码:`CacheSafeParams`(src/utils/forkedAgent.ts）打包五项必须对齐的字段：

1. system prompt 内容（最核心）;
2. userContext(如 CLAUDE.md 内容）;
3. systemContext（环境信息）;
4. 工具池的顺序和定义（序列化进请求，顺序不能变）;
5. 对话历史前缀（决定从哪分叉）。

**精妙细节**:Fork agent 的 `getSystemPrompt` **直接返回空字符串**——不是偷懒，而是直接用父已渲染好的 prompt 字节，重新生成可能差一个字符就丢缓存。

其他定义：`tools: ['*']`（用父的完整工具池）、`maxTurns: 200`、`model: 'inherit'`、`permissionMode: 'bubble'`（权限弹窗浮到父终端）。

**适用边界**:
- 用 Fork：需要父的完整上下文但不想污染父主循环（如生成 PR 描述、/btw 总结）;
- 用常规 subagent：有明确专业分工、system prompt 定制的（Explore、Plan);
- **Fork 与 Coordinator 模式互斥**(Coordinator 的 worker 本来就异步，职责重叠）。

**工程启示**:**成本优化本身就是能力的一部分**——成本降到 10%,subagent 就能派得更频繁，能力边界反而扩大。

---

## 五、Coordinator 模式：真正的多 Agent 并行

**启用**：编译时 feature 开关 + 运行时 `CLAUDE_CODE_COORDINATOR_MODE=1`，两者同时满足。

**核心变化**:主 agent 从「全能选手」退化成「纯协调者」，只做三件事——**派 worker、收结果、合成答案**(system prompt 强制约束）。

### 协调者工具箱

派 worker（立即返回 ID)/ 创建解散团队 / 给 worker 发消息（SendMessage)/ 合成最终输出 / 停止 worker。
注意：源码里 `INTERNAL_WORKER_TOOLS` 常量是**给 worker 的黑名单**——这些协调专用工具从 worker 工具池摘掉，让 worker 只管干活（递归防护：worker 不能派 worker，系统保持「一个协调者 + 一堆 worker」的扁平形态）。
另:**agent-teams 模式和 Coordinator 模式是两个独立开关**，前者管双向消息，后者是编排模式。

### 并行是超能力

> "Parallelism is your superpower. Workers are async. Launch independent workers concurrently whenever possible."

派 worker 的工具调用可在**同一条 assistant 消息里出现多次，底层并发执行**。串行派 3 个用户等十分钟，并行只要三分钟多一点。

### 任务流水线四阶段

| 阶段 | 谁做 | 目的 |
|---|---|---|
| 调研 | Workers（并行） | 调查代码库、理解问题 |
| **合成** | **协调者本人** | 读懂发现、写成实现规格 |
| 实现 | Workers | 按规格修改提交 |
| 验证 | Workers | 测试改动是否真的工作 |

**关键哲学：协调者必须「理解」而不能「转发」。** 不要偷懒让 worker「based on your findings, implement the fix」——转发型协调者没有存在价值。

### Continue vs Spawn（老 worker 还是新 worker?)

- 新任务与 worker 现有上下文高度相关 → **续命老 worker**（它已「知道」那些文件，续命比新派省钱）;
- 不相关或之前走偏了 → 派新 worker（避免旧上下文干扰）;
- **验证类工作永远派新 worker**——不能让刚写完代码的 worker 自己验自己（需要新鲜眼光）。

### Coordinator vs 常规 subagent 对比

| 维度 | 常规 subagent | Coordinator 模式 |
|---|---|---|
| 主 agent 角色 | 全能选手 | 纯协调者 |
| 执行方式 | 同步（2 分钟后转后台） | 默认异步 |
| 并发程度 | 偶尔并发 | 最大化并发 |
| 适合场景 | 单任务+临时帮手 | 大任务+高并发拆解 |
| 系统形态 | 父子树 | 协调者+worker 扁平层 |

---

## 六、5 条 Multi-Agent 设计原则（可直接搬进面试答案)

1. **上下文隔离要按字段粒度做**：对着父 agent 每项状态问「子拿它干啥？会不会影响父?」——克隆缓存、关写权限、留任务通路、深度计数;
2. **通信走消息，不走函数调用**：父→子写信箱下轮自取；子→父 XML 伪装用户消息。天然异步、并发、兼容 agentic loop、可持久化;
3. **工具权限要分级管控**：全局黑名单（防递归）+ 类型黑名单（自定义更严）+ 异步白名单;
4. **缓存友好是一种架构能力**：设计 subagent 时考虑 prompt 前缀能否复用父的缓存，省 80-90% 成本;
5. **并行优先 + 协调者合成**：异步消息打底、协调者亲自理解合成，不当传话筒；扁平不递归（层级限制在两层）。

---

## 七、面试速答要点

- 三套机制：常规 Subagent（父子）/ Fork Subagent（字节级复用父 prompt 缓存，成本降 90%)/ Coordinator（主从，主 agent 只做派 worker、收结果、合成）;
- 隔离：工具三道准入门 + 上下文按字段粒度（克隆/关闭/留通路/深度+1);
- 通信：默认形态单向（子→父 task-notification XML 伪装用户消息 + 2 分钟 auto-background);agent-teams 模式补齐父→子 SendMessage 信箱，双向消息驱动，子停了还能从 transcript 唤醒;
- 并发：同一 assistant 消息多个派 worker 调用并发执行；验证任务永远派新 worker。

---

## 关键数字速查

| 数字 | 含义 |
|---|---|
| 2 分钟 | auto-background 转后台阈值（120_000ms) |
| 5 层 | subagent 嵌套深度报警阈值（示例） |
| 10% | prompt 缓存命中后的价格 |
| 90% | Fork 复用缓存节省的成本 |
| 200 轮 | Fork agent 的 maxTurns |
| 上万 token | Claude Code system prompt 规模（Fork 要复用的对象） |

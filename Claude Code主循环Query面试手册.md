# Claude Code 主循环 Query 面试手册

> 整理自《Claude Code 主循环 Query 图解：一轮对话是怎么跑起来的？》(公众号 @小林 coding)
> 核心文件：`query.ts`,1729 行，其中约 80% 在处理「异常路径」——这才是工业级 Agent 和 demo 的区别。

---

## 〇、面试官想考什么

「介绍一下 Claude Code 主循环 Query 的流程?」——考点不是皮毛的「调模型、跑工具」，而是：

- 模型吐字时怎么控？（流式 + 并行）
- 出错了怎么兜？（十几种退出原因）
- 状态怎么跨轮不丢？（State 对象）

---

## 一、主循环是什么

用户敲「帮我修复这个 bug」后，模型第一次回复往往不是答案，而是「我要先看那个文件」(tool_use)；系统读文件塞回去再问，模型又说「我要跑测试」……来来回回直到模型说「修好了」。这个**「调模型 → 看结果 → 决定下一步」的循环就是主循环（Query Loop)，是 Agent 的心脏**。

朴素写法 `while(true){ 调模型; if(用工具) 执行工具; else break }` 能跑通 demo，但生产环境会被千锤百炼出无数 bug:Ctrl+C 中断、工具跑挂、输出截断、上下文爆炸、跑 50 轮不收敛……1729 行代码 80% 在兜这些异常路径。

---

## 二、四层调用链:ask → QueryEngine → query → queryLoop

类比自来水：水厂净化 → 街道主管线 → 小区水箱加压 → 家里水龙头。

| 层 | 职责 | 类比 |
|---|---|---|
| `ask` | SDK 一次性调用入口，只做一件事：new QueryEngine 并转交 | 水龙头 |
| `QueryEngine.submitMessage` | 管会话状态：消息历史、文件缓存、权限拒绝记录（跨轮持久化「记账本」)；处理斜杠命令、组装系统提示 | 小区水箱 |
| `query` | **流式包装层**（异步生成器），把对话生命周期的收尾工作与核心循环解耦 | 主管线 |
| `queryLoop` | 核心 `while(true)` 循环本体 | 水厂 |

**关键机制：每层用 `yield*` 接力**,queryLoop 抛出的每个事件都能原封不动冒泡到最外层 ask 的调用方——这是「边干边吐」流水线贯通的关键。

### 为什么必须用异步生成器（async generator)?

- 普通 async 函数 = 接一桶水：等接满整桶端走才能用（等几十秒突然蹦一大段）;
- 异步生成器 = 水龙头本身：模型每吐一个字符、每跑完一个工具立刻流出来；
- 这不仅是用户体感，更是后面「流式 + 并行工具执行」的前提。

---

## 三、主循环转一圈：五步骨架

```typescript
while (true) {
  // 1. 准备消息：必要时压缩（被动触发！)
  const messagesForQuery = maybeCompact(state.messages)
  // 2. 流式调大模型，边收边处理
  for await (const chunk of callModel(messagesForQuery)) {
    yield chunk                      // 文字块即时抛给用户
    if (chunk 是 tool_use 块) { toolUseBlocks.push(chunk); needsFollowUp = true }
  }
  // 3. 判断继续还是结束
  if (!needsFollowUp) return { reason: 'completed' }
  // 4. 执行所有 tool_use(可并行/串行)
  const toolResults = await runTools(toolUseBlocks)
  // 5. 结果作为 user 角色消息塞回历史，更新 state,continue 进下一轮
}
```

**第一步的细节：压缩是被动触发，不是主动预防。** 先按原样发，只有 API 返回 `prompt_too_long`(413）拒收才补救压缩一次重试；压完还塞不下就退出。`hasAttemptedReactiveCompact` 标志位防止同一轮内反复压缩陷入死循环（新一轮重置）。为什么？压缩本身要烧 token，能不压就不压。

**消息滚雪球**：每轮消息历史变长（assistant 回复 + 工具结果），第 20 轮发给模型的历史包含前 19 轮全部内容——所以压缩机制必须存在。

**模型回话的两种块**（概念铺垫）:
- **文字块**：模型在跟你说话；
- **tool_use 块**：结构化工具调用请求 `{工具名:"Read", 参数:{...}}`。

---

## 四、决策点：该用工具还是该停下来？

**简单到出乎意料：就看返回里有没有 tool_use 块。**

- 有 tool_use → `needsFollowUp = true` → 执行工具 → 继续循环；
- 没有 tool_use → 模型在跟你说话 → `return { reason: 'completed' }`。

为什么这么朴素？**因为「决定下一步做什么」本身就是大模型的工作**，看它吐出的是文字还是工具调用请求就知道意图了。

### 退出原因（reason)：源码 17 种，代表性的 8 种

| reason | 含义 |
|---|---|
| `completed` | 正常完成（最小的一支） |
| `max_turns` | 轮数封顶，强行止损防跑飞烧钱 |
| `aborted_streaming` | 模型吐字时用户按 Ctrl+C |
| `aborted_tools` | 工具跑到一半用户按 Ctrl+C |
| `prompt_too_long` | 消息历史太长，压缩也救不回 |
| `max_output_tokens_recovery` | 输出截断，续写 3 次仍不行 |
| `stop_hook_prevented` | 用户自定义 Stop Hook 拦下（如 push 前 lint 没过） |
| `image_error` | 图片格式错误 |

**核心认知：工业级主循环兜的不是「正常退出」，而是十几种「异常退出」。** 每种异常都要：识别错误状态 → 必要清理（取消工具、解锁资源）→ 返回结构化 reason 供外层上报/重试/提示。

---

## 五、最骚的设计：流式 + 并行工具执行

**朴素做法**：等模型流完 → 收齐 tool_use → 挨个执行 → 塞回。模型吐字时工具干等，工具跑时模型干等。

**Claude Code 的做法(`StreamingToolExecutor`)**:

```typescript
// 模型流式输出时，每识别到一个完整 tool_use 块就立刻丢后台开跑
streamingToolExecutor.addTool(toolBlock, message)
// 模型流完后，把已完成的结果一次性收上来
const toolUpdates = streamingToolExecutor.getRemainingResults()
```

点菜类比：你报一个菜名，服务员立刻冲后厨喊一份；你点完时红烧肉已经炒了一半。**把模型生成时间和工具执行时间高度重叠，榨干每一段空闲时间。**

**并行边界（重要）**:
- 只读工具（Read/Grep/Glob)→ 随便并行；
- 会改状态的工具（Edit/Write/Bash)→ 必须串行（防并发覆盖）;
- 该属性在工具定义时声明；**漏写则 fail-closed 兜底默认串行**——宁可错杀让你慢一点，也绝不并发踩踏。

---

## 六、容错三件套：出错时凭什么让你毫无察觉

### 6.1 跨轮状态对象 State（不用全局变量糊弄）

```typescript
type State = {
  messages: Message[]                   // 累积的对话历史
  turnCount: number                     // 当前第几轮
  maxOutputTokensRecoveryCount: number  // 输出截断已恢复几次
  hasAttemptedReactiveCompact: boolean  // 本轮是否已压缩过
}
```

所有跨轮字段打包成一个状态对象，每轮开头读出、结尾构造新的写回。后两个字段本质是**计数器/防重复标志（同一轮内只让某件事发生一次），目的就是避免无限循环**。摊在明处 → 每一轮的「现场」一目了然，能调试的状态才是可靠的状态。

### 6.2 工具跑挂了，对话凭什么继续？（合成假 tool_result)

- **API 死规定**：每个 `tool_use` 块必须有配对的 `tool_result` 块，否则下一轮请求直接被拒收（「工具调用 ID 对不上」)。
- 孤立 tool_use 的场景：网络断了工具没跑 / 工具跑到一半 Ctrl+C / 模型降级切换后前一个模型的 tool_use 没机会执行。
- **方案**：合成一个假的 tool_result 塞回去（`yieldMissingToolResultBlocks`)，标明 `is_error: true`、内容是错误消息、附上对应 tool_use_id。
- 这叫「**先认错，再继续**」：协议是死的，但主循环不会卡死，模型还知道「上个工具失败了」能做出合理决策。

### 6.3 模型输出被截断：两段式恢复

背景：常规 max_output_tokens 默认 32k，灰度场景压到 8k 控费。

1. **第一段：静默升档**。首次触发 8k 截断 → 不报错，悄悄把上限调到 64k 重试这一轮，用户完全无感知；
2. **第二段：nudge 续写**。64k 还截断 → 在下一轮消息里塞提示：「Output token limit hit. Resume directly — no apology, no recap...」让模型从断点直接续写、把剩余工作拆小；
3. **3 次上限**：续写 3 次还不行 → 退出，reason = `max_output_tokens_recovery`（配合 State 里的计数器防死循环）。

**核心认知：工业级的容错不是「报错弹窗」，而是用户根本感觉不到出过错。**

---

## 七、四大设计哲学（收尾金句）

1. **边干边吐，感知前置**：异步生成器让每个中间事件即时抛出，「响应快」的体感由此而来；
2. **状态显式管理**：该计数的字段全摊在 State 对象里，不藏闭包变量；
3. **引擎层不掺业务**：主循环不知道工具具体在干嘛，新增能力零侵入，往工具池塞个新工具即可；
4. **错误恢复优先于优雅**：宁可丑陋地合成假 tool_result，也不能优雅地崩溃。

---

## 八、面试速答模板

> 「Claude Code 主循环不能简单写成 while true。调用链分四层：ask 是 SDK 入口、QueryEngine 管会话状态、query 用异步生成器做流式包装、queryLoop 是循环本体，层间用 yield* 接力让事件冒泡。每轮做五件事：准备消息（压缩是被动触发，被 API 拒收才压）、流式调模型、看有没有 tool_use 块决定继续还是结束、执行工具、塞回结果。工具有个很妙的设计：StreamingToolExecutor 在模型流式输出时就把识别出的完整 tool_use 丢后台并行跑，只读工具并行、写工具串行，漏标就 fail-closed 默认串行。退出原因有 17 种——正常完成只是最小的一支，大头是各种异常：Ctrl+C 中断、prompt_too_long、max_turns 止损、Stop Hook 拦截等。容错上，跨轮状态全摊在一个 State 对象里防无限循环；工具跑挂会合成带 is_error 的假 tool_result 满足 API 的配对死规定；输出截断走两段式恢复——先静默把 8k 升档到 64k，再 nudge 模型断点续写，3 次不行才退出。一句话总结：错误恢复优先于优雅，用户根本感觉不到出过错。」

---

## 关键数字速查

| 数字 | 含义 |
|---|---|
| 1729 行 | 主循环文件规模，~80% 处理异常路径 |
| 4 层 | ask / QueryEngine / query / queryLoop |
| 17 种 | 退出原因 reason 总数 |
| 8k → 64k | 输出截断时静默升档的 token 上限 |
| 3 次 | 续写重试上限 / 同轮压缩次数上限 |

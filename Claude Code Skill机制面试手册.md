# Claude Code Skill 机制面试手册

> 整理自《Claude Code Skill 原理图解:几个 Markdown 文件,怎么做到按需加载?》(公众号 @小林 coding)
> 面试高频题:「Skill 和提示词有什么区别?」+ 追问「那直接把内容塞进 system prompt 不就行了?」

---

## 〇、面试速答（先背这个)

> 「Skill 的内容确实就是提示词，这没错。但区别不在内容，在**加载方式**。提示词是一股脑全塞进上下文，skill 走的是**渐进式披露（Progressive Disclosure)**，分层按需加载：平时上下文里只常驻一行描述，模型自己判断这活用得上，正文才会被注入，更细的参考文档还能再懒一层、真用到才去读。所以我的理解是——**提示词是知识本身，skill 是知识的加载策略**。」

听到「渐进式披露」四个字，面试官大概率就坐直了。

---

## 一、Skill 是什么

- 一个 skill = 一个目录：必须叫 `SKILL.md` 的主文件 + 可选的参考文档（reference.md/forms.md)+ 可选脚本（scripts/)。全是纯文本，没二进制、没安装脚本。
- SKILL.md 开头是 YAML **frontmatter**，最重要两个字段:**name**（名字）+ **description**（干什么的、什么时候该用）；正文就是操作指南。可选进阶字段:allowed-tools、model。
- 类比：名校实习生入职——不做开颅手术灌知识，而是扔一本**入职手册**。模型能力没变，手头多了份「你家业务的说明书」。
- Anthropic 已将其定名 **Agent Skills** 并开源，意图成为行业通用标准。

---

## 二、为什么需要渐进式披露（那道追问的答案)

算笔账:50 个 skill × 平均 3000 token = **15 万 token**,20 万窗口被手册占掉 3/4；而且当前对话可能一个都用不上。所以:**skill 不能全文常驻上下文，但不放进去模型又看不见它**——这个死结的解法就是渐进式披露。

### 三层渐进式披露（用到哪层才翻开哪层)

| 层 | 内容 | 开销 |
|---|---|---|
| 1. 常驻「书脊」 | 所有 skill 的 name+description 拼成清单，随 system-reminder 进上下文，正文一个字不带 | 预算只有窗口的 **1%**(≈2000 token)；单条描述上限 **250 字符** |
| 2. 选中才抽书 | 模型判断需要时调用 Skill 工具,SKILL.md 完整正文才注入对话 | 用到才花几千 token |
| 3. 附录更懒 | reference.md/scripts 连第二层都不进;SKILL.md 只留指路的话，模型真遇到才自己 Read 进来 | 按需付费，无需专门机制（复用 Read 工具） |

**效果：从 15 万 token 降到 2000,75% 的窗口占用变成 1%。**

**第一层的降级方案**（装几百个 skill 时）：先全量描述 → 超预算则按剩余空间等比截短 → 最极端时第三方 skill 只留名字、描述全砍。**官方内置 skill(bundledIndices）的描述永远不被截断**——「亲儿子待遇」。

**第二层注入的精妙细节**:skill 正文不走 system prompt，而是**包装成一条 user message**（带 `isMeta: true`，界面不显示）插入对话流。原因：改 system prompt 会导致 **Prompt Cache 大面积失效**每轮重新计费，追加消息是纯增量、不伤缓存。

> Skill 骨子里就是一套**上下文的懒加载系统**。

---

## 三、Claude 怎么知道该用哪个 skill?（选得准)

候选方案：embedding 相似度检索？意图分类模型？关键词匹配？——**Claude Code 全都没用。**

答案：把「书脊清单」摆在模型眼前，**相信模型自己的阅读理解**。每轮生成前模型本来就要通读上下文，清单在、需求也在，「需求跟哪行描述对得上」正是语言模型最擅长的事。整个选择过程没有一行检索代码，全发生在模型的一次前向推理里。

- **description 是唯一广告位**：由 `description` + 可选 `when_to_use` 拼成，是模型做判断的全部依据；写得含糊就坐一辈子冷板凳;
- **防「想起来但懒得用」**:SkillTool 工具说明里有硬性指令——「When a skill matches the user's request, this is a **BLOCKING REQUIREMENT**: invoke the relevant Skill tool BEFORE generating any other response」，且「绝不允许嘴上提到 skill 却不真正调用」;
- **贯穿始终的哲学**（呼应检索用 grep、记忆用 markdown)：能靠模型自身理解力解决的，绝不外挂一套额外系统——外挂系统多一处故障点、多一摊维护；模型的理解力随升级白嫖变强;
- **软肋**:skill 到几百个、描述被截得只剩名字时，匹配准确率会掉（几十量级内碰不到天花板）。

---

## 四、/ 斜杠命令和 skill 是同一个东西

源码内部 slash command 和 skill 已统一为一套 **Command 体系**：同一数据结构、同一加载执行路径。工具说明直接挑明:「When users reference a "slash command" or "/<something>", they are referring to a skill.」

**一个 skill 天生两个触发入口**：人敲 `/名字` 手动触发；模型判断匹配自主触发。

**frontmatter 两个开关**:
- `disable-model-invocation: true` → 模型禁调，只有人能触发。场景:**危险操作**（如「一键发布上线」，必须人来扣扳机）;
- `user-invocable: false` → 从斜杠菜单消失，只有模型能调。场景:**纯背景知识**（如「本项目数据库表结构说明」)。

---

## 五、skill 不只是静态文档：参数 + 动态内容

1. **`$ARGUMENTS` 占位符**:`/style-check 只查命名问题` → 参数原样填进正文，模型自主调用也能传参；
2. **`` !`命令` `` 内嵌命令**:skill 被调用那一刻先执行命令、把输出填进正文再交给模型。如官方 /commit 内嵌 `` !`git diff` ``——模型睁眼就看到真实 diff;`!`cat $ARGUMENTS`` 直接把待检查代码怼进 prompt。

**安全防线**:**来自 MCP 服务器的远程 skill 永远不执行内嵌命令**(MCP 是远程内容不可信）——只有本地目录的 skill 才享受动态展开。

---

## 六、skill 的五个来源与撞名规则

| 来源 | 位置 | 说明 |
|---|---|---|
| 官方内置 | 随 Claude Code 发行 | 如 commit |
| 用户级 | `~/.claude/skills/` | 跨项目跟随 |
| 项目级 | 项目 `.claude/skills/` | **随 git 仓库走**——新同事 clone 即就位，团队知识从口耳相传的 wiki 变成 agent 可执行的活文档 |
| 插件 | 随插件打包 | |
| MCP | 远程服务器动态提供 | 不执行内嵌命令 |

**撞名**：按固定顺序扫描，**先到先得**，后来的重名跳过并记日志。项目级排在用户级**后面**加载——别指望在项目里覆盖用户级同名 skill，起名错开才是正道。

---

## 七、怎么写好一个 skill（三条心法)

1. **description 按广告词标准写，重点写「什么时候用我」**：放弃功能罗列，用用户嘴里会说出来的动词。反例:「功能强大的代码质量工具，支持多种语言」；正例:「检查代码是否符合团队风格规范。当用户要求检查代码风格、评审代码时使用」;
2. **SKILL.md 管今天，references 管细节**：主动利用第三层——主文件只写高频主干流程，低频/超长细节拆到 references，正文留指路的话。糙但好用的标准:**SKILL.md 超 500 行就该拆附录**;
3. **重复劳动写成脚本，别让模型每次现想**：固定机械操作直接放 scripts/ 脚本，正文一句「执行这个脚本」。模型现想有失败率，脚本跑一万次一个样——**能固化成脚本的就别让模型现想**。

---

## 八、收尾金句

> Skill 表面是「几个 Markdown 文件」，拆开看是一整套精密的上下文调度系统:「放得下」靠三层渐进式披露（常驻开销压到窗口 1%),「选得准」靠一行 description + 模型自己的阅读理解，检索系统一概没上。

---

## 关键数字速查

| 数字 | 含义 |
|---|---|
| 1% | skill 清单占上下文窗口的预算(SKILL_BUDGET_CONTEXT_PERCENT = 0.01) |
| 250 字符 | 清单中单条 description 硬上限(MAX_LISTING_DESC_CHARS) |
| 15 万 → 2000 token | 50 个 skill 全文常驻 vs 渐进式披露的开销对比 |
| 500 行 | SKILL.md 超过这个长度就该拆 references |
| 5 个 | skill 来源：内置/用户级/项目级/插件/MCP |
| 2 个 | 触发入口（人敲 / 模型自主）与开关(disable-model-invocation / user-invocable) |

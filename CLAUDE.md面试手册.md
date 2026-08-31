# CLAUDE.md 面试手册

> 整理自《CLAUDE.md 指南：Claude Code 的项目记忆该怎么写？》(公众号 @小林 coding)

---

## 一、核心概念：CLAUDE.md 是什么？

- **本质**：一个放在项目根目录的普通 markdown 文件，文件名固定为 `CLAUDE.md`。
- **机制**：每次启动 Claude Code 会话时，它会被**自动完整加载**进上下文，作为整个对话的「ground truth」（默认成立的前提）。不是可选提示，而是默认前提。
- **与 README 的区别**（高频考点）：
  - README 是写给**人**看的，Claude 默认不会主动读；
  - CLAUDE.md 是写给 **agent** 看的，每次会话自动加载、每行都吃 token。
  - 一句话：**CLAUDE.md 不是文档，是给 Claude 的配置 / 入职手册**。

**面试金句**：「CLAUDE.md 是给 Agent 的入职手册，不是给人的 README。」

---

## 二、为什么写多了反而废？（关键数据）

来源：SFEIR Institute 实测。

| 写法 | 规则遵守率 |
|---|---|
| 单文件 ≤ 200 行 | ≈ 92% |
| 单文件 > 400 行 | 掉到 ≈ 70% |
| 拆成 5 个 30 行模块文件放 `.claude/rules/` | 升到 ≈ 96% |

**两个原因**：

1. **Token 经济**：CLAUDE.md 每次启动都完整加载，行数越多越挤压对话、思考、工具结果的上下文空间。
2. **注意力稀释**：模型注意力有限，规则一多每条权重被摊薄；超过 300 行后「记不住」成常态。

**黄金线：200 行以内，推荐 80 行左右起步。**

---

## 三、三类「负资产」内容，直接删

1. **复述型**：把架构文档整段复制进来 → 改为一行指路：`项目架构详见 docs/architecture.md`。
2. **愿望型**：「我们希望测试覆盖率 90%」→ 只写当下实际执行的规则；「PR 前必须跑 `npm test`」是规则，「希望大家多写测试」是 PUA。
3. **术语表型**：通用术语 Claude 都懂；团队特有黑话放 `docs/glossary.md`。

---

## 四、有效规则的四个原则（铁律）

> **短、具体、告诉为什么、持续更新**

| 模糊写法（无效） | 具体写法（有效） |
|---|---|
| 测试一下你的修改 | 提交前跑 `npm test` |
| 保持目录整洁 | API 处理函数放在 `src/api/handlers/` |
| 别把构建搞挂了 | 推代码前跑 `npm run typecheck` |
| 用好的命名 | 组件 PascalCase，工具函数 kebab-case |

- **具体 = 可验证**：Claude 能自检（是不是 2 空格？是不是 .ts 文件？）。
- **告诉为什么**（最关键）：例「不要写入生产数据库，**因为去年测试清空过 users 表出过事故**」→ Claude 知道规则边界，能对 staging 等场景做正确判断，而不是机械执行。
- **持续更新**：当活文档维护。Claude 同一错误犯 **2 次以上**就加一条防御规则；过时规则要删——**「错误的规则比没有规则更糟」**。
- **没有标准模板**，原则吃透后按项目裁剪，别迷信任何「最佳实践」。

---

## 五、分层组织：CLAUDE.md 是一个生态

加载机制（源码 `src/utils/claudemd.ts`）：从当前目录一路向上爬到文件系统根，再反向逐层读取所有 `CLAUDE.md` 和 `.claude/CLAUDE.md`，**全部合并**喂给模型。

**三层结构**：

```
~/.claude/
└── CLAUDE.md          # 全局：跨项目个人偏好（如「用中文回复」）
my-project/
├── CLAUDE.md          # 项目根：技术栈、目录、命令、硬约束（每次加载，大头）
├── frontend/
│   └── CLAUDE.md      # 前端模块约定（按需加载）
└── backend/
    └── CLAUDE.md      # 后端模块约定（按需加载）
```

**进阶：`.claude/rules/` 模块化 + path-scoped rules**

```
.claude/rules/
├── testing.md       # 测试规则
├── api-design.md    # 接口设计规则
├── security.md      # 安全规则
└── ui-components.md # UI 组件约定
```

每个文件聚焦一个主题、≤30 行，顶部用 YAML frontmatter 声明作用路径：

```markdown
---
paths: ["**/*.test.ts", "**/*.spec.ts"]
---
# 测试规则
- 用 describe / it，不用 test()
- mock 外部依赖必须用 vi.mock
```

Claude 只在改匹配路径的文件时才加载该规则 → **少加载、按需加载，效果比一坨更好（96%）**。

**演进路径**：`CLAUDE.md` 起步 → 长了拆 `rules/` → 高频工作流写 `commands/` → 可复用能力封装 `skills/`。

---

## 六、跨工具复用：AGENTS.md

- OpenAI Codex 的对应物是 `AGENTS.md`，作用、写法、加载机制几乎一样，所有原则照搬即可。
- **双工具项目的巧妙做法**：规则全写 `AGENTS.md`，`CLAUDE.md` 只留一行：

```
@AGENTS.md
```

`@文件名` 是引用指令，Claude Code 加载时会顺藤读入 AGENTS.md → **一份文件、两个工具、零重复维护**。

- **坑：规则冲突**。官方原话：「如果两条规则互相矛盾，Claude 可能会随便挑一条。」需每 1–2 周 review 一次，清理过时/冲突规则。

---

## 七、工具链：/init 起步、/memory 维护、Plan Mode

| 命令/功能 | 作用 |
|---|---|
| `/init` | 自动扫描代码库，生成 CLAUDE.md 草稿（技术栈、目录、命令）→ review 后删错补漏。「五分钟时间，永久受益」 |
| `/memory` | 会话中快速编辑 CLAUDE.md；或直接说「记一下这条规则」自动追加 |
| Plan Mode（Shift+Tab ×2） | 复杂任务先出计划再执行；**计划会把 CLAUDE.md 规则全考虑进去**，好的 CLAUDE.md 直接决定计划质量。3 个文件以上的改动强烈建议先切 Plan Mode |

---

## 八、推荐模板（6 段式，≤80 行）

```markdown
# CLAUDE.md
## 1. Project Overview      # 2-3 行：是啥项目 + 技术栈 + 定位
## 2. Commands              # 最常用命令：install / dev / test / typecheck / lint
## 3. Architecture          # 三句话指路，详见 docs/architecture.md（不复述）
## 4. Conventions           # 团队真实在用的约定（命名、返回格式、错误处理）
## 5. Hard Constraints      # 硬约束，写 why（如「不写生产库，去年出过事故」）
## 6. Gotchas               # 新人踩过的坑（价值最高，Claude 无法从代码推断）
```

**细节**：总行数 ~50 行留扩展空间；架构段只指路不复述；硬约束带 why；Gotchas 价值最高。

---

## 九、面试速答模板（背下来）

> 「CLAUDE.md 每次启动都会被完整加载进上下文，规则一多反而稀释模型注意力。社区实测数据是 200 行 92% 遵守率，400 行掉到 70%。我的做法是项目根 CLAUDE.md 控制在 80 行以内，按模块拆到 `.claude/rules/` 下用 path-scoped 加载，配合 `/init` 起步和 `/memory` 维护，规则遵守率明显上来了。」

---

## 十、三句话精华（电梯版）

1. CLAUDE.md 是给 Agent 的入职手册，不是给人的 README。
2. 200 行是黄金线，每行都吃 token，多写不如不写；复述型、愿望型、术语表型直接删。
3. 具体可验证、告诉 why、持续更新——三条铁律压过一切技巧。

---

### 参考资料
- Anthropic 官方文档：https://docs.anthropic.com/en/docs/claude-code/claude-md
- claudeguide.io：How to Write Effective CLAUDE.md Files
- claude-codex.fr：Mastering CLAUDE.md（6 段式结构来源）
- SFEIR Institute：The CLAUDE.md Memory System Deep Dive（实测数据来源）

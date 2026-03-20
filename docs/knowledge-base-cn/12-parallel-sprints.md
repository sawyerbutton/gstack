# 12. 并行 Sprint 实战

> 一个 Sprint 是强大的。十个并行 Sprint 是一种完全不同的工作方式。本文解析 gstack 如何通过会话感知、端口隔离、共享状态和 vendored symlink 机制支持 10-15 个并行 Sprint 同时运行，以及如何从 2-3 个 Sprint 起步逐步扩展到这种工作模式。

---

## 为什么并行 Sprint 是变革性的

gstack 的每个技能都很强大：`/office-hours` 产出设计文档，`/plan-eng-review` 锁定架构，`/qa` 测试并修复 bug，`/ship` 创建 PR。但如果你只能串行使用它们——一个任务完成再开始下一个——你只是用 AI 加速了一个人的工作。

并行 Sprint 改变的不是速度，而是**工作模式**。想象一下：

```
Sprint A: /office-hours  → 回答问题 → 设计文档批准
Sprint B: /plan-eng-review → 审查架构 → 回答一个 AskUserQuestion
Sprint C: /qa → 自动运行中... → 报告到了
Sprint D: /ship → 测试通过 → PR URL 出来了
Sprint E: /office-hours → 等你回答 Q3
```

五个不同的功能，同时在推进。你不是在写代码——你是在做决策。每个 Sprint 需要你的时候才需要你，其余时间它在自主工作。

Garry Tan 描述这种模式为"像 CEO 管理团队一样管理多个 Claude Code 会话"。每个会话是一个"团队成员"，你做方向性决策，它们执行。

---

## Sprint 结构如何支撑并行性

没有流程，10 个 agent 就是 10 个混乱源。gstack 的 Sprint 结构天然支持并行执行，原因在于：

**1. 每个技能是自包含的。** `/office-hours` 不需要知道 `/qa` 在做什么。每个技能有自己的输入（CLAUDE.md、git log、设计文档）和输出（设计文档、测试计划、PR）。

**2. 共享状态在文件系统中。** `~/.gstack/projects/$SLUG/` 是所有技能的汇合点。设计文档、测试计划、审查日志——所有跨技能的信息都在这里，通过文件名约定（`{user}-{branch}-{type}-{datetime}.md`）自然分离。

**3. 每个 Sprint 在自己的分支上。** Git 分支提供了天然的隔离。Sprint A 在 `feature/auth-refactor`，Sprint B 在 `feature/billing-ui`，Sprint C 在 `feature/api-v2`。它们的文件变更、commits、测试运行互不干扰。

---

## 会话感知（Session Tracking）

### 机制

gstack 的 preamble（每个技能启动时执行的初始化脚本）包含会话跟踪代码：

```bash
mkdir -p ~/.gstack/sessions
touch ~/.gstack/sessions/"$PPID"
_SESSIONS=$(find ~/.gstack/sessions -mmin -120 -type f 2>/dev/null | wc -l | tr -d ' ')
find ~/.gstack/sessions -mmin +120 -type f -delete 2>/dev/null || true
```

这段代码做了三件事：
1. 在 `~/.gstack/sessions/` 下创建一个以父进程 PID 命名的文件
2. 统计过去 2 小时内活跃的会话数量（`-mmin -120`）
3. 清理超过 2 小时不活跃的会话文件

每个 Claude Code 实例有一个唯一的父进程 PID。当你开 10 个终端窗口各自运行 Claude Code 时，`~/.gstack/sessions/` 下会有 10 个文件。

### 多会话时的 Re-ground 机制

当检测到 3+ 活跃会话时，每个 AskUserQuestion 都会重新锚定上下文。AskUserQuestion 的格式规范要求：

```
1. Re-ground: 说明项目、当前分支（使用 preamble 打印的 _BRANCH 值——
   不是对话历史或 gitStatus 中的分支）、以及当前计划/任务。(1-2 句)
2. Simplify: 用一个聪明的 16 岁少年能理解的简单英语解释问题。
3. Recommend: RECOMMENDATION: Choose [X] because [one-line reason]
4. Options: A) ... B) ... C) ...
```

Re-ground 步骤在单会话时也有，但在多会话时尤其关键：当你在 10 个窗口之间切换时，你不记得哪个对话在讨论什么。Re-ground 确保每次 AI 需要你注意力的时候，你立刻知道：

- 这是**哪个项目**
- 在**哪个分支**上
- 在做**什么事**

格式规范还有一句关键的设计理念：

```
Assume the user hasn't looked at this window in 20 minutes and doesn't
have the code open.
```

假设用户已经 20 分钟没看这个窗口了，也没有打开代码。这就是并行 Sprint 的现实——你不是全程盯着每个 Sprint，你是在多个 Sprint 之间跳转。

---

## 端口隔离

### 随机端口选择

gstack 的浏览器守护进程（browse daemon）使用 10000-60000 之间的随机端口：

```typescript
// browse/src/server.ts
// Random port with retry
const MIN_PORT = 10000;
const MAX_PORT = 60000;
const port = MIN_PORT + Math.floor(Math.random() * (MAX_PORT - MIN_PORT));
```

每个工作空间都有自己的 Chromium 实例和自己的端口。状态文件存储在项目根目录下：

```
/code/project-a/.gstack/browse.json    → random port (10000-60000)
/code/project-b/.gstack/browse.json    → random port (10000-60000)
/code/project-c/.gstack/browse.json    → random port (10000-60000)
```

**零配置，零端口冲突。** 旧的方案（扫描 9400-9409）在多工作空间场景下经常崩溃。随机端口 + 碰撞重试（最多 5 次）从根本上解决了这个问题。

### 每个工作空间完全隔离

```
Workspace A                    Workspace B
┌─────────────────────┐       ┌─────────────────────┐
│ Claude Code session  │       │ Claude Code session  │
│ Branch: feat/auth    │       │ Branch: feat/billing │
│                      │       │                      │
│ Browse daemon :34721 │       │ Browse daemon :51083 │
│ Chromium instance A  │       │ Chromium instance B  │
│                      │       │                      │
│ .gstack/browse.json  │       │ .gstack/browse.json  │
└─────────────────────┘       └─────────────────────┘
          │                              │
          └──────────┬───────────────────┘
                     │
            ~/.gstack/projects/$SLUG/
            (共享的设计文档、测试计划、审查日志)
```

每个工作空间的浏览器实例是独立的——它们有自己的 cookie、自己的导航历史、自己的截图输出目录。但它们共享 `~/.gstack/projects/` 下的项目状态。

---

## 跨会话的信息流

### `~/.gstack/projects/` 作为共享状态

这是并行 Sprint 的关键协调机制。所有技能都往这个目录读写：

```
~/.gstack/projects/acme-dashboard/
├── alice-feat-auth-design-20260320-143022.md       ← /office-hours 产出
├── alice-feat-auth-test-plan-20260320-150315.md     ← /plan-eng-review 产出
├── alice-feat-auth-test-outcome-20260320-155508.md  ← /qa 产出
├── bob-feat-billing-design-20260320-141505.md       ← 另一个 Sprint
├── feat-auth-reviews.jsonl                          ← 审查日志
├── feat-billing-reviews.jsonl                       ← 另一个分支的审查日志
└── ceo-plans/
    ├── 2026-03-20-auth-refactor.md                  ← CEO Review 持久化
    └── archive/                                     ← 旧的 CEO Plans
```

文件名约定是 `{user}-{branch}-{type}-{datetime}.md`，这确保了：
- 同一分支的多次运行可以追踪历史（datetime 后缀）
- 不同分支的产出物不会互相覆盖
- 下游技能可以找到最近的相关文档（`ls -t ... | head -1`）

### 设计文档跨会话发现

`/office-hours` 的 Phase 2.5（Related Design Discovery）和所有审查技能的 Design Doc Check 都利用这个共享目录：

```bash
# /plan-eng-review 的 Design Doc Check
DESIGN=$(ls -t ~/.gstack/projects/$SLUG/*-$BRANCH-design-*.md 2>/dev/null | head -1)
[ -z "$DESIGN" ] && DESIGN=$(ls -t ~/.gstack/projects/$SLUG/*-design-*.md 2>/dev/null | head -1)
```

先找这个分支的设计文档；找不到就找这个项目的任何设计文档。这意味着 Sprint A 中 `/office-hours` 产出的设计文档会被 Sprint B 中 `/plan-eng-review` 自动发现和使用——即使它们在不同的 Claude Code 会话中运行。

### 审查日志跨对话持久化

```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"plan-eng-review",
"timestamp":"2026-03-20T14:30:00","status":"clean","unresolved":0,
"critical_gaps":0,"mode":"FULL_REVIEW","commit":"abc1234"}'
```

审查日志写入 `$BRANCH-reviews.jsonl`。当你在一个新的 Claude Code 会话中运行 `/ship` 时，它会读取这些日志来决定审查就绪状态——即使审查是在几小时前的另一个会话中完成的。

---

## Vendored Symlink 风险

### 开发 gstack 时的特殊情况

`CLAUDE.md` 中有一段专门讨论 vendored symlink 的警告：

```
When developing gstack, .claude/skills/gstack may be a symlink back to this
working directory (gitignored). This means skill changes are live immediately —
great for rapid iteration, risky during big refactors where half-written skills
could break other Claude Code sessions using gstack concurrently.
```

如果你是 gstack 的贡献者，`.claude/skills/gstack` 可能是一个指向你工作目录的 symlink。这意味着：

- 模板变更 + `bun run gen:skill-docs` **立即影响所有 gstack 调用**
- 破坏性的 SKILL.md.tmpl 变更可以**搞坏并发的 gstack 会话**
- 在大重构期间，半写完的技能可能被其他窗口的 Claude Code 读取

检查方法：

```bash
ls -la .claude/skills/gstack
```

如果是 symlink，在大重构期间应该移除它（`rm .claude/skills/gstack`），让全局安装的 `~/.claude/skills/gstack/` 被使用。

这个问题对 gstack 用户不适用（他们使用的是安装好的副本），但对并行开发 gstack 本身的贡献者来说是一个重要的风险。

---

## 实际的限制

### 10-15 是当前的实际上限

并行 Sprint 不是无限可扩展的。当前的实际限制在 10-15 个并行 Sprint，原因是：

**认知负荷：** 每个 Sprint 偶尔需要你做决策（AskUserQuestion）。如果 15 个 Sprint 同时需要你的注意力，你的决策质量会下降。

**资源消耗：** 每个 Claude Code 会话消耗 API 配额。15 个并行会话意味着 15 倍的 API 调用。

**上下文切换：** 即使有 Re-ground 机制，在 15 个不同的功能方向之间切换仍然需要认知努力。

### 每个 Sprint 的时间线

```
Think    → /office-hours      ~15 min
Plan     → /plan-eng-review   ~10 min
Build    → implement code     ~15 min
Review   → /qa                ~10 min
Test     → tests              ~5 min
Ship     → /ship              ~5 min
                              ─────────
                              ~60 min total per sprint
```

但关键在于：10 个 Sprint 并行运行时，挂钟时间仍然是约 60 分钟——因为你的瓶颈不是 AI 的执行时间（大部分步骤是非交互式的），而是你自己做决策的时间。

```
串行执行 10 个 Sprint:  10 × 60min = 600 min (10 hours)
并行执行 10 个 Sprint:  ~60-90 min (wall clock)
                        ──────────
                        压缩比: ~7-10x
```

---

## 如何开始并行 Sprint

### 从 2-3 个开始

不要试图一下子跳到 10 个并行 Sprint。从 2-3 个开始：

**1. 一个 Sprint 一个 feature branch**

```bash
# Terminal 1
git checkout -b feature/auth-refactor
# 启动 Claude Code, 运行 /office-hours

# Terminal 2
git checkout -b feature/billing-ui
# 启动 Claude Code, 运行 /plan-eng-review

# Terminal 3
git checkout -b feature/api-v2
# 启动 Claude Code, 运行 /qa
```

**2. 不同阶段的 Sprint**

最有效的并行模式是让 Sprint 处于不同阶段：

```
Sprint A: /office-hours   ← 需要你回答问题
Sprint B: /qa             ← 自动运行中，不需要你
Sprint C: /ship           ← 自动运行中，等 PR URL
```

当 Sprint A 在等你回答 Q3 的时候，你可以去看 Sprint C 的 PR URL，或者检查 Sprint B 的 QA 报告。

**3. 做决策，让其余的运行**

你的角色是**方向控制**，不是执行。当 Sprint 需要你的时候（AskUserQuestion），做决策。当它不需要你的时候，让它自主运行。

### 扩展到 5-8 个

当你习惯了 2-3 个的节奏后：

- 开始用窗口管理器（tiling window manager）或 tmux 来组织终端
- 给每个终端窗口一个有意义的名字（feature name，不是 "Terminal 1"）
- 建立一个检查节奏：每 5-10 分钟扫一遍所有窗口，处理等待中的决策
- 让非交互式步骤在后台运行（`/ship` 和 `/qa` 大部分时间不需要你）

### 扩展到 10-15 个

这是 CEO 级别的工作模式：

```
10-15 个并行 Sprint
每个 Sprint: think → plan → build → review → test → ship
每个约 30-60 分钟
但它们并行运行，所以挂钟时间仍然是 ~60 分钟
```

在这个级别，你需要：

- **优先级排序：** 不是所有 Sprint 都同等重要。有些是关键路径，有些是探索性的
- **批量决策：** 集中处理所有等待中的 AskUserQuestion，而不是逐个切换
- **信任自动化：** `/ship` 说测试通过了就是通过了。不要手动验证每个 Sprint 的每个步骤
- **利用共享状态：** 一个 Sprint 的 `/office-hours` 设计文档可以影响另一个 Sprint 的架构决策

---

## 经济学

### 产出量

这种工作模式产出的代码量是惊人的：

```
10 个并行 Sprint × 平均 1000 LOC/Sprint = 10,000 LOC/天
```

这不是玩具代码——它经过了 `/plan-eng-review` 的架构审查、`/qa` 的测试验证、`/ship` 的覆盖率审计。

Garry Tan 在 CHANGELOG 中分享的数据：**60 天，600,000 行生产代码，35% 是测试。** 这是一个人（兼职）加 AI 的产出。

### 压缩比

| 维度 | 人工团队 | CC + gstack 并行 Sprint | 压缩比 |
|------|---------|------------------------|--------|
| 一个功能从想法到 PR | 2-4 周 | ~60 分钟 | ~200-400x |
| 10 个功能从想法到 PR | 20-40 周 | ~60 分钟（并行） | ~2000-4000x |
| 每日代码产出 | ~200-500 LOC/人/天 | 10,000+ LOC/天 | ~20-50x |
| QA + 修复周期 | 3-5 天 | ~10 分钟 | ~400x |

这些数字看起来不真实，但它们建立在一个关键洞察上：**AI 时代的瓶颈不是执行，而是决策。** 并行 Sprint 让你把决策时间本身并行化——当你在等 Sprint A 的测试跑完时，你在为 Sprint B 做架构决策。

### 从 20 人团队到 1 人 + AI

传统软件开发中，一个 20 人团队大约有 5 个工程师在写代码，5 个在审查代码，3 个在做 QA，2 个在做项目管理，2 个在做设计，3 个在做其他事情。

gstack 并行 Sprint 模式下：
- 写代码 → AI（实现）
- 审查代码 → `/plan-eng-review` + `/plan-ceo-review`
- QA → `/qa`
- 项目管理 → Sprint 结构本身（TODOS.md、设计文档、审查日志）
- 设计 → `/plan-design-review` + `/office-hours`
- **你** → 做所有的决策

这不是"AI 替代了 19 个人"——这是"AI 让 1 个人能以 20 人团队的产出量工作"。区别在于：那 1 个人必须有 CEO 级别的决策能力。gstack 的技能体系（office hours → review trilogy → qa → ship）提供了结构，让这种决策能力可以被系统地发挥。

---

## 常见陷阱

### 1. 不要在同一个分支上开多个 Sprint

每个 Sprint 需要自己的分支。两个 Sprint 操作同一个分支会导致 commit 交错、合并冲突、测试状态混乱。

### 2. 不要忽略 Review 门槛

并行 Sprint 的诱惑是跳过审查直接 ship。不要。`/ship` 的 Review Readiness Dashboard 存在是有原因的——跳过 Eng Review 的快速修复往往在下一个 Sprint 中变成阻塞问题。

### 3. 不要在所有 Sprint 上同时做决策密集的阶段

最高效的并行模式是**阶段交错**：

```
好的模式：
Sprint A: /office-hours (决策密集)
Sprint B: /qa (自动运行)
Sprint C: /ship (自动运行)
Sprint D: implementing code (自动运行)

差的模式：
Sprint A: /office-hours Q3 (等你回答)
Sprint B: /office-hours Q2 (等你回答)
Sprint C: /plan-eng-review Step 0 (等你决定)
Sprint D: /plan-ceo-review opt-in ceremony (等你决定)
```

### 4. 定期同步基分支

并行 Sprint 运行时间越长，与基分支的偏离越大。定期（至少每天）让每个 Sprint 合并基分支。`/ship` 的 Step 2 会自动做这件事，但如果 Sprint 还没到 ship 阶段，手动 merge 也很重要。

---

## 总结：一种新的工作方式

并行 Sprint 不只是"更快"——它是一种质的变化。你从"一个人写代码"变成"一个人运营一个 AI 工程团队"。gstack 的技能体系提供了流程骨架（office hours → review → qa → ship），会话感知提供了上下文保持，端口隔离和共享状态提供了基础设施。

但最终，这种工作模式的限制不在技术，而在人：你能同时追踪多少个功能方向？你能多快做出高质量的决策？你对自己产品的理解有多深？

这些问题没有技术解决方案。它们需要的是创始人级别的判断力——而这正是 `/office-hours` 从第一天就试图帮你建立的东西。

---

*本文是 gstack 深度知识库的最后一篇。返回 [阅读路径指南](00-reading-guide.md) 查看完整索引。*

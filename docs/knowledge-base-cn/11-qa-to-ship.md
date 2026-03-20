# 11. 从 `/qa` 到 `/ship`：自动化发布的完整链路

> 从找到 bug 到发布 PR，整条链路是自动化的。`/qa` 测试、修复、生成回归测试、产出健康分数；`/ship` 合并基分支、跑测试、审查 diff、版本号自增、生成 CHANGELOG、拆分 bisectable commits、推送、创建 PR。用户说 "/ship"，下一件看到的就是 PR URL。本文拆解这两个技能如何协作构成完整的发布链路。

---

## `/qa` 深度拆解

### 三个层级

`/qa` 不是"找所有 bug"——它按严重程度分层：

| 层级 | 修复范围 | 场景 |
|------|---------|------|
| Quick | 只修 Critical + High | 快速冒烟测试，30 秒 |
| Standard（默认） | + Medium | 常规 QA，5-15 分钟 |
| Exhaustive | + Low/Cosmetic | 全面 QA，含视觉细节 |

层级决定了 Phase 7 Triage 中哪些问题会被修复，哪些被标记为"deferred"。

### Diff-aware 模式：最常见的使用场景

当用户在 feature branch 上说 `/qa` 而不提供 URL 时，自动进入 diff-aware 模式。这是最常见的场景——开发者刚写完代码，想验证它能用。

```bash
git diff main...HEAD --name-only    # 什么文件变了
git log main..HEAD --oneline         # 什么 commit 在这个分支上
```

然后从变更的文件反推受影响的页面/路由：
- Controller/route 文件 → 它们服务哪些 URL 路径
- View/template/component 文件 → 哪些页面渲染它们
- Model/service 文件 → 哪些页面使用这些 model
- CSS/style 文件 → 哪些页面包含这些样式表
- API 端点 → 直接用 `$B js "await fetch('/api/...')"` 测试

### Clean Working Tree 要求

`/qa` 需要 clean working tree，因为每个 bug 修复需要自己的原子 commit。如果 tree 是 dirty 的，它会停下来问：

```
A) Commit my changes — 提交所有当前变更，然后开始 QA
B) Stash my changes  — stash，跑 QA，完了 pop 回来
C) Abort             — 我自己清理
```

推荐 A，因为未提交的工作应该作为 commit 保存，然后才让 QA 添加自己的修复 commits。

### Test Plan Context：从上游技能获取测试计划

在依赖 git diff 启发式之前，`/qa` 会先检查更丰富的测试计划来源：

```bash
source <(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)
ls -t ~/.gstack/projects/$SLUG/*-test-plan-*.md 2>/dev/null | head -1
```

如果 `/plan-eng-review` 产出了测试计划 artifact，`/qa` 会用它作为主要测试输入——这比 git diff 启发式精确得多，因为 Eng Review 已经分析了确切的测试范围和意图。

### Phases 1-6：QA 基线

前六个阶段构成 QA 基线：

1. **Phase 1: Initialize** —— 找到 browse binary、创建输出目录、启动计时
2. **Phase 2: Authenticate** —— 如有需要，处理登录（填表单、导入 cookies、2FA 等待）
3. **Phase 3: Orient** —— 获取应用地图（截图、链接、控制台错误、框架检测）
4. **Phase 4: Explore** —— 系统地访问每个页面，按深度优先级（核心功能 > 次要页面）
5. **Phase 5: Document** —— 发现问题时立即记录（交互 bug 需 before/after 截图）
6. **Phase 6: Wrap Up** —— 计算健康分数、写 Top 3、控制台健康汇总

### Phase 7：Triage

按严重程度排序所有发现的问题，根据选定的层级决定修复哪些：

```
Quick:      修 Critical + High。标记 Medium/Low 为 "deferred"
Standard:   修 Critical + High + Medium。标记 Low 为 "deferred"
Exhaustive: 修所有，包括 Cosmetic/Low
```

无法从源代码修复的问题（第三方 widget bug、基础设施问题）无论层级都标记为 "deferred"。

### Phase 8：Fix Loop —— 核心机制

这是 `/qa` 最复杂的阶段。对于每个可修复的问题，按严重程度排序执行：

```
8a. Locate → 8b. Fix → 8c. Commit → 8d. Re-test → 8e. Classify → 8e.5 Regression Test → 8f. Self-Regulate
```

#### 8a. 定位源码

```bash
# Grep 错误消息、组件名、路由定义
# Glob 匹配受影响页面的文件模式
```

只修改与问题直接相关的文件。

#### 8b. 修复

阅读源代码，理解上下文，做**最小修复**——不重构周围代码、不添加功能、不"改进"不相关的东西。

#### 8c. 原子提交

```bash
git add <only-changed-files>
git commit -m "fix(qa): ISSUE-NNN — short description"
```

**一个 commit 对应一个修复。永远不捆绑多个修复。**

#### 8d. 重新测试

导航回受影响的页面，拍 before/after 截图对，检查控制台，用 `snapshot -D` 验证变更有预期效果。

#### 8e. 分类

三种分类：
- **verified**：重新测试确认修复有效，没有引入新错误
- **best-effort**：修复已应用但无法完全验证（需要 auth 状态、外部服务等）
- **reverted**：检测到回归 → `git revert HEAD` → 标记为 "deferred"

#### 8e.5. 回归测试生成

这是 `/qa` 最有技术含量的步骤。跳过条件：分类不是 "verified"、或纯 CSS 修复、或没有测试框架。

流程：

**1. 研究项目现有的测试模式：** 读取最接近修复的 2-3 个测试文件。精确匹配文件命名、导入、断言风格、describe/it 嵌套、setup/teardown 模式。回归测试必须看起来像同一个开发者写的。

**2. 追踪 bug 的代码路径，然后写测试：** 在写测试之前追踪数据流：

```
- 什么输入/状态触发了 bug？（精确的前置条件）
- 它走了什么代码路径？（哪些分支、哪些函数调用）
- 在哪里断了？（精确的行/条件）
- 什么其他输入可能命中同一代码路径？（修复周围的边界情况）
```

测试**必须**：
- 设置触发 bug 的前置条件
- 执行暴露 bug 的动作
- 断言正确的行为（不是 "it renders" 或 "it doesn't throw"）
- 包含完整的归因注释

```javascript
// Regression: ISSUE-NNN — {what broke}
// Found by /qa on {YYYY-MM-DD}
// Report: .gstack/qa-reports/qa-report-{domain}-{date}.md
```

**3. 运行测试：** 只运行新测试文件。通过 → commit。失败 → 修一次。还失败 → 删除测试，defer。

#### 8f. WTF-likelihood 自我调节

每 5 个修复（或任何 revert 之后），计算 WTF-likelihood 启发式：

```
WTF-LIKELIHOOD:
  从 0% 开始
  每次 revert:                +15%
  每次修复涉及 >3 个文件:      +5%
  超过 15 个修复后:            每个额外修复 +1%
  所有剩余都是 Low severity:   +10%
  修改了不相关的文件:          +20%
```

**如果 WTF > 20%：** 立即停止。给用户看到目前为止做了什么，问是否继续。

**硬上限：50 个修复。** 超过 50 个修复后无条件停止。

这个机制防止 AI 陷入"越修越坏"的螺旋——当 revert 开始累积、修复开始扩散到不相关的文件时，继续下去的风险超过了收益。

### Phase 9：Final QA

所有修复完成后：
1. 对所有受影响的页面重新运行 QA
2. 计算最终健康分数
3. **如果最终分数比基线更差：突出警告** —— 有什么退步了

### Phase 10：Report —— 双输出

报告写入两个位置：

- **本地：** `.gstack/qa-reports/qa-report-{domain}-{YYYY-MM-DD}.md`
- **项目范围：** `~/.gstack/projects/{slug}/{user}-{branch}-test-outcome-{datetime}.md`

```
.gstack/qa-reports/
├── qa-report-{domain}-{YYYY-MM-DD}.md    # 结构化报告
├── screenshots/
│   ├── initial.png                        # 着陆页注释截图
│   ├── issue-001-step-1.png               # 每个问题的证据
│   ├── issue-001-result.png
│   ├── issue-001-before.png               # 修复前
│   ├── issue-001-after.png                # 修复后
│   └── ...
└── baseline.json                          # 用于回归模式
```

摘要包含：总问题数、修复数（verified: X, best-effort: Y, reverted: Z）、deferred 问题数、健康分数变化量。

### qa-only 变体

`/qa-only` 使用完全相同的测试方法论（Phases 1-6），但**永远不修任何东西**。只产出报告。不读源代码、不编辑文件、不在报告中建议修复。它的唯一工作是报告什么是坏的。

---

## `/ship` 深度拆解

### 全自动哲学

`/ship` 的核心设计原则用一句话概括：

```
The goal is: user says /ship, next thing they see is the PR URL.
```

除了几个必须停下的情况外，整个流程是非交互式的：

**必须停下的：** 在基分支上（中止）、无法自动解决的合并冲突、测试失败、需要用户判断的 ASK 项、MINOR/MAJOR 版本号提升

**永远不停的：** 未提交的变更（直接包含）、版本号选择（自动选 MICRO 或 PATCH）、CHANGELOG 内容（自动从 diff 生成）、commit message 审批（自动提交）、多文件变更集（自动拆分 bisectable commits）

### Step 1：Pre-flight + Review Readiness Dashboard

预检查不只是看 git status——它检查审查就绪状态。

如果 Eng Review 不是 "CLEAR"：
1. 先检查是否有之前的 override（这个分支已经被批准跳过过）
2. 如果没有 override，问用户：
   - A) 直接发
   - B) 中止——先跑 /plan-eng-review
   - C) 变更太小不需要 eng review

如果用户选择 A 或 C，决策会被持久化：

```bash
echo '{"skill":"ship-review-override","timestamp":"...","decision":"ship_anyway"}' \
  >> ~/.gstack/projects/$SLUG/$BRANCH-reviews.jsonl
```

这样同一个分支的后续 `/ship` 运行不会再问。

### Step 2：合并基分支（测试之前）

```bash
git fetch origin <base> && git merge origin/<base> --no-edit
```

在测试之前合并基分支，这样测试跑在合并后的状态上。简单冲突（VERSION、schema.rb、CHANGELOG 排序）尝试自动解决。复杂冲突则停下来展示。

### Step 2.5：Test Framework Bootstrap

如果项目没有测试框架，`/ship` 会提供引导选项。这确保即使是全新的项目也能有测试。

### Step 3：运行测试（在合并后的代码上）

并行运行测试套件：

```bash
bin/test-lane 2>&1 | tee /tmp/ship_tests.txt &
npm run test 2>&1 | tee /tmp/ship_vitest.txt &
wait
```

**任何测试失败：停下来。不继续。**

### Step 3.25：Eval Suites（条件性）

只在 prompt 相关文件变更时运行。检查 diff 是否触及：
- `app/services/*_prompt_builder.rb`
- `config/system_prompts/*.txt`
- `test/evals/**/*`
- 等等

如果没有匹配："No prompt-related files changed — skipping evals."

如果匹配：用 `EVAL_JUDGE_TIER=full`（Opus 级别评估）运行受影响的评估套件。`/ship` 是发布前门槛，所以永远用最高级别。

### Step 3.4：Test Coverage Audit —— 覆盖率审计

这一步不是简单地看覆盖率百分比——它**追踪每一条代码路径**。

**1. 追踪每条变更的代码路径：**

对每个变更的文件，读取完整文件（不只是 diff hunk），追踪数据流：
- 输入从哪里来？
- 什么转换它？
- 它去哪里？
- 每一步可能出什么错？

**2. 产出 ASCII 覆盖率图：**

```
CODE PATH COVERAGE
===========================
[+] src/services/billing.ts
    │
    ├── processPayment()
    │   ├── [★★★ TESTED] Happy path + card declined — billing.test.ts:42
    │   ├── [GAP]         Network timeout — NO TEST
    │   └── [GAP]         Invalid currency — NO TEST

USER FLOW COVERAGE
===========================
[+] Payment checkout flow
    │
    ├── [★★★ TESTED] Complete purchase — checkout.e2e.ts:15
    ├── [GAP]         Double-click submit — NO TEST
    └── [GAP]         Navigate away during payment — NO TEST

─────────────────────────────────
COVERAGE: 5/12 paths tested (42%)
GAPS: 7 paths need tests
─────────────────────────────────
```

质量评分：
- ★★★ 测试了行为 + 边界情况 + 错误路径
- ★★  测试了正确行为，只有 happy path
- ★   冒烟测试 / 存在性检查

**3. 为未覆盖的路径生成测试：** 上限 30 条代码路径、20 个生成的测试、每个测试 2 分钟探索上限。

### Step 3.5：Pre-Landing Review

审查 diff 中测试无法捕获的结构性问题：

1. 读取 `.claude/skills/review/checklist.md`
2. 跑完整 diff
3. 两轮审查：
   - **Pass 1 (CRITICAL)：** SQL 与数据安全、LLM 输出信任边界
   - **Pass 2 (INFORMATIONAL)：** 其余所有类别
4. 分类为 AUTO-FIX 或 ASK
5. 自动修复所有 AUTO-FIX 项
6. 如有 ASK 项，一次 AskUserQuestion 呈现

如果任何修复被应用：commit 并**停下来让用户重新运行 `/ship`** 以在修复后的代码上重新测试。

### Step 4：Version Bump 自动决策

```
MICRO (4th digit): < 50 lines changed, trivial tweaks
PATCH (3rd digit): 50+ lines changed, bug fixes, small-medium features
MINOR (2nd digit): ASK the user — major features
MAJOR (1st digit): ASK the user — milestones or breaking changes
```

MICRO 和 PATCH 自动决定。MINOR 和 MAJOR 必须问用户。版本格式是 4 位数：`MAJOR.MINOR.PATCH.MICRO`。提升一位重置其右边所有位为 0。

### Step 5：CHANGELOG 自动生成

从**分支上的所有 commits**（不只是最近的）和完整 diff 自动生成。分类为 Added / Changed / Fixed / Removed。**绝不问用户描述变更。**

### Step 5.5：TODOS.md 自动更新

交叉引用 diff 和 commit 历史，自动检测已完成的 TODO 项：
- 匹配 commit 消息与 TODO 标题
- 检查 TODO 引用的文件是否在 diff 中
- 检查 TODO 描述的工作是否与功能变更匹配

**保守原则：只有 diff 中有明确证据时才标记为完成。** 不确定就不动。

完成的项目被移到 `## Completed` 部分，附注版本号和日期。

### Step 6：Bisectable Commits

目标是创建对 `git bisect` 友好的小而逻辑化的 commits：

```
提交顺序（早先 → 晚后）：
1. Infrastructure: 迁移、配置、路由
2. Models & services: 新模型、服务、concerns（带测试）
3. Controllers & views: 控制器、视图、JS/React 组件（带测试）
4. VERSION + CHANGELOG + TODOS.md: 永远在最后一个 commit
```

规则：
- 一个 model 和它的测试文件在同一个 commit
- 每个 commit 必须独立有效——没有 broken imports
- 如果总 diff 很小（< 50 行跨 < 4 文件），单个 commit 就行

### Step 6.5：Verification Gate —— 铁律实施

```
IRON LAW: NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE.
```

在推送之前，如果代码在 Step 3 测试运行后发生了变更（审查修复等），必须重新运行测试：

```
防止合理化：
- "应该能用了"     → 运行它。
- "我有信心"       → 信心不是证据。
- "我之前已经测试过了" → 代码从那以后变了。再测一次。
- "这是个微不足道的变更" → 微不足道的变更搞垮生产环境。
```

**"在没有验证的情况下声称工作完成是不诚实，不是效率。"**

### Step 7-8：Push + PR 创建

推送并创建 PR，PR body 包含结构化的内容：

```
## Summary
## Test Coverage        ← 来自 Step 3.4 的覆盖率图
## Pre-Landing Review   ← 来自 Step 3.5 的发现
## Design Review        ← 如果有前端变更
## Eval Results         ← 如果 evals 运行了
## Greptile Review      ← 如果有 Greptile 评论
## TODOS               ← 完成/创建/重组的 TODOS
## Test plan
```

### Step 8.5：Auto-invoke `/document-release`

PR 创建后，自动同步项目文档：
- 读取项目中所有 `.md` 文件
- 交叉引用 diff
- 更新任何漂移的文档（README、ARCHITECTURE、CONTRIBUTING、CLAUDE.md 等）
- 如果有更新，commit 并推送到同一分支

**这一步是自动的。不问用户确认。** 用户运行 `/ship`，文档自动保持最新。

---

## `/qa` 和 `/ship` 的连接

两个技能之间有三条信息流：

### 1. Test Plan：从 Eng Review 流向 QA

```
/plan-eng-review → 写入 test-plan-*.md → /qa 读取作为测试输入
```

### 2. QA Findings：流入 Ship 的 Pre-Landing Review

`/qa` 产出的报告和健康分数可以作为 `/ship` 的 Step 3.5 Pre-Landing Review 的参考。如果 `/qa` 发现了问题并修复了它们，那些修复 commits 已经在分支上，`/ship` 的测试会验证它们。

### 3. Review Log：跨技能状态共享

```bash
~/.gstack/projects/$SLUG/$BRANCH-reviews.jsonl
```

每个审查技能（CEO Review、Eng Review、Design Review、QA）都往这个 JSONL 文件写入状态。`/ship` 在 Step 1 读取它来决定审查就绪状态。

---

## 完整链路的时间线

一个典型的从想法到 PR 的完整链路：

```
时间线（并行执行时的挂钟时间）
═══════════════════════════════════════════
T+0min   /office-hours 开始
T+15min  设计文档批准
T+15min  /plan-eng-review 开始
T+25min  架构锁定，测试计划写入
T+25min  开始实现代码
T+50min  代码完成
T+50min  /qa 开始
T+60min  QA 报告 + 修复完成
T+60min  /ship 开始
T+65min  测试通过，coverage audit 完成
T+70min  PR 创建，文档同步
═══════════════════════════════════════════
总计: ~70 分钟，从想法到 PR
```

这个时间线在人工团队中通常是 2-4 周。

---

*下一篇：[并行 Sprint 实战](12-parallel-sprints.md)*

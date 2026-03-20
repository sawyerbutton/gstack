# 10. Review 三件套拆解：CEO / 工程 / 设计

> gstack 的审查体系不是一个审查，而是三个视角。CEO Review 问"我们该建什么"，Eng Review 问"我们该怎么建"，Design Review 问"它该给人什么感觉"。三个视角互补，覆盖了一个产品决策的所有维度。本文逐一拆解每个审查的内部机制，展示它们各自的认知模型、交互设计和产出物格式，最后分析它们如何链接成一条完整的审查链。

---

## 三个视角，为什么互补

大多数代码审查工具只有一个维度：代码质量。gstack 把审查拆成三个角色，每个角色有自己的认知模式、关注重点和输出格式：

| 维度 | 技能 | 核心问题 | 认知模式 |
|------|------|---------|---------|
| 产品/战略 | `/plan-ceo-review` | 我们该建什么？ | 10x 思维、反转检验、时间深度 |
| 工程 | `/plan-eng-review` | 我们该怎么建？ | 无聊优先、增量优于革命、系统优于英雄 |
| 设计 | `/plan-design-review` | 它该给人什么感觉？ | 层级即服务、减法优先、信任在像素级别构建 |

这种拆分反映了一个真实的认知事实：一个人很难同时用三种思维方式审视同一个计划。CEO 在考虑"这够不够大胆"的时候，工程师在考虑"这在凌晨三点值班的时候好不好调试"，设计师在考虑"这个空状态会让用户产生什么感受"。把它们拆开，每个维度都能得到全力专注的审视。

三个审查之间还有一个微妙但重要的权力关系：Eng Review 是唯一的**强制发布门槛**。CEO Review 和 Design Review 是推荐的但可选的。这反映了一个务实的判断——代码质量是不可妥协的底线，而产品方向和设计品质虽然重要，但在某些情况下（比如小修复、配置变更）可以合理跳过。

---

## CEO Review (`/plan-ceo-review`) 深度拆解

### 哲学定位：不是橡皮图章

模板的开场白直接定义了姿态：

```
You are not here to rubber-stamp this plan. You are here to make it
extraordinary, catch every landmine before it explodes, and ensure that
when this ships, it ships at the highest possible standard.
```

CEO Review 不是在问"这个计划行不行"，而是在问"这个计划怎么才能变得非凡"。这个姿态上的差异决定了整个审查的深度和广度。

### 四种模式的详细对比

CEO Review 最独特的设计是四种模式。每种模式的审查姿态、复杂度问题、产出物完全不同：

```
┌────────────────────────────────────────────────────────────────────┐
│                        MODE COMPARISON                             │
├─────────────┬──────────────┬──────────────┬──────────┬────────────┤
│             │  EXPANSION   │  SELECTIVE   │  HOLD    │ REDUCTION  │
├─────────────┼──────────────┼──────────────┼──────────┼────────────┤
│ Scope       │ Push UP      │ Hold + offer │ Maintain │ Push DOWN  │
│ Recommend   │ Enthusiastic │ Neutral      │ N/A      │ N/A        │
│ posture     │              │              │          │            │
│ 10x check   │ Mandatory    │ Cherry-pick  │ Optional │ Skip       │
│ Platonic    │ Yes          │ No           │ No       │ No         │
│ ideal       │              │              │          │            │
│ Complexity  │ "Is it big   │ "Is it right │ "Is it   │ "Is it the │
│ question    │  enough?"    │  + what else │  too     │  bare      │
│             │              │  is tempting"│ complex?"│  minimum?" │
│ CEO plan    │ Written      │ Written      │ Skipped  │ Skipped    │
│ Error map   │ Full + chaos │ Full + chaos │ Full     │ Critical   │
│             │  scenarios   │ for accepted │          │ paths only │
└─────────────┴──────────────┴──────────────┴──────────┴────────────┘
```

模式选择有上下文感知的默认值。全新功能默认进入 EXPANSION 模式，鼓励大胆思考；功能增强默认进入 SELECTIVE EXPANSION 模式，在稳定基础上探索扩展；Bug 修复和重构默认进入 HOLD SCOPE 模式，专注于做好当前工作；涉及超过 15 个文件的庞大计划会建议进入 REDUCTION 模式，先砍到最小可行版本。

### Opt-in 仪式：用户始终掌控

在 EXPANSION 和 SELECTIVE EXPANSION 模式下，审查中发现的每个范围扩展机会都通过 AskUserQuestion **逐一呈现**给用户。这是 CEO Review 最精妙的交互设计——它绝不会悄悄地把功能塞进计划。

EXPANSION 模式的流程是：先描述 10x 愿景和 Platonic Ideal（"如果世界上最好的工程师有无限时间和完美品味，这个系统会是什么样？"），然后从这些愿景中提炼出具体的范围提案，每个提案作为独立的 AskUserQuestion 呈现。推荐姿态是热情的——解释为什么值得做。

SELECTIVE EXPANSION 模式的流程类似，但推荐姿态是中性的——呈现机会、说明工作量和风险、让用户决定。不带偏见。

每个扩展提案有三个选项：A) 加入本次计划范围；B) 推迟到 TODOS.md；C) 跳过。被接受的项目立即成为后续所有审查章节的计划范围。被拒绝的进入"NOT in scope"文档。

模板中有一条铁律强调这一点：

```
Once the user selects a mode, COMMIT to it. Do not silently drift toward
a different mode. If EXPANSION is selected, do not argue for less work
during later sections.
```

### 十个审查章节

选定模式并确认范围后，CEO Review 进入十个完整的审查章节。这十个章节的覆盖范围之广令人印象深刻——从架构到安全、从性能到部署、从可观测性到长期轨迹，每个章节都有自己的评估框架和强制输出格式。

每个章节结束后都有一个 `STOP` 指令——逐个问题用 AskUserQuestion 与用户交互。这确保了审查不是单向输出，而是双向对话。

### Error & Rescue Map：最关键的产出物

第二章节（Error & Rescue Map）是整个 CEO Review 中最有辨识度的产出物。它要求对每个可能失败的新方法、服务或代码路径填写一张完整的二维表格：

```
METHOD/CODEPATH          | WHAT CAN GO WRONG           | EXCEPTION CLASS
-------------------------|-----------------------------|-----------------
ExampleService#call      | API timeout                 | TimeoutError
                         | API returns 429             | RateLimitError
                         | API returns malformed JSON  | JSONParseError
                         | DB connection pool exhausted| ConnectionPoolExhausted
                         | Record not found            | RecordNotFound

EXCEPTION CLASS              | RESCUED?  | RESCUE ACTION          | USER SEES
-----------------------------|-----------|------------------------|------------------
TimeoutError                 | Y         | Retry 2x, then raise   | "暂时不可用"
RateLimitError               | Y         | Backoff + retry         | 无（透明处理）
JSONParseError               | N ← GAP   | —                      | 500 error ← 问题
ConnectionPoolExhausted      | N ← GAP   | —                      | 500 error ← 问题
```

这张表的设计哲学是**零静默失败**：每个失败模式都必须对系统、团队和用户可见。如果一个失败可以静默发生，那就是计划中的关键缺陷。

规则也非常严格：Catch-all 错误处理（`rescue StandardError`、`catch (Exception e)`）永远被标记为代码异味。每个被捕获的错误必须做三件事之一——带退避重试、优雅降级并给用户可见消息、或附加上下文后重新抛出。"吞掉并继续"几乎永远不可接受。

### 四条数据路径

对于每个新数据流，必须追踪四条路径——Happy Path（数据正确流动）、Nil Path（输入为空/缺失时会怎样）、Empty Path（输入存在但为空/零长度时会怎样）、Error Path（上游调用失败时会怎样）。这四条路径的完整追踪确保了不只是"正常情况能用"，而是"所有情况都有明确的行为"。

### CEO Plan 持久化与推广到 docs/designs/

在 EXPANSION 和 SELECTIVE EXPANSION 模式下，审查产出的愿景和决策会被持久化到磁盘：

```bash
~/.gstack/projects/$SLUG/ceo-plans/{date}-{feature-slug}.md
```

文档包含完整的 10x Check 愿景、Platonic Ideal 描述、以及所有范围决策的表格。审查结束时还有一个重要的推广步骤——如果愿景足够引人注目，可以将 CEO Plan 推广到项目仓库的 `docs/designs/` 目录，从个人参考变为团队共享的设计文档。

---

## Eng Review (`/plan-eng-review`) 深度拆解

### Step 0：范围挑战——审查从质疑开始

Eng Review 的第一步不是审查代码质量——是挑战计划本身的范围。五个核心问题：

1. **现有代码杠杆：** 什么现有代码已经部分或完全解决了每个子问题？能否从现有流程中捕获输出而不是构建平行的流程？
2. **最小变更集：** 达成目标的最小变更集是什么？标记所有可以推迟而不阻塞核心目标的工作
3. **复杂度异味：** 如果计划涉及超过 8 个文件或引入超过 2 个新类/服务，视为异味并质疑
4. **TODOS 交叉引用：** 有没有被推迟的项阻塞了这个计划？有没有推迟的项可以捎带完成？
5. **完整性检查：** 计划是做完整版还是走捷径？用 AI 辅助编码时，完整性的成本降低了 10-100 倍

复杂度阈值——8 个文件或 2 个新类——不是随意数字。它代表了一个经验判断：超过这个阈值，计划很可能过度设计了，同样的目标可以用更少的移动部件达成。

如果复杂度检查触发，Eng Review 会主动通过 AskUserQuestion 推荐范围缩减——解释什么过度构建了、提出最小版本、问是否缩减还是继续。如果没有触发，直接展示 Step 0 的发现并进入正式审查。

### 认知模式：工程经理的思维本能

Eng Review 嵌入了 15 个认知模式，每个都有明确的思想来源和实际应用场景。这些不是清单项——模板明确说它们是"instincts"，是经验丰富的工程领导者经过多年发展出来的思维本能。

核心模式包括：**Boring by default**（每家公司大约有三个创新代币，其他一切都应该是成熟技术——来自 McKinley 的《Choose Boring Technology》）；**Incremental over revolutionary**（绞杀者模式而不是大爆炸，金丝雀发布而不是全量上线——来自 Fowler）；**Systems over heroes**（为凌晨三点值班的疲惫人类设计，不是为最佳状态的最优工程师设计）；**Essential vs accidental complexity**（在添加任何东西之前问：这是在解决真实问题还是我们自己制造的问题？——来自 Brooks 的《No Silver Bullet》）；**Make the change easy, then make the easy change**（先重构，再实现——来自 Beck）。

审查时，评估架构用"boring by default"思维，审查测试用"systems over heroes"思维，评估复杂度时问 Brooks 的问题。这些认知模式不是孤立应用的，而是融入审查的每一个判断中。

### 强制 ASCII 图表与图表维护

模板不只要求产出图表，还要求**维护图表**：

```
Diagram maintenance is part of the change. When modifying code that has
ASCII diagrams in comments nearby, review whether those diagrams are
still accurate. Stale diagrams are worse than no diagrams.
```

过时的图表比没有图表更危险——它们会积极误导读者。在审查过程中遇到的过时图表，即使在直接范围之外，也必须标记出来。

### Test Plan Artifact：连接 Eng Review 和 QA

Eng Review 完成测试审查后，会写一个测试计划 artifact。这个 artifact 被存储在 `~/.gstack/projects/{slug}/{user}-{branch}-test-plan-{datetime}.md`，内容包含受影响的页面和路由、需要验证的关键交互、边界情况、关键端到端路径。

这是 Eng Review 和 `/qa` 之间的关键连接点。`/qa` 默认依靠 git diff 启发式来决定测试什么——这是有损的。有了 Eng Review 产出的测试计划，`/qa` 知道了确切的测试范围和意图，测试质量大幅提升。

### Failure Modes Registry 和 Lake Score

每个新代码路径都必须填入 Failure Modes Registry：

```
CODEPATH | FAILURE MODE   | RESCUED? | TEST? | USER SEES?     | LOGGED?
---------|----------------|----------|-------|----------------|--------
```

任何一行同时满足 RESCUED=N、TEST=N、USER SEES=Silent 的条件 → 标记为 **CRITICAL GAP**。

Lake Score 是完成摘要中的一个独特指标，追踪在整个审查过程中有多少推荐选择了完整版本（boil the lake）而不是捷径。它是 Completeness Principle 的量化度量——如果 Lake Score 是 8/10，意味着十个决策点中有八个选择了完整实现。

---

## Design Review (`/plan-design-review`) 拆解

### 核心机制：0-10 评分 + 修复到 10

Design Review 的独特之处不在于它审查什么（大多数设计审查都会看信息架构、交互状态等），而在于**它如何审查**。0-10 评分法创造了一个迭代循环：

```
评分 → 差距分析 → 修复 → 重新评分 → 再修复 → 直到 10 或用户说够了
```

这不是"看看有什么问题然后报告"——这是"找到差距然后直接把计划改好"。Design Review 的输出不是一份问题清单，而是**一个更好的计划**。

模板中的描述非常明确："The output of this skill is a better plan, not a document about the plan."（这个技能的输出是一个更好的计划，不是一份关于计划的文档。）

### 七个审查 Pass 与每个 Pass 的深度

七个 Pass 覆盖了设计审查的所有维度，每个都有自己的评分标准和修复目标：

**Pass 1: Information Architecture** —— 审查信息层级。用户先看到什么、后看到什么、最后看到什么？应用"约束崇拜"原则：如果你只能展示三样东西，哪三样最重要？产出包含屏幕/页面结构和导航流程的 ASCII 图。

**Pass 2: Interaction State Coverage** —— 审查五种交互状态：Loading、Empty、Error、Success、Partial。对每个 UI 功能生成覆盖表。核心原则是"空状态是功能"——"没有找到项目"不是设计，每个空状态需要温暖、一个主要行动按钮、和上下文说明。

**Pass 3: User Journey & Emotional Arc** —— 审查用户的情感体验。生成用户旅程故事板，应用时间地平线设计——5 秒的本能反应（视觉冲击）、5 分钟的行为体验（使用流程是否顺畅）、5 年的反思关系（产品在用户生活中的位置）。

**Pass 4: AI Slop Risk** —— 这是 Design Review 最独特也最有争议的章节。它专门检测 AI 生成的通用 UI 模式——那些看起来"干净现代"但实际上和每个 SaaS 模板一模一样的设计。"Cards with icons"要质疑它和通用模板有什么不同；"Hero section"要质疑什么让它感觉像这个产品；"Clean, modern UI"直接判定为毫无意义的描述，必须替换为实际的设计决策。

**Pass 5: Design System Alignment** —— 如果项目有 DESIGN.md，检查计划是否对齐。如果没有，标记为缺口并推荐先运行 `/design-consultation`。

**Pass 6: Responsive & Accessibility** —— 审查响应式设计和无障碍性。响应式不是"在移动端堆叠"——每个视口需要有意图的布局变更。无障碍性包括键盘导航模式、ARIA 地标、触摸目标尺寸（最小 44px）、色彩对比要求。

**Pass 7: Unresolved Design Decisions** —— 找出会在实现时困扰开发者的模糊之处。以表格形式呈现每个需要做的决策以及如果推迟会发生什么后果。比如"空状态长什么样？"如果推迟，工程师就会发布"No items found."这种敷衍的文本。

### Completion Summary 与分数追踪

Design Review 的完成摘要追踪每个 Pass 的前后分数变化：

```
| Pass 1  (Info Arch)  | 4/10 → 8/10 after fixes                |
| Pass 2  (States)     | 3/10 → 9/10 after fixes                |
| Pass 3  (Journey)    | 6/10 → 8/10 after fixes                |
| Pass 4  (AI Slop)    | 2/10 → 7/10 after fixes                |
| Overall design score | 3.75/10 → 8/10                          |
```

如果所有 Pass 都达到 8 分以上，宣布"计划设计完成，实现后运行 /design-review 做视觉 QA"。如果有低于 8 分的，注明什么未解决以及为什么（用户选择推迟）。

---

## 审查链接：如何从一个审查流向下一个

### Review Log：跨技能的状态持久化

每个审查结束后都会写入 review log：

```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"plan-ceo-review",
"timestamp":"TIMESTAMP","status":"STATUS","unresolved":N,
"critical_gaps":N,"mode":"MODE","commit":"COMMIT"}'
```

这个日志被 `/ship` 的 Review Readiness Dashboard 消费。Dashboard 知道哪些审查已完成、哪些有未解决的问题、哪些可能过时了（通过比较 commit hash）。

### 审查推荐逻辑

每个审查结束后会读取 dashboard 输出，智能推荐下一步：

- **CEO Review 之后：** 如果 Eng Review 没有被全局跳过（`skip_eng_review` 设置），推荐运行 Eng Review 作为必需的发布门槛。如果这次 CEO Review 扩展了范围或改变了架构方向，强调需要一次新的 Eng Review。如果检测到 UI 范围，同时推荐 Design Review。
- **Eng Review 之后：** 如果有 UI 变更且没有 Design Review 存在，建议运行 Design Review。如果这是重大产品变更且没有 CEO Review，软性提及 CEO Review 的价值。
- **Design Review 之后：** 推荐 Eng Review 作为必需门槛。只在发现根本性产品方向问题（初始分数低于 4/10、信息架构有重大结构问题、质疑是否在解决正确的问题）时才推荐 CEO Review。

### 过时检测机制

每个审查日志记录了当时的 commit hash。当后续审查或 `/ship` 读取 dashboard 时，会比较当前 commit 和审查时的 commit。如果差异显著，标记审查为"可能过时"。这确保了审查链不是一次性的仪式，而是一个活的系统——代码变了，审查也需要更新。

---

## 三套认知模式的深层对比

三个审查最深层的差异不在于它们审查什么，而在于它们**如何思考**。

**CEO 的认知模式**侧重战略决策：反转检验问"什么会让我们失败"（Munger）；聚焦即减法从 350 个产品砍到 10 个（Jobs）；速度校准判断 70% 信息就够做决定了、大部分事情是双向门（Bezos）；时间深度用 5-10 年的弧线思考；创始人模式判断深度参与是扩展还是约束了团队的思考（Chesky/Graham）。

**Eng 的认知模式**侧重工程执行：无聊优先除非有充分理由否则用成熟技术（McKinley）；增量优于革命用绞杀者模式替代大爆炸重写（Fowler）；系统优于英雄为疲惫人类设计而不是最佳状态的最优工程师；本质与偶然复杂度区分真实问题和自造问题（Brooks）；拥有你在生产环境中的代码不要在开发和运维之间建墙（Majors）。

**Design 的认知模式**侧重用户体验：层级即服务回答用户先看什么后看什么；同理心即模拟运行心理模型想象信号差、单手操作、老板在看的场景（Norman）；减法优先如果一个 UI 元素不值得它的像素就砍掉（Rams/Maeda）；有原则的品味让"这感觉不对"可以追溯到一个被打破的原则（Zhuo）；为信任而设计在像素级别构建或侵蚀用户信任（Gebbia）。

同一个计划，三种思维方式看到完全不同的问题。CEO 看到"这不够大胆"，Eng 看到"这个错误路径没有处理"，Design 看到"这个空状态会让用户困惑"。这就是三件套的价值——它不是重复审查，而是从三个正交维度确保计划的完整性。

---

## 实际的工作流建议

**小变更（少于 50 行）：** 只跑 Eng Review，甚至可能被 `/ship` 的审查门槛自动跳过。选项 C（变更太小不需要审查）存在就是为了这个场景。

**中型功能：** Eng Review 必跑。如果有 UI 变更，加 Design Review。CEO Review 可选但对产品方向的清晰度有帮助。

**大型产品方向变更：** 全套链路。`/office-hours` 产出设计文档 → CEO Review 用 EXPANSION 模式探索 10x 愿景 → Eng Review 锁定架构和测试计划 → Design Review 确保 UI 规格完整。这看起来很重，但在 AI 辅助下每个审查大约 15-30 分钟，总共大约 1-2 小时——相比一个走错方向的两周 Sprint，这是极好的投资回报率。

---

*下一篇：[从 `/qa` 到 `/ship`：自动化发布链路](11-qa-to-ship.md)*

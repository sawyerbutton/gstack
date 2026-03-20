# 05. Skill 即 Prompt 的架构哲学

> 大多数"AI 工作流编排"系统都在发明新的 DSL、新的 DAG 引擎、新的运行时。gstack 的回答是：Markdown 够了。一个 `.md` 文件就是整个工作流的定义、文档和运行时。这个选择看起来简陋，却蕴含着对 AI Agent 本质的深刻理解。

---

## 一、为什么 Markdown 足以编码完整的工程工作流

传统工作流系统的思路是：定义状态机、声明依赖关系、编排并发执行。它们假设执行者是一台不理解上下文的机器。但 AI Agent 不同——它能读懂自然语言，能根据上下文做判断，能在模糊指令中找到确定性。

gstack 的洞察是：**AI Agent 的行为完全由它读到的 Prompt 决定。SKILL.md 文件不是"文档"——它就是 Prompt 本身。**

看一个具体例子。`ship/SKILL.md.tmpl` 中定义了完整的发布流程：

```markdown
# Ship: Fully Automated Ship Workflow

You are running the `/ship` workflow. This is a **non-interactive,
fully automated** workflow. Do NOT ask for confirmation at any step.

**Only stop for:**
- On the base branch (abort)
- Merge conflicts that can't be auto-resolved
- Test failures
- Pre-landing review finds ASK items that need user judgment

**Never stop for:**
- Uncommitted changes (always include them)
- Version bump choice (auto-pick MICRO or PATCH)
- CHANGELOG content (auto-generate from diff)
```

这不是伪代码，不是 YAML 配置，不是 Python 脚本。它是用英语写的工作流定义——但 Claude 读到它之后，会精确地按照这些规则执行。"Only stop for" 和 "Never stop for" 就是条件分支。"Do NOT ask for confirmation" 就是控制流。

这种方式带来了几个关键优势：

1. **零运行时依赖**：不需要工作流引擎、不需要容器、不需要编排层。Claude 本身就是执行引擎。
2. **人类可审计**：任何人都能读懂一个 Skill 在做什么。不需要学习 DSL。
3. **迭代速度极快**：修改工作流 = 修改文本。不需要重新编译、不需要重新部署。
4. **天然支持模糊逻辑**："如果变更显然是微小的（少于 20 行、拼写修正、仅配置变更），选择 C"——这种判断用传统代码写起来极其复杂，但自然语言表达毫不费力。

---

## 二、模板系统：从 .tmpl 到 SKILL.md 的编译流水线

gstack 的 Skill 文件并不是手写的 Markdown，而是通过模板系统生成的。这套系统的核心是 `scripts/gen-skill-docs.ts`，它实现了一个从 `.tmpl` 模板到 `.md` 最终文件的编译流水线：

```
.tmpl 模板文件
     │
     ▼
gen-skill-docs.ts（模板编译器）
     │  ┌─ commands.ts（命令注册表）
     │  ├─ snapshot.ts（快照标志元数据）
     │  └─ 内置占位符解析器
     ▼
SKILL.md（最终生成的 Prompt 文件）
```

模板文件使用 `{{PLACEHOLDER}}` 语法标记需要动态生成的内容。当前系统支持的所有占位符定义在一个统一的解析器映射中：

```typescript
const RESOLVERS: Record<string, (ctx: TemplateContext) => string> = {
  COMMAND_REFERENCE: generateCommandReference,
  SNAPSHOT_FLAGS:    generateSnapshotFlags,
  PREAMBLE:          generatePreamble,
  BROWSE_SETUP:      generateBrowseSetup,
  BASE_BRANCH_DETECT: generateBaseBranchDetect,
  QA_METHODOLOGY:    generateQAMethodology,
  DESIGN_METHODOLOGY: generateDesignMethodology,
  DESIGN_REVIEW_LITE: generateDesignReviewLite,
  REVIEW_DASHBOARD:  generateReviewDashboard,
  TEST_BOOTSTRAP:    generateTestBootstrap,
};
```

每个占位符背后是一个纯函数，接收 `TemplateContext`（包含 Skill 名称、模板路径、目标 host 类型），返回一段 Markdown 文本。

### 占位符的设计哲学

以 `{{PREAMBLE}}` 为例，它是所有 Skill 共享的"序言"，由多个子模块组合而成：

```typescript
function generatePreamble(ctx: TemplateContext): string {
  return [
    generatePreambleBash(ctx),     // 运行时检查脚本
    generateUpgradeCheck(ctx),     // 版本更新检测
    generateLakeIntro(),           // 完整性原则介绍
    generateTelemetryPrompt(ctx),  // 遥测设置
    generateAskUserFormat(ctx),    // AskUserQuestion 格式规范
    generateCompletenessSection(), // 完整性原则详述
    generateContributorMode(),     // 贡献者模式
    generateCompletionStatus(),    // 完成状态协议
  ].join('\n\n');
}
```

这意味着修改 Preamble 的任何一部分——比如调整 AskUserQuestion 的格式要求——会自动传播到所有 22 个 Skill。这就是模板系统的核心价值：**一处修改，全局生效**。

### 从数据源到文档的编译

`{{COMMAND_REFERENCE}}` 的生成过程展示了"源码即文档"的理念。它直接从 `commands.ts` 中读取命令注册表：

```typescript
function generateCommandReference(_ctx: TemplateContext): string {
  const groups = new Map<string, Array<{...}>>();
  for (const [cmd, meta] of Object.entries(COMMAND_DESCRIPTIONS)) {
    const list = groups.get(meta.category) || [];
    list.push({ command: cmd, description: meta.description, usage: meta.usage });
    groups.set(meta.category, list);
  }
  // ... 生成 Markdown 表格
}
```

添加一个新的浏览器命令只需要在 `commands.ts` 中加一行，运行 `bun run gen:skill-docs`，命令参考表就会自动更新。SNAPSHOT_FLAGS 同理——快照标志的元数据数组同时驱动 CLI 参数解析、文档生成和测试验证。**真正的单一数据源（Single Source of Truth）**。

---

## 三、为什么生成的文件要提交到 Git

一个自然的问题是：既然 SKILL.md 可以从模板生成，为什么不在运行时动态生成？为什么要把生成的文件提交到 Git？

gstack 选择"生成时编译 + 提交到 Git"有三个关键原因：

**1. Claude 在加载时读取，不是在运行时生成**

Claude Code 的 Skill 系统在会话开始时读取 SKILL.md 文件。它不会去运行 `gen-skill-docs.ts`。如果生成文件不存在，Skill 就无法工作。提交到 Git 确保了任何人 `git clone` 之后就能直接使用。

**2. CI 可以验证新鲜度**

`gen-skill-docs.ts` 支持 `--dry-run` 模式：生成到内存，然后与已提交的文件对比。如果不一致，说明有人修改了模板但忘记重新生成，测试会失败：

```bash
bun run gen:skill-docs --dry-run  # 如果 .md 与 .tmpl 不同步，exit 1
```

**3. Git blame 可追溯**

每次生成文件的变更都有 commit 记录。当一个 Skill 的行为发生变化时，`git blame SKILL.md` 可以精确定位到是哪次提交、哪个人修改了模板。如果是运行时生成，这种追溯能力就消失了。

---

## 四、Preamble 系统：每个 Skill 的共同大脑

Preamble 是注入到每个 Skill 开头的共享模块，它实现了跨 Skill 的一致行为。让我们逐一分析其核心功能：

### 更新检查

```bash
_UPD=$(~/.claude/skills/gstack/bin/gstack-update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD" || true
```

Preamble 的第一步就是检查 gstack 是否有新版本。如果检测到更新，Claude 会读取 `gstack-upgrade/SKILL.md` 并引导用户升级。

### 会话追踪

```bash
mkdir -p ~/.gstack/sessions
touch ~/.gstack/sessions/"$PPID"
_SESSIONS=$(find ~/.gstack/sessions -mmin -120 -type f 2>/dev/null | wc -l | tr -d ' ')
find ~/.gstack/sessions -mmin +120 -type f -delete 2>/dev/null || true
```

每次 Skill 运行时，都会在 `~/.gstack/sessions/` 下创建一个以 PID 命名的文件。通过计算 2 小时内的活跃会话文件数量，gstack 知道用户正在运行多少个并发会话。这个信息被 AskUserQuestion 格式规范使用——当会话数较多时，说明用户在多线程工作，提问需要更加自包含。

### 贡献者模式

```bash
_CONTRIB=$(~/.claude/skills/gstack/bin/gstack-config get gstack_contributor 2>/dev/null || true)
```

当 `_CONTRIB` 为 `true` 时，Claude 会在每个主要工作流步骤结束后反思工具体验。如果发现可改进之处（比如 `$B js "await fetch(...)"` 曾因缺少 async 包装而报错），它会自动提交 field report 到 `~/.gstack/contributor-logs/`。这是一个自我改进的反馈循环。

### AskUserQuestion 格式规范

Preamble 定义了严格的提问格式：

```
1. Re-ground: 说明项目名、当前分支、当前任务
2. Simplify: 用 16 岁聪明人能懂的语言解释问题
3. Recommend: 给出推荐选项及理由，包含完整性评分
4. Options: 字母编号的选项列表
```

其中有一句关键指令："假设用户已经 20 分钟没看这个窗口了，也没有打开代码。" 这确保了每次提问都包含足够的上下文，即使用户已经切换到其他任务。

---

## 五、SKILL.md.tmpl 写作规则

CLAUDE.md 中定义了一套严格的模板写作规则，这些规则源于对 Claude 执行方式的深刻理解：

### 规则一：用自然语言表达逻辑和状态

```markdown
<!-- 错误 -->
```bash
BASE_BRANCH=$(gh pr view --json baseRefName -q .baseRefName 2>/dev/null)
if [ -z "$BASE_BRANCH" ]; then
  BASE_BRANCH=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)
fi
```

<!-- 正确 -->
1. Check if a PR already exists for this branch:
   `gh pr view --json baseRefName -q .baseRefName`
2. If no PR exists, detect the repo's default branch:
   `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`
3. If both commands fail, fall back to `main`.
```

每个 bash 代码块运行在独立的 shell 中——变量不会在块之间持久化。因此，不能用 shell 变量传递状态，而应该用自然语言告诉 Claude 需要记住什么。

### 规则二：bash 块必须自包含

每个代码块应该能独立执行。如果需要前面步骤的上下文，应该在代码块上方的散文中重申。

### 规则三：用英语表达条件

```markdown
<!-- 不要这样写 -->
if [ "$REVIEW_STATUS" = "CLEAR" ]; then
  # ...
elif [ "$REVIEW_STATUS" = "NOT_CLEARED" ]; then
  # ...
fi

<!-- 要这样写 -->
If the Eng Review is NOT "CLEAR":
1. Check for a prior override on this branch
2. If no override exists, use AskUserQuestion
3. If the user chooses A or C, persist the decision
```

### 规则四：不要硬编码分支名

使用 `{{BASE_BRANCH_DETECT}}` 占位符动态检测基础分支。在散文中引用为"the base branch"，在代码块中使用 `<base>` 占位符。

---

## 六、平台无关设计

gstack 的 Skill 模板遵循一个严格的原则：**永远不要假设项目使用什么框架、什么语言、什么目录结构。**

实现这个原则的三步机制：

```
1. 读取 CLAUDE.md → 查找项目特定配置
2. 如果缺失 → AskUserQuestion 询问用户
3. 持久化答案到 CLAUDE.md → 下次不再询问
```

`{{TEST_BOOTSTRAP}}` 占位符是这个机制的最佳示例。它不会假设项目用 Jest 还是 Vitest，用 RSpec 还是 Minitest，而是：

1. 检查 CLAUDE.md 中是否已有测试命令配置
2. 如果没有，扫描 `package.json`、`Gemfile` 等文件自动检测
3. 如果检测失败，询问用户
4. 将结果写回 CLAUDE.md

这意味着 gstack 在 Rails 项目、Next.js 项目、Python 项目中都能工作——它让项目拥有自己的配置，gstack 只是读取它。

---

## 七、多 Host 支持：同一模板，不同输出

模板编译器支持 `--host` 参数，可以为不同的 AI Agent 平台生成适配的 SKILL.md：

```typescript
const HOST_PATHS: Record<Host, HostPaths> = {
  claude: {
    skillRoot: '~/.claude/skills/gstack',
    localSkillRoot: '.claude/skills/gstack',
    binDir: '~/.claude/skills/gstack/bin',
    browseDir: '~/.claude/skills/gstack/browse/dist',
  },
  codex: {
    skillRoot: '~/.codex/skills/gstack',
    localSkillRoot: '.agents/skills/gstack',
    binDir: '~/.codex/skills/gstack/bin',
    browseDir: '~/.codex/skills/gstack/browse/dist',
  },
};
```

对于 Codex host，编译器还会执行额外的转换：剥离 Claude 特有的 frontmatter 字段（`allowed-tools`、`hooks`），替换路径引用，并注入安全性建议。同一套模板，为不同平台生成语义等价的 Prompt。

---

## 八、核心洞察

让我们回到最初的问题：为什么 Markdown 够了？

答案是：**因为 AI Agent 的行为完全由它读到的文本决定。**

传统软件的行为由编译后的二进制代码决定。你需要编程语言、编译器、运行时。但 AI Agent 不同——它的"代码"就是自然语言。SKILL.md 文件不是"描述工作流的文档"，它**就是工作流本身**。

gstack 的模板系统做了一件看起来平凡却意义深远的事情：它把"编写 AI Agent 行为"这件事，变成了"编辑 Markdown 文件"。这意味着：

- 调试 Agent 行为 = 读一个文本文件
- 修改 Agent 行为 = 编辑一个文本文件
- 复用 Agent 行为 = 复制一个文本文件
- 审查 Agent 行为 = `git diff` 一个文本文件

不需要新的编程范式。不需要可视化编排工具。不需要 Agent 框架。Markdown 就是编程语言，Claude 就是编译器和运行时。

这就是"Skill 即 Prompt"的架构哲学：**不要在 AI 和用户之间增加抽象层——直接用 AI 能理解的语言描述你想要的行为。**

---

*下一篇：[Sprint 流水线与信息流](06-sprint-pipeline.md)*

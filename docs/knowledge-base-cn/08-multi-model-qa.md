# 08. 多模型协作与质量保证

> 一个 AI 模型的盲点恰好是另一个模型的强项。gstack 不把赌注押在单一模型上——它让 Claude 和 Codex 独立审查同一段代码，然后对比发现。同时，它构建了一套三层测试体系来确保 gstack 自身的质量：免费的静态验证、中等成本的 E2E 测试、昂贵但精准的 LLM 判定。

---

## 一、/codex：三种模式的多模型协作

`/codex` Skill 是 gstack 与 OpenAI Codex CLI 的集成层。它不是简单地"调用另一个 AI"——它定义了三种精确的协作模式，每种模式有不同的推理强度和使用场景。

### 模式一：Review（代码审查，带通过/失败门控）

```bash
codex review --base <base> -c 'model_reasoning_effort="high"' --enable web_search_cached
```

Review 模式让 Codex 独立审查当前分支的 diff。关键词是"独立"——它不知道 Claude 的 `/review` 发现了什么，它从零开始分析。

审查结果有一个明确的门控机制：

```
如果输出包含 [P1]（优先级 1 问题）→ 门控 FAIL
如果只有 [P2] 或无发现 → 门控 PASS
```

推理强度设为 `high`——足够深入但不至于太慢。Diff 审查需要深度但不需要穷举。

### 模式二：Challenge（对抗模式，最高推理强度）

```bash
codex exec "<adversarial-prompt>" -s read-only -c 'model_reasoning_effort="xhigh"' --enable web_search_cached --json
```

Challenge 模式是对抗性的。提示词明确告诉 Codex 扮演攻击者和混沌工程师：

```
Your job is to find ways this code will fail in production.
Think like an attacker and a chaos engineer. Find edge cases,
race conditions, security holes, resource leaks, failure modes,
and silent data corruption paths. Be adversarial. Be thorough.
No compliments — just the problems.
```

推理强度设为 `xhigh`——最大计算力。当试图破坏代码时，你希望模型思考得尽可能深入。

用户可以指定焦点领域：`/codex challenge security` 会让 Codex 专注于注入向量、权限提升、数据暴露等安全问题。

### 模式三：Consult（咨询，带会话连续性）

```bash
# 新会话
codex exec "<prompt>" -s read-only -c 'model_reasoning_effort="high"' --enable web_search_cached --json
# 继续会话
codex exec resume <session-id> "<prompt>" -s read-only ...
```

Consult 模式支持多轮对话。会话 ID 保存在 `.context/codex-session-id` 文件中：

```bash
mkdir -p .context
# 保存 JSONL 事件流中 thread.started 事件的 thread_id
```

下次运行 `/codex` 时，它会检测到已有会话并询问：

```
You have an active Codex conversation from earlier.
A) Continue the conversation (Codex remembers the prior context)
B) Start a new conversation
```

推理强度设为 `high`——对话场景需要平衡深度和速度。

### 推理强度的精确调节

三种模式使用不同的推理强度不是随意选择。这是对"计算资源 vs 任务需求"的精确校准：

| 模式 | 推理强度 | 原因 |
|------|---------|------|
| Review | `high` | 彻底但不慢。Diff 审查从深度中受益，但不需要最大计算量 |
| Challenge | `xhigh` | 最大推理能力。试图破坏代码时，你希望模型思考得尽可能深入 |
| Consult | `high` | 深度与速度的平衡，适合对话场景 |

---

## 二、交叉模型分析

当 `/review`（Claude）和 `/codex review`（OpenAI）都审查了同一个分支时，gstack 会自动生成交叉模型分析：

```
CROSS-MODEL ANALYSIS:
  Both found: [两个模型都发现的问题]
  Only Codex found: [仅 Codex 发现的问题]
  Only Claude found: [仅 Claude /review 发现的问题]
  Agreement rate: X% (N/M total unique findings overlap)
```

这种分析的价值在于：**不同的 AI 模型有不同的盲点。**

Claude 可能更擅长理解复杂的业务逻辑和上下文关系，而 Codex 可能更擅长发现底层的并发问题和边界条件。当两个模型都发现同一个问题时，你可以高度确信这是真实问题。当只有一个模型发现时，值得仔细审视——它可能是真正的洞察，也可能是误报。

`/codex` 的 SKILL.md 模板明确指出：

```markdown
After presenting, note any points where Codex's analysis differs from
your own understanding. If there is a disagreement, flag it:
"Note: Claude Code disagrees on X because Y."
```

这不是简单的"展示两个输出"——Claude 会主动分析分歧点并解释原因。

---

## 三、为什么不同 AI 模型能发现不同的问题

这个现象的根本原因在于训练数据和对齐方式的差异。不同的模型在不同的代码模式上有不同的"直觉"：

- **训练数据分布不同**：Claude 和 GPT 系列模型的训练语料有重叠但不完全相同，这导致它们对不同类型的代码模式有不同的敏感度
- **推理偏好不同**：即使面对相同输入，不同模型可能从不同角度分析代码——一个可能先关注数据流，另一个可能先关注控制流
- **对齐目标不同**：不同厂商的对齐训练强调不同方面，这影响模型对"什么是问题"的判断标准

gstack 利用这种多样性，而不是试图消除它。两个独立审查 > 一个审查做两遍。

---

## 四、三层测试体系

gstack 自身的质量保证建立在一套三层测试体系上，每层有不同的成本、速度和覆盖范围：

```
Tier 1: 静态验证          免费, <2s     每次提交前必跑
  │     skill-validation   SKILL.md 结构 + 命令一致性
  │     gen-skill-docs     模板生成质量检查
  │     browse tests       快照 + 集成测试
  │
  ▼
Tier 2: E2E 端到端        ~$3.85/run   发布前必跑
  │     claude -p          真实 Claude 会话执行 Skill
  │     codex E2E          真实 Codex CLI 执行 Skill
  │
  ▼
Tier 3: LLM 判定          ~$0.15/run   发布前必跑
        LLM-as-judge       用 Claude 评估 SKILL.md 质量
```

### Tier 1：免费的静态验证

`bun test` 运行所有 Tier 1 测试。这些测试不调用任何 LLM API，不启动浏览器，不花钱。它们验证的是结构性正确——SKILL.md 是否与源码一致、命令参考表是否完整、快照标志是否与代码匹配。

典型的验证逻辑：从 `commands.ts` 读取所有命令 → 从 `SKILL.md` 解析命令参考表 → 对比两者是否一致。如果有人在 `commands.ts` 中添加了新命令但忘记运行 `gen-skill-docs`，这个测试就会失败。

### Tier 2：真实的 E2E 测试

E2E 测试通过 `claude -p`（Claude Code 的 programmatic 模式）实际执行 Skill：

```typescript
export async function runSkillTest(options: {
  prompt: string;
  workingDirectory: string;
  maxTurns?: number;
  allowedTools?: string[];
  timeout?: number;
}): Promise<SkillTestResult> {
  const args = [
    '-p',
    '--output-format', 'stream-json',
    '--verbose',
    '--dangerously-skip-permissions',
    '--max-turns', String(maxTurns),
    '--allowed-tools', ...allowedTools,
  ];

  const proc = Bun.spawn(
    ['sh', '-c', `cat "${promptFile}" | claude ${args.map(a => `"${a}"`).join(' ')}`],
    { cwd: workingDirectory, stdout: 'pipe', stderr: 'pipe' }
  );
  // ...
}
```

session-runner 把 prompt 写入临时文件，通过 stdin 管道传给 `claude -p`。输出是 NDJSON 格式的事件流——每个工具调用、每次 AI 回复都是一个 JSON 事件。

实时进度追踪通过解析 NDJSON 流实现：

```typescript
// 实时输出到 stderr
const progressLine = `  [${elapsed}s] turn ${liveTurnCount} tool #${liveToolCount}: ` +
                     `${item.name}(${truncate(JSON.stringify(item.input || {}), 80)})\n`;
process.stderr.write(progressLine);
```

同时写入心跳文件 `~/.gstack-dev/e2e-live.json`，外部监控可以实时查看当前正在运行的测试。

每个 E2E 测试有超时竞赛机制——如果 Claude 在指定时间内没有完成，进程被杀死：

```typescript
const timeoutId = setTimeout(() => {
  timedOut = true;
  proc.kill();
}, timeout);
```

### Tier 3：LLM 作为评判者

LLM-judge 测试用 Claude 来评估 SKILL.md 的质量——命令参考是否完整、工作流描述是否清晰、与源码是否一致。这不是简单的字符串比较，而是语义级别的评估。

---

## 五、Diff-based 测试选择

运行全部 E2E 测试大约需要 $3.85。每次提交都跑全部测试太贵了。gstack 的解决方案是基于 diff 的测试选择：

```typescript
/**
 * E2E test touchfiles — keyed by testName.
 * Each test lists the file patterns that, if changed, require the test to run.
 */
export const E2E_TOUCHFILES: Record<string, string[]> = {
  'browse-basic':    ['browse/src/**'],
  'browse-snapshot': ['browse/src/**'],
  'qa-quick':        ['qa/**', 'browse/src/**'],
  'qa-b6-static':    ['qa/**', 'browse/src/**', 'browse/test/fixtures/qa-eval.html',
                       'test/fixtures/qa-eval-ground-truth.json'],
  'review-sql-injection': ['review/**', 'test/fixtures/review-eval-vuln.rb'],
  'ship-base-branch':     ['ship/**'],
  // ...
};
```

每个测试声明自己依赖的文件模式。测试选择算法：

```
1. git diff 获取变更文件列表
2. 如果任何变更文件匹配全局 touchfile → 运行所有测试
3. 否则，对每个测试检查其 touchfile 模式是否匹配变更文件
4. 只运行匹配的测试
```

全局 touchfile 包含测试基础设施本身：

```typescript
export const GLOBAL_TOUCHFILES = [
  'test/helpers/session-runner.ts',
  'test/helpers/codex-session-runner.ts',
  'test/helpers/eval-store.ts',
  'test/helpers/llm-judge.ts',
  'scripts/gen-skill-docs.ts',
  'test/helpers/touchfiles.ts',
  'browse/test/test-server.ts',
];
```

如果修改了 `session-runner.ts`（E2E 测试的运行器），所有 E2E 测试都会被触发——因为运行器的变更可能影响任何测试的执行方式。

用户可以通过 `bun run eval:select` 预览哪些测试会被选中：

```bash
bun run eval:select
# 输出: 基于当前 diff，以下测试将运行：
#   browse-basic (browse/src/server.ts changed)
#   browse-snapshot (browse/src/snapshot.ts changed)
# 以下测试将跳过：
#   qa-quick (no matching changes)
#   review-sql-injection (no matching changes)
```

---

## 六、Eval 持久化与跨运行对比

每次测试运行的结果都被持久化到 `~/.gstack-dev/evals/` 目录：

```typescript
// 文件命名格式
const filename = `${version}-${safeBranch}-${this.tier}-${dateStr}.json`;
// 例如: 0.9.0-main-e2e-20260316-152300.json
```

每个 eval 文件包含完整的测试数据：

```typescript
export interface EvalResult {
  schema_version: number;
  version: string;
  branch: string;
  git_sha: string;
  timestamp: string;
  hostname: string;
  tier: 'e2e' | 'llm-judge';
  total_tests: number;
  passed: number;
  failed: number;
  total_cost_usd: number;
  total_duration_ms: number;
  tests: EvalTestEntry[];
}
```

### 增量保存

`EvalCollector` 在每个测试完成后执行增量保存（`savePartial()`）。这意味着即使测试运行中途崩溃，已完成的测试结果不会丢失：

```typescript
addTest(entry: EvalTestEntry): void {
  this.tests.push(entry);
  this.savePartial();  // 原子写入：先写 .tmp 再 rename
}
```

### 自动对比

`finalize()` 方法在所有测试完成后自动寻找前一次运行进行对比：

```typescript
const prevFile = findPreviousRun(this.evalDir, this.tier, git.branch, filepath);
if (prevFile) {
  const prevResult = JSON.parse(fs.readFileSync(prevFile, 'utf-8'));
  const comparison = compareEvalResults(prevResult, result, prevFile, filepath);
  process.stderr.write(formatComparison(comparison) + '\n');
}
```

对比优先同分支，回退到任何分支：

```typescript
// Sort by timestamp descending
entries.sort((a, b) => b.timestamp.localeCompare(a.timestamp));
// Prefer same branch
const sameBranch = entries.find(e => e.branch === branch);
if (sameBranch) return sameBranch.file;
// Fallback: any branch
return entries[0].file;
```

对比输出不仅显示通过/失败变化，还追踪效率变化——turn 数、时长、成本、检测率：

```
vs previous: main/eval (2026-03-15 14:23)
──────────────────────────────────────────────────────────────────
  qa-b6-static                PASS  → PASS   =$0.45→$0.38  5→4t  32→28s
  review-sql-injection        PASS  → PASS   =$0.22→$0.25  3→3t  18→20s
  ship-base-branch            FAIL  → PASS   ↑$0.31→$0.28  4→3t  25→22s
──────────────────────────────────────────────────────────────────
  Status: 1 improved, 0 regressed, 2 unchanged
  Cost:   -$0.07
  Duration: -5s

  Takeaway:
    Fixed: "ship-base-branch" now passes.
    "qa-b6-static": 1 fewer turns (20% more efficient), 4s faster.
```

`generateCommentary()` 函数自动解读数字变化的含义：

```typescript
if (turnsDelta < 0) {
  insights.push(`${Math.abs(turnsDelta)} fewer turns (${Math.abs(turnsPct)}% more efficient)`);
} else {
  insights.push(`${turnsDelta} more turns (${turnsPct}% less efficient)`);
}
```

手动对比和汇总也有专用命令：

```bash
bun run eval:compare   # 对比最近两次运行
bun run eval:summary   # 跨所有运行的聚合统计
bun run eval:list      # 列出所有历史运行
```

---

## 七、多 Agent 支持

gstack v0.9.0 开始支持在多个 AI Agent 平台上运行：Claude Code、Codex CLI、Gemini CLI、Cursor。

安装时通过 `--host` 参数指定目标平台：

```bash
./setup --host auto    # 自动检测已安装的 Agent
./setup --host claude  # 仅 Claude Code
./setup --host codex   # 仅 Codex CLI
```

`auto` 模式检测逻辑：

```bash
INSTALL_CLAUDE=0
INSTALL_CODEX=0
if [ "$HOST" = "auto" ]; then
  command -v claude >/dev/null 2>&1 && INSTALL_CLAUDE=1
  command -v codex >/dev/null 2>&1 && INSTALL_CODEX=1
  # 如果都没找到，默认 Claude
  if [ "$INSTALL_CLAUDE" -eq 0 ] && [ "$INSTALL_CODEX" -eq 0 ]; then
    INSTALL_CLAUDE=1
  fi
fi
```

不同平台的 Skill 安装方式不同：

- **Claude Code**：SKILL.md 放入 `~/.claude/skills/gstack/`，子 Skill 通过符号链接注册到 `~/.claude/skills/`
- **Codex CLI**：SKILL.md 放入 `~/.codex/skills/gstack/`，同时生成 `.agents/skills/gstack-*` 格式的 Codex 专用 Skill

模板编译器（`gen-skill-docs.ts`）为不同 host 生成适配的 SKILL.md——替换路径引用、剥离 Claude 特有的 frontmatter 字段（如 `allowed-tools`、`hooks`）。

Codex E2E 测试使用独立的 session runner（`codex-session-runner.ts`），直接调用 Codex CLI 而非 Claude CLI：

```typescript
export const E2E_TOUCHFILES = {
  // Codex E2E (tests skills via Codex CLI)
  'codex-discover-skill':  ['codex/**', '.agents/skills/**',
                             'test/helpers/codex-session-runner.ts'],
  'codex-review-findings': ['review/**', '.agents/skills/gstack-review/**',
                             'codex/**', 'test/helpers/codex-session-runner.ts'],
};
```

---

## 八、质量保证的核心原则

gstack 的质量保证体系遵循一个核心原则：**免费抓住 95% 的问题，只在需要判断力的地方使用 LLM。**

```
Tier 1 (免费, <2s)
  ├─ SKILL.md 是否与源码同步？          → 字符串比较
  ├─ 命令参考表是否完整？                → 集合对比
  ├─ 快照标志是否与代码匹配？            → 元数据验证
  └─ 模板生成是否无错误？                → 编译检查

Tier 2 (~$3.85, 带 diff 选择)
  ├─ Claude 能否成功执行 Skill？         → 进程退出码
  ├─ 浏览器引擎是否报错？                → 错误模式匹配
  └─ Skill 输出是否符合预期？            → 结构化检查

Tier 3 (~$0.15)
  └─ SKILL.md 的质量是否足够高？          → LLM 语义评估
```

绝大多数问题——命令缺失、路径错误、模板未同步——可以通过免费的字符串比较和集合运算检测。只有"SKILL.md 写得好不好"这种主观判断才需要动用 LLM。

E2E 测试也不需要每次都全跑。Diff-based 选择确保只有受影响的测试才会执行。修改了 `review/SKILL.md.tmpl`？只跑 review 相关的测试。修改了 `session-runner.ts`？全部重跑——因为测试基础设施的变更可能影响任何测试。

这种分层策略让 gstack 在保持高质量的同时，把持续集成的成本控制在可接受的范围内：

```bash
bun test             # 每次提交前运行 — 免费, <2s
bun run test:evals   # 发布前运行 — 最多 ~$4/run
```

免费测试抓住结构性问题。付费测试验证端到端行为。LLM 判定确保语义质量。三层防线，每层有明确的成本和覆盖范围。

---

## 九、E2E 失败的归因协议

当 E2E 测试失败时，一个常见的陷阱是"这不是我们的变更导致的"。gstack 的 CLAUDE.md 对此有严格要求：

```
Required before attributing a failure to "pre-existing":
1. Run the same eval on main and show it fails there too
2. If it passes on main but fails on the branch — it IS your change
3. If you can't run on main, say "unverified — may or may not be related"

"Pre-existing" without receipts is a lazy claim. Prove it or don't say it.
```

这个协议确保了每次失败都被认真对待。在 AI Agent 的系统中，变更之间有"隐形耦合"——修改 preamble 文本会影响 Agent 行为，修改一个 helper 会改变时序，重新生成 SKILL.md 会偏移 prompt 上下文。不能仅凭直觉判断"这不相关"，必须用证据证明。

---

*下一篇：[`/office-hours` 完全拆解](09-office-hours-deep-dive.md)*

# 07. 浏览器引擎：给 AI 一双眼睛

> AI Agent 可以读代码、写代码、运行测试——但它看不到页面。它不知道按钮是不是歪了，不知道表单提交后发生了什么，不知道用户看到的是什么。gstack 的浏览器引擎解决了这个问题：一个为 AI Agent 设计的、持久化的、毫秒级响应的无头 Chromium。

---

## 一、"I SEE THE ISSUE"——为什么 AI Agent 需要浏览器

考虑一个典型场景：你让 AI Agent 修复一个前端 bug。它读了代码，理解了逻辑，做了修改。但它怎么知道修复有效？

传统方式是运行测试。但很多 UI 问题——布局错位、交互卡顿、视觉回归——没有对应的测试，也很难用测试覆盖。人类开发者会打开浏览器看一眼。AI Agent 需要同样的能力。

gstack 的浏览器引擎让 AI Agent 能做到：

```bash
$B goto http://localhost:3000/login    # 导航到页面
$B snapshot -i                          # 看到所有交互元素
$B fill @e3 "test@example.com"          # 填写表单
$B click @e5                            # 点击提交
$B snapshot -D                          # 看到页面变化（unified diff）
$B console                              # 检查 JS 错误
$B screenshot /tmp/after-login.png      # 截图作为证据
```

每条命令约 100ms。从"打开页面"到"得到结论"，整个过程不到 2 秒。

---

## 二、守护进程模型：3-5 秒 vs 100 毫秒

最直观的浏览器方案是：每次命令都启动一个 Chromium 实例，执行完关闭。这种方式的问题是 Chromium 的冷启动需要 3-5 秒。如果一次 QA 测试需要 50 个浏览器命令，仅启动开销就要 150-250 秒。

gstack 的方案是守护进程模型（daemon model）：

```
首次调用（~3s）          后续调用（~100ms）
┌──────────┐            ┌──────────┐
│ CLI      │            │ CLI      │
│ 启动服务器│            │ 读取状态  │
│ 等待就绪  │            │ HTTP POST│
│ 发送命令  │            │ 收到响应  │
└──────────┘            └──────────┘
     │                       │
     ▼                       ▼
┌─────────────────────────────────────┐
│  Server (Bun.serve)                 │
│  ├─ BrowserManager                  │
│  │  └─ Chromium (持久化运行)         │
│  ├─ ConsoleBuffer (环形缓冲)        │
│  ├─ NetworkBuffer (环形缓冲)        │
│  └─ DialogBuffer  (环形缓冲)        │
└─────────────────────────────────────┘
```

第一次调用 `$B` 时，CLI 检测到没有运行中的服务器，在后台启动一个。之后的每次调用都只是一个 HTTP POST——发送命令，接收结果。Chromium 实例在后台持续运行，cookies、登录状态、标签页全部保持。

30 分钟无活动后自动关闭：

```typescript
const idleCheckInterval = setInterval(() => {
  if (Date.now() - lastActivity > IDLE_TIMEOUT_MS) {
    console.log(`[browse] Idle for ${IDLE_TIMEOUT_MS / 1000}s, shutting down`);
    shutdown();
  }
}, 60_000);
```

---

## 三、完整架构：CLI → HTTP → Server → CDP → Chromium

gstack 浏览器引擎的架构是一个清晰的分层模型：

```
┌──────────────────────────────────────────────────────────┐
│  Claude Code / AI Agent                                  │
│  执行: $B goto https://example.com                       │
└─────────────────────┬────────────────────────────────────┘
                      │ Bash 进程
                      ▼
┌──────────────────────────────────────────────────────────┐
│  CLI (browse/src/cli.ts → 编译为 browse/dist/browse)      │
│  1. 读取 .gstack/browse.json（端口 + token）              │
│  2. 如果服务器不存在 → 后台启动                            │
│  3. HTTP POST localhost:{port}/command                    │
│     Authorization: Bearer {token}                        │
│  4. 输出响应到 stdout / stderr                            │
└─────────────────────┬────────────────────────────────────┘
                      │ HTTP (localhost only)
                      ▼
┌──────────────────────────────────────────────────────────┐
│  Server (browse/src/server.ts — Bun.serve)               │
│  - 验证 Bearer token                                     │
│  - 路由命令到 read/write/meta 处理器                      │
│  - 管理环形缓冲区（console/network/dialog）               │
│  - 每秒异步刷新日志到磁盘                                 │
└─────────────────────┬────────────────────────────────────┘
                      │ Playwright API
                      ▼
┌──────────────────────────────────────────────────────────┐
│  BrowserManager (browse/src/browser-manager.ts)          │
│  - Chromium 生命周期管理                                   │
│  - 标签页管理（Map<id, Page>）                             │
│  - Ref Map（@e1 → Playwright Locator）                    │
│  - Dialog 自动处理                                        │
│  - Handoff / Resume 状态管理                               │
└─────────────────────┬────────────────────────────────────┘
                      │ Chrome DevTools Protocol
                      ▼
┌──────────────────────────────────────────────────────────┐
│  Chromium (headless)                                     │
└──────────────────────────────────────────────────────────┘
```

命令被分为三类，每类由独立模块处理：

```typescript
export const READ_COMMANDS = new Set([
  'text', 'html', 'links', 'forms', 'accessibility',
  'js', 'eval', 'css', 'attrs', 'console', 'network',
  'cookies', 'storage', 'perf', 'dialog', 'is',
]);

export const WRITE_COMMANDS = new Set([
  'goto', 'back', 'forward', 'reload',
  'click', 'fill', 'select', 'hover', 'type', 'press',
  'scroll', 'wait', 'viewport', 'cookie', ...
]);

export const META_COMMANDS = new Set([
  'tabs', 'tab', 'newtab', 'closetab',
  'status', 'stop', 'restart',
  'screenshot', 'pdf', 'responsive', 'chain', 'diff',
  'url', 'snapshot', 'handoff', 'resume',
]);
```

这个分类不仅是代码组织——它决定了权限模型。READ 命令不修改页面状态，WRITE 命令会，META 命令操作浏览器自身。

---

## 四、状态文件：多工作区隔离

CLI 与服务器之间通过状态文件通信：

```typescript
interface ServerState {
  pid: number;           // 服务器进程 ID
  port: number;          // HTTP 监听端口
  token: string;         // Bearer token
  startedAt: string;     // 启动时间
  serverPath: string;    // 服务器脚本路径
  binaryVersion?: string; // 二进制版本哈希
}
```

状态文件位于项目根目录的 `.gstack/browse.json` 中（不是 `/tmp`）。这意味着**每个项目有自己独立的浏览器实例**。你可以同时在两个项目中运行 `/qa`，它们的 Chromium 实例互不干扰。

端口在 10000-60000 范围内随机选取，进一步确保隔离：

```typescript
async function findPort(): Promise<number> {
  const MIN_PORT = 10000;
  const MAX_PORT = 60000;
  const MAX_RETRIES = 5;
  for (let attempt = 0; attempt < MAX_RETRIES; attempt++) {
    const port = MIN_PORT + Math.floor(Math.random() * (MAX_PORT - MIN_PORT));
    try {
      const testServer = Bun.serve({ port, fetch: () => new Response('ok') });
      testServer.stop();
      return port;
    } catch { continue; }
  }
  throw new Error(`No available port after ${MAX_RETRIES} attempts`);
}
```

---

## 五、版本自动重启

当 gstack 更新时，正在运行的浏览器服务器可能使用的是旧版本代码。CLI 通过版本哈希检测这种不一致：

```
构建时: git rev-parse HEAD → browse/dist/.version

CLI 启动时:
  1. 读取 .gstack/browse.json 获取 binaryVersion
  2. 读取 browse/dist/.version 获取当前版本
  3. 如果不匹配 → 杀死旧服务器 → 启动新服务器
```

这确保了用户永远在使用最新版本的浏览器引擎，无需手动重启。

---

## 六、Ref 系统：@e1, @e2 与无 DOM 突变的元素选择

Ref 系统是 gstack 浏览器引擎最精巧的设计之一。问题是：AI Agent 怎么"指向"页面上的一个元素？

CSS 选择器（如 `#login-btn`）需要事先知道元素的 id 或 class。对于 AI Agent 来说，它只能看到"页面上有一个登录按钮"，不知道它的 CSS 选择器是什么。

gstack 的方案是 **Ref 系统**：

```
1. page.locator(scope).ariaSnapshot() → YAML 式无障碍树
2. 解析树，为每个元素分配 @e1, @e2, ... 引用
3. 为每个引用构建 Playwright Locator（getByRole + nth）
4. 存储 Map<string, Locator> 在 BrowserManager 上
5. 返回紧凑文本输出，每行一个元素带 @ref
```

运行 `$B snapshot -i` 的输出类似：

```
@e1 [heading] "Welcome" [level=1]
@e2 [textbox] "Email"
@e3 [textbox] "Password"
@e4 [button] "Log in"
```

之后 AI Agent 可以直接说 `$B click @e4` 或 `$B fill @e2 "test@example.com"`。

### 为什么不修改 DOM

很多浏览器自动化方案会向 DOM 注入自定义属性（如 `data-test-id`），然后通过这些属性选择元素。gstack 明确拒绝了这种方式。原因在 `snapshot.ts` 的注释中：

```typescript
/**
 * Architecture (Locator map — no DOM mutation):
 *   1. page.locator(scope).ariaSnapshot() → YAML-like accessibility tree
 *   2. Parse tree, assign refs @e1, @e2, ...
 *   3. Build Playwright Locator for each ref (getByRole + nth)
 *   4. Store Map<string, Locator> on BrowserManager
 */
```

不修改 DOM 意味着：浏览器引擎观察到的页面与用户看到的完全一致。不会因为注入代码而改变页面行为、触发副作用、或者被 CSP 策略阻止。

### @c 引用：cursor-interactive 元素

ARIA 无障碍树不包含所有可交互元素。很多 Web 应用使用 `<div>` 加 `cursor:pointer` 或 `onclick` 来创建可点击元素。`-C` 标志扫描这些非 ARIA 元素并分配 `@c1, @c2, ...` 引用：

```bash
$B snapshot -C    # 查找 cursor:pointer, onclick, tabindex 元素
$B click @c1      # 点击 cursor-interactive 引用
```

### Ref 失效检测

导航后 ref 会失效——因为页面已经变了。CLI 在导航命令后会提醒："Refs are invalidated on navigation — run `snapshot` again after `goto`."

---

## 七、安全模型

浏览器引擎处理敏感数据——用户的 cookies、登录凭证、会话状态。安全模型有多层防护：

**仅 localhost 监听**：服务器只绑定 `127.0.0.1`，不可从网络访问。

**Bearer Token 认证**：每次启动时生成随机 UUID 作为 token：

```typescript
const AUTH_TOKEN = crypto.randomUUID();

function validateAuth(req: Request): boolean {
  const header = req.headers.get('authorization');
  return header === `Bearer ${AUTH_TOKEN}`;
}
```

只有知道 token 的进程（通过读取 `.gstack/browse.json`）才能发送命令。

**Cookie 安全**：`cookie-import-browser` 功能可以从用户的真实浏览器（Chrome、Arc、Brave、Edge）导入 cookies。解密过程遵循严格的安全原则：

```typescript
/**
 * Decryption pipeline (Chromium macOS "v10" format):
 *   Ciphertext = encrypted_value[3:]
 *   Plaintext = AES-128-CBC-decrypt(key, iv, ciphertext)
 */
```

- Cookie 数据库以只读模式打开
- 解密密钥在内存中缓存，不写入磁盘
- 每次会话独立派生密钥

---

## 八、日志架构：三个环形缓冲区

浏览器的控制台输出、网络请求、对话框事件需要被记录。但 AI Agent 的会话可能持续很长时间，日志量可能很大。gstack 使用环形缓冲区（CircularBuffer）解决这个问题：

```typescript
/**
 * CircularBuffer<T>: O(1) insert ring buffer with fixed capacity.
 *
 *   ┌───┬───┬───┬───┬───┬───┐
 *   │ 3 │ 4 │ 5 │   │ 1 │ 2 │  capacity=6, head=4, size=5
 *   └───┴───┴───┴───┴─▲─┴───┘
 *                      │
 *                    head (oldest entry)
 *
 *   push() writes at (head+size) % capacity, O(1)
 *   toArray() returns entries in insertion order, O(n)
 *   totalAdded keeps incrementing past capacity (flush cursor)
 */
```

三个独立的缓冲区分别用于：

1. **Console Buffer**：捕获 `console.log`、`console.error`、`console.warn`
2. **Network Buffer**：记录所有 HTTP 请求和响应
3. **Dialog Buffer**：记录 alert、confirm、prompt 对话框

每秒异步刷新到磁盘（`.gstack/browse-{console,network,dialog}.log`），但不阻塞主线程：

```typescript
const flushInterval = setInterval(flushBuffers, 1000);
```

`totalAdded` 计数器持续递增，作为"刷新游标"——确保即使缓冲区已满，新条目仍然能被刷新到磁盘。

---

## 九、错误哲学：错误是给 AI Agent 看的

gstack 浏览器引擎的错误信息不是为人类设计的——它们为 AI Agent 设计：

```typescript
function wrapError(err: any): string {
  const msg = err.message || String(err);
  if (err.name === 'TimeoutError' || msg.includes('Timeout')) {
    if (msg.includes('locator.click') || msg.includes('locator.fill')) {
      return `Element not found or not interactable within timeout. ` +
             `Check your selector or run 'snapshot' for fresh refs.`;
    }
    if (msg.includes('page.goto') || msg.includes('Navigation')) {
      return `Page navigation timed out. The URL may be unreachable ` +
             `or the page may be loading slowly.`;
    }
  }
  if (msg.includes('resolved to') && msg.includes('elements')) {
    return `Selector matched multiple elements. ` +
           `Be more specific or use @refs from 'snapshot'.`;
  }
  return msg;
}
```

每条错误信息都包含**下一步操作建议**。不是"TimeoutError: waiting for locator"，而是"Element not found — run 'snapshot' for fresh refs"。AI Agent 读到错误后知道该做什么。

---

## 十、崩溃恢复

Chromium 可能崩溃。gstack 不尝试自我修复——它选择快速失败：

```typescript
// browser-manager.ts
this.browser.on('disconnected', () => {
  console.error('[browse] FATAL: Chromium process crashed or was killed.');
  console.error('[browse] Console/network logs flushed to .gstack/browse-*.log');
  process.exit(1);
});
```

服务器进程退出后，下一次 CLI 调用会检测到服务器不存在（PID 不再存活），自动启动新实例。整个恢复过程对 AI Agent 透明。

设计哲学是：**不要隐藏故障**。与其尝试在损坏的状态上继续运行，不如干净地重启。

---

## 十一、Handoff 机制：把浏览器交给人类

有些问题 AI Agent 无法解决——CAPTCHA、2FA、复杂的登录流程。Handoff 机制让 AI 把浏览器"交给"用户：

```typescript
async handoff(message: string): Promise<string> {
  // 1. 保存当前状态（cookies, localStorage, URLs）
  const state = await this.saveState();

  // 2. 启动新的有头浏览器（用户可见）
  const newBrowser = await chromium.launch({ headless: false });

  // 3. 将状态恢复到新浏览器
  // （cookies、localStorage、所有标签页和 URL）

  // 4. 关闭旧的无头浏览器
  await oldBrowser.close();

  return `HANDOFF: Browser opened at ${currentUrl}\n` +
         `MESSAGE: ${message}\n` +
         `STATUS: Waiting for user. Run 'resume' when done.`;
}
```

用户在可见的 Chrome 窗口中完成需要人工操作的步骤，然后 AI 运行 `$B resume` 夺回控制权。Resume 会重新运行 snapshot，获取人类操作后的页面状态。

---

## 十二、为什么选择 Bun

gstack 选择 Bun 而非 Node.js 有几个关键原因：

1. **编译二进制**：`bun build --compile` 生成独立可执行文件，不需要安装 Node.js 或 Bun 运行时
2. **内置 SQLite**：cookie 导入功能需要读取 Chromium 的 SQLite 数据库，Bun 的原生 SQLite 支持避免了额外依赖
3. **原生 TypeScript**：无需 tsc 编译步骤，直接运行 .ts 文件
4. **内置 HTTP Server**：`Bun.serve` 是零依赖的 HTTP 服务器，不需要 Express 或 Fastify
5. **启动速度**：Bun 的冷启动时间远快于 Node.js，这对于"每个命令都是独立 CLI 调用"的模型至关重要

最终产出是一个 `browse/dist/browse` 二进制文件。它包含完整的 CLI + 服务器启动逻辑。用户运行 `$B goto https://example.com`，这个二进制文件负责一切——找到或启动服务器，发送命令，返回结果。

---

## 十三、总结：浏览器引擎的设计原则

回顾 gstack 浏览器引擎的全部设计，可以提炼出几条核心原则：

1. **持久化优于冷启动**：守护进程模型将延迟从秒级降到毫秒级
2. **文件系统优于内存**：状态文件确保了多进程、多工作区的协调
3. **快速失败优于自我修复**：Chromium 崩溃时退出进程，不尝试恢复
4. **不修改被观察者**：不注入 DOM，不修改页面，保持观察的纯净性
5. **错误信息面向 AI**：每条错误都包含可操作的下一步建议
6. **安全性分层**：localhost + bearer token + 只读 DB + 内存密钥缓存

这些原则的共同指向是：**浏览器引擎不是一个通用工具，它是专门为 AI Agent 设计的感知器官**。它的每个设计决策都围绕着一个问题：怎样让 AI Agent 更快、更准、更安全地"看到"页面？

---

*下一篇：[多模型协作与质量保证](08-multi-model-qa.md)*

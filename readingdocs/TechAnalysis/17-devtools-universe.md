# 17 - UniverseManager：DevTools 引擎的双栈复用

> 关键源码：`src/DevtoolsUtils.ts`、`src/DevToolsConnectionAdapter.ts`、`src/McpContext.ts`

## 1. 问题：Puppeteer 和 DevTools 是两套世界

Puppeteer 和 Chrome DevTools 前端都是浏览器自动化的"上层框架"，但它们有不同的模型：

| Puppeteer | DevTools 前端 |
| --- | --- |
| Page / Frame / ElementHandle | Target / Model / Context |
| 事件用 EventEmitter | 事件用 Observer + 明确定义的 Types |
| 简单 CDPSession 抽象 | 完整的 TargetManager + 多层 Model |
| 适合自动化 | 适合深度调试（Source Maps、Stack Traces、Debugger） |

有些能力只有 DevTools 前端能做：

- **源码映射级别的堆栈解析**：`Debugger.paused` 事件 + DebuggerWorkspaceBinding；
- **Issue Aggregator**：CORS / Mixed Content / COEP 等复杂问题的聚合；
- **Trace Engine + Insights**：性能分析；
- **HeapSnapshot Proxy**：内存分析。

项目需要"借 Puppeteer 驱动浏览器，借 DevTools 深度分析"。**UniverseManager** 就是那座桥。

## 2. 核心架构

```mermaid
flowchart TB
    subgraph Puppeteer
      P1["puppeteer.Browser"] --> P2["puppeteer.Page"]
      P2 --> P3["createCDPSession()"]
    end

    subgraph DevTools 子系统
      D1["Universe 一个独立沙箱"]
      D2["TargetManager 管理 Target"]
      D3["Target（frame）"]
      D4["DebuggerModel / RuntimeModel / NetworkModel"]
    end

    P3 -->|包装| A["PuppeteerDevToolsConnection 适配器"]
    A --> D3
    D3 --> D2
    D2 --> D1
    D1 -->|context.get(DevTools.TargetManager)| D2
    D1 -->|context.get(DebuggerWorkspaceBinding)| BINDING["源码映射绑定"]

    UM["UniverseManager"] -->|Page → Universe| D1
    UM -->|listen targetcreated/destroyed| P1
```

每一个 Puppeteer Page，都会有一个**对应的** DevTools Universe（它自己的"DevTools 实例"）。两者通过 `PuppeteerDevToolsConnection` 适配器用同一个 CDPSession 通信，**不开两次协议会话**。

## 3. UniverseManager 的核心职责

```55:127:src/DevtoolsUtils.ts
export class UniverseManager {
  readonly #browser: Browser;
  readonly #createUniverseFor: TargetUniverseFactoryFn;
  readonly #universes = new WeakMap<Page, TargetUniverse>();

  /** Guard access to #universes so we don't create unnecessary universes */
  readonly #mutex = new Mutex();

  async init(pages: Page[]) {
    try {
      await this.#mutex.acquire();
      const promises = [];
      for (const page of pages) {
        promises.push(this.#createUniverseFor(page).then(targetUniverse => this.#universes.set(page, targetUniverse)));
      }
      this.#browser.on('targetcreated', this.#onTargetCreated);
      this.#browser.on('targetdestroyed', this.#onTargetDestroyed);
      await Promise.all(promises);
    } finally {
      this.#mutex.release();
    }
  }

  get(page: Page): TargetUniverse | null {
    return this.#universes.get(page) ?? null;
  }
  ...
}
```

**关键设计**：

- `WeakMap<Page, TargetUniverse>`：Page 被 GC 时 Universe 自动释放；
- `Mutex`：防止同一 Page 被并发创建两个 Universe（比如 init 和 targetcreated 同时发生）；
- **懒创建**：只在需要时（需要深度分析）才创建 Universe，避免资源浪费；
- **自动跟随**：`targetcreated` / `targetdestroyed` 事件让 Universe 生命周期自动跟随 Page。

## 4. DEFAULT_FACTORY：一个 Universe 怎么建

```129:158:src/DevtoolsUtils.ts
const DEFAULT_FACTORY: TargetUniverseFactoryFn = async (page: Page) => {
  const settingStorage = new DevTools.Common.Settings.SettingsStorage({});
  const universe = new DevTools.Foundation.Universe.Universe({
    settingsCreationOptions: {
      syncedStorage: settingStorage,
      globalStorage: settingStorage,
      localStorage: settingStorage,
      settingRegistrations: DevTools.Common.SettingRegistration.getRegisteredSettings(),
    },
    overrideAutoStartModels: new Set([DevTools.DebuggerModel]),
  });

  const session = await page.createCDPSession();
  const connection = new PuppeteerDevToolsConnection(session);

  const targetManager = universe.context.get(DevTools.TargetManager);
  targetManager.observeModels(DevTools.DebuggerModel, SKIP_ALL_PAUSES);

  const target = targetManager.createTarget(
    'main',
    '',
    'frame' as any,
    /* parentTarget */ null,
    session.id(),
    undefined,
    connection,
  );
  return {target, universe};
};
```

**逐段解释**：

### 4.1 SettingsStorage 内存化

DevTools 默认从 localStorage 读设置，MCP 没有 localStorage，就**给它一个内存 storage**。MCP 实例不持久化 DevTools 设置，每次启动都是全新状态。

### 4.2 overrideAutoStartModels

Universe 默认"按需启动 Model"，但 DebuggerModel 要立刻启用才能捕获 stack trace。`overrideAutoStartModels` 强制拉起。

### 4.3 PuppeteerDevToolsConnection

关键的"适配层"（详见第 5 节），把 Puppeteer 的 CDPSession 假装成 DevTools 的 CDPConnection。

### 4.4 SKIP_ALL_PAUSES

```160:173:src/DevtoolsUtils.ts
// We don't want to pause any DevTools universe session ever on the MCP side.
//
// Note that calling `setSkipAllPauses` only affects the session on which it was
// sent. This means DevTools can still pause, step and do whatever. We just won't
// see the `Debugger.paused`/`Debugger.resumed` events on the MCP side.
const SKIP_ALL_PAUSES = {
  modelAdded(model: DevTools.DebuggerModel): void {
    void model.agent.invoke_setSkipAllPauses({skip: true});
  },
  modelRemoved(): void { /* Do nothing. */ },
};
```

**非常重要的细节**：启用 DebuggerModel 后默认会在断点处 pause 住页面。MCP 要的是 **被动监听**（读 stack trace、读 scope），不能让页面卡住。`setSkipAllPauses` 让 CDP 层跳过所有 pause，但保留事件分发。

注释也很教科书：**设置只对本 session 生效，用户如果开了真正的 DevTools 面板，他们的 debugger 还能正常用。**

## 5. PuppeteerDevToolsConnection：适配器模式

```19:113:src/DevToolsConnectionAdapter.ts
export class PuppeteerDevToolsConnection implements DevTools.CDPConnection.CDPConnection {
  readonly #connection: puppeteer.Connection;
  readonly #observers = new Set<DevTools.CDPConnection.CDPConnectionObserver>();
  readonly #sessionEventHandlers = new Map<string, puppeteer.Handler<unknown>>();

  constructor(session: puppeteer.CDPSession) {
    this.#connection = session.connection()!;

    session.on(CDPSessionEvent.SessionAttached, this.#startForwardingCdpEvents.bind(this));
    session.on(CDPSessionEvent.SessionDetached, this.#stopForwardingCdpEvents.bind(this));
    this.#startForwardingCdpEvents(session);
  }

  send<T extends DevTools.CDPConnection.Command>(method, params, sessionId): Promise<...> {
    if (sessionId === undefined) throw new Error('Attempting to send on the root session. This must not happen');
    const session = this.#connection.session(sessionId);
    if (!session) throw new Error('Unknown session ' + sessionId);
    return session.send(method as any, params)
      .then(result => ({result}))
      .catch(error => ({error})) as any;
  }

  observe(observer): void { this.#observers.add(observer); }
  unobserve(observer): void { this.#observers.delete(observer); }

  #startForwardingCdpEvents(session: puppeteer.CDPSession): void {
    const handler = this.#handleEvent.bind(this, session.id()) as puppeteer.Handler<unknown>;
    this.#sessionEventHandlers.set(session.id(), handler);
    session.on('*', handler);  // 订阅所有 CDP 事件
  }

  #stopForwardingCdpEvents(session: puppeteer.CDPSession): void {
    const handler = this.#sessionEventHandlers.get(session.id());
    if (handler) session.off('*', handler);
  }

  #handleEvent(sessionId, type, event): void {
    if (typeof type === 'string' && type !== CDPSessionEvent.SessionAttached && type !== CDPSessionEvent.SessionDetached) {
      this.#observers.forEach(observer =>
        observer.onEvent({method: type as DevTools.CDPConnection.Event, sessionId, params: event}),
      );
    }
  }
}
```

**适配器设计**：

- **接口实现**：`implements DevTools.CDPConnection.CDPConnection`，DevTools 把它当成原生连接；
- **协议转换**：Puppeteer 的 `session.send(method, params)` ↔ DevTools 期望的 `{result} | {error}` 联合体；
- **事件桥接**：`session.on('*', handler)` 订阅所有 CDP 事件，转发给 DevTools observers；
- **嵌套 session 自动管理**：`SessionAttached`/`SessionDetached` 让子 frame 的会话也自动加入桥接；
- **Rolled protocol 容忍**：`method as any` —— Puppeteer 和 DevTools 的协议类型版本可能不完全同步，用 any 跳过编译检查（注释明确声明）。

**注释里最精妙的一句**：

> We don't have to recursively listen for 'sessionattached' as the "root" CDP session sees all child session attached events, regardless how deeply nested they are.

意即：**只在根 session 监听 SessionAttached 就够了**，不需要递归注册子 session——Puppeteer 会把所有层级的 session attach 事件冒泡到根。

## 6. SymbolizedError：实战价值

UniverseManager 的价值在 `SymbolizedError` 里体现得最充分：

```253:290:src/DevtoolsUtils.ts
static async fromDetails(opts): Promise<SymbolizedError> {
  const message = SymbolizedError.#getMessage(opts.details);
  if (!opts.includeStackAndCause || !opts.devTools) {
    return new SymbolizedError(message, opts.resolvedStackTraceForTesting, opts.resolvedCauseForTesting);
  }

  let stackTrace: DevTools.StackTrace.StackTrace.StackTrace | undefined;
  if (opts.details.stackTrace) {
    try {
      stackTrace = await createStackTrace(opts.devTools, opts.details.stackTrace, opts.targetId);
    } catch { /* ignore */ }
  }

  // TODO: Turn opts.details.exception into a JSHandle and retrieve the 'cause' property.
  //       If its an Error, recursively create a SymbolizedError.
  let cause: SymbolizedError | undefined;
  if (opts.details.exception) {
    try {
      const causeRemoteObj = await SymbolizedError.#lookupCause(...);
      if (causeRemoteObj) cause = await SymbolizedError.fromError({...});
    } catch { /* Ignore */ }
  }
  return new SymbolizedError(message, stackTrace, cause);
}
```

用 DevTools Universe 做到：

1. **源码映射**：把 minified 代码的 `at Object.xyz (app.js:1:123)` 翻译成 `at handleSubmit (src/Login.tsx:45:10)`；
2. **Error cause chain**：JavaScript 的 `new Error("foo", {cause: err})` 可以连锁；
3. **异步栈还原**：跨 Promise 边界的完整 stack。

**没有 UniverseManager，这些能力就得自己实现 sourcemap 解析——不现实**。

## 7. Script 等待：源码映射加载完成

```398:448:src/DevtoolsUtils.ts
const signal = AbortSignal.timeout(1_000);
await Promise.all(
  [...scriptIds].map(id =>
    waitForScript(model, id, signal)
      .then(script => model.sourceMapManager().sourceMapForClientPromise(script))
      .catch(),
  ),
);

...

async function waitForScript(model, scriptId, signal) {
  while (true) {
    if (signal.aborted) throw signal.reason;
    const script = model.scriptForId(scriptId);
    if (script) return script;

    await new Promise((resolve, reject) => {
      signal.addEventListener('abort', () => reject(signal.reason), {once: true});
      void model.once('ParsedScriptSource' as any).then(resolve);
    });
  }
}
```

**Stack trace 正确性的细节**：

- DevTools 本身会"先给 stack，sourcemap 到了再更新"——但 MCP 不会触发更新事件；
- 所以 MCP 在构造 stack trace **之前**，主动等到所有 sourcemap 加载完成；
- 1 秒超时兜底，避免没 sourcemap 的脚本挂死。

## 8. 为什么这个设计值得学

**适配两个强大框架的典范**：

| 维度 | 做法 |
| --- | --- |
| 协议互通 | 适配器模式 + `implements` 接口 |
| 事件桥接 | `on('*', handler)` 订阅全部事件 |
| 生命周期同步 | `targetcreated/destroyed` + WeakMap |
| 状态隔离 | 每 Page 一个 Universe，SettingsStorage 内存化 |
| 能力开关 | `overrideAutoStartModels` + `SKIP_ALL_PAUSES` 只启需要的 Model |
| 错误容忍 | 协议版本不一致用 `as any` 跳过编译 |
| 文档注释 | 对"设计决策"（比如 SessionAttached 的传递性）有充分解释 |

## 9. 可迁移经验

| 经验 | 说明 |
| --- | --- |
| 适配器模式让两个库互通 | `implements` + 协议翻译，无需修改任一方 |
| 订阅根 session 的全部事件 | `session.on('*', ...)` 是 Puppeteer 级别的隐藏武器 |
| 复用官方重型库 | 源码映射、trace 分析等复杂能力不自研 |
| 内存化 SettingsStorage | 快速初始化和彻底隔离 |
| 配合 Mutex 保护懒初始化 | 避免并发重复创建 |

## 10. 延伸阅读

- 源码映射 stack trace 用在哪 → [18-formatter-system.md](./18-formatter-system.md)（ConsoleFormatter）
- 性能 trace 如何使用 Universe → [16-performance-trace-crux.md](./16-performance-trace-crux.md)
- Mutex 的用法 → [03-mutex-serialization.md](./03-mutex-serialization.md)

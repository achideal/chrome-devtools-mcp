# 16 - 性能 Trace 采集与 CrUX 集成

> 关键源码：`src/tools/performance.ts`、`src/trace-processing/parse.ts`、`src/DevtoolsUtils.ts`

## 1. 问题空间

性能优化是 DevTools MCP 的核心卖点之一，但它比"点按钮"复杂 3 个数量级：

1. **数据量爆炸**：一次 5 秒的 trace，`traceEvents` 动辄 50MB、几十万条；
2. **数据低信号**：原始事件大部分是内部时序，Agent 无法直接理解；
3. **需要 Insight 引擎**：LCP 在哪？INP 是多少？Long tasks 集中在哪些函数？—— 这些都需要专用分析器；
4. **Lab data 不够**：还要结合真实用户的 CrUX 数据才完整。

项目选择的解决方案：**直接复用 DevTools 前端的 Trace Engine + Insights + Formatter**，而不是自己实现。

## 2. 总体流程

```mermaid
flowchart TB
    A["performance_start_trace"] -->|"about:blank 清状态"| B["page.tracing.start(categories)"]
    B --> C["reload 到目标 URL（若 reload=true）"]
    C --> D["autoStop：5 秒后停；否则 Agent 调 stop_trace"]
    D --> E["performance_stop_trace"]
    E --> F["page.tracing.stop() 拿 buffer"]
    F --> G["可选 gzip + saveFile"]
    F --> H["parseRawTraceBuffer"]
    H --> I["TraceEngine.parse -> parsedTrace + insights"]
    I --> J["populateCruxData 拉 CrUX 字段数据"]
    J --> K["context.storeTraceRecording"]
    K --> L["response.attachTraceSummary"]
    L --> M["PerformanceTraceFormatter 生成摘要"]

    N["performance_analyze_insight"] -->|"lastRecording"| O["getInsightOutput"]
    O --> P["response.attachTraceInsight"]
```

## 3. Trace 采集精选 categories

```73:90:src/tools/performance.ts
const categories = [
  '-*',                         // 禁用默认
  'blink.console',
  'blink.user_timing',
  'devtools.timeline',
  'disabled-by-default-devtools.screenshot',
  'disabled-by-default-devtools.timeline',
  'disabled-by-default-devtools.timeline.invalidationTracking',
  'disabled-by-default-devtools.timeline.frame',
  'disabled-by-default-devtools.timeline.stack',
  'disabled-by-default-v8.cpu_profiler',
  'disabled-by-default-v8.cpu_profiler.hires',
  'latencyInfo',
  'loading',
  'disabled-by-default-lighthouse',
  'v8.execute',
  'v8',
];
```

这个列表**与 DevTools TimelineController 和 Lighthouse 的 gatherer 保持同步**。注释明确指出源文件——保证当 DevTools 升级能记得同步更新。

**关键细节**：

- `-*` 先禁用所有，再白名单加回来；
- 包括 CPU profile、invalidation tracking、frames、screenshot，数据全面；
- `disabled-by-default-*` 是 Chrome 默认关闭的高开销 category，trace 期间才开。

## 4. Reload 前清状态

```63:69:src/tools/performance.ts
if (request.params.reload) {
  // Before starting the recording, navigate to about:blank to clear out any state.
  await page.pptrPage.goto('about:blank', {
    waitUntil: ['networkidle0'],
  });
}
```

为什么 **先 `about:blank`**？

- 如果直接 reload，记录会包含"老页面卸载"的污染数据；
- 跳到空白页后 reload 相当于冷启动，数据干净；
- `waitUntil: ['networkidle0']` 确保空白页完全加载（防止竞态）。

## 5. autoStop 机制

```101:113:src/tools/performance.ts
if (request.params.autoStop) {
  await new Promise(resolve => setTimeout(resolve, 5_000));
  await stopTracingAndAppendOutput(page.pptrPage, response, context, request.params.filePath);
} else {
  response.appendResponseLine(
    `The performance trace is being recorded. Use performance_stop_trace to stop it.`,
  );
}
```

5 秒是 **约定俗成的性能 trace 长度**——覆盖了 FCP/LCP/TBT/CLS 等指标的测量窗口。自动停省去 Agent 的一次工具调用。

如果是"长操作"场景（比如用户在页面上连点几次），可以设 `autoStop=false`，Agent 自己决定什么时候停。

## 6. 并发保护

```52:58:src/tools/performance.ts
if (context.isRunningPerformanceTrace()) {
  response.appendResponseLine(
    'Error: a performance trace is already running. Use performance_stop_trace to stop it. Only one trace can be running at any given time.',
  );
  return;
}
context.setIsRunningPerformanceTrace(true);
```

**浏览器限制**：同一时间只能有一个 trace session。项目在 Context 层面维护一个 bool 避免 Agent 连发两次 start。`finally` 里重置：

```222:224:src/tools/performance.ts
} finally {
  context.setIsRunningPerformanceTrace(false);
}
```

即便解析失败，flag 也会被清——**资源必须对称释放**。

## 7. Trace Engine：复用 DevTools

```10:10:src/trace-processing/parse.ts
const engine = DevTools.TraceEngine.TraceModel.Model.createWithAllHandlers();
```

`DevTools.TraceEngine` 就是 Chrome DevTools 前端的 trace 引擎（bundled 在 `src/third_party/lighthouse-devtools-mcp-bundle.js`）。它能：

- 解析所有 category 的事件；
- 生成 Thread、Network、Renderer 等领域模型；
- 产出 Insights（LCP breakdown、CLS 原因、INP heavy tasks 等）。

**项目完全没有自己实现 Trace 解析**——直接用 Chrome 官方版本。好处：

- 永远和 DevTools UI 保持一致；
- 未来 Chrome 加新 Insight 自动获得；
- 零维护成本。

## 8. 解析 + 错误处理

```27:71:src/trace-processing/parse.ts
export async function parseRawTraceBuffer(buffer): Promise<TraceResult | TraceParseError> {
  engine.resetProcessor();
  if (!buffer) return {error: 'No buffer was provided.'};

  const asString = new TextDecoder().decode(buffer);
  if (!asString) return {error: 'Decoding the trace buffer returned an empty string.'};

  try {
    const data = JSON.parse(asString) as {traceEvents: Event[]} | Event[];
    const events = Array.isArray(data) ? data : data.traceEvents;
    await engine.parse(events);
    const parsedTrace = engine.parsedTrace();
    if (!parsedTrace) return {error: 'No parsed trace was returned from the trace engine.'};
    const insights = parsedTrace?.insights ?? null;
    return {parsedTrace, insights};
  } catch (e) {
    const errorText = e instanceof Error ? e.message : JSON.stringify(e);
    logger(`Unexpected error parsing trace: ${errorText}`);
    return {error: errorText};
  }
}
```

**Result 类型**：返回 `TraceResult | TraceParseError`，调用方用 `traceResultIsSuccess()` 类型窄化。比 `throw` 更友好——trace 解析经常失败（老浏览器的格式变化），不能崩溃整个工具。

**`engine.resetProcessor()`**：每次解析前重置，避免多次 trace 之间状态串台。

**双格式兼容**：`{traceEvents: Event[]} | Event[]` ——支持新老两种 trace JSON 格式。

## 9. 摘要生成：PerformanceTraceFormatter

```79:88:src/trace-processing/parse.ts
export function getTraceSummary(result: TraceResult): string {
  const focus = DevTools.AgentFocus.fromParsedTrace(result.parsedTrace);
  const formatter = new DevTools.PerformanceTraceFormatter(focus);
  const summaryText = formatter.formatTraceSummary();
  return `## Summary of Performance trace findings:
${summaryText}

## Details on call tree & network request formats:
${extraFormatDescriptions}`;
}
```

- **AgentFocus**：DevTools 提供的"给 Agent 看"的 trace 视图封装（文件名本身暗示是为 AI 设计的）；
- **PerformanceTraceFormatter.formatTraceSummary()**：输出针对 Agent 优化的 markdown 摘要（LCP、CLS、主线程耗时 Top N 等）；
- **extraFormatDescriptions**：追加一段格式说明，Agent 看到"怎么读这些数字"也清晰。

## 10. CrUX 集成

```227:265:src/tools/performance.ts
async function populateCruxData(result: TraceResult): Promise<void> {
  const cruxManager = DevTools.CrUXManager.instance();
  cruxManager.setEndpointForTesting(
    'https://chromeuxreport.googleapis.com/v1/records:queryRecord?key=AIzaSyBn5gimNjhiEyA_euicSKko6IlD3HdgUfk',
  );
  const cruxSetting = DevTools.Common.Settings.Settings.instance()
    .createSetting('field-data', {enabled: true});
  cruxSetting.set({enabled: true});

  const urls = [...(result.parsedTrace.insights?.values() ?? [])].map(c => c.url.toString());
  urls.push(result.parsedTrace.data.Meta.mainFrameURL);
  const urlSet = new Set(urls);

  if (urlSet.size === 0) return;

  const cruxData = await Promise.all(
    Array.from(urlSet).map(async url => {
      const data = await cruxManager.getFieldDataForPage(url);
      return data;
    }),
  );

  result.parsedTrace.metadata.cruxFieldData = cruxData;
}
```

**CrUX = Chrome User Experience Report**，Google 收集的真实用户（Real User Monitoring）性能数据。结合 Lab data（trace）+ Field data（CrUX）才能客观评估页面：

- Lab：测试机、特定网络、1 次运行；
- Field：数千万真实用户的 28 天聚合。

**实现要点**：

- `setEndpointForTesting(..., key=AIzaSy...)`：公开 API key（注释"Yes, we're aware this API key is public"）。API key 只是 Google 用于计费/限流，不是安全凭据；
- **去重后并发查询**（`Set` + `Promise.all`）：节省请求；
- 结果挂到 `parsedTrace.metadata.cruxFieldData`，后续 Formatter 自动拾起。

## 11. 可关闭

```630:633:src/bin/chrome-devtools-mcp-cli-options.ts
performanceCrux: {
  type: 'boolean',
  default: true,
  describe: 'Set to false to disable sending URLs from performance traces to CrUX API...',
},
```

`--no-performance-crux` 让隐私敏感用户关掉——trace 里的 URL 不会发到 Google。启动期会打印一条 disclaimer 告知。

## 12. Insights 按需深入

```142:177:src/tools/performance.ts
export const analyzeInsight = definePageTool({
  name: 'performance_analyze_insight',
  annotations: {category: ToolCategory.PERFORMANCE, readOnlyHint: true},
  schema: {
    insightSetId: zod.string().describe(...),
    insightName: zod.string().describe('e.g., "DocumentLatency" or "LCPBreakdown"'),
  },
  handler: async (request, response, context) => {
    const lastRecording = context.recordedTraces().at(-1);
    if (!lastRecording) {
      response.appendResponseLine('No recorded traces found. Record a performance trace so you have Insights to analyze.');
      return;
    }
    response.attachTraceInsight(lastRecording, request.params.insightSetId, request.params.insightName);
  },
});
```

- Summary 里列了可用的 `(insightName, insightKey)` 对；
- Agent 看到想深入的 insight → 调 `performance_analyze_insight`；
- 只拿最近一次 trace（`recordedTraces().at(-1)`）；
- **零额外数据获取**——analyze 只是"从已保存的 trace 中提取指定 insight 的详细输出"。

## 13. 压缩落盘

```187:208:src/tools/performance.ts
if (filePath && traceEventsBuffer) {
  let dataToWrite: Uint8Array = traceEventsBuffer;
  if (filePath.endsWith('.gz')) {
    dataToWrite = await new Promise((resolve, reject) => {
      zlib.gzip(traceEventsBuffer, (error, result) => error ? reject(error) : resolve(result));
    });
  }
  const file = await context.saveFile(dataToWrite, filePath, filePath.endsWith('.gz') ? '.json.gz' : '.json');
  response.appendResponseLine(`The raw trace data was saved to ${file.filename}.`);
}
```

Agent 可以传 `filePath="trace.json.gz"`，**zlib.gzip** 原生压缩（通常 10x 压缩比），50MB 变 5MB。`saveFile` 自动加扩展名（`.json.gz`）。

## 14. 可迁移经验

| 经验 | 说明 |
| --- | --- |
| 复用专业库而非重造轮子 | Trace engine、Formatter 全借官方实现 |
| Summary + Detail 分离 | 先看 summary，按需 analyze insight |
| 并发状态布尔锁 | 防止同时跑多个 trace |
| 先 about:blank 再开测 | 清除历史噪声的标准做法 |
| Result 类型避免 throw | 解析失败用 `error` 字段而非异常 |
| 集成 Lab + Field data | 性能分析的黄金标准 |
| gzip 大 blob | 50MB → 5MB 轻松 |
| disclaimer 提示隐私行为 | 主动告知 URL 会发到 CrUX |

## 15. 延伸阅读

- 格式化如何输出 → [18-formatter-system.md](./18-formatter-system.md)
- 文件落盘与 token 节省 → [09-token-optimization.md](./09-token-optimization.md)
- DevTools 引擎复用 → [17-devtools-universe.md](./17-devtools-universe.md)

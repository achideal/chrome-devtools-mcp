# 09 - Token 优化策略：分页 / 摘要 / 文件引用

> 关键源码：`src/utils/pagination.ts`、`src/formatters/NetworkFormatter.ts`、`src/formatters/SnapshotFormatter.ts`、`src/tools/performance.ts`、`src/tools/screenshot.ts`

## 1. 为什么 Token 是第一性问题

对 LLM 服务商的定价：

- 输入 token：$1~15 / M；
- 输出 token：$3~75 / M。

一个 Chrome 页面：

- DOM HTML：100K ~ 2M token（完全喂不进去）
- Performance Trace：30MB JSON（一条跟踪动不动就超 context）
- 100 个网络请求：每个带 header/body，堆起来 50K token

**MCP 设计原则之一就是 Token-Optimized**（见 `docs/design-principles.md`）。项目实现了 4 条组合拳。

## 2. 第一招：分页（Pagination）

用 `src/utils/pagination.ts` 的通用函数管住列表：

```22:62:src/utils/pagination.ts
const DEFAULT_PAGE_SIZE = 20;

export function paginate<Item>(items, options?): PaginationResult<Item> {
  const total = items.length;
  if (!options || noPaginationOptions(options)) {
    return {items, currentPage: 0, totalPages: 1, ...};
  }
  const pageSize = options.pageSize ?? DEFAULT_PAGE_SIZE;
  const totalPages = Math.max(1, Math.ceil(total / pageSize));
  const {currentPage, invalidPage} = resolvePageIndex(options.pageIdx, totalPages);
  const startIndex = currentPage * pageSize;
  const pageItems = items.slice(startIndex, startIndex + pageSize);
  ...
}
```

在工具 schema 上统一暴露分页参数：

```42:71:src/tools/network.ts
pageSize: zod.number().int().positive().optional().describe('Maximum number of requests to return. When omitted, returns all requests.'),
pageIdx: zod.number().int().min(0).optional().describe('Page number to return (0-based). When omitted, returns the first page.'),
```

**关键设计**：

- 不传参 → 返回全部（backward compatible）；
- 传参 → 自动分页；
- 无效页码不报错，**降级到第 0 页并给出提示**："Invalid page number provided. Showing first page."；
- 响应里永远告诉 Agent 当前位置：`"Showing 1-20 of 73 (Page 1 of 4). Next page: 1"`，让 Agent 自主决定要不要翻页。

## 3. 第二招：语义摘要而非原始数据

### 3.1 网络请求

一条请求的"列表项"：

```259:264:src/formatters/NetworkFormatter.ts
function convertNetworkRequestConciseToString(data): string {
  return `reqid=${data.requestId} ${data.method} ${data.url} [${data.status}]${data.selectedInDevToolsUI ? ` [selected in the DevTools Network panel]` : ''}`;
}
```

只输出 `reqid + method + URL + status`。Agent 想看详情就调 `get_network_request(reqid)`，这时候才返回完整 headers/body。

### 3.2 Lighthouse

原始 JSON 300KB，`McpResponse.format` 只取**分类 score + 审计通过数 + 报告文件路径**：

```874:887:src/McpResponse.ts
for (const score of summary.scores) {
  response.push(`- ${score.title}: ${(score.score ?? 0) * 100} (${score.id})`);
}
response.push(`Passed: ${summary.audits.passed}`);
response.push(`Failed: ${summary.audits.failed}`);
```

### 3.3 Performance Trace

Trace 原始事件可能几百万条，**只在第一次总结时 attachTraceSummary**，Agent 想挖某个 Insight 再调 `performance_analyze_insight(insightName)`，这是典型的**懒加载 + 按需深入**。

### 3.4 Accessibility Snapshot

默认 `interestingOnly: true`，只保留交互元素（button、link、textbox 等）。`verbose: true` 才拉完整树（见 `TextSnapshot.create`）。

## 4. 第三招：大数据落盘 + 路径回传

设计原则 7：**Reference over Value**。

### 4.1 截图

```96:107:src/tools/screenshot.ts
if (request.params.filePath) {
  const result = await context.saveFile(screenshot, request.params.filePath, `.${format}`);
  response.appendResponseLine(`Saved screenshot to ${result.filename}.`);
} else if (screenshot.length >= 2_000_000) {  // 2MB 自动落盘
  const {filepath} = await context.saveTemporaryFile(screenshot, `screenshot.${request.params.format}`);
  response.appendResponseLine(`Saved screenshot to ${filepath}.`);
} else {
  response.attachImage({mimeType: `image/${format}`, data: Buffer.from(screenshot).toString('base64')});
}
```

三层分支：

- Agent 指定 filePath → 存到指定位置；
- 超过 2MB → **自动**落盘（避免 base64 爆炸）；
- 否则 → 内联 base64（让支持图像的客户端直接显示）。

### 4.2 网络请求 body

```87:110:src/formatters/NetworkFormatter.ts
if (this.#options.requestFilePath) {
  if (data) {
    const result = await this.#options.saveFile(Buffer.from(data), this.#options.requestFilePath, '.network-request');
    this.#requestBodyFilePath = result.filename;
  } else {
    this.#requestBody = requestBodyNotAvailableMessage;
  }
} else {
  if (data) {
    this.#requestBody = getSizeLimitedString(data, BODY_CONTEXT_SIZE_LIMIT);
  }
  ...
}
```

- Agent 提供 `requestFilePath/responseFilePath` → 保存到文件；
- 否则内联但**截断到 10KB**：`getSizeLimitedString(text, sizeLimit)`。

### 4.3 Trace

```186:209:src/tools/performance.ts
const traceEventsBuffer = await page.tracing.stop();
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

Trace 可以 **gzip 压缩**后落盘。`ensureExtension` 函数保证文件扩展名正确。

## 5. 第四招：截断与省略

```252:257:src/formatters/NetworkFormatter.ts
function getSizeLimitedString(text: string, sizeLimit: number) {
  if (text.length > sizeLimit) {
    return text.substring(0, sizeLimit) + '... <truncated>';
  }
  return text;
}
```

**10KB** 是一个魔数，但很合理——Agent 通常只需要看 body 的开头判断内容类型和错误信息。

其他截断：

- `BODY_CONTEXT_SIZE_LIMIT = 10000`：HTTP body；
- 二进制 body 直接显示 `<binary data>`；
- 空 body 显示 `<empty response>`；
- 不可用 body 显示 `<Request/Response body not available anymore>`。

**语义化的占位符**比截断后的乱码更友好。

## 6. 敏感信息脱敏

```164:176:src/formatters/NetworkFormatter.ts
#redactNetworkHeaders(headers: Record<string, string>) {
  const headersList = Object.entries(headers).map(item => ({name: item[0], value: item[1]}));
  const redacted = DevTools.NetworkRequestFormatter.sanitizeHeaders(headersList);
  return redacted.reduce<Record<string, string>>((acc, item) => {
    acc[item.name] = item.value;
    return acc;
  }, {});
}
```

`--redactNetworkHeaders` 开关启用后，Authorization / Cookie 等敏感 header 被脱敏。**既节省 token（脱敏值短）又降低泄露风险**。

## 7. 参数命名的隐式编码

```25:33:src/telemetry/ClearcutLogger.ts
export const PARAM_BLOCKLIST = new Set(['uid', 'reqid', 'msgid']);
```

工具暴露给 Agent 的参数用 **短名**：

| 长名 | 短名 |
| --- | --- |
| networkRequestId | reqid |
| elementUid | uid |
| consoleMessageId | msgid |

- 每次 Agent 调用都用短名，节省 token；
- 遥测侧把这些"高熵 ID"加入 blocklist，不收集具体值。

## 8. Trace URL 合集去重 + 串行请求

```244:265:src/tools/performance.ts
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
```

去重后并发查 CrUX，减少 URL 重复请求浪费。

## 9. Token 优化的层级全景

```mermaid
flowchart TB
    subgraph 原始数据
      HTML["页面 HTML 2MB"]
      TRACE["Performance trace 30MB"]
      NET["100 个网络请求 50K"]
      SS["截图 PNG 5MB"]
    end

    subgraph 第一层：结构化
      HTML --> AX["a11y 快照 2~20KB"]
      TRACE --> SUM["TraceSummary 关键指标 1KB"]
      NET --> LIST["reqid + URL + status 5KB"]
      SS --> IMG["base64 内联 < 2MB"]
    end

    subgraph 第二层：分页
      LIST -->|pageSize=20| P1["Page 1 / 4 约 1.5KB"]
    end

    subgraph 第三层：落盘
      TRACE --> FILE["trace.json.gz 5MB 只回传路径"]
      SS --> FILE2["/tmp/.../screenshot.png"]
      NET --> FILE3["body.network-response"]
    end

    subgraph 第四层：详情按需拉取
      P1 -->|"get_network_request"| DETAIL["完整 headers/body 10KB 截断"]
      SUM -->|"performance_analyze_insight"| INSIGHT["单个 Insight 2KB"]
    end
```

## 10. 可迁移经验

| 经验 | 应用场景 |
| --- | --- |
| 默认全量、带参分页 | 简单列表 API 的平滑升级路径 |
| 无效页自动降级 | 比报错更友好 |
| 2MB 自动落盘 + 1 条提示 | 任何大 blob 返回都可借鉴 |
| 10KB body 截断 + `<truncated>` 占位 | 日志/响应脱敏的模板 |
| 短参数名 | token 成本敏感的 API 设计 |
| 语义化占位符 | `<binary data>` / `<not available>` 比 `null` 更友好 |

## 11. 延伸阅读

- Response 层如何调度 → [08-mcp-response-builder.md](./08-mcp-response-builder.md)
- 分页实现细节 → `src/utils/pagination.ts`
- Trace 处理 → [16-performance-trace-crux.md](./16-performance-trace-crux.md)

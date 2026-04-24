# 02 - 工具注册系统与 `defineTool` / `definePageTool` 设计

> 关键源码：`src/tools/ToolDefinition.ts`、`src/tools/tools.ts`、`src/tools/*.ts`

## 1. 问题背景

项目暴露 **30+ 个工具**，它们有三类差异：

1. 有的作用于「浏览器全局」（`list_pages`、`install_extension`），有的作用于「某一页面」（`click`、`take_screenshot`）；
2. 有的需要根据 CLI 参数**动态调整 schema**（例如启用 `experimentalPageIdRouting` 时要加 `pageId` 字段）；
3. Schema 必须强类型且可以在运行期被遍历（遥测、文档生成都需要）。

如果每个工具自己去调 `server.registerTool(...)`，会出现：注册逻辑散落、类型推断丢失、CLI 开关到处写 `if`。

本项目用一套极小但极巧妙的**工厂 + 类型重载**搞定了这三件事。

## 2. 三层类型结构

```ts
BaseToolDefinition<Schema>    // name/description/annotations/schema
   ├── ToolDefinition<Schema>  // 加 handler(Request, Response, Context)
   └── PageToolDefinition<Schema> // handler 签名多一个 page: ContextPage
         └── DefinedPageTool    // 带运行期标记 pageScoped: true
```

- `ToolDefinition` 负责"全局作用域"的工具；
- `PageToolDefinition` 是 page-scoped 的子类型，编译后打上 `pageScoped: true` 这个运行期标记，注册器据此决定要不要给 handler 注入 page。

```287:305:src/tools/ToolDefinition.ts
interface PageToolDefinition<
  Schema extends zod.ZodRawShape = zod.ZodRawShape,
> extends BaseToolDefinition<Schema> {
  handler: (
    request: Request<Schema> & {page: ContextPage},
    response: Response,
    context: Context,
  ) => Promise<void>;
}

export type DefinedPageTool<Schema extends zod.ZodRawShape = zod.ZodRawShape> =
  PageToolDefinition<Schema> & {
    pageScoped: true;
    ...
  };
```

## 3. `defineTool` / `definePageTool` 的两形态

### 3.1 静态定义形态

直接传对象：

```45:84:src/tools/input.ts
export const click = definePageTool({
  name: 'click',
  description: `Clicks on the provided element`,
  annotations: {category: ToolCategory.INPUT, readOnlyHint: false},
  schema: {
    uid: zod.string().describe('...'),
    dblClick: dblClickSchema,
    includeSnapshot: includeSnapshotSchema,
  },
  handler: async (request, response) => { ... },
});
```

### 3.2 工厂函数形态

当 schema 依赖 CLI 参数时，包成一个 `(args?) => ToolDefinition`：

```17:66:src/tools/script.ts
export const evaluateScript = defineTool(cliArgs => {
  return {
    name: 'evaluate_script',
    description: `Evaluate a JavaScript function ...${cliArgs?.categoryExtensions ? ' or service worker' : ''}`,
    schema: {
      function: zod.string().describe(...),
      ...(cliArgs?.experimentalPageIdRouting ? pageIdSchema : {}),
      ...(cliArgs?.categoryExtensions ? {serviceWorkerId: zod.string().optional()} : {}),
    },
    handler: ...,
  };
});
```

两种形态共用同一个 API —— 通过 **函数重载 + typeof 运行期分支** 实现：

```259:285:src/tools/ToolDefinition.ts
export function defineTool<Schema extends zod.ZodRawShape>(
  definition: ToolDefinition<Schema>,
): ToolDefinition<Schema>;
export function defineTool<Schema, Args extends ParsedArguments = ParsedArguments>(
  definition: (args?: Args) => ToolDefinition<Schema>,
): (args?: Args) => ToolDefinition<Schema>;
export function defineTool(definition) {
  if (typeof definition === 'function') {
    const factory = definition;
    return (args: Args) => factory(args);
  }
  return definition;
}
```

**学习点**：`defineTool` 本身几乎不做事情，它的价值在于 **TypeScript 签名**——它是"类型见证人"，让外层的 `createTools` 能安全地 `Object.values(consoleTools)` 后做统一处理。

## 4. 工具聚合与裁剪

```27:62:src/tools/tools.ts
export const createTools = (args: ParsedArguments) => {
  const rawTools = args.slim
    ? Object.values(slimTools)
    : [
        ...Object.values(consoleTools),
        ...Object.values(emulationTools),
        ...Object.values(extensionTools),
        ...
      ];

  const tools = [];
  for (const tool of rawTools) {
    if (typeof tool === 'function') {
      tools.push(tool(args) as unknown as ToolDefinition);
    } else {
      tools.push(tool as ToolDefinition);
    }
  }

  tools.sort((a, b) => a.name.localeCompare(b.name));
  return tools;
};
```

值得注意的点：

- **扁平合并 + 动态构造**：每一类工具以模块导出对象被 `Object.values` 扁平化，函数形态会被 `tool(args)` 运行期展开；
- **按 name 排序**：确保 `tools/list` 输出稳定，对 Agent 的 prompt 缓存和日志对比非常重要；
- **Slim 分支单独走**：整套工具集完全替换，避免在通用逻辑里塞条件。

## 5. Schema 动态拼装的三种手法

项目大量使用**对象展开 + 条件对象**来拼 schema：

```ts
// 1) 简单条件展开
...(cliArgs?.experimentalPageIdRouting ? pageIdSchema : {})

// 2) 公共 schema 复用
export const pageIdSchema = { pageId: zod.number().optional().describe('...') };
export const timeoutSchema = { timeout: zod.number().int().optional().transform(v => v && v <= 0 ? undefined : v) };

// 3) Transform 做语义归一
transform(value => value && value <= 0 ? undefined : value)
```

这种写法比 `zod.object({...}).extend({...})` 更轻量，且每个字段的 `describe` 字符串**直接成为 Agent 的调用文档**，减少了一份 README 或 JSON Schema 注释。

## 6. 注册器如何识别 pageScoped

`registerTool` 里关键一段：

```208:233:src/index.ts
if ('pageScoped' in tool && tool.pageScoped) {
  const page = serverArgs.experimentalPageIdRouting && params.pageId && !serverArgs.slim
    ? context.getPageById(params.pageId)
    : context.getSelectedMcpPage();
  response.setPage(page);
  await tool.handler({params, page}, response, context);
} else {
  await tool.handler({params}, response, context);
}
```

- `definePageTool` 注入的运行期属性 `pageScoped: true` 成为注册器的"发牌条件"；
- 注册器自己解决"实验性路由（多页）"和"默认选中页"两种语义——**工具 handler 无须感知这一层复杂度**；
- 如果 `experimentalPageIdRouting` 启用，schema 会追加 `pageIdSchema`，Agent 可以明确指定 pageId。

## 7. 结构化设计的 5 条经验

| 经验 | 具体体现 |
| --- | --- |
| 类型优先 | `BaseToolDefinition<Schema extends zod.ZodRawShape>` 让 params 自动推断为 `zod.objectOutputType<Schema>` |
| 标记类而非继承树 | `pageScoped: true` 运行期标记比多态更适合 JSON 序列化友好的场景 |
| CLI 参数注入 | 工厂形态 `(args) => ToolDefinition` 把 schema 的动态性压缩到一处 |
| 描述即文档 | 每个字段的 `.describe()` 同时服务 Agent prompt 和生成的 markdown 文档 |
| 列表稳定 | `tools.sort()` 保证 `tools/list` 的可预测性 |

## 8. 架构图

```mermaid
flowchart LR
    A["工具模块 console.ts / input.ts / ..."] -->|"Object.values"| B["rawTools: mixed(static|factory)"]
    B -->|"typeof === function ? f(args) : v"| C["tools: ToolDefinition[]"]
    C -->|"sort by name"| D["createTools 返回"]
    D -->|"registerTool 循环"| E["server.registerTool"]
    E -->|"根据 pageScoped 分流"| F["不同 handler 调用签名"]
```

## 9. 延伸阅读

- 整体生命周期 → [01-mcp-server-architecture.md](./01-mcp-server-architecture.md)
- Schema 如何驱动遥测 → [19-zod-schema-telemetry.md](./19-zod-schema-telemetry.md)
- 类别过滤 → [12-tool-categories-feature-flags.md](./12-tool-categories-feature-flags.md)

# chrome-devtools-mcp 源码深度技术分析

本目录是对 `chrome-devtools-mcp`（Chrome DevTools for Agents）项目源码的深度拆解，从中提炼出 20 个最具学习价值的技术主题。每一篇文档独立聚焦一个维度，覆盖：

- 整体架构与请求生命周期
- 核心子系统的实现原理
- 工程化落地方案
- 性能与 Token 优化策略
- 风险规避与编码最佳实践

## 主题总览

| # | 文档 | 维度 |
| --- | --- | --- |
| 01 | [MCP Server 整体架构与请求生命周期](./01-mcp-server-architecture.md) | 架构设计 |
| 02 | [工具注册系统与 `defineTool` / `definePageTool` 设计](./02-tool-definition-system.md) | 设计思路 |
| 03 | [Mutex 串行化与「单一工具执行」保证](./03-mutex-serialization.md) | 并发控制 |
| 04 | [McpContext：上下文中心化与多页面状态管理](./04-mcp-context.md) | 状态管理 |
| 05 | [PageCollector：跨导航资源采集与稳定 ID 体系](./05-page-collector.md) | 事件系统 |
| 06 | [WaitForHelper：智能等待（导航 + DOM 稳定）](./06-wait-for-helper.md) | 自动化可靠性 |
| 07 | [TextSnapshot：可访问性树序列化与 UID 映射](./07-text-snapshot.md) | DOM 抽象 |
| 08 | [McpResponse：响应构建器与结构化/文本双输出](./08-mcp-response-builder.md) | 编码模式 |
| 09 | [Token 优化策略：分页 / 摘要 / 文件引用](./09-token-optimization.md) | 性能优化 |
| 10 | [浏览器连接策略：Launch / Connect / AutoConnect](./10-browser-connection.md) | 技术选型 |
| 11 | [Watchdog 子进程与遥测解耦架构](./11-watchdog-telemetry.md) | 系统设计 |
| 12 | [工具分类与条件注册的特性开关体系](./12-tool-categories-feature-flags.md) | 工程化 |
| 13 | [Slim 模式与 Agent 能力的渐进式暴露](./13-slim-mode-progressive-complexity.md) | 产品设计 |
| 14 | [Daemon IPC 架构：CLI 复用 MCP 服务](./14-daemon-ipc.md) | 架构复用 |
| 15 | [自动更新检查与非阻塞后台任务模式](./15-update-check-background-task.md) | 工程实践 |
| 16 | [性能 Trace 采集与 CrUX 集成](./16-performance-trace-crux.md) | 性能分析 |
| 17 | [UniverseManager：DevTools 引擎双栈复用](./17-devtools-universe.md) | 技术集成 |
| 18 | [Formatter 体系与结构化输出双通道](./18-formatter-system.md) | 数据呈现 |
| 19 | [Zod Schema 驱动的工具声明与隐私友好埋点](./19-zod-schema-telemetry.md) | Schema 驱动 |
| 20 | [错误处理、自愈式报错与资源清理模式](./20-error-handling-cleanup.md) | 鲁棒性 |

## 阅读路径建议

- **想了解整体架构** → 01 → 04 → 08 → 12
- **关注自动化可靠性** → 05 → 06 → 07 → 20
- **关注 LLM / Agent 设计思路** → 02 → 09 → 13 → 19
- **关注工程化与运维** → 10 → 11 → 14 → 15
- **关注性能专业领域** → 16 → 17 → 18

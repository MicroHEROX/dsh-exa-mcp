# 标准术语表

> `dsh-exa-mcp` 工程统一术语。英文术语保留原文，中文为约定译名；参考列指向官方文档或本工程的实测说明。

## 一、DeepSeek Harness / Cordis

| 术语 | 说明 | 参考 |
|---|---|---|
| dsh | DeepSeek Harness 的 CLI 与运行时；本工程按 `@deepseek-ai/dsh` 0.2.2-rc.3 实测 | [官方仓库](https://github.com/deepseek-ai/deepseek-harness) |
| Cordis | dsh 的插件框架：插件向共享上下文贡献服务、类型化事件与可逆效果 | [cordis-primer](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md) |
| plugin（插件） | Cordis 的可挂载模块。本 bundle 自身不含运行时模块，只插入官方 mcp-client 行 | [config.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/config.md) |
| bundle（组合包） | 以 npm 包分发的一层配置：`package.json` 声明 `dsh.bundle.patch`，由 profile 叠加 | [publish.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.md) |
| profile | `$DSH_HOME/profiles/<name>/` 下的可运行组合；`dsh.profile.bundles` 列出叠加层 | 同上 |
| patch（补丁层） | 顶层 YAML 数组（`insert` / 按 id 覆盖），加载时合成进配置树 | [cordis-primer#loader-configuration](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md) |
| overlay | `--patch <path>` 传入的临时补丁层，优先级高于 profile 层 | [publish.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.md) |
| entry / row（行） | 配置树中的一行：`id` + `name`（包名/插件名）+ 可选 `config`、`disabled` | [cordis-primer](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md) |
| `!!js` | YAML 标签，标记加载时求值的 JS 表达式；只用于插件 `config`（及 `disabled`），entry 元数据保持静态 | [cordis-primer](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md) |
| ctx | 插件上下文：服务注册表、事件总线、效果注册 | [cordis-primer](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md) |
| service（服务） | 注册在 ctx 上的能力，如 `ctx.tools`、`ctx.systemPrompt` | [architecture.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md) |
| inject（注入） | 插件声明依赖的服务；激活前等待其就绪 | [service.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/framework/service.md) |
| effect（效果） | 插件注册的可逆副作用，卸载时自动回滚 | [cordis-primer](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md) |
| HMR | 热替换：修改配置后无需重启进程即可重载插件 | [publish.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.md) |
| session log | 追加式会话事件日志；当前文件名为 `session.v4.jsonl.zstd`，模型上下文的事实来源 | [architecture.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md) |
| tool registry | 工具注册表 `ctx.tools`；模型可见的工具定义注册于此 | [tools.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/tools.md) |
| DSH_HOME | dsh 数据目录（默认 `~/.dsh`）：profiles、sessions、storages、credentials | [app-boot](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/app-boot/README.md) |
| `--dump-config` | 离线合成并打印最终配置树；不启动、不求值 `!!js`，表达式以原文保留 | [app-boot](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/app-boot/README.md) |
| plugin manager | 插件管理器：安装、停用、卸载、更新 bundle，并读取 locale/icon 显示元数据 | [plugin-manager](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/client/ui-plugin-manager) |
| `engines.dsh` | 作者声明的 DSH 兼容 SemVer 范围；DSH 先读它，缺失时才回退到 DSH peer 依赖 | [package-manifest](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/util/package-manifest) |

## 二、MCP / mcp-client

| 术语 | 说明 | 参考 |
|---|---|---|
| MCP | Model Context Protocol，模型上下文协议 | <https://modelcontextprotocol.io/> |
| Streamable HTTP | MCP 传输方式之一（HTTP + SSE）；`mcp.exa.ai` 使用此方式 | [mcp-client README](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/mcp/mcp-client) |
| stdio | MCP 传输方式之一（子进程 stdin/stdout） | 同上 |
| `@deepseek-ai/dsh-mcp-client` | dsh 的 MCP 客户端桥：连接外部 server，把工具注册到 `ctx.tools` | 同上 |
| serverName | mcp-client 的命名空间，`[A-Za-z0-9_-]{1,32}`，同一注册作用域内唯一 | 同上 |
| 公开工具名 | `mcp__<serverName>__<rawName>`；非法/超长名称会做确定性归一化 | 同上 |
| `toolCallTimeoutMs` | 单次 `tools/call` 或资源请求超时；默认 60000 ms | 同上 |
| `startupTimeoutMs` | 可选的初始模板解析、协商、连接与工具同步总预算；未设时用 SDK 默认 | 同上 |
| `closeTimeoutMs` | 关闭传输与本地清理预算；默认 5000 ms | 同上 |
| `failOnStartupError` | 初始连接/同步失败时是否拒绝插件激活；默认 false | 同上 |
| `maxInstructionBytes` | server instructions 的 UTF-8 字节上限；默认 32768 | 同上 |
| `expandTemplates` | 是否在运行时解析 `${credential:REF}` 并脱敏诊断；默认 false | 同上 |
| reconnect | 断线重连策略：`enabled` / `initialDelayMs` / `maxDelayMs` / `maxAttempts` | 同上 |
| `tools/list_changed` | MCP 通知：服务器工具集变化后桥自动重同步 | 同上 |
| JSON-RPC / -32602 | MCP 调用错误码；-32602 表示参数校验失败 | <https://modelcontextprotocol.io/specification> |
| `@deepseek-ai/dsh-mcp-manager` | 运行时 MCP server 管理器；Web/Desktop 组合挂载，可用设置增删 server | [mcp-manager](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/mcp/mcp-manager) |

## 三、Exa

| 术语 | 说明 | 参考 |
|---|---|---|
| Exa | 网页搜索与全文抓取服务 | <https://exa.ai> |
| `mcp.exa.ai/mcp` | Exa 托管的 MCP 端点（Streamable HTTP） | [Exa MCP 文档](https://exa.ai/docs/reference/exa-mcp) |
| 免费额度（free tier） | 匿名使用、限流（429）；默认只暴露 `web_search_exa` 和 `web_fetch_exa` | [exa-mcp-server](https://github.com/exa-labs/exa-mcp-server) |
| API Key | 在 <https://dashboard.exa.ai/api-keys> 申请；本插件通过 `x-api-key` 发送 | 同上 |
| OAuth | 托管端点的登录流程（`?login`）；dsh mcp-client 不实现，改用 API Key | 同上 |
| `web_search_exa` | 基础语义搜索；必填 `query` 与 `objective`，可选 `numResults` | [API.md](API.md) |
| `web_fetch_exa` | 按 URL 批量抓取全文；必填 `urls`，可选 `maxCharacters` | 同上 |
| `web_search_advanced_exa` | 高级筛选搜索；匿名可用，但需 `?tools=` 白名单启用 | 同上 |
| `agent_run` | Exa Agent 多步研究；需 API Key + `?tools=` 白名单，按用量计费（dsh 不支持 OAuth） | 同上 |
| `?tools=` | MCP URL 查询参数：显式启用工具；它替换默认集合，需把基础工具一起写上 | [Exa MCP 文档](https://exa.ai/docs/reference/exa-mcp) |
| `?defaultSearchType=` | MCP URL 查询参数：默认检索模式，如 `fast` | 同上 |
| `objective` | `web_search_exa` 的必填字段：说明本轮搜索目标、优先/排除的文档与要提取的事实 | [API.md](API.md) |

## 四、本插件约定

| 术语 | 约定 |
|---|---|
| 行 id | `mcp-exa`（覆盖、停用或排查时按此 id 定位） |
| serverName | `exa` |
| 工具命名空间 | `mcp__exa__*` |
| key 环境变量 | `EXA_API_KEY`（有则带 `x-api-key`，无则匿名） |
| 默认超时 | `toolCallTimeoutMs: 180000` |
| 启动失败策略 | `failOnStartupError: false`（可降级启动，桥自动重连） |
| 显示资源 | `locale/en.json`、`locale/zh.json`、`icon.svg`；详情页回退 README |
| 兼容声明 | 不声明 `engines.dsh`；以实测版本和 `--dump-config` 为准 |

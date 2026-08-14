# 标准术语表

> dsh-exa-mcp 工程涉及的统一术语。英文术语保留原文，中文为约定译名。来源标注为 dsh 官方文档或本插件实测。

## 一、DeepSeek Harness / Cordis 侧

| 术语 | 说明 | 参考 |
|---|---|---|
| dsh | DeepSeek Harness 的 CLI 名（`npx @deepseek-ai/dsh`） | [官方 README](https://github.com/deepseek-ai/deepseek-harness#readme) |
| Cordis | dsh 的插件框架：插件向共享上下文贡献服务、类型化事件与可逆效果 | [cordis-primer](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md) |
| 插件（plugin） | Cordis 中一个可挂载的模块（导出 `name` 与 `apply(ctx, config)`） | [config.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/config.md) |
| bundle（组合包） | npm 包形态的配置层，`package.json` 声明 `dsh.bundle.patch` | [publish.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.md) |
| profile | `$DSH_HOME/profiles/<name>/` 下的可运行组合，`dsh.profile.bundles` 列出叠加的 bundle | 同上 |
| patch（补丁层） | 顶层 YAML 数组（`insert` / 按 id 覆盖），在加载时合成进配置树 | [cordis-primer#loader-configuration](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md) |
| overlay | 命令行 `--patch <path>` 传入的临时补丁层，优先级最高 | [publish.md 加载顺序](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.md) |
| entry / row（行） | 配置树中的一行：`id` + `name`（插件名）+ `config`（可选）+ `disabled`（可选） | [cordis-primer](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md) |
| `!!js` | YAML 标签，标记一个 JS 表达式，在**插件 config**（及 `disabled`）加载时求值；作用域含 `process` 与已注入的 `ctx.<service>` | [cordis-tutorial/05-config](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-tutorial/05-config.md) |
| ctx | 插件上下文（Context）：服务注册表、事件总线、效果注册 | [cordis-primer](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md) |
| 服务（service） | 注册在 ctx 上的能力（如 `ctx.tools`、`ctx.llm`、`ctx.sessions`） | [architecture.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md) |
| inject（注入） | 插件声明依赖的服务，激活前框架等待其就绪 | [service.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/framework/service.md) |
| 事件（event） | 类型化广播；会话事件持久化为 session log，代理事件实时分发 | [events.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/framework/events.md) |
| 效果（effect） | 插件在 ctx 上注册的可逆副作用，插件卸载时自动回滚 | [cordis-primer](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md) |
| HMR | 热替换：编辑配置后无需重启进程即可重载插件 | [publish.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.md) |
| session log | 追加式会话事件日志（`session.jsonl.zstd`，多帧 zstd 压缩），模型上下文的事实来源；Web UI 可导出明文 zip | [architecture.md#session-log](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md) |
| tool registry | 工具注册表 `ctx.tools`；模型可见工具即注册于其中的定义 | [tools.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/tools.md) |
| seam（能力缝） | 可替换能力的三件套：服务定义 + 服务提供者 + 消费者 | [capability-seams.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/capability-seams.md) |
| LLM adapter | 模型适配器（如 `@deepseek-ai/dsh-llm-deepseek`），注册在 `ctx.llm` | [llm-streaming.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/llm-streaming.md) |
| credentials（凭据库） | `ctx.credentials` 管理的本地凭据（如 `~/.dsh/.credentials.yaml`），经 `apiKeyEnv` 引用 | [credentials.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/credentials.md) |
| apiKeyEnv | 适配器配置字段：从凭据库/环境变量解析 API key 的变量名 | [verify-config-source-ownership](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/verify-config-source-ownership.ts) |
| DSH_HOME | dsh 数据目录（默认 `~/.dsh`）：profiles / sessions / storages / settings / credentials | [app-boot](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/app-boot/README.md) |
| `--dump-config` | 离线合成并打印最终配置树（不启动、不求值 `!!js`） | [app-boot#renderConfigDump](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/app-boot/README.md) |
| turn / step | turn = 一次任务回合；step = 一次模型请求 + 其调用的工具 | [architecture.md#turn-flow](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md) |
| dsh-plugin topic | 官方约定的 GitHub 主题标签，用于插件可发现性 | [官方 README](https://github.com/deepseek-ai/deepseek-harness#readme) |

## 二、MCP / mcp-client 桥侧

| 术语 | 说明 | 参考 |
|---|---|---|
| MCP | Model Context Protocol，模型上下文协议 | <https://modelcontextprotocol.io/> |
| Streamable HTTP | MCP 传输方式之一（HTTP + SSE），`mcp.exa.ai` 使用此方式 | [mcp-client README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md) |
| stdio | MCP 传输方式之一（子进程 stdin/stdout） | 同上 |
| `@deepseek-ai/dsh-mcp-client` | dsh 的 MCP 客户端桥插件：连接外部 MCP 服务器并把工具注册到 `ctx.tools` | 同上 |
| serverName | mcp-client 配置的命名空间，`[A-Za-z0-9_-]{1,32}`，全实例唯一 | 同上 |
| 公开工具名 | `mcp__<serverName>__<rawName>`（超长/非法字符时确定性哈希归一化） | 同上 |
| `failOnStartupError` | 初始连接/同步失败时是否让插件激活失败（默认 false：仅记录并进入重连） | 同上 |
| reconnect | 断线自动重连策略（`initialDelayMs`/`maxDelayMs`/`maxAttempts`） | 同上 |
| toolCallTimeoutMs | 单次 `callTool` 超时（默认 60000 ms） | 同上 |
| `tools/list_changed` | MCP 通知：服务器工具集变化后 mcp-client 自动重同步 | 同上 |
| JSON-RPC / -32602 | MCP 调用错误码（-32602 = 参数校验失败） | <https://spec.modelcontextprotocol.io/> |

## 三、Exa 侧

| 术语 | 说明 | 参考 |
|---|---|---|
| Exa | 网页搜索/抓取服务提供商 | <https://exa.ai> |
| `mcp.exa.ai/mcp` | Exa 托管的 MCP 端点（Streamable HTTP） | [Exa MCP 文档](https://exa.ai/docs/reference/exa-mcp) |
| 免费额度（free tier） | 匿名使用，限流（429），默认仅 `web_search_exa` + `web_fetch_exa` | [exa-mcp-server](https://github.com/exa-labs/exa-mcp-server) |
| API Key | `https://dashboard.exa.ai/api-keys` 申请；经 `x-api-key` / `Authorization: Bearer` / `?exaApiKey=` 传递 | 同上 |
| OAuth | 托管端点的登录流程（`?login`），dsh 桥不支持 | 同上 |
| `web_search_exa` | 基础搜索工具（query + numResults） | 本插件实测 schema，见 [API.md](API.md) |
| `web_fetch_exa` | 网页全文抓取工具（urls 数组 + maxCharacters） | 同上 |
| `web_search_advanced_exa` | 高级筛选搜索（27 参数）；**匿名也可用**，需 `?tools=` 白名单启用 | 实测见 [API.md](API.md) |
| `agent_run` | Exa Agent 多步研究；需鉴权（API key/OAuth）+ `?tools=` 启用，按用量计费 | 实测见 [API.md](API.md) |
| `?tools=` / `?defaultSearchType=` | MCP URL 查询参数：工具白名单 / 默认检索模式 | [Exa MCP 文档](https://exa.ai/docs/reference/exa-mcp) |

## 四、本插件约定

| 术语 | 约定 |
|---|---|
| 行 id | `mcp-exa`（覆盖/禁用时按此 id 打补丁） |
| serverName | `exa` |
| 工具命名空间 | `mcp__exa__*` |
| key 环境变量 | `EXA_API_KEY`（有则带 key，无则匿名） |
| 默认超时 | `toolCallTimeoutMs: 180000` |

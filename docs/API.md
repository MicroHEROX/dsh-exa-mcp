# API 列表

> `dsh-exa-mcp` 0.2.0 对外接口：安装/加载、`mcp-exa` 行配置、模型可见工具、dsh 侧可观察面。配置字段按 `@deepseek-ai/dsh-mcp-client` 0.2.2-rc.3 的 schema 与实测整理；工具 schema 以 2026-11-16 对 `https://mcp.exa.ai/mcp` 的 `tools/list` 为准。

## 1. 安装与加载接口

| 方式 | 命令 / 文件 | 说明 |
|---|---|---|
| 插件管理器 | 设置 → 插件 → 安装，填包名、GitHub spec 或本地目录 | 写入 profile 的 `dependencies` 与 `dsh.profile.bundles` |
| CLI | `dsh plugin --profile <name> add <包名/路径/github:...>` | 需要 pnpm；`--profile` 必须放在转发的 pnpm 参数之前 |
| 一次性 overlay | `dsh web --patch <path>/cordis.patch.yml` | 不安装，仅本次运行生效 |
| 持久化免安装 | 把 `insert` 块合并进 `$DSH_HOME/profiles/<name>/cordis.patch.yml` | 长期生效；不要与已安装 bundle 叠加同一行 |
| 卸载 | `dsh plugin --profile <name> remove dsh-exa-mcp` | 只改 profile，不动 dsh 安装 |
| 清单 | `package.json` 的 `dsh.bundle.patch: "./cordis.patch.yml"` | 声明 bundle 的 patch 层 |

要求：dsh 自带 `@deepseek-ai/dsh-mcp-client`；`dsh plugin` 需要 pnpm。安装后可用 `dsh --profile <name> --dump-config` 确认 `# == dsh-exa-mcp` 与 `- id: mcp-exa` 出现。

## 2. 配置接口（`mcp-exa` 行）

下表列出本插件实际设置的字段，以及 mcp-client 0.2.2-rc.3 中与本场景相关的可选字段；“未设置”表示沿用桥的默认值。

| 字段 | 类型 | 本插件值 | 说明 |
|---|---|---|---|
| `serverName` | string | `exa` | 必填；`[A-Za-z0-9_-]{1,32}`，同一作用域唯一 |
| `transport` | `"streamable-http"` | `streamable-http` | 必填；Exa 托管端点协议 |
| `url` | string | `https://mcp.exa.ai/mcp` | 可追加 `?tools=` / `?defaultSearchType=` |
| `headers` | object | `!!js` 条件表达式 | 有 `EXA_API_KEY` → `{"x-api-key": <key>}`；无 → `{}` |
| `toolCallTimeoutMs` | number | `180000` | 单次工具调用超时；默认 60000 |
| `startupTimeoutMs` | number | 未设置 | 可选；初始连接与工具同步预算，未设时用 SDK 默认 |
| `closeTimeoutMs` | number | 未设置 | 可选；关闭清理预算，默认 5000 |
| `failOnStartupError` | boolean | `false` | 初始失败只记录并重连；`true` 会拒绝激活 |
| `maxInstructionBytes` | number | 未设置 | 可选；server instructions 上限，默认 32768 |
| `expandTemplates` | boolean | 未设置 | 可选；解析 `${credential:REF}`，默认 false |
| `reconnect.*` | object | 未设置 | 默认 `enabled: true`、`initialDelayMs: 500`、`maxDelayMs: 30000`、`maxAttempts: 10` |

鉴权表达式：

```yaml
headers: !!js 'process.env.EXA_API_KEY ? { "x-api-key": process.env.EXA_API_KEY } : {}'
```

覆盖时按 id 打补丁，不要再用 `insert`；`config` 会被整体替换，必须写全：

```yaml
- id: mcp-exa
  name: '@deepseek-ai/dsh-mcp-client'
  config:
    serverName: exa
    transport: streamable-http
    url: 'https://mcp.exa.ai/mcp?tools=web_search_exa,web_fetch_exa,web_search_advanced_exa'
    headers: !!js 'process.env.EXA_API_KEY ? { "x-api-key": process.env.EXA_API_KEY } : {}'
    toolCallTimeoutMs: 180000
    failOnStartupError: false
```

注意：`startupTimeoutMs`、`closeTimeoutMs`、`expandTemplates`、`maxInstructionBytes` 是较新 mcp-client 才有的字段；旧版 DSH 可能不认识，跨版本使用前先查对应版本的 schema。

## 3. 工具接口（模型可见）

命名：`mcp__exa__<rawName>`。匿名默认只暴露 `web_search_exa` 与 `web_fetch_exa`；高级工具需 `?tools=` 白名单，`agent_run` 还需 API Key。

### 3.1 `mcp__exa__web_search_exa`

语义化网页搜索，返回可读正文/摘要。

```json
{
  "type": "object",
  "properties": {
    "query": { "type": "string", "minLength": 1 },
    "objective": { "type": "string", "minLength": 1, "maxLength": 4096 },
    "numResults": { "type": "number" }
  },
  "required": ["query", "objective"],
  "additionalProperties": false
}
```

`objective` 说明本轮搜索的目标、应优先/排除的文档、要提取的具体事实；当前端点把它列为必填。

### 3.2 `mcp__exa__web_fetch_exa`

按已知 URL 抓取干净 Markdown，支持批量。

```json
{
  "type": "object",
  "properties": {
    "urls": { "type": "array", "items": { "type": "string" } },
    "maxCharacters": { "type": "number", "minimum": 1 }
  },
  "required": ["urls"],
  "additionalProperties": false
}
```

`maxCharacters` 默认 3000。

### 3.3 `mcp__exa__web_search_advanced_exa`

高级筛选搜索；匿名可用，但必须通过 URL 白名单启用，例如：

```text
https://mcp.exa.ai/mcp?tools=web_search_exa,web_fetch_exa,web_search_advanced_exa
```

`?tools=` 是替换语义：列出的工具覆盖默认集合，所以要把基础工具一起写上。当前 schema 约 28 个参数，按用途分组：

- 基础：`query`（必填）、`numResults`、`type`、`category`
- 域名：`includeDomains`、`excludeDomains`
- 时间：`startPublishedDate`、`endPublishedDate`、`startCrawlDate`、`endCrawlDate`、`maxAgeHours`
- 文本：`includeText`、`excludeText`、`textMaxCharacters`、`contextMaxCharacters`
- 摘要/高亮：`enableSummary`、`summaryQuery`、`enableHighlights`、`highlightsMaxCharacters`、`highlightsNumSentences`、`highlightsPerUrl`、`highlightsQuery`
- 其他：`userLocation`、`moderation`、`additionalQueries`、`livecrawlTimeout`、`subpages`、`subpageTarget`

完整语义以 [Exa MCP 文档](https://exa.ai/docs/reference/exa-mcp) 和实时 `tools/list` 为准。

### 3.4 `mcp__exa__agent_run`

多步研究/列表构建/结构化输出；需要 API Key + `?tools=` 白名单。匿名白名单调用返回 `-32000 Authentication required`。按用量计费。

关键参数（Exa 文档与历史实测）：`query`（与 `runId` 二选一）、`runId`（`agent_run_*`，续跑）、`systemPrompt`、`outputSchema`、`input`。返回 `{success, id, status, outputReady, output}`；`output.structured` 按 `outputSchema` 输出，`output.grounding` 带引用。耗时数秒至数分钟，受 `toolCallTimeoutMs` 约束。

## 4. 运行时接口（dsh 侧可观察面）

### 4.1 会话日志

`tool/call` 与 `tool/result` 记录工具名、参数、结果或 `isError`。当前会话文件为 `session.v4.jsonl.zstd`，具体字段随 session format 版本变化；排查时以实际文件/Web UI「Session log」导出为准，不要依赖旧版固定路径。

### 4.2 进程日志

mcp-client 以 `mcp-client(<serverName>): ...` 前缀输出重连、恢复与最终失败；本插件为 `mcp-client(exa)`。

## 5. 外部接口（不在本插件范围）

- Exa REST API（`api.exa.ai`）：本插件经 MCP 端点访问，不直接调用 REST。
- Exa OAuth（`auth.exa.ai` / `?login`）：dsh mcp-client 不实现，请用 API Key。

## 6. 版本

本文件不重复维护版本矩阵；以 [PROJECT.md §7](PROJECT.md#7-版本兼容) 为准。

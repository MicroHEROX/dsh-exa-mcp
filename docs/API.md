# API 列表

> dsh-exa-mcp 对外接口全集：安装/加载接口、配置接口、工具接口、运行时接口。除注明外均为**实测验证**（dsh 0.1.0-rc.6 + 真实 Exa 端点）。

## 1. 安装与加载接口

| 方式 | 命令 / 文件 | 说明 |
|---|---|---|
| 标准安装 | `dsh plugin --profile <name> add <path-or-package-or-github>` | 初始化/更新 profile，pnpm 链接，追加 bundle 到 `dsh.profile.bundles` |
| 卸载 | `dsh plugin --profile <name> remove dsh-exa-mcp` | 移除依赖与 bundle 层，profile 其余部分不变 |
| 一次性覆盖 | `dsh web --patch <path>/cordis.patch.yml` | 不安装，仅本次运行生效 |
| 持久化（免安装） | 合并 insert 块到 `$DSH_HOME/profiles/<name>/cordis.patch.yml` | 长期生效 |
| npm 清单 | `package.json` 的 `dsh.bundle.patch: "./cordis.patch.yml"` | 声明 bundle 的补丁层 |

要求：dsh CLI 自带 `@deepseek-ai/dsh-mcp-client`；`dsh plugin` 需要机器上安装 pnpm。

## 2. 配置接口（`mcp-exa` 行）

行定义（继承 `@deepseek-ai/dsh-mcp-client` 的 Config schema，逐字段校验）：

| 字段 | 类型 | 必填 | 本插件值 | 说明 |
|---|---|---|---|---|
| `serverName` | string | 是 | `exa` | `[A-Za-z0-9_-]{1,32}`，决定 `mcp__exa__*` 命名 |
| `transport` | `"streamable-http"` | 是 | `streamable-http` | Exa 托管端点协议 |
| `url` | string | 是 | `https://mcp.exa.ai/mcp` | 可追加 `?tools=` / `?defaultSearchType=` 调参 |
| `headers` | object | 否 | `!!js` 条件表达式 | 有 `EXA_API_KEY` → `{"x-api-key": <key>}`；无 → `{}` |
| `toolCallTimeoutMs` | number | 否 | `180000` | 适配长耗时的 agent 研究 |
| `failOnStartupError` | boolean | 否 | （默认 false） | 初始失败仅记录并进入重连 |
| `reconnect.*` | object | 否 | （默认） | `enabled`/`initialDelayMs`/`maxDelayMs`/`maxAttempts` |

覆盖方法（dsh patch 层语义，需重写完整 config）：

```yaml
- id: mcp-exa
  name: '@deepseek-ai/dsh-mcp-client'
  config:
    serverName: exa
    transport: streamable-http
    url: 'https://mcp.exa.ai/mcp?tools=web_search_exa,web_fetch_exa,agent_run'
    headers: !!js 'process.env.EXA_API_KEY ? { "x-api-key": process.env.EXA_API_KEY } : {}'
    toolCallTimeoutMs: 300000
```

## 3. 工具接口（模型可见）

命名：`mcp__exa__<rawName>`。匿名层实测为 2 个；有 key 后可经 `?tools=` 白名单启用更多。

### 3.1 `mcp__exa__web_search_exa`（匿名可用，实测）

- 描述：语义化网页搜索，返回干净文本结果（标题/URL/发布时间/高亮）
- Schema（JSON Schema draft-07）：

```json
{
  "type": "object",
  "properties": {
    "query": { "type": "string", "minLength": 1,
      "description": "Natural language search query. Should be a semantically rich description of the ideal page, not just keywords. Optionally include category:<type> (company, people) to focus results" },
    "numResults": { "type": "number",
      "description": "Number of search results to return (default: 10)" }
  },
  "required": ["query"],
  "additionalProperties": false
}
```

### 3.2 `mcp__exa__web_fetch_exa`（匿名可用，实测）

- 描述：按已知 URL 抓取网页干净 markdown，支持批量
- Schema：

```json
{
  "type": "object",
  "properties": {
    "urls": { "type": "array", "items": { "type": "string" },
      "description": "URLs to read. Batch multiple URLs in one call." },
    "maxCharacters": { "type": "number", "minimum": 1,
      "description": "Maximum characters to extract per page (default: 3000)" }
  },
  "required": ["urls"],
  "additionalProperties": false
}
```

### 3.3 `mcp__exa__web_search_advanced_exa`（需鉴权，未实测）

- 高级筛选搜索：日期/域名/分类/实时抓取（livecrawl）等；需 API key，可经 `?tools=` 白名单启用
- 来源：[Exa MCP 文档](https://exa.ai/docs/reference/exa-mcp)

### 3.4 `mcp__exa__agent_run`（需鉴权，未实测）

- Exa Agent 多步自主研究，按使用量计费；需 API key，白名单示例 `?tools=web_search_exa,web_fetch_exa,agent_run`
- 来源：[Exa MCP 文档](https://exa.ai/docs/reference/exa-mcp)

## 4. 运行时接口（dsh 侧可观察面）

### 4.1 会话日志事件（session log）

| 事件 | 关键字段（实测） | 说明 |
|---|---|---|
| `tool/call` | `data.name`、`data.arguments`（JSON 字符串）、`data.callId`、`data.turn/step` | 工具调用记录，`name` 为 `mcp__exa__*` 公开名 |
| `tool/result` | `data.message.content[0].content[0].text`、`...isError` | 结果或错误；`isError: true` 表示 MCP 错误（如 Exa 401/-32602）透传 |

### 4.2 日志（stderr）

`mcp-client(<serverName>): ...` 前缀：重连（warn）、恢复（info）、最终失败（error）。

## 5. 外部接口（不在本插件范围内）

- Exa REST API（`api.exa.ai`）：本插件经 MCP 端点访问，不直接调用 REST
- Exa OAuth（`auth.exa.ai`）：dsh mcp-client 桥不支持，请使用 API key

## 6. 版本

| 组件 | 版本 |
|---|---|
| `dsh-exa-mcp` | **0.1.0**（npm 名与 GitHub 仓库名一致；Releases 标签同步） |
| `@deepseek-ai/dsh`（CLI） | ≥ 0.1.0-rc.5；实测 0.1.0-rc.6 |
| `@deepseek-ai/dsh-mcp-client` | ^0.1.0-rc.6（CLI 随附，解析自 dsh 安装） |
| Exa MCP 端点服务端 | 3.2.1（2026-08-14 实测 `initialize` 返回） |
| MCP 协议版本 | 2025-06-18 |
| Node.js | 实测 v24.16.0（建议 ≥ 22） |

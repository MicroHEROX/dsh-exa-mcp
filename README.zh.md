# Exa Search MCP for DeepSeek Harness

<p align="center">
  <a href="README.md"><b>English</b></a>
</p>

<p align="center">
  <a href="https://github.com/MicroHEROX/dsh-exa-mcp/blob/main/LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/license-MIT-blue.svg"></a>
  <a href="https://github.com/topics/dsh-plugin"><img alt="dsh-plugin" src="https://img.shields.io/badge/dsh-plugin-8A2BE2"></a>
  <a href="https://github.com/MicroHEROX/dsh-exa-mcp"><img alt="stars" src="https://img.shields.io/github/stars/MicroHEROX/dsh-exa-mcp"></a>
</p>

面向 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（`dsh`）的第三方插件 bundle：把 [Exa](https://exa.ai) 的神经网页搜索与全文抓取接入 Agent。它通过 dsh 自带的 MCP 客户端桥连接 Exa 托管的 [MCP 端点](https://mcp.exa.ai/mcp)（Streamable HTTP），把 Exa 工具注册为 `exa` 命名空间下的原生工具。

```text
mcp__exa__web_search_exa   ·  mcp__exa__web_fetch_exa
mcp__exa__web_search_advanced_exa  ·  mcp__exa__agent_run   （需白名单；Agent 另需 API Key）
```

- **dsh 进程内不运行第三方代码**——上游是 Exa 官方托管端点。
- **纯配置 bundle**——一层 patch、一行插件，无构建步骤、无运行时模块。
- **显示资源齐全**——中英文标题/描述与原创 SVG 图标，插件管理器可直接展示。

---

## 快速开始

### 1. 安装

**方式 A —— 插件管理器（推荐）。** 打开 **设置 → 插件 → 安装**，填包名、GitHub spec 或本地目录。等价的 CLI：

```sh
npm install -g pnpm
dsh plugin --profile web add github:MicroHEROX/dsh-exa-mcp
dsh web
```

> GitHub 网络不稳时，`github:` 安装可能静默漏掉 bundle 层。用 `dsh --profile web --dump-config | grep -A2 "== dsh-exa-mcp"` 确认；本地 `link:`/`file:` 安装与 `--patch` 不受影响。完整修复步骤见 [docs/SOLUTIONS.md §1.7](docs/SOLUTIONS.md#17-github-安装偶发不生效或卸载残留)。

**方式 B —— 一次性 overlay，不安装：**

```sh
dsh web --patch /path/to/dsh-exa-mcp/cordis.patch.yml
```

**方式 C —— 免安装长期生效：** 把 `cordis.patch.yml` 中唯一的 `insert` 块合并进 `$DSH_HOME/profiles/<name>/cordis.patch.yml`（或 `$DSH_HOME/cordis.patch.yml`，对全部 profile 生效）。

运行时只会装入 `cordis.patch.yml`、`locale/`、`icon.svg`、README 与 LICENSE；`docs/` 留在仓库，不进入安装环境。

### 2. 设置 Exa API Key（可选）

托管端点**匿名可用**（免费额度、限流、仅基础工具）。要提升额度并启用高级工具，在 [dashboard](https://dashboard.exa.ai/api-keys) 申请 key，启动 dsh 前设置：

```sh
export EXA_API_KEY="your-key"        # macOS / Linux
$env:EXA_API_KEY = "your-key"        # Windows PowerShell
```

插件仅在变量存在时附加 `x-api-key`，否则匿名回退。切勿把 key 写进任何 patch 文件。

### 3. 验证

1. 启动 `dsh web`（已安装 bundle 或叠加 overlay）。
2. 提问：*“用 Exa 找一下 DeepSeek Harness 最新的 release notes 并总结。”*
3. 确认模型调用了 `mcp__exa__web_search_exa`（带 `query` 和 `objective`），必要时再调 `mcp__exa__web_fetch_exa`。

---

## 做了什么

- 通过 dsh 自带的 `@deepseek-ai/dsh-mcp-client`（`streamable-http`）连接 `https://mcp.exa.ai/mcp`。
- 把 Exa 广告的每个工具注册为 `mcp__exa__<tool>`；监听 `tools/list_changed` 自动重同步。
- 仅在 `EXA_API_KEY` 存在时附加 `x-api-key`——不发送 `undefined` 头，不落盘凭据。
- 用 `toolCallTimeoutMs: 180000` 适配长研究调用；重连策略保持桥默认。
- 保持 `failOnStartupError: false`：Exa 故障不会拒绝插件激活，桥记录并重连。

## 不做什么

- 不下载、托管或监督 Exa 服务器。
- 不实现 OAuth（`?login`），请用 API Key。
- 不桥接 MCP resources / prompts，只消费工具。
- 不管理 Exa 套餐/计费，不存储你的 key。
- 不修改 dsh 安装目录；安装/卸载只改 `$DSH_HOME/profiles/`。

## 配置

| 目标 | 做法 |
|---|---|
| 匿名搜索 + 抓取 | 什么都不用做；免费额度、限流、2 个工具 |
| 提高额度 | 设置 `EXA_API_KEY` 后重启 dsh |
| 启用高级搜索 | 覆盖 `url`，加 `?tools=web_search_exa,web_fetch_exa,web_search_advanced_exa` |
| 启用 Exa Agent | `?tools=` 加 `agent_run` 并使用 API Key；按用量计费 |
| 改默认检索模式 | `url` 加 `?defaultSearchType=fast` |

按 id 覆盖 `mcp-exa` 行（不要写 `insert`；`config` 整体替换）。完整字段与示例见 [docs/API.md §2](docs/API.md#2-配置接口mcp-exa-行)。

## 卸载

```sh
dsh plugin --profile <name> remove dsh-exa-mcp
dsh --profile <name> --dump-config | grep -c "dsh-exa-mcp"   # 期望 0
```

`--patch` overlay 去掉启动参数即可；合并进 profile patch 的删除 `mcp-exa` 块。卸载不会触碰 dsh 安装或其他 bundle。

---

## 版本兼容

| 组件 | 版本 | 说明 |
|---|---|---|
| `dsh-exa-mcp` | **0.2.0** | 见 [Releases](https://github.com/MicroHEROX/dsh-exa-mcp/releases) |
| DeepSeek Harness CLI | 设计兼容 **≥ 0.1.0-rc.5**；实测 **0.2.2-rc.3** | 随 CLI 附带 mcp-client |
| MCP 桥 | 由 dsh 解析；实测 **0.2.2-rc.3** | 无需单独安装 |
| Exa MCP 端点 | server **3.5.1**（2026-11-16） | Exa 托管；以 `tools/list` 为准 |
| MCP 协议 | `2025-06-18`（自动协商） | — |
| Node.js | dsh 要求 **≥ 24.9** | 插件本身未声明 `engines` |
| 平台 | Windows / macOS / Linux | 纯配置，无平台代码 |

dsh 处于 developer preview。升级 dsh 后请重跑 [docs/SOLUTIONS.md §4](docs/SOLUTIONS.md#四验证与取证) 的检查。

## 安全

- 唯一密钥是 `EXA_API_KEY`：装载时从环境读取，以 `x-api-key` 发给 Exa；插件不落盘。
- dsh 进程内不执行第三方代码。
- 本仓库不含 key、本地路径或机器数据。

## 文档

- [工程文档](docs/PROJECT.md) · [术语表](docs/GLOSSARY.md) · [API 列表](docs/API.md) · [解决方案](docs/SOLUTIONS.md)
- DeepSeek Harness：<https://github.com/deepseek-ai/deepseek-harness>
- Exa MCP 文档：<https://exa.ai/docs/reference/exa-mcp>

## License

[MIT](LICENSE)。非 DeepSeek 或 Exa 官方产品。

## 致谢

为 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（DeepSeek AI 出品、基于 [Cordis](https://github.com/cordiverse/paper) 的“万物皆插件”harness）而构建。感谢：

- [Exa MCP Server](https://github.com/exa-labs/exa-mcp-server)——本插件连接的托管搜索端点
- [Model Context Protocol](https://modelcontextprotocol.io/)——本 bundle 所讲的协议
- [@deepseek-ai/dsh-mcp-client](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/mcp/mcp-client)——MCP 桥
- [Cordis](https://github.com/cordiverse/cordis) 及其插件生态

# Exa Search MCP for DeepSeek Harness

<p align="center">
  <a href="README.md"><b>English</b></a> ·
  <a href="#中文"><b>中文</b></a>
</p>

<p align="center">
  <a href="https://github.com/MicroHEROX/dsh-exa-mcp/blob/main/LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/license-MIT-blue.svg"></a>
  <a href="https://github.com/topics/dsh-plugin"><img alt="dsh-plugin" src="https://img.shields.io/badge/dsh-plugin-8A2BE2"></a>
  <a href="https://github.com/MicroHEROX/dsh-exa-mcp"><img alt="stars" src="https://img.shields.io/github/stars/MicroHEROX/dsh-exa-mcp"></a>
</p>

面向 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（`dsh`）的第三方插件：将 Exa 的[神经网络网页搜索与抓取](https://exa.ai)能力接入 agent。它通过 dsh CLI 自带的 MCP 客户端桥（`@deepseek-ai/dsh-mcp-client`）连接 Exa 托管的 [MCP 端点](https://mcp.exa.ai/mcp)（Streamable HTTP），把 Exa 工具以 `exa` 命名空间注册为原生工具。

```
mcp__exa__web_search_exa   ·  mcp__exa__web_fetch_exa
mcp__exa__web_search_advanced_exa  ·  mcp__exa__agent_run   （需 API Key）
```

- 零运行负担：进程内不运行任何第三方代码——上游是 Exa 官方托管端点
- 纯配置 bundle：一层 patch，无构建步骤、无运行时 API
- 完全不动 deepseek-harness 安装：只向组合后的 `cordis.yml` 增加一行

---

## 快速开始

### 1. 安装

**方式 A —— 安装为插件 bundle（推荐，需要 [pnpm](https://pnpm.io/zh-CN/installation)）：**

```sh
npm install -g pnpm
dsh plugin --profile web add github:MicroHEROX/dsh-exa-mcp
dsh web
```

> **`github:` 安装后请验证** —— 当到 github.com 的网络不稳定时，`github:`（git 协议）安装可能静默失败或完成后 `dsh.profile.bundles` 未同步（本地 `link:`/`file:` 安装不受影响）。一键检查并修复：
>
> ```sh
> # 检查：bundle 层是否出现
> dsh --profile web --dump-config | grep -A2 "== dsh-exa-mcp"
> # 未出现则执行修复（追加到 profile manifest）：
> node -e "const fs=require('fs');const p=process.env.DSH_HOME+'/profiles/web/package.json';const j=JSON.parse(fs.readFileSync(p,'utf8'));if(!(j.dsh.profile.bundles||[]).includes('dsh-exa-mcp')){j.dsh.profile.bundles=[...(j.dsh.profile.bundles||[]),'dsh-exa-mcp'];fs.writeFileSync(p,JSON.stringify(j,null,2)+'\n','utf8');console.log('fixed: dsh-exa-mcp appended');}else{console.log('already present');}"
> ```
>
> 该时序问题已反馈官方（[Discussions #656](https://github.com/deepseek-ai/deepseek-harness/discussions/656)）；`--patch` overlay 不受影响。

**方式 B —— 一次性 overlay，不安装：**

```sh
dsh web --patch /path/to/dsh-exa-mcp/cordis.patch.yml
```

**方式 C —— 免安装长期生效：** 把 `cordis.patch.yml` 中唯一的 `insert` 块合并进 `$DSH_HOME/profiles/<name>/cordis.patch.yml`（或 `$DSH_HOME/cordis.patch.yml` 对全部 profile 生效）。

> 安装行为说明：pnpm 的 git 安装按 `files` 字段打包——`docs/` **不会**进入你的运行环境，只安装运行时需要的 `cordis.patch.yml`；文档完整保留在本仓库。

### 2. 设置 Exa API Key（可选）

托管端点**匿名可用**（免费额度，限流，仅基础工具）。要提升限额并解锁高级搜索 / Exa Agent，请在 [dashboard](https://dashboard.exa.ai/api-keys) 申请 key 并设置环境变量：

```sh
export EXA_API_KEY="your-key"        # macOS / Linux
$env:EXA_API_KEY = "your-key"        # Windows PowerShell
```

插件在加载时**自动判断**：有 `EXA_API_KEY` → 附加 `x-api-key` 请求头；无 → 匿名模式。切勿把 key 写进任何 patch 文件。

### 3. 验证

1. 启动 `dsh web`（已安装 bundle 或叠加 overlay）。
2. 稍等初始发现完成（异步）。
3. 提问：*"Use Exa to find the latest release notes of the DeepSeek Harness project on GitHub and summarize them."*
4. 确认模型调用了 `mcp__exa__web_search_exa`（必要时 `web_fetch_exa` 获取全文）并基于结果作答。

---

## 做了什么

- 通过 dsh CLI 官方随附的 `@deepseek-ai/dsh-mcp-client`（`streamable-http`）连接 `https://mcp.exa.ai/mcp`
- 将 Exa 发布的每个工具按规范注册为 `mcp__exa__<tool>`；监听 `tools/list_changed` 自动重同步
- 自动鉴权：仅在 `EXA_API_KEY` 存在时附加 `x-api-key`（优雅回落匿名，绝不发送坏的 `undefined` 头）
- 针对搜索场景调优：`toolCallTimeoutMs: 180000` 适配长耗时研究任务；重连策略保持桥默认
- 严格遵循 dsh 插件规范：bundle 清单（`dsh.bundle.patch`）、patch 层合成、按 id 覆盖（`mcp-exa`）、`!!js` 仅用于 config 表达式

## 没做什么

- **不**下载、托管或监督任何 Exa 服务器——上游是 Exa 托管端点
- **不**实现 OAuth 登录（`https://mcp.exa.ai/mcp?login`）——dsh 桥无 OAuth 流程，请用 API key
- **不**桥接 MCP 的 resources / prompts——harness 只消费 MCP 工具
- **不**代管 Exa 套餐、计费或存储你的 key——key 只存在于你的环境变量
- **不**修改你的 deepseek-harness 安装——安装/卸载只触碰 `$DSH_HOME/profiles/`

## 能走的路线

| 路线 | 做法 |
|---|---|
| 匿名搜索 + 抓取 | 什么都不用做——免费额度、限流、2 个工具 |
| 完整工具集（高级搜索、Agent） | 设置 `EXA_API_KEY` 后重启；Agent 需 `?tools=` 白名单（见下） |
| 工具白名单 / 默认检索模式 | 覆盖 `mcp-exa` 行的 `url`：`?tools=web_search_exa,web_fetch_exa,agent_run` 或 `?defaultSearchType=fast`（见 [docs/API.md](docs/API.md)） |
| 多 MCP 服务器并存 | 增加更多 `mcp-client` 行，使用唯一 `serverName` |
| 热重载 | 编辑 patch 层中的行——HMR 免重启重连 |
| 卸载 | `dsh plugin --profile <name> remove dsh-exa-mcp`——profile 与基础 bundle 保持完好 |

## 不能走的路线（设计如此）

| 路线 | 原因 |
|---|---|
| OAuth 登录流程 | dsh mcp-client 不实现 OAuth 握手——请用 API key |
| MCP resources / prompts | harness 只桥接工具 |
| 每次请求动态切换鉴权 | `EXA_API_KEY` 在配置求值时决定（启动 / HMR 时），非调用时 |
| bundle 已装又叠加同一 `--patch` | dsh 会 fail loud：`duplicate loader entry id: mcp-exa`——二选一 |
| 把 key 写进 patch 文件 | key 属于环境变量；提交 key 即泄露 |

## 卸载

**Bundle 安装**（经 `dsh plugin add` 安装的）：

```sh
dsh plugin --profile <name> remove dsh-exa-mcp
```

验证无残留：

```sh
dsh --profile <name> --dump-config | grep -c "dsh-exa-mcp"   # 期望 0
```

> 若在**网络不稳**时经 `github:` 安装，`remove` 可能残留悬空的 `dsh-exa-mcp` 条目于 `dsh.profile.bundles`，导致 profile 启动失败（`cannot resolve profile bundle "dsh-exa-mcp"`）。用下面命令移除该条目（node 写入，无 BOM）：
>
> ```sh
> node -e "const fs=require('fs');const p=process.env.DSH_HOME+'/profiles/<name>/package.json';const j=JSON.parse(fs.readFileSync(p,'utf8'));j.dsh.profile.bundles=(j.dsh.profile.bundles||[]).filter(b=>b!=='dsh-exa-mcp');fs.writeFileSync(p,JSON.stringify(j,null,2)+'\n','utf8')"
> ```

**Overlay / 手动方式**（未装 bundle）：

- `--patch` overlay：从启动命令中去掉 `--patch <path>/cordis.patch.yml` 参数即可——不产生任何持久残留
- 合并进 profile patch 文件的：从 `$DSH_HOME/profiles/<name>/cordis.patch.yml`（或 `$DSH_HOME/cordis.patch.yml`，对全部 profile 生效）中删除 `mcp-exa` 块（或整个 `insert` 列表）

可选：不再使用 Exa 时，从环境中移除 `EXA_API_KEY`。

卸载不会触碰 deepseek-harness 安装或其他任何 bundle——只编辑 `$DSH_HOME` 下的 profile 目录。

---

## 版本兼容

| 组件 | 版本 | 说明 |
|---|---|---|
| `dsh-exa-mcp`（本插件） | **0.1.0** | 见 [Releases](https://github.com/MicroHEROX/dsh-exa-mcp/releases) |
| DeepSeek Harness CLI（`@deepseek-ai/dsh`） | **≥ 0.1.0-rc.5**，实测 **0.1.0-rc.7** | CLI 随附本 bundle 挂载的 `@deepseek-ai/dsh-mcp-client` 桥 |
| MCP 桥（`@deepseek-ai/dsh-mcp-client`） | `^0.1.0-rc.7`（由 dsh CLI 解析） | 无需单独安装 |
| Exa MCP 端点（`mcp.exa.ai/mcp`） | 服务端 **3.2.1**（2026-08-14 实测） | 由 Exa 维护，可能随时变化 |
| MCP 协议版本 | `2025-06-18` | 自动协商 |
| Node.js | 实测 **v24.16.0**；建议 ≥ 22 | dsh 本身未声明 `engines` 范围 |
| 平台 | Windows / macOS / Linux | 纯配置 bundle，无平台差异代码 |

dsh 处于 developer preview，迭代较快。升级 dsh 后请重跑 [docs/SOLUTIONS.md](docs/SOLUTIONS.md) 的验证清单。

---

## 安全

- 唯一涉及的密钥是 `EXA_API_KEY`：加载时从环境读取，以 `x-api-key` 头发给 Exa，本插件不落盘任何文件
- dsh 进程内不执行第三方代码——插件只是基于官方桥的声明式配置
- 本仓库不含任何 key、本地路径或机器数据

## License

[MIT](LICENSE)。非 DeepSeek 或 Exa 官方产品。

## 致谢

为 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（DeepSeek AI 出品、基于 [Cordis](https://github.com/cordiverse/paper) 的"万物皆插件"harness）而构建。感谢：

- [Exa MCP Server](https://github.com/exa-labs/exa-mcp-server) —— 本插件连接的托管搜索端点
- [Model Context Protocol SDK](https://github.com/modelcontextprotocol/modelcontextprotocol) —— 插件所讲协议
- [@deepseek-ai/dsh-mcp-client](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/mcp/mcp-client) —— 本 bundle 使用的 MCP 桥
- [Cordis](https://github.com/cordiverse/cordis) 及其插件生态 —— dsh 的底层框架

感谢 DeepSeek Harness 团队与所有被本项目使用、参考的开源项目。

## 文档

- [工程文档](docs/PROJECT.md) · [术语表](docs/GLOSSARY.md) · [API 列表](docs/API.md) · [解决方案与坑](docs/SOLUTIONS.md)
- DeepSeek Harness：<https://github.com/deepseek-ai/deepseek-harness>
- Exa MCP 文档：<https://exa.ai/docs/reference/exa-mcp>

# dsh-exa-mcp 工程文档

> 面向 DeepSeek Harness（`dsh`）的第三方 Exa 搜索插件。本文档描述工程结构、规范符合性、验证方法与发布流程。

## 1. 项目定位

- **类型**：第三方插件 bundle（纯配置型，无运行时代码）
- **功能**：通过 dsh 自带的 MCP 客户端桥（`@deepseek-ai/dsh-mcp-client`）连接托管 Exa MCP 端点（`https://mcp.exa.ai/mcp`，Streamable HTTP），将 Exa 工具以 `mcp__exa__*` 命名注册为模型可用工具
- **不做什么**：不修改任何 deepseek-harness 安装文件；不在进程内运行第三方代码；不接管 Exa 的账号、计费与鉴权（由用户通过环境变量提供）

## 2. 目录结构

```
dsh-exa-mcp/
├── package.json          # npm 包 + dsh bundle 清单（dsh.bundle.patch）
├── cordis.patch.yml      # 唯一 patch 层：插入 mcp-exa 行
├── README.md             # 英文使用说明
├── README.zh.md          # 中文使用说明
├── LICENSE               # MIT
└── docs/
    ├── PROJECT.md        # 本文档
    ├── GLOSSARY.md       # 标准术语表
    ├── API.md            # API 列表（配置接口 + 工具接口 + 运行时接口）
    └── SOLUTIONS.md      # 解决方案文档（坑、疑难、方法论、官方文档地址）
```

## 3. 核心机制（单一职责）

`cordis.patch.yml` 是插件的全部内容——一个 `insert` 块，插入一行 id 为 `mcp-exa` 的 mcp-client 实例：

```yaml
- insert:
    - id: mcp-exa
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: exa
        transport: streamable-http
        url: https://mcp.exa.ai/mcp
        headers: !!js 'process.env.EXA_API_KEY ? { "x-api-key": process.env.EXA_API_KEY } : {}'
        toolCallTimeoutMs: 180000
```

要点：

| 字段 | 说明 |
|---|---|
| `name` | 解析到 dsh CLI 随附的 `@deepseek-ai/dsh-mcp-client`（无需声明依赖） |
| `serverName: exa` | 工具命名空间，决定公开名 `mcp__exa__<tool>` |
| `transport/url` | Exa 托管端点为 Streamable HTTP 协议 |
| `headers` | 加载时求值的 `!!js` 表达式：有 `EXA_API_KEY` 自动附 `x-api-key`，无则空对象（匿名） |
| `toolCallTimeoutMs` | 180 s，适配长耗时的 agent 研究任务 |

## 4. 规范符合性（依据 dsh 官方文档）

| 规范点 | 依据文档 |
|---|---|
| bundle 清单 `dsh.bundle.patch` | [publish.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.md) |
| patch YAML 方言（`entryListSchema` + `!!js`） | [cordis-primer.md#loader-configuration](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md) |
| `!!js` 仅用于插件 `config`，元数据保持静态 | [verify-cordis-config](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/verify-cordis-config.ts) 规则 |
| mcp-client 配置契约 | [@deepseek-ai/dsh-mcp-client README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md) |
| 按 id 覆盖（`mcp-exa`） | [publish.md 加载顺序](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.md) |
| 工具命名 `mcp__<serverName>__<rawName>` | [mcp-client README#Tool-naming](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md) |

已实测通过的真实 CLI 验证（dsh 0.1.0-rc.6）：`--dump-config` 合成、headless 全链路调用、web 启动、安装/卸载/重装（详见 [SOLUTIONS.md](SOLUTIONS.md#4-验证方法论)）。

## 5. 开发与构建

```sh
# 无需构建：纯配置 bundle，无编译步骤
# 本地冒烟（不安装）：
dsh web --patch "$PWD/cordis.patch.yml"

# 标准安装（需 pnpm）：
npm install -g pnpm
dsh plugin --profile web add ./dsh-exa-mcp
```

## 6. 发布

```sh
npm pack          # 产出 dsh-exa-mcp-0.1.0.tgz
# 或发布到 npm registry 后：dsh plugin add dsh-exa-mcp
# 或推送到 GitHub 后：dsh plugin add github:you/dsh-exa-mcp
```

### 安装内容与文档的关系（GitHub 安装）

- **仓库**（git）包含全部文件：`cordis.patch.yml`、包文件与 `docs/` 四份文档——发布 GitHub 时必须全部提交
- **运行时安装**（pnpm git 依赖 / npm 打包）只按 `package.json` 的 `files` 字段打包：仅 `cordis.patch.yml` 等运行时文件进入 `node_modules`，**`docs/` 不会下载安装到运行环境**（已用真实 git 依赖协议实测验证）
- 结论：文档随仓库分发（可在线阅读/克隆），但不污染用户运行环境；修改 `files` 时请保持 `docs/` 排除在外

发布建议：在 GitHub 仓库添加 `dsh-plugin` topic（官方约定，见 [官方 README](https://github.com/deepseek-ai/deepseek-harness#readme)）。

## 7. 版本兼容

| 组件 | 版本 | 说明 |
|---|---|---|
| `dsh-exa-mcp` | **0.1.0** | 语义化版本；破坏性变更走 minor 升版 |
| `@deepseek-ai/dsh`（CLI） | **≥ 0.1.0-rc.5**，实测 **0.1.0-rc.6** | 0.1.0-rc.5 起随 CLI 附带 `@deepseek-ai/dsh-mcp-client` |
| `@deepseek-ai/dsh-mcp-client` | `^0.1.0-rc.6`（由 CLI 解析） | 无需声明依赖 |
| Exa MCP 端点 | server 3.2.1（2026-08-14 实测） | Exa 托管，可能变化；以 `tools/list` 返回为准 |
| MCP 协议 | 2025-06-18 | 自动协商 |
| Node.js | 实测 v24.16.0；建议 ≥ 22 | dsh 无 engines 声明 |
| 平台 | Windows / macOS / Linux | 纯配置，无平台代码 |

- dsh 处于 developer preview，可能发生破坏性变更；升级 dsh 后建议重跑 [SOLUTIONS.md](SOLUTIONS.md) 的验证清单
- 本插件版本记录于 GitHub Releases 标签，与 `package.json` 的 `version` 一致

## 8. 相关链接

- 官方仓库：<https://github.com/deepseek-ai/deepseek-harness>
- 本插件文档：[术语表](GLOSSARY.md) · [API 列表](API.md) · [解决方案](SOLUTIONS.md)
- Exa MCP 官方文档：<https://exa.ai/docs/reference/exa-mcp>
- Exa MCP 服务端源码：<https://github.com/exa-labs/exa-mcp-server>

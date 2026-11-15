# dsh-exa-mcp 工程文档

> 面向 DeepSeek Harness（`dsh`）的第三方 Exa 搜索 bundle。当前版本 **0.2.0**，按 **dsh 0.2.2-rc.3** 实测整理。本文描述工程结构、规范符合性、验证方法与发布流程；配置字段与工具接口见 [API.md](API.md)，坑与验证细节见 [SOLUTIONS.md](SOLUTIONS.md)。

## 1. 项目定位

- **类型**：第三方插件 bundle，纯配置，无运行时代码；附带插件管理器使用的 locale 与图标。
- **功能**：通过 dsh 自带的 `@deepseek-ai/dsh-mcp-client` 连接 Exa 托管 MCP 端点（`https://mcp.exa.ai/mcp`，Streamable HTTP），把 Exa 工具注册为 `mcp__exa__*`。
- **不做什么**：不在 dsh 进程内运行第三方代码；不接管 Exa 账号、计费或鉴权；不实现 OAuth；不修改 dsh 安装目录。

## 2. 目录结构

```text
dsh-exa-mcp/
├── package.json          # npm 清单 + dsh.bundle.patch
├── cordis.patch.yml      # 唯一 patch 层：插入 mcp-exa 行
├── locale/en.json        # 插件管理器显示名/描述（英文）
├── locale/zh.json        # 插件管理器显示名/描述（中文）
├── icon.svg              # 36×36 透明 SVG 图标
├── README.md             # 英文使用说明（插件详情页的回退文档）
├── README.zh.md          # 中文使用说明
├── LICENSE               # MIT
└── docs/
    ├── PROJECT.md        # 本文档
    ├── GLOSSARY.md       # 标准术语表
    ├── API.md            # 配置、工具与运行时接口
    └── SOLUTIONS.md      # 坑、疑难与验证方法论
```

`package.json` 的 `files` 只打包运行时需要的文件：patch、locale、icon、README 与 LICENSE；`docs/` 随仓库分发，不进入用户运行环境。插件管理器未声明 `meta.detail`，详情页回退到 README。

## 3. 核心机制

`cordis.patch.yml` 的全部内容是一个 `insert` 块，插入 id 为 `mcp-exa` 的 mcp-client 实例：

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
        failOnStartupError: false
```

要点：

| 字段 | 说明 |
|---|---|
| `name` | 解析到 dsh 随附的 `@deepseek-ai/dsh-mcp-client`，无需声明依赖 |
| `serverName: exa` | 工具命名空间，决定 `mcp__exa__*` |
| `transport/url` | Exa 托管端点使用 Streamable HTTP |
| `headers` | 加载时求值：有 `EXA_API_KEY` → `x-api-key`；无 → `{}`（匿名），不发送 `undefined` |
| `toolCallTimeoutMs` | 180 s，适配长耗时的研究调用 |
| `failOnStartupError: false` | Exa 故障或离线不拒绝插件激活；桥记录错误并重连 |

`failOnStartupError` 设为 `true` 会让初始连接失败直接拒绝激活，便于把“没有 Exa 工具”当故障处理；本插件选择可降级启动。

Web/Desktop profile 还会挂载 `@deepseek-ai/dsh-mcp-manager`，用于在设置中运行时增删 MCP server。固定 Exa 端点的部署仍适合普通 `mcp-client` 行；若同时在管理器里添加 Exa，必须使用不同的 `serverName`，否则同一作用域内命名冲突。

## 4. 插件规范与显示资源

| 规范点 | 本工程做法 |
|---|---|
| bundle 清单 | `package.json` 声明 `dsh.bundle.patch: "./cordis.patch.yml"` |
| patch 方言 | 顶层数组 + `insert`；按 id 覆盖时不再写 `insert` |
| `!!js` | 只用于插件 `config`（本工程仅 `headers`）；entry 元数据保持静态 |
| 按 id 覆盖 | 目标 id `mcp-exa`；`config` 整体替换，不深合并 |
| 工具命名 | `mcp__<serverName>__<rawName>` |
| 显示元数据 | `locale/en.json`、`locale/zh.json` 的 `meta.title` / `meta.description`；`icon.svg` 通过 `./icon` 导出 |
| 详情文档 | 不声明 `meta.detail`，使用 README 回退；未附带预览图，避免为详情页引入无实际内容的资源 |
| 兼容声明 | 不声明 `engines.dsh`。本包设计上兼容 0.1.x 与 0.2.x 的配置契约，本次实测 0.2.2-rc.3；不写范围可避免 prerelease 版本被范围误挡。若将来声明，DSH 会先读 `engines.dsh` 再回退到 DSH peer 依赖 |

图标为原创 SVG：36×36 viewBox、透明背景、无 `<text>`、无脚本、无外链，符合插件管理器的图标格式与体积限制。

## 5. 开发与验证

无需构建。常用检查：

```sh
dsh --version
dsh --profile web --dump-config | grep -A8 "== dsh-exa-mcp"
npm pack --dry-run          # 检查实际打包文件
```

本地联调（未安装 bundle 时）：

```sh
dsh web --patch "$PWD/cordis.patch.yml"
```

验证分层：

| 层 | 手段 | 证明什么 |
|---|---|---|
| 清单与资源 | JSON 解析、SVG 结构检查、`npm pack --dry-run` | 清单、locale、icon、`files` 正确 |
| patch 语法 | YAML 解析；`dsh --dump-config` | 行完整、`!!js` 保留、按 id 合成 |
| 端点 | MCP `initialize` + `tools/list` | 匿名工具集与服务器版本 |
| 启动 | `dsh --profile web`（隔离 `DSH_HOME`） | mcp-client 连接与工具注册 |
| 行为 | 会话日志中的 `tool/call` / `tool/result` | 真实调用、参数与错误路径 |
| 生命周期 | 插件管理器安装/停用/卸载/重装 | 可逆性与残留 |

界面渲染（插件详情页、图标、中英文标题）需要浏览器控制；无控制时只能核对资源与导出，不能宣称已渲染。

## 6. 发布

```sh
npm pack                    # 生成 dsh-exa-mcp-0.2.0.tgz
npm publish                 # 发布到 npm registry
```

或从 GitHub 安装：

```sh
dsh plugin --profile web add github:MicroHEROX/dsh-exa-mcp
```

发布时必须提交仓库全部文件；npm/pnpm 只按 `files` 打包，`docs/` 不会安装到用户环境。若替换已安装的包，插件管理器可能返回 `restart-required`；按返回状态处理，不要仅凭 HMR 推断新包已生效。

## 7. 版本兼容

| 组件 | 版本 | 说明 |
|---|---|---|
| `dsh-exa-mcp` | **0.2.0** | 语义化版本；破坏性变更走 minor 升版 |
| DeepSeek Harness CLI | 设计兼容 `>= 0.1.0-rc.5`；**实测 0.2.2-rc.3** | 0.1.0-rc.5 起随 CLI 附带 mcp-client |
| `@deepseek-ai/dsh-mcp-client` | 由 DSH 解析；实测 **0.2.2-rc.3** | 本插件使用的字段在该版本仍受支持 |
| Exa MCP 端点 | server **3.5.1**（2026-11-16 探测） | Exa 托管，可能变化；以 `tools/list` 为准 |
| MCP 协议 | `2025-06-18`（探测值） | 自动协商 |
| Node.js | DSH 要求 **≥ 24.9**；桌面实测 24.21.0 | 插件本身无 `engines` 声明 |
| 平台 | Windows / macOS / Linux | 纯配置，无平台代码 |

dsh 处于 developer preview。升级 dsh 后建议重跑 [SOLUTIONS.md §4](SOLUTIONS.md) 的验证矩阵；跨版本升级前先读运行安装目录内的 `docs/upgrade-guide/README.md`。

## 8. 相关链接

- 本插件：[README](../README.md) · [术语表](GLOSSARY.md) · [API](API.md) · [解决方案](SOLUTIONS.md)
- DSH 仓库：<https://github.com/deepseek-ai/deepseek-harness>
- DSH 发布：<https://github.com/deepseek-ai/deepseek-harness/releases>
- mcp-client 文档：<https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/mcp/mcp-client>
- Exa MCP 文档：<https://exa.ai/docs/reference/exa-mcp>
- Exa MCP 服务端：<https://github.com/exa-labs/exa-mcp-server>

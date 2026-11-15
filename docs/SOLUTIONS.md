# 解决方案与坑

> 本工程的坑、疑难与验证方法。每条按「现象 → 原因 → 解决」组织；版本基线为 **dsh 0.2.2-rc.3 / Exa server 3.5.1 / Windows**（2026-11-16）。跨版本使用时先确认对应版本的 mcp-client schema 与 Exa `tools/list`。

## 一、配置与加载

### 1.1 `!!js` 只能用于 `config`（及 `disabled`）

- **现象**：把 `!!js` 写在 entry 的 `name`、`id` 等元数据上，表达式不生效或被当成普通值。
- **原因**：Loader 只对插件 `config` 和 `disabled` 求值；其他 entry 元数据保持字面量。
- **解决**：表达式只放 `config` 值；条件启用/禁用用 `disabled: !!js '...'`，不要在 `config` 外做动态拼接。

### 1.2 bundle 已安装又叠加同一 `--patch` → 启动失败

- **现象**：`duplicate loader entry id: mcp-exa`，profile 起不来。
- **原因**：bundle 层已插入 `mcp-exa`，overlay 又插入同 id；dsh 对重复 id fail loud。
- **解决**：二者取其一。已装 bundle 就直接 `dsh --profile <name>`；不装才用 `--patch`。按 id 覆盖时不写 `insert`。

### 1.3 `EXA_API_KEY` 未设置时发送坏 header

- **现象**：Exa 返回鉴权错误，或请求带字面 `"undefined"`。
- **原因**：`headers: { x-api-key: !!js 'process.env.EXA_API_KEY' }` 在变量未设置时求值为 `undefined`，被序列化成字符串。
- **解决**：用条件表达式返回空对象：

```yaml
headers: !!js 'process.env.EXA_API_KEY ? { "x-api-key": process.env.EXA_API_KEY } : {}'
```

匿名回退仍可用（免费额度、限流）。验证方法见 §4.3。

### 1.4 工具集、鉴权与 `?tools=` 白名单

- **现象**：① 匿名 `tools/list` 只有 2 个工具；② 设置 `EXA_API_KEY` 后默认工具集不变；③ 匿名看不到 advanced；④ 加了 `?tools=` 反而丢掉基础工具。
- **原因**：Exa 的默认工具集与鉴权解耦；可选工具必须用 URL `?tools=` 显式启用，且该参数是**替换**语义，不是追加。
- **解决**：
  - 默认：`web_search_exa`、`web_fetch_exa`（匿名可用）。
  - 高级搜索：`?tools=web_search_exa,web_fetch_exa,web_search_advanced_exa`（匿名即可用）。
  - Agent：`?tools=web_search_exa,web_fetch_exa,agent_run` + API Key；匿名调用报 `-32000 Authentication required`。
  - 覆盖 `mcp-exa` 的 `url` 时把需要的基础工具一起列出。

### 1.5 `serverName` 冲突

- **现象**：`serverName "exa" is already in use by another mcp-client instance`。
- **原因**：同一注册作用域内两个 mcp-client 实例占用同一命名空间；0.2.x 的 `mcp-manager` 里添加同名 server 也会冲突。
- **解决**：给其中一个改 `serverName`；若用 `mcp-manager` 管理 Exa，保留本插件的 `mcp-exa` 行时不要再添加同名 server，或卸载本 bundle 后统一由管理器管理。

### 1.6 `dsh plugin` 需要 pnpm

- **现象**：`dsh plugin ...` 报 pnpm 找不到。
- **原因**：`dsh plugin` 把参数转发给 pnpm。
- **解决**：`npm install -g pnpm`；或用 `--patch` / 手动合并 patch 的免安装路线。`--profile` 必须放在转发的 pnpm 参数之前。

### 1.7 `github:` 安装偶发不生效或卸载残留

- **现象**：网络不稳时 `dsh plugin add github:MicroHEROX/dsh-exa-mcp` 可能安装失败，或安装完成但 `dsh.profile.bundles` 未追加；`remove` 后可能残留悬空 bundle 引用，启动报 `cannot resolve profile bundle`。
- **原因**：git 依赖的 HEAD 探测/checkout 与 `reconcilePlugins` 存在时序窗口；不是插件配置错误。本地 `link:`/`file:` 安装不受影响。
- **解决**：网络不稳时优先本地路径或 `--patch`；安装后跑 `dsh --profile <name> --dump-config | grep "== dsh-exa-mcp"` 确认；卸载后检查 `dsh.profile.bundles` 并移除残留项。该问题已反馈官方（[Discussions #656](https://github.com/deepseek-ai/deepseek-harness/discussions/656)）。

## 二、运行时与调用

### 2.1 搜索返回 429

- **现象**：`HTTP 429` 或免费额度限流。
- **原因**：匿名档限流。
- **解决**：设置 `EXA_API_KEY` 后重启 dsh；申请地址 <https://dashboard.exa.ai/api-keys>。

### 2.2 超时：区分工具调用与启动

- **现象**：长研究任务报工具超时；或启动时连接 Exa 很慢。
- **原因**：`toolCallTimeoutMs` 只约束单次 `tools/call`；启动连接/发现另有预算。
- **解决**：
  - 工具调用：本插件已设 `toolCallTimeoutMs: 180000`；仍不够按 id 覆盖。
  - 启动：0.2.x 可选 `startupTimeoutMs`；未设时用 SDK 默认。不要用 `toolCallTimeoutMs` 解决启动慢。
  - 关闭：`closeTimeoutMs` 默认 5000，通常不需要改。

### 2.3 MCP 错误透传（-32602 / isError）

- **现象**：工具结果 `isError: true`，内容是 Exa 的校验/鉴权错误。
- **原因**：mcp-client 把 MCP 错误原样经注册表错误路径返回；这是正确行为。
- **解决**：无需吞错；错误文本通常已含可操作细节（如 `urls expected array`、缺 `objective`）。

### 2.4 OAuth 不可用

- **现象**：想用 `https://mcp.exa.ai/mcp?login` 走登录流程。
- **原因**：dsh mcp-client 不实现 OAuth 握手。
- **解决**：改用 API Key（`x-api-key`）；OAuth 只在 Exa 自己的客户端里可用。

### 2.5 Exa 工具 schema 会漂移（当前必填 `objective`）

- **现象**：`web_search_exa` 调用报 `-32602`，提示缺参数；旧文档只写了 `query`。
- **原因**：Exa 端点升级后，`web_search_exa` 的必填字段变为 `query` + `objective`（2026-11-16 的 server 3.5.1 实测）。
- **解决**：以实时 `tools/list` 为准；调用时同时给 `query` 和 `objective`。升级 Exa 服务端后重跑 §4.1 的 schema 检查，不要只信旧文档。

## 三、Windows 工程环境

### 3.1 PowerShell 写 JSON 的 BOM 问题

- **现象**：手动改 profile 的 `package.json` 后 dsh 报 `SyntaxError: Unexpected token '﻿'`。
- **原因**：Windows PowerShell 5.1 的 `ConvertTo-Json | Set-Content -Encoding UTF8` 写 BOM；dsh 读取 JSON 不做 BOM 剥离。
- **解决**：用 Node 写无 BOM UTF-8：

```sh
node -e "const fs=require('fs');const p=process.env.DSH_HOME+'/profiles/<name>/package.json';const j=JSON.parse(fs.readFileSync(p,'utf8'));fs.writeFileSync(p,JSON.stringify(j,null,2)+'\n','utf8')"
```

### 3.2 端口占用

- **现象**：启动 web profile 报 `EADDRINUSE: 127.0.0.1:<port>`。
- **原因**：已有 dsh 或其他进程占用端口。
- **解决**：`Get-NetTCPConnection -LocalPort <port>` 找 pid；确认归属后停止。测试优先用隔离 `DSH_HOME` 和 `--port 0`，不要杀正在使用的实例。

### 3.3 卸载后的包管理器残留

- **现象**：`remove` 后 profile 仍留有 `node_modules` junction 或 `pnpm-lock.yaml`；声明式状态已干净。
- **原因**：`link:`/`file:` 与 pnpm 的包管理器产物不由插件管理器完全清理。
- **解决**：这些残留不会加载插件；要逐字节还原，先在改动前快照 profile 文件，再按快照恢复。手工清理前先核对归属。

## 四、验证与取证

### 4.1 分层验证矩阵

| 层 | 手段 | 证明什么 |
|---|---|---|
| 清单/资源 | JSON 解析、SVG 结构、`npm pack --dry-run` | locale/icon/`files` 正确 |
| 配置契约 | YAML 解析；`dsh --profile <p> --dump-config` | 行完整、`!!js` 保留、按 id 合成 |
| 端点 | MCP `initialize` + `tools/list` | 匿名工具集、serverInfo 版本 |
| 启动 | 隔离 `DSH_HOME` 启动 web/headless | 连接与工具注册 |
| 行为 | 会话日志 `tool/call` / `tool/result` | 真实调用、参数、错误路径 |
| 生命周期 | 安装/停用/卸载/重装 | 可逆性与残留 |

### 4.2 会话日志与导出

当前会话文件为 `session.v4.jsonl.zstd`，是多帧 zstd；不同 Node 版本的解压行为不完全一致。优先用 Web UI 的「Session log」导出明文 zip，或用明确支持多帧的 zstd 读取器。会话日志是 `tool/call` / `tool/result` 的权威记录，但字段随 session format 版本变化，断言前先看实际结构。

### 4.3 A/B 鉴权测试

- 匿名：不设 `EXA_API_KEY`，搜索/抓取成功。
- 伪 key：设置一个明显无效的 key，Exa 返回 `401 Invalid API key`——证明 header 确实送达并被使用。
- 恢复：清掉临时环境变量，重启后回到匿名档。不要在测试脚本或文档中写入真实 key。

### 4.4 隔离原则

- 测试用临时 `DSH_HOME`，不碰真实 `~/.dsh`（含凭据、会话）。
- 不在 patch/文档中写真实 key；key 只经环境变量注入。
- 清理前核对目录归属与时间戳；不要删除来源不明的 profile/session 数据。

## 五、官方文档索引

| 主题 | 位置 |
|---|---|
| DSH 仓库/发布 | <https://github.com/deepseek-ai/deepseek-harness> · <https://github.com/deepseek-ai/deepseek-harness/releases> |
| 当前安装的升级指南 | 运行安装目录内 `@deepseek-ai/dsh/docs/upgrade-guide/README.md` |
| mcp-client | <https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/mcp/mcp-client> |
| mcp-manager | <https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/mcp/mcp-manager> |
| Cordis 入门 | <https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md> |
| 插件打包/安装 | <https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.md> |
| Exa MCP 文档 | <https://exa.ai/docs/reference/exa-mcp> |
| Exa MCP 服务端 | <https://github.com/exa-labs/exa-mcp-server> |
| MCP 规范 | <https://modelcontextprotocol.io/specification> |

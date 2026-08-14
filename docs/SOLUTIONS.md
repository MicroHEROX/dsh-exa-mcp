# 解决方案文档

> 本工程的坑（pitfalls）、疑难问题、解决方法与验证方法论。每条含「现象 → 原因 → 解决」与对应官方/本仓库文档地址（如适用）。所有条目均经真实环境验证（dsh 0.1.0-rc.6）。

## 一、配置与加载类

### 1.1 `!!js` 表达式求值位置受限（关键坑）

- **现象**：`!!js` 写在插件 `config` 之外（如 entry 元数据）不生效或行为异常
- **原因**：dsh 的 Loader 只对 `config` 字段（及 `disabled`）插值表达式；历史上 `disabled` 也出现过"表达式对象被当真值"的事故
- **解决**：`!!js` 只用于 `config` 值；条件性启用/禁用请用 overlay 或 `disabled: !!js '...'`
- **官方文档**：[cordis-primer.md#loader-configuration](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md) · 事故复盘：[postmortem 0002](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/postmortem/0002-js-expression-disabled-filesystem-tools.md)

### 1.2 bundle 已安装又叠加 `--patch` → 启动失败

- **现象**：`dsh --profile <p> --patch <bundle>/cordis.patch.yml` 报 `duplicate loader entry id: mcp-exa`
- **原因**：bundle 层已插入 `mcp-exa`，overlay 再插入同名行；dsh 对重复 id fail loud（防重复注册）
- **解决**：二者取其一——装了 bundle 就直接 `dsh --profile <p>`；不装就用 `--patch`
- **实测**：dsh 0.1.0-rc.6 复现并确认

### 1.3 EXA_API_KEY 未设置时发坏 header

- **现象**：`headers: { x-api-key: !!js 'process.env.EXA_API_KEY' }` 在未设变量时发送字面 `"undefined"`
- **原因**：JS 表达式求值后为 `undefined`，被序列化成字符串头
- **解决**：条件表达式 `!!js 'process.env.EXA_API_KEY ? { "x-api-key": process.env.EXA_API_KEY } : {}'` —— 无 key 时空对象（匿名免费额度），有 key 自动附头
- **验证**：A/B 实测——匿名搜索成功；伪 key 被 Exa 拒为 `401 Invalid API key`（证明 header 送达且被使用）

### 1.4 工具集与鉴权/白名单的关系（匿名、有 key、白名单三者）

- **现象**：① 匿名 `tools/list` 只有 2 个工具；② 设置 `EXA_API_KEY` 后工具集**不变**；③ 匿名下 advanced 也**不可见**
- **原因**：Exa 服务端对"默认工具集"的界定与鉴权**解耦**——可选工具必须经 URL `?tools=` 白名单显式启用（实测：`?tools=...,web_search_advanced_exa` 后 3 工具；`?tools=...,agent_run` 后 agent_run 出现）
- **解决**：覆盖 `mcp-exa` 行的 `url` 加上所需白名单；`agent_run` 还需 API key（匿名白名单调用报 `-32000 Authentication required`）
- **实测**（2026-08-14，真实 API key）：匿名与带 key 默认均 2 工具；**advanced 匿名即可用**（白名单即启用）；agent_run 需 key + 白名单
- **官方文档**：[Exa MCP 文档](https://exa.ai/docs/reference/exa-mcp)（"Tool enablement (optional)" 一节）

### 1.5 `serverName` 冲突

- **现象**：报 `serverName "exa" is already in use by another mcp-client instance`
- **原因**：同一进程内两个 mcp-client 实例占用同一命名空间
- **解决**：改名其中一个实例的 `serverName`
- **官方文档**：[mcp-client README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md)

### 1.6 安装/卸载需要 pnpm

- **现象**：`dsh plugin ...` 报 pnpm 找不到
- **原因**：`dsh plugin` 转发给 pnpm 执行
- **解决**：`npm install -g pnpm`；或走 `--patch` / 合并 profile patch 的免安装路线
- **官方文档**：[publish.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.md)

### 1.7 `dsh plugin add`（github: 协议）偶发不生效——网络抖动时序

- **现象**：`dsh plugin --profile <p> add github:MicroHEROX/dsh-exa-mcp` 偶发：依赖未写入、或已写入但 `dsh.profile.bundles` 未追加；`remove` 后偶发 bundles 残留悬空引用导致 `cannot resolve profile bundle` 启动失败
- **根因（实测定位）**：**不是插件问题，也不是 CLI 稳定 bug**——github.com 网络抖动（ECONNRESET/ETIMEDOUT 的 git HEAD 探测重试）窗口内：① pnpm 安装失败（`dsh` 只在 stderr 打一行 `pnpm failed`，易被忽略）；② pnpm 输出 Done 但 git checkout 异步落地，`reconcilePlugins` 跑在空 node_modules 上 → 不追加，此后不再补偿；③ 中途失败回滚 manifest 依赖项。网络正常时（实测 5s 完成）add/remove/reconcile **全流程正确**
- **解决**：① 网络不稳时优先 `--patch` overlay 或本地路径安装（`link:`/`file:` 无网络依赖，始终正常）；② github: 安装后按 README 提供的一行命令**验证并修复** manifest；③ 已上报官方建议（Discussions #656：reconcile 等待 git checkout 落地、pnpm 失败提示更醒目）
- **附带**：手动维护 manifest 的 BOM 问题见 1.8

### 1.8 PowerShell `ConvertTo-Json` 写 UTF-8 BOM 破坏 dsh JSON

- **现象**：手动改 profile `package.json` 后 `dsh` 报 `SyntaxError: Unexpected token '﻿'`
- **原因**：PS 5.1 `ConvertTo-Json | Set-Content -Encoding UTF8` 写 BOM；dsh 的 JSON 读取不做 BOM 剥离
- **解决**：用 node `fs.writeFileSync(p, JSON.stringify(j,null,2)+'\n', 'utf8')` 无 BOM 写入
- **备注**：仅影响手工维护 profile manifest 的工程操作，不影响 `dsh plugin` 本身

## 二、运行时与调用类

### 2.1 搜索报 429

- **现象**：`HTTP 429` / 免费额度限流提示
- **原因**：匿名层限流
- **解决**：设置 `EXA_API_KEY`（[申请地址](https://dashboard.exa.ai/api-keys)）
- **官方文档**：[Exa MCP 文档](https://exa.ai/docs/reference/exa-mcp)

### 2.2 工具调用超时（agent 研究类任务）

- **现象**：`toolCallTimeoutMs` 到期中止
- **原因**：默认 60 s 对多步 agent 研究不够
- **解决**：本插件默认 180000；仍不够按 id 覆盖 `toolCallTimeoutMs`
- **官方文档**：[mcp-client README#Config](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md)

### 2.3 MCP 错误透传（-32602 / isError）

- **现象**：`tool/result` 携带 `isError: true`，内容为 Exa 的校验/鉴权错误原文
- **原因**：mcp-client 把 MCP 错误原样经注册表错误路径返回（**正确行为**，曾用于发现 mock 参数名错误）
- **解决**：无需处理；错误信息已含可操作细节（如 `urls expected array`）
- **官方文档**：[mcp-client README#Behavior](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md)

### 2.4 OAuth 不可用

- **现象**：想用 `mcp.exa.ai/mcp?login` 登录流程
- **原因**：dsh mcp-client 桥不实现 OAuth 握手
- **解决**：改用 API key（`x-api-key` header）
- **官方文档**：[mcp-client README#Known-Limitations](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md)

### 2.5 Web UI「Session log」面板会触发会话导出下载

- **现象**：点击 Web UI 顶部的 **Session log** 按钮后，浏览器自动下载 `dsh-session-session-<uuid>.zip` 到下载目录
- **原因**：dsh 自带的会话导出功能（导出当前会话为 zip，内含**明文** `session.jsonl`），非异常行为
- **解决/利用**：无需处理；导出物是明文 jsonl，可用于快速核对会话内容（免去多帧 zstd 解压）
- **实测**：dsh 0.1.0-rc.6 Web UI，导出内容与落盘会话日志一致，无敏感数据（key 零泄漏）

## 三、工程环境类（Windows 开发坑）

### 3.1 PowerShell 5.1 读 UTF-8 无 BOM 脚本乱码

- **现象**：`.ps1` 里含中文路径时路径失效（`WorkingDirectory 无效` / ENOENT）
- **原因**：PS 5.1 按 ANSI 读取无 BOM 的 UTF-8 文件
- **解决**：脚本内避免中文路径（用 ASCII 副本），或保存为带 BOM 的 UTF-8
- **备注**：dsh 本体与插件不受影响，仅影响测试脚本编写

### 3.2 端口占用（EADDRINUSE: 127.0.0.1:3080）

- **现象**：启动 web profile 失败
- **原因**：已有 dsh 实例占用 3080（或测试残留进程）
- **解决**：`Get-NetTCPConnection -LocalPort 3080` 找到 pid；普通权限 `Stop-Process`，管理员进程需 UAC 提权 `taskkill /F /PID`
- **注意**：杀死他人/正在使用的实例前先确认（本工程测试一律使用隔离 DSH_HOME + 临时 CLI）

### 3.3 GitHub 安装与文档分发

- **机制**：pnpm 的 git 依赖按 npm 打包规则安装——只打包 `package.json` 的 `files` 字段列出的文件；`docs/` 不在其中，因此**不会进入运行环境**（实测 `node_modules/dsh-exa-mcp/` 仅含 `cordis.patch.yml` 与包文件）
- **要点**：`files` 保持最小集（**不要**加入 `docs/`）；文档随 git 仓库完整分发
- **详见**：[PROJECT.md §6](PROJECT.md)

### 3.4 `profiles/node_modules` 等残留

- **现象**：测试后 `$DSH_HOME` 出现模板 profile / node_modules / 空会话目录
- **原因**：profile 初始化（initProfile）会自动生成模板文件与会话
- **解决**：正式验证使用隔离 `DSH_HOME`（临时目录）；真实 home 只做用户级操作；清理前先核对内容归属（解压会话日志确认）

## 四、验证方法论

### 4.1 分层验证矩阵（本插件实测流程）

| 层 | 手段 | 证明什么 |
|---|---|---|
| 配置契约 | 用官方 `@deepseek-ai/cordis-plugin-include` 的 `entryListSchema` 解析 patch；用发布版 `dsh-mcp-client` 的 `Config` schema 校验 | YAML 方言、config 合法性 |
| 表达式 | 复刻 Loader 求值（`with(ctx){eval(expr)}`）双路径断言 | `EXA_API_KEY` 有无 → headers 正确 |
| 合成 | `dsh --profile <p> --patch ... --dump-config` | bundle 层合成、行完整、`!!js` 保留 |
| 启动 | web/headless 真实 boot | mcp-client `await connection.ready` 通过 ⇒ 连接+发现成功 |
| 行为 | mock LLM + 会话日志断言 | 工具注册名、真实调用、结果/错误路径 |
| 生命周期 | `dsh plugin add / remove / add` | 安装、卸载、重装幂等 |

### 4.2 行为测试的关键技巧

- **mock LLM**（OpenAI 兼容 SSE，本地端口）：无 API key 也能驱动真实 agent 循环；mock 记录请求 `tools` 数组 → 直接断言模型可见工具集与 schema；有状态 mock（search → fetch → 总结）验证多步调用
- **会话日志即真相**：`tool/call`（`data.name`/`data.arguments`）、`tool/result`（文本 + `isError`）结构化断言
- **注意**：`session.jsonl.zstd` 是**多帧** zstd——Node `zstdDecompressSync` 只解第一帧；用 python `zstandard` 的 `stream_reader` 解全部帧（本工程踩过）；或经 Web UI「Session log」导出明文 jsonl（见 2.5）
- **A/B 鉴权测试**：伪 key → Exa `401 Invalid API key`，与匿名成功对照，证明 header 路径
- **web UI 全交互**（WebBridge 驱动真实浏览器）：选工作区 → 发消息 → UI 消息流出现 `Tool call mcp__exa__*` 卡片（含参数）→ 回复与统计（轮/步/耗时）→ 落盘会话日志与 mock 请求双重印证；UI 的"选择工作区"输入框是 `readOnly`，需先经目录选择器
- **`agent_run` 实测要点**：需 API key + `?tools=` 白名单；返回 `runId`（`agent_run_*`）支持续跑；`outputSchema` 可约束输出（`structured` + `grounding` 引用回传）；匿名白名单调用报 `-32000 Authentication required`；实调用按用量计费（本工程仅跑 1 次最小任务，12.1s 完成）

### 4.3 安全与隔离原则

- 测试一律使用**隔离 DSH_HOME** + 临时安装的 CLI，不触碰真实 `~/.dsh`（含凭据、会话）
- 不在 patch/文档中写入真实 key；key 只经环境变量注入
- 清理残留前核对归属（会话/存储内容解压确认，时间戳交叉验证）

## 五、官方文档索引（本文档引用）

| 主题 | 地址 |
|---|---|
| dsh 总览/安装 | <https://github.com/deepseek-ai/deepseek-harness#readme> |
| 架构 | <https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md> |
| Cordis 入门 | <https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md> |
| 插件配置 | <https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/config.md> |
| 打包与安装（bundle/profile） | <https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.md> |
| 工具开发 | <https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/tool.md> |
| 工具注册表子系统 | <https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/tools.md> |
| mcp-client 桥 | <https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md> |
| 凭据子系统 | <https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/credentials.md> |
| 配置目录（生成） | <https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/config-catalog.md> |
| CLI 参考 | <https://github.com/deepseek-ai/deepseek-harness/blob/master/apps/cli/reference/README.md> |
| `!!js` 事故复盘 | <https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/postmortem/0002-js-expression-disabled-filesystem-tools.md> |
| Exa MCP 官方 | <https://exa.ai/docs/reference/exa-mcp> |
| Exa MCP 服务端源码 | <https://github.com/exa-labs/exa-mcp-server> |

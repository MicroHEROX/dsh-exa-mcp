# Exa Search MCP for DeepSeek Harness

<p align="center">
  <a href="#english"><b>English</b></a> ·
  <a href="README.zh.md"><b>中文</b></a>
</p>

<p align="center">
  <a href="https://github.com/MicroHEROX/dsh-exa-mcp/blob/main/LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/license-MIT-blue.svg"></a>
  <a href="https://github.com/topics/dsh-plugin"><img alt="dsh-plugin" src="https://img.shields.io/badge/dsh-plugin-8A2BE2"></a>
  <a href="https://github.com/MicroHEROX/dsh-exa-mcp"><img alt="stars" src="https://img.shields.io/github/stars/MicroHEROX/dsh-exa-mcp"></a>
</p>

A third-party plugin that brings [Exa](https://exa.ai) — the neural web search and fetch engine — into [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (`dsh`). It connects to Exa's hosted [MCP endpoint](https://mcp.exa.ai/mcp) (Streamable HTTP) through the MCP client bridge that ships with the `dsh` CLI, and registers Exa's tools as native agent tools under the `exa` namespace.

```
mcp__exa__web_search_exa   ·  mcp__exa__web_fetch_exa
mcp__exa__web_search_advanced_exa  ·  mcp__exa__agent_run   (with an API key)
```

- Zero install footprint: no code runs in the dsh process — the upstream is Exa's official hosted endpoint.
- Pure configuration bundle: one patch layer, no build step, no runtime API.
- Never touches your deepseek-harness installation; it only adds one row to the composed `cordis.yml`.

---

## Quick Start

### 1. Install

**Option A — install as a plugin bundle (recommended, requires [pnpm](https://pnpm.io/install)):**

```sh
npm install -g pnpm
dsh plugin --profile web add github:MicroHEROX/dsh-exa-mcp
dsh web
```

**Option B — one-off overlay, no install:**

```sh
dsh web --patch /path/to/dsh-exa-mcp/cordis.patch.yml
```

**Option C — keep it permanently without installing:** merge the single `insert` block from `cordis.patch.yml` into `$DSH_HOME/profiles/<name>/cordis.patch.yml` (or `$DSH_HOME/cordis.patch.yml` for every profile).

> Install behavior: pnpm installs git dependencies through the `files` field, so `docs/` is **not** installed into your runtime — only `cordis.patch.yml` is. The docs live in this repository.

### 2. Provide an Exa API key (optional)

The hosted endpoint works **anonymously** on Exa's free tier (rate-limited, basic tools only). To lift limits and unlock advanced search / Exa Agent, [create an API key](https://dashboard.exa.ai/api-keys) and set it in the environment:

```sh
export EXA_API_KEY="your-key"        # macOS / Linux
$env:EXA_API_KEY = "your-key"        # Windows PowerShell
```

The plugin **auto-detects** the key at load time: `EXA_API_KEY` set → sends `x-api-key`; unset → anonymous. Never put the key into any patch file.

### 3. Verify

1. Start `dsh web` (with the bundle or overlay applied).
2. Wait a moment for initial discovery (it is asynchronous).
3. Ask: *"Use Exa to find the latest release notes of the DeepSeek Harness project on GitHub and summarize them."*
4. Confirm the model called `mcp__exa__web_search_exa` (and `web_fetch_exa` for full text) and answered from the results.

---

## What This Plugin Does

- Connects `https://mcp.exa.ai/mcp` via `@deepseek-ai/dsh-mcp-client` (`streamable-http` transport), the official bridge shipped with the dsh CLI.
- Registers every tool Exa advertises under the spec-compliant name `mcp__exa__<tool>`; re-syncs automatically on `tools/list_changed` notifications.
- Auto-authenticates: attaches `x-api-key` only when `EXA_API_KEY` is present (graceful anonymous fallback — no broken `undefined` header).
- Tunes the bridge for search workloads: `toolCallTimeoutMs: 180000` for long research tasks; reconnect policy left at bridge defaults.
- Follows dsh plugin conventions exactly: bundle manifest (`dsh.bundle.patch`), patch-layer composition, per-id override (`mcp-exa`), `!!js` config expressions only.

## What This Plugin Does NOT Do

- Does **not** download, host, or supervise any Exa server — the upstream is Exa's hosted endpoint.
- Does **not** implement OAuth login (`https://mcp.exa.ai/mcp?login`) — the dsh MCP bridge has no OAuth flow; use an API key.
- Does **not** bridge MCP resources or prompts — the harness only consumes MCP tools.
- Does **not** choose Exa plans, manage billing, or store your API key — the key lives in your environment only.
- Does **not** modify your deepseek-harness installation — installing/uninstalling only touches `$DSH_HOME/profiles/`.

## Routes That Work

| Route | How |
|---|---|
| Anonymous search + fetch | Do nothing — free tier, rate-limited, 2 tools |
| Full tool set (advanced search, Agent) | Set `EXA_API_KEY` + restart; whitelist Agent via `?tools=` (below) |
| Tool whitelist / default search type | Override the `mcp-exa` row's `url` with `?tools=web_search_exa,web_fetch_exa,agent_run` or `?defaultSearchType=fast` (see [docs/API.md](docs/API.md#2-配置接口mcp-exa-行)) |
| Multiple MCP servers | Add more `mcp-client` rows with unique `serverName` values |
| Hot reload | Edit the row in a patch layer — HMR reconnects without process restart |
| Uninstall | `dsh plugin --profile <name> remove dsh-exa-mcp` — profile and base bundles stay intact |

## Routes That Do NOT Work (by design)

| Route | Why not |
|---|---|
| OAuth login flow | dsh mcp-client does not implement the OAuth handshake — use an API key |
| MCP resources / prompts from Exa | The harness bridges tools only |
| Per-request auth switching | The `EXA_API_KEY` decision is made at config evaluation (startup / HMR), not per call |
| Using the same patch twice (bundle installed **and** `--patch`) | dsh fails loud with `duplicate loader entry id: mcp-exa` — pick one method |
| API key in patch files | Keys belong in environment variables; committing them is a leak |

---

## Security

- The only secret involved is `EXA_API_KEY`, read from the environment at load time; it is sent to Exa as the `x-api-key` header and never written to any file by this plugin.
- No third-party code executes inside the dsh process — the plugin is declarative configuration over the official bridge.
- This repository contains no keys, no local paths, and no machine-specific data.

## License

[MIT](LICENSE). Not an official DeepSeek or Exa product.

## Acknowledgements

Built for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) by DeepSeek AI — the "everything is a plugin" harness built on [Cordis](https://github.com/cordiverse/paper). Powered by:

- [Exa MCP Server](https://github.com/exa-labs/exa-mcp-server) — the hosted web search endpoint
- [Model Context Protocol SDK](https://github.com/modelcontextprotocol/modelcontextprotocol) — the protocol this plugin speaks through
- [@deepseek-ai/dsh-mcp-client](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/mcp/mcp-client) — the MCP bridge used by this bundle
- [Cordis](https://github.com/cordiverse/cordis) and its plugin ecosystem — the framework under dsh

Thank you to the DeepSeek Harness team and every open-source project this work builds on.

## Documentation

- [Project documentation](docs/PROJECT.md) · [Glossary](docs/GLOSSARY.md) · [API reference](docs/API.md) · [Solutions & pitfalls](docs/SOLUTIONS.md)
- DeepSeek Harness: <https://github.com/deepseek-ai/deepseek-harness>
- Exa MCP docs: <https://exa.ai/docs/reference/exa-mcp>

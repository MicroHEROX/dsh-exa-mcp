# Exa Search MCP for DeepSeek Harness

<p align="center">
  <a href="README.zh.md"><b>中文</b></a>
</p>

<p align="center">
  <a href="https://github.com/MicroHEROX/dsh-exa-mcp/blob/main/LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/license-MIT-blue.svg"></a>
  <a href="https://github.com/topics/dsh-plugin"><img alt="dsh-plugin" src="https://img.shields.io/badge/dsh-plugin-8A2BE2"></a>
  <a href="https://github.com/MicroHEROX/dsh-exa-mcp"><img alt="stars" src="https://img.shields.io/github/stars/MicroHEROX/dsh-exa-mcp"></a>
</p>

A third-party plugin bundle that brings [Exa](https://exa.ai) — neural web search and full-page fetch — into [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (`dsh`). It mounts Exa's hosted [MCP endpoint](https://mcp.exa.ai/mcp) (Streamable HTTP) through the MCP client bridge shipped with `dsh`, and registers Exa's tools as native agent tools under the `exa` namespace.

```text
mcp__exa__web_search_exa   ·  mcp__exa__web_fetch_exa
mcp__exa__web_search_advanced_exa  ·  mcp__exa__agent_run   (whitelist; agent also needs a key)
```

- **No third-party code in the dsh process** — the upstream is Exa's official hosted endpoint.
- **Pure configuration bundle** — one patch row; no build step, no runtime module.
- **Display-ready** — English/Chinese titles and descriptions plus an original SVG icon for the Plugin Manager.

---

## Quick Start

### 1. Install

**Option A — Plugin Manager (recommended).** Open **Settings → Plugins → Install** and use the package name, a GitHub spec, or a local directory. The CLI equivalent:

```sh
npm install -g pnpm
dsh plugin --profile web add github:MicroHEROX/dsh-exa-mcp
dsh web
```

> If GitHub is unstable, a `github:` install can silently miss the bundle layer. Verify with `dsh --profile web --dump-config | grep -A2 "== dsh-exa-mcp"`; local `link:`/`file:` installs and `--patch` overlays are unaffected. The full repair procedure is in [docs/SOLUTIONS.md §1.7](docs/SOLUTIONS.md#17-github-安装偶发不生效或卸载残留).

**Option B — One-off overlay, no install:**

```sh
dsh web --patch /path/to/dsh-exa-mcp/cordis.patch.yml
```

**Option C — Permanent without a bundle:** merge the single `insert` block from `cordis.patch.yml` into `$DSH_HOME/profiles/<name>/cordis.patch.yml` (or `$DSH_HOME/cordis.patch.yml` for every profile).

Installation packs only `cordis.patch.yml`, `locale/`, `icon.svg`, the READMEs, and LICENSE into your runtime; `docs/` stays in this repository.

### 2. Provide an Exa API key (optional)

The endpoint works **anonymously** on Exa's free tier (rate-limited, basic tools only). For higher limits and advanced tools, [create an API key](https://dashboard.exa.ai/api-keys) and set it before launching dsh:

```sh
export EXA_API_KEY="your-key"        # macOS / Linux
$env:EXA_API_KEY = "your-key"        # Windows PowerShell
```

The plugin attaches `x-api-key` only when the variable is present; otherwise it falls back to anonymous access. Never put the key into a patch file.

### 3. Verify

1. Start `dsh web` with the bundle or overlay applied.
2. Ask: *"Use Exa to find the latest DeepSeek Harness release notes and summarize them."*
3. Confirm the model called `mcp__exa__web_search_exa` (with `query` and `objective`) and, if needed, `mcp__exa__web_fetch_exa`.

---

## What It Does

- Connects `https://mcp.exa.ai/mcp` via `@deepseek-ai/dsh-mcp-client` (`streamable-http`), the official bridge shipped with dsh.
- Registers every tool Exa advertises as `mcp__exa__<tool>`; re-syncs on `tools/list_changed`.
- Attaches `x-api-key` only when `EXA_API_KEY` is set — no `undefined` header, no credential stored on disk.
- Uses `toolCallTimeoutMs: 180000` for long research calls; reconnect behavior stays at bridge defaults.
- Keeps `failOnStartupError: false`: an Exa outage does not reject plugin activation; the bridge logs and reconnects.

## Limits

- Does **not** host, download, or supervise an Exa server.
- Does **not** implement OAuth (`?login`); use an API key.
- Does **not** bridge MCP resources or prompts — tools only.
- Does **not** manage Exa billing or store your key.
- Does **not** modify the dsh installation; install/uninstall only touches `$DSH_HOME/profiles/`.

## Configuration

| Goal | How |
|---|---|
| Anonymous search + fetch | Do nothing; free tier, rate-limited, 2 tools |
| Higher limits | Set `EXA_API_KEY` and restart dsh |
| Enable advanced search | Override `url` with `?tools=web_search_exa,web_fetch_exa,web_search_advanced_exa` |
| Enable Exa Agent | Add `agent_run` to `?tools=` and use an API key; billing applies |
| Change default search type | Add `?defaultSearchType=fast` to `url` |

Override the `mcp-exa` row by id from a later patch layer (no `insert`; `config` is replaced wholesale). Full fields and examples: [docs/API.md §2](docs/API.md#2-配置接口mcp-exa-行).

## Uninstall

```sh
dsh plugin --profile <name> remove dsh-exa-mcp
dsh --profile <name> --dump-config | grep -c "dsh-exa-mcp"   # expect 0
```

For `--patch` overlays, drop the flag. For a merged patch, delete the `mcp-exa` block. Uninstalling never touches the dsh installation or other bundles.

---

## Version Compatibility

| Component | Version | Notes |
|---|---|---|
| `dsh-exa-mcp` | **0.2.0** | See [releases](https://github.com/MicroHEROX/dsh-exa-mcp/releases) |
| DeepSeek Harness CLI | designed for **≥ 0.1.0-rc.5**; tested on **0.2.2-rc.3** | Ships the `@deepseek-ai/dsh-mcp-client` bridge |
| MCP bridge | resolved from dsh; tested **0.2.2-rc.3** | No separate install |
| Exa MCP endpoint | server **3.5.1** (2026-11-16) | Exa-managed; `tools/list` is authoritative |
| MCP protocol | `2025-06-18` (negotiated) | Auto-negotiated |
| Node.js | dsh requires **≥ 24.9** | Plugin itself declares no `engines` |
| Platform | Windows / macOS / Linux | Config-only; no platform code |

dsh is in developer preview. After a dsh upgrade, re-run the checks in [docs/SOLUTIONS.md §4](docs/SOLUTIONS.md#四验证与取证).

## Security

- The only secret is `EXA_API_KEY`, read from the environment at load time and sent to Exa as `x-api-key`; the plugin never writes it to disk.
- No third-party code executes inside the dsh process.
- This repository contains no keys, no local paths, and no machine-specific data.

## Documentation

- [Project documentation](docs/PROJECT.md) · [Glossary](docs/GLOSSARY.md) · [API reference](docs/API.md) · [Solutions & pitfalls](docs/SOLUTIONS.md)
- DeepSeek Harness: <https://github.com/deepseek-ai/deepseek-harness>
- Exa MCP docs: <https://exa.ai/docs/reference/exa-mcp>

## License

[MIT](LICENSE). Not an official DeepSeek or Exa product.

## Acknowledgements

Built for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) by DeepSeek AI — the "everything is a plugin" harness built on [Cordis](https://github.com/cordiverse/paper). Powered by:

- [Exa MCP Server](https://github.com/exa-labs/exa-mcp-server) — the hosted search endpoint
- [Model Context Protocol](https://modelcontextprotocol.io/) — the protocol this bundle speaks
- [@deepseek-ai/dsh-mcp-client](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/mcp/mcp-client) — the MCP bridge
- [Cordis](https://github.com/cordiverse/cordis) and its plugin ecosystem

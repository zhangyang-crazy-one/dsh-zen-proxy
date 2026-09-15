# dsh-zen-proxy

A [DeepSeek Harness (dsh)](https://github.com/deepseek-ai/deepseek-harness) plugin: an in-process, OpenAI-compatible proxy that injects the official OpenCode Zen client headers on upstream requests, enabling **Zen free models** in dsh without the `429 FreeUsageLimitError`.

## Why

OpenCode Zen's free-tier gateway rate-limits third-party clients by fingerprinting HTTP identity headers. Requests that look like the official `opencode` CLI get the normal free quota; anonymous or third-party clients are pinned to a very low fallback bucket and immediately rejected with:

```json
{"type":"FreeUsageLimitError","message":"Rate limit exceeded. Please try again later."}
```

dsh's LLM adapters (e.g. `dsh-llm-pi-ai`) are designed to always send their own attribution `User-Agent` (`deepseek-harness/<version> ...`) and filter any configured `user-agent` out — so no dsh configuration can make the gateway see the official client. This plugin closes that gap with a tiny local proxy that lives inside the dsh process (started/stopped with the profile) and rewrites the identity headers before forwarding to `https://opencode.ai/zen/v1`.

### Verified behavior

Smoke-tested against `https://opencode.ai/zen/v1/chat/completions`:

| Headers sent | Result |
|---|---|
| no official headers | `429` FreeUsageLimitError |
| `x-opencode-*` only | `429` ❌ |
| `x-opencode-*` + official `User-Agent` | `200` ✅ |
| `x-opencode-*` + dsh's own UA | `429` ❌ |

> The gateway validates the `User-Agent` content, not just its presence. The `x-opencode-*` headers alone are not enough.

## Install

```powershell
dsh plugin --profile web add "https://github.com/Yee-h/dsh-zen-proxy.git"
```


Or from a local checkout:

```powershell
dsh plugin --profile web add "file:C:/path/to/dsh-zen-proxy"
```

### TUI (dsh-tui)

The plugin uses only `ctx.effect` and `ctx.logger` with no service
injections, so it mounts unchanged in terminal compositions. [`dsh-tui`](https://github.com/zhangyang-crazy-one/dsh-tui)
bundles this file as `dist/zen-proxy.js` and mounts it in its shipped
`cordis.patch.yml`; point the `opencode` provider at the proxy like below
and it works in the TUI with no extra setup.

```yaml
# dsh-tui cordis.patch.yml
- insert:
    - id: zen-proxy
      name: "./dist/zen-proxy.js"
      config:
        host: 127.0.0.1
        port: 4097
```

## Configure

### 1. Register the plugin (profile patch)

Add to `$DSH_HOME/profiles/<profile>/cordis.patch.yml`:

```yaml
- insert:
    - id: zen-proxy
      name: 'dsh-zen-proxy'
      config:
        host: 127.0.0.1
        port: 4097
        upstreamHost: opencode.ai
        upstreamBasePath: /zen/v1
```

### 2. Point the opencode provider at the proxy

In `$DSH_HOME/settings.yaml`:

```yaml
llm-pi-ai:
  providers:
    opencode:
      apiKeyEnv: OPENCODE_API_KEY
      baseURL: http://127.0.0.1:4097/v1
```

Keep your `sk-zen-*` key in `$DSH_HOME/.credentials.yaml` (or the environment variable named by `apiKeyEnv`).

Restart dsh. The proxy listens on `http://127.0.0.1:4097/v1` for as long as the profile runs and closes automatically on shutdown — no external process, no startup scripts.

## How it works

```
dsh (dsh-llm-pi-ai)
   │  POST http://127.0.0.1:4097/v1/chat/completions
   ▼
dsh-zen-proxy (in-process)
   │  rewrites identity headers, forwards to https://opencode.ai/zen/v1
   ▼
OpenCode Zen gateway → official-client quota → 200
```

Headers injected on every upstream request:

| Header | Value |
|---|---|
| `User-Agent` | `opencode/1.15.5 ai-sdk/provider-utils/4.0.23 runtime/bun/1.3.14` |
| `x-opencode-client` | `cli` |
| `x-opencode-project` | `global` |
| `x-opencode-session` | `ses_` + random (per request) |
| `x-opencode-request` | `msg_` + random (per request) |

The `Authorization` header and the request body (model, messages, `stream: true`, etc.) pass through unchanged; SSE streaming responses are relayed as-is.

## Configuration reference

| Field | Default | Description |
|---|---|---|
| `host` | `127.0.0.1` | Bind address |
| `port` | `4097` | Listen port |
| `upstreamHost` | `opencode.ai` | Zen upstream host |
| `upstreamBasePath` | `/zen/v1` | Zen upstream base path |
| `userAgent` | `opencode/1.15.5 ai-sdk/provider-utils/4.0.23 runtime/bun/1.3.14` | Official CLI user-agent |
| `clientHeader` | `cli` | `x-opencode-client` value |
| `projectHeader` | `global` | `x-opencode-project` value |

## Smoke test

```powershell
$body = '{"model":"deepseek-v4-flash-free","messages":[{"role":"user","content":"hi"}],"max_tokens":16}'
Invoke-RestMethod -Uri "http://127.0.0.1:4097/v1/chat/completions" -Method Post `
  -Headers @{ "Authorization" = "Bearer sk-zen-你的密钥" } `
  -ContentType "application/json" -Body $body
```

Expect HTTP 200 with a real completion.

## Notes & limitations

- **Free-tier quota still applies**: the free allowance is shared per IP + client fingerprint (community reports ~200 requests/day) and is managed dynamically by OpenCode. Headers only restore the official-client tier; they cannot create quota.
- **Occasional 429s can still happen** when the server is overloaded even with correct headers — retry or switch to another free model.
- **Recharging does NOT unlock free models** (known issue): it only unlocks paid models.
- Only the tool-calling HTTP surface (`/v1/chat/completions` and `GET /v1/models`) is proxied; resources and prompts are out of scope.
- This plugin deliberately keeps dsh's own attribution mechanism intact: dsh still sends its real identity to the local proxy; the identity rewrite happens at the network boundary.

## License

MIT

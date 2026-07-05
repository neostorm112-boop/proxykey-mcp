# proxykey MCP server

> Issue, rotate and revoke API-key passes from your AI agent — without the
> agent ever seeing a real key.

**proxykey** is a credential vault and proxy: your real API keys (OpenAI,
Anthropic, Telegram, Stripe, 25+ providers) are encrypted with AES-256-GCM
and never leave the server. Apps and agents use revocable virtual keys —
**passes** (`vlt_…`) — with per-pass IP binding, rate limits and full request
logs. This MCP server lets an agent manage those passes.

- Website: https://proxykey.org/en/
- Panel: https://app.proxykey.org
- Docs: https://app.proxykey.org/docs.html
- MCP endpoint (Streamable HTTP): `https://mcp.proxykey.org/mcp`

## Why give an agent this instead of a key

Anything that enters a model's context can leak — through logs, traces or
prompt injection. With proxykey the agent gets a *tool*, not a secret:

- the agent can **create / rotate / revoke / inspect** passes;
- a **"read the real key" operation does not exist** in the toolset;
- only a human can enter the original key, in the panel;
- a leaked pass is a non-event: wrong IP → 403, over limit → 429,
  revocation is one click and doesn't touch the original.

## Quick start

1. Sign in at https://app.proxykey.org (GitHub OAuth or email magic link, free).
2. Open **MCP** → create a token (`mcp_…`).
3. Connect your client:

**Claude Code**
```bash
claude mcp add --transport http proxykey https://mcp.proxykey.org/mcp \
  --header "Authorization: Bearer mcp_YOUR_TOKEN"
```

**Claude Desktop / Cursor (mcp.json)**
```json
{
  "mcpServers": {
    "proxykey": {
      "url": "https://mcp.proxykey.org/mcp",
      "headers": { "Authorization": "Bearer mcp_YOUR_TOKEN" }
    }
  }
}
```

## Tools (13)

| Tool | What it does |
|---|---|
| `list_providers` | Provider catalogue (slugs, auth models) |
| `list_secrets` | Stored keys (metadata only — never values) |
| `get_manual_secret_setup` | Link for a human to enter a key |
| `list_passes` | All passes with status, limits, binding |
| `create_pass` | Issue a pass for an existing secret |
| `create_pending_pass` | Issue a pass *before* the key exists; a human fills the key in later, the pass activates automatically |
| `update_pass` | Change limits, IP mode, expiry |
| `revoke_pass` / `delete_pass` | Kill a pass instantly / remove a revoked one |
| `rotate_pass` | Reissue the token, same settings |
| `rebind_pass_ip` | Reset IP learning |
| `get_pass_logs` / `get_pass_stats` | Per-pass request log and usage stats |

## The proxy itself

Point your client at the proxy and use the pass as the API key — only the
host and the key change:

```bash
# before
curl https://api.openai.com/v1/chat/completions -H "Authorization: Bearer sk-..."
# after
curl https://api.proxykey.org/p/openai/v1/chat/completions -H "Authorization: Bearer vlt_openai_..."
```

Streaming (SSE), request bodies and headers pass through unchanged.
Telegram bots keep their URL shape (`/p/telegram-bot/<pass>/getMe`).
Full reference: https://app.proxykey.org/docs.html

## Security model (short version)

- Secrets: AES-256-GCM envelope encryption, decrypted in memory per request,
  never logged, never returned by any API after creation.
- Pass tokens and MCP tokens are stored hashed (SHA-256).
- MCP surface cannot create or read secret values, or toggle body logging —
  those stay human-only in the panel.
- Request logs contain metadata only — no auth headers, no key material.

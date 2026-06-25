# HirePlus Claude Plugin

A [Claude Code plugin](https://code.claude.com/docs/en/plugins) that connects Claude to the
**Hyrewyse / HirePlus** recruitment platform. Installing it does two things in one step:

1. **Wires up the MCP server** — Claude connects to the platform's OAuth-protected MCP server and
   gains its recruiting tools (`list_jobs`, `get_job`, `create_job`, `list_resumes`, `create_resume`,
   `upload_resume`, `add_interview_questions`, `get_workflow_status`).
2. **Installs the `hyrewyse-recruiting` skill** — teaches Claude *how* to use those tools well,
   especially the "client-side intelligence" write tools where Claude extracts the structured data
   and scores candidates itself.

## Install

```shell
/plugin marketplace add 91social/hireplus-claude-plugin
/plugin install hireplus@91social
```

Then restart Claude Code and authenticate:

```shell
/mcp                      # select "hyrewyse" → Authenticate
# or, from your shell (Claude Code v2.1.186+):
claude mcp login hyrewyse
```

A browser opens the Hyrewyse consent page — **sign in with your Hyrewyse email and password** and the
tools become available. No Node.js or extra bridge process required — Claude Code talks to the server
natively over HTTP.

## What's inside

```
.claude-plugin/marketplace.json        # marketplace catalog (this repo)
plugins/hireplus/
├── .claude-plugin/plugin.json         # plugin manifest
├── .mcp.json                          # MCP server: native HTTP + static-client OAuth
└── skills/hyrewyse-recruiting/        # the bundled skill (SKILL.md + tool reference)
```

The MCP server is declared as a **native HTTP server with a pre-configured (static) OAuth client**.
The Hyrewyse server uses statically pre-registered OAuth clients (no Dynamic Client Registration), and
Claude Code supports that directly via the `oauth.clientId` + `oauth.callbackPort` fields — no
`mcp-remote` bridge and no Node.js needed:

```json
{
  "mcpServers": {
    "hyrewyse": {
      "type": "http",
      "url": "https://hireplus-5ie3l.ondigitalocean.app/mcp",
      "oauth": {
        "clientId": "hyrewyse-mcp",
        "callbackPort": 6274
      }
    }
  }
}
```

`callbackPort` is fixed to `6274` so the redirect Claude Code uses —
`http://localhost:6274/callback` — exactly matches the redirect registered for the `hyrewyse-mcp`
client on the server (Claude Code's OAuth callback path is always `/callback`).

Equivalent to adding it by hand with:

```shell
claude mcp add --transport http hyrewyse \
  https://hireplus-5ie3l.ondigitalocean.app/mcp \
  --client-id hyrewyse-mcp --callback-port 6274
```

## Server-side prerequisite (one-time, platform operators)

For the OAuth handshake to succeed, the deployed Hyrewyse backend must trust the plugin's client. In
**DigitalOcean App Platform → Settings → Environment Variables**, ensure:

```bash
APP_URL=https://hireplus-5ie3l.ondigitalocean.app
MCP_OAUTH_CLIENTS=[{"clientId":"hyrewyse-mcp","redirectUris":["http://localhost:6274/callback","http://127.0.0.1:6274/callback"],"name":"Hyrewyse MCP Client"},{"clientId":"claude-desktop","redirectUris":["https://claude.ai/api/mcp/auth_callback"],"name":"Claude"}]
```

- `APP_URL` must be the exact public origin — `MCP_PUBLIC_BASE_URL=${APP_URL}`, and if it's empty the
  server audience-binds tokens to `localhost` and Claude gets a 401/audience mismatch.
- `hyrewyse-mcp` (redirect `http://localhost:6274/callback`) is the client this plugin uses. Its
  redirect must match `callbackPort` in `.mcp.json` exactly. `claude-desktop` (redirect
  `https://claude.ai/api/mcp/auth_callback`) is optional — only for adding the server through Claude
  Desktop's Connectors GUI instead of Claude Code.
- Redeploy after changing env vars.

> **Even-cleaner long-term option:** add RFC 7591 Dynamic Client Registration to the backend. Then the
> plugin could drop `oauth.clientId`/`callbackPort` entirely and clients would self-register.

## Updating

Push changes to this repo, then users run:

```shell
/plugin marketplace update 91social
```

## License

MIT

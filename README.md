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

Then restart Claude Code. On first use a browser opens the Hyrewyse consent page — **sign in with your
Hyrewyse email and password** and the tools become available.

> Requires Node.js (the MCP connection runs `npx mcp-remote` under the hood).

## What's inside

```
.claude-plugin/marketplace.json        # marketplace catalog (this repo)
plugins/hireplus/
├── .claude-plugin/plugin.json         # plugin manifest
├── .mcp.json                          # MCP server: mcp-remote bridge → Hyrewyse, static OAuth client
└── skills/hyrewyse-recruiting/        # the bundled skill (SKILL.md + tool reference)
```

The MCP server is declared as a stdio **`mcp-remote` bridge** rather than a native remote URL because
the Hyrewyse server uses statically pre-registered OAuth clients (no Dynamic Client Registration), so
the client id and redirect must be fixed:

```json
{
  "mcpServers": {
    "hyrewyse": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote",
        "https://hireplus-5ie3l.ondigitalocean.app/mcp",
        "--static-oauth-client-info", "{\"client_id\":\"claude-mcp-remote\"}",
        "--port", "3334"
      ]
    }
  }
}
```

## Server-side prerequisite (one-time, platform operators)

For the OAuth handshake to succeed, the deployed Hyrewyse backend must trust the plugin's client. In
**DigitalOcean App Platform → Settings → Environment Variables**, ensure:

```bash
APP_URL=https://hireplus-5ie3l.ondigitalocean.app
MCP_OAUTH_CLIENTS=[{"clientId":"claude-mcp-remote","redirectUris":["http://localhost:3334/oauth/callback","http://127.0.0.1:3334/oauth/callback","http://localhost:3334/callback"],"name":"Claude (mcp-remote)"},{"clientId":"claude-desktop","redirectUris":["https://claude.ai/api/mcp/auth_callback"],"name":"Claude"}]
```

- `APP_URL` must be the exact public origin — `MCP_PUBLIC_BASE_URL=${APP_URL}`, and if it's empty the
  server audience-binds tokens to `localhost` and Claude gets a 401/audience mismatch.
- `claude-mcp-remote` (with the `:3334` redirect) backs this plugin. `claude-desktop` (with the
  `claude.ai/api/mcp/auth_callback` redirect) is for adding the server through Claude Desktop's
  Connectors GUI instead. Keep any other clients (e.g. the MCP Inspector) in the same array.
- Redeploy after changing env vars.

> **Cleaner long-term option:** add RFC 7591 Dynamic Client Registration to the backend. Then this
> plugin could declare the server natively (`{"type":"http","url":".../mcp"}`) with no `mcp-remote`
> bridge, no static client, and no fixed port.

## Updating

Push changes to this repo, then users run:

```shell
/plugin marketplace update 91social
```

## License

MIT

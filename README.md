# HirePlus Claude Plugin

A [Claude Code plugin](https://code.claude.com/docs/en/plugins) that connects Claude to the
**Hyrewyse / HirePlus** recruitment platform. Installing it does two things in one step:

1. **Wires up the MCP server** — Claude connects to the platform's OAuth-protected MCP server and
   gains its recruiting tools (`list_jobs`, `get_job`, `create_job`, `list_resumes`, `create_resume`,
   `upload_resume`, `get_screening_questions`, `get_interview_questions`,
   `generate_screening_questions`, `generate_interview_questions`, `add_interview_questions`,
   `get_workflow_status`).
2. **Installs the recruiting skills and commands:**
   - `hyrewyse-recruiting` — the core playbook: teaches Claude *how* to use the tools well,
     especially the "client-side intelligence" write tools where Claude extracts the structured
     data and scores candidates itself.
   - `generate-screening-questions` — quick recruiter phone-screen questions for a candidate
     (platform-generated or Claude-authored).
   - `generate-interview-questions` — deep round-1 technical/behavioral questions tailored to a
     candidate's resume, evaluation gaps, and the JD (platform-generated or Claude-authored).
   - `interview-oncall` — silent interview co-pilot: confirms the JD and candidate with the main
     panel member, listens to the live interview without interrupting, then scores the candidate's
     answers against the stored questions' key points, the resume, and the JD.
   - `/hireplus:ListScreeningQuestions` and `/hireplus:ListInterviewQuestions` — list a
     candidate's stored question sets by category.

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
tools become available. No Node.js or extra bridge process required — Claude talks to the server
natively over HTTP.

> **Claude Desktop app:** the same plugin installs there too. After installing, open the **hyrewyse**
> connector and click **Connect** — Claude Desktop self-registers its OAuth client via DCR and opens
> the same consent page. Nothing to configure.

## What's inside

```
.claude-plugin/marketplace.json        # marketplace catalog (this repo)
plugins/hireplus/
├── .claude-plugin/plugin.json         # plugin manifest
├── .mcp.json                          # MCP server: native HTTP + DCR (self-registering OAuth)
├── commands/
│   ├── ListScreeningQuestions.md      # /hireplus:ListScreeningQuestions <candidate> [job]
│   └── ListInterviewQuestions.md      # /hireplus:ListInterviewQuestions <candidate> [job]
└── skills/
    ├── hyrewyse-recruiting/           # core playbook (SKILL.md + tool reference)
    ├── generate-screening-questions/  # recruiter phone-screen question generation
    ├── generate-interview-questions/  # round-1 interview question generation
    └── interview-oncall/              # silent listen → analyze interview co-pilot
```

The MCP server is declared as a **native HTTP server**. The Hyrewyse backend implements
[RFC 7591 Dynamic Client Registration](https://datatracker.ietf.org/doc/html/rfc7591), so Claude —
both Claude Code and the Claude Desktop app — **self-registers its own OAuth client on first
connect**. No client id, no callback port, no `mcp-remote` bridge, and no Node.js needed:

```json
{
  "mcpServers": {
    "hyrewyse": {
      "type": "http",
      "url": "https://hireplus-5ie3l.ondigitalocean.app/mcp"
    }
  }
}
```

On first authenticate, Claude discovers the server's OAuth metadata
(`/.well-known/oauth-authorization-server`), registers a client at the advertised
`registration_endpoint`, then runs the standard PKCE authorization-code flow against the consent page.

Equivalent to adding it by hand with:

```shell
claude mcp add --transport http hyrewyse \
  https://hireplus-5ie3l.ondigitalocean.app/mcp
```

## Server-side prerequisite (one-time, platform operators)

With Dynamic Client Registration the backend onboards clients itself — there is **no per-client
allow-list to maintain**. Two things must be in place on the deployed backend:

1. The `mcp_oauth_clients` table migration is applied (where DCR stores self-registered clients).
2. The public origin is set correctly. In **DigitalOcean App Platform → Settings → Environment
   Variables**:

   ```bash
   APP_URL=https://hireplus-5ie3l.ondigitalocean.app
   ```

- `APP_URL` must be the exact public origin (`MCP_PUBLIC_BASE_URL=${APP_URL}`). If it's empty the
  server audience-binds tokens to `localhost` and Claude gets a 401 / audience mismatch.
- `MCP_OAUTH_CLIENTS` is now **optional** — DCR registers clients dynamically. Set it only if you
  also want a fixed, pre-registered client id (e.g. for a script or a non-DCR client).
- Redeploy after applying the migration or changing env vars.

## Updating

Push changes to this repo, then users run:

```shell
/plugin marketplace update 91social
```

## License

MIT

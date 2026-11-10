# Installing the TaxLocus MCP server

TaxLocus is a remote MCP server. There is nothing to download, build or run.

- **Server URL:** `https://api.taxlocus.com/mcp`
- **Transport:** Streamable HTTP
- **Auth:** OAuth 2.1 with PKCE. The server answers an unauthenticated request with `401` and
  `WWW-Authenticate: Bearer resource_metadata="https://api.taxlocus.com/.well-known/oauth-protected-resource"`
  (RFC 9728), and the authorization server supports dynamic client registration, so an OAuth-capable
  client signs you in by itself. No API key or environment variable to set.

The first time your client connects, a browser window opens. Enter your email, click the sign-in link
we send you, and approve access. Open the link in the same browser you started in. If you don't have a
TaxLocus account yet, a free one is created for you (1,000 lookups a month, no card).

## Claude Code

```bash
claude mcp add --transport http taxlocus https://api.taxlocus.com/mcp
```

Then run `/mcp` inside Claude Code, select **taxlocus** and choose **Authenticate** to sign in.
Add `--scope user` to make it available in every project.

## Claude.ai and Claude Desktop

1. Open **Settings → Connectors**.
2. Click **Add custom connector**, name it `TaxLocus` and paste `https://api.taxlocus.com/mcp`.
3. Click **Connect** and sign in.

Connectors you add on claude.ai also appear in Claude Desktop.

## Cursor

Add this to `~/.cursor/mcp.json` (all projects) or `.cursor/mcp.json` (one project):

```json
{
  "mcpServers": {
    "taxlocus": {
      "url": "https://api.taxlocus.com/mcp"
    }
  }
}
```

Open **Cursor Settings → MCP** and click **Connect** (or **Needs login**) next to taxlocus to sign in.

## VS Code (GitHub Copilot agent mode)

Add this to `.vscode/mcp.json` in your workspace, or run **MCP: Add Server** from the Command Palette:

```json
{
  "servers": {
    "taxlocus": {
      "type": "http",
      "url": "https://api.taxlocus.com/mcp"
    }
  }
}
```

Or from a terminal:

```bash
code --add-mcp '{"name":"taxlocus","type":"http","url":"https://api.taxlocus.com/mcp"}'
```

Start the server from `mcp.json` or the MCP view; VS Code asks you to sign in on first start.

## Clients without OAuth: API key

A client that can't do OAuth can send a key instead. Create one at https://taxlocus.com/keys and send
it as a header:

```json
{
  "mcpServers": {
    "taxlocus": {
      "type": "http",
      "url": "https://api.taxlocus.com/mcp",
      "headers": { "Authorization": "Bearer tlx_live_YOUR_KEY" }
    }
  }
}
```

Keep the key out of shared or committed config files.

## Check it works

Ask: *"What's the sales tax rate at 414 E 12th St, Kansas City, MO 64106?"* The answer should come
from the `lookup_rate` tool, with a breakdown by jurisdiction and a source link for each line.

## Help

support@taxlocus.com · https://taxlocus.com/mcp

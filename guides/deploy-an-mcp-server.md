# Deploy an MCP server

Deploy a Model Context Protocol server on Brimble so AI clients (Claude, Cursor, IDE agents) can connect to it over the internet, without it running on your laptop.

## Prerequisites

- A Git repository containing an MCP server. The MCP SDK is available for TypeScript, Python, and other languages.
- A Brimble account.
- The MCP server should expose itself over HTTP (Streamable HTTP transport). MCP servers that only speak stdio don't work for remote deployment.

## What you get

After deploy, your MCP server is reachable at `https://<project-name>.brimble.app` (or your custom domain). Any MCP client that supports remote servers can connect to it.

## Step 1: Make sure the server speaks HTTP

Verify your MCP server uses the HTTP/SSE or Streamable HTTP transport, and reads its port from the `PORT` environment variable.

Example, TypeScript MCP server:

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express from "express";

const server = new McpServer({ name: "acme-mcp", version: "1.0.0" });
// ... register tools, resources, prompts ...

const app = express();
app.use(express.json());
app.post("/mcp", async (req, res) => {
  const transport = new StreamableHTTPServerTransport({});
  res.on("close", () => transport.close());
  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

const port = Number(process.env.PORT) || 3000;
app.listen(port, () => console.log(`MCP listening on ${port}`));
```

The path can be anything (`/mcp` is conventional). Whatever you pick, MCP clients connect to it.

## Step 2: Create the project

1. In the dashboard, click **New project**.
2. Connect your repository.
3. Under **Service type**, choose **MCP server**.
4. Brimble auto-detects the install, build, and start commands. Override if needed.
5. Pick a region close to your primary clients.

## Step 3: Set authentication

Most MCP servers shouldn't be open to the internet. Pick one:

**Option A — Password-protect the deployment.** Brimble can require basic auth at the edge. Enable it under **Settings** → **Access** and set a username/password. Clients pass `Authorization: Basic …` on every request.

**Option B — App-level auth.** Implement bearer-token validation in your server code. The MCP SDK supports auth headers; verify them before handling requests.

Option B is more flexible (per-user tokens, rotation) but you write the validation code. Option A is one click.

## Step 4: Deploy

Click **Deploy**. The build runs the same as any web service. Once active, your MCP server is live at the project URL.

## Step 5: Connect from a client

In Claude Desktop, edit `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "acme": {
      "url": "https://acme-mcp.brimble.app/mcp",
      "headers": {
        "Authorization": "Bearer <your-token>"
      }
    }
  }
}
```

In Cursor, add the server under **Settings** → **MCP Servers**.

Restart the client. The MCP tools, resources, and prompts you registered should appear.

## Verification

From your terminal:

```bash
curl -X POST https://<project-name>.brimble.app/mcp \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-token>" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

You should get back a list of the tools your server registered.

## Logging

The dashboard streams stdout and stderr from the MCP server in real time. Useful when debugging which tools an agent is calling and with what arguments.

If your server's logging is verbose, log selectively — runtime logs accumulate and are visible to anyone with project access.

## Troubleshooting

**Client can't connect.** Confirm the server is listening on `process.env.PORT` and that the path in the URL (`/mcp` or whatever you chose) matches what your code handles.

**Client connects but no tools appear.** Tools are registered before `server.connect()` is called. Check that registration happens at module-load time, not lazily inside a handler.

**502 from `/mcp`.** Your server isn't responding within Brimble's timeout. Check the runtime logs for crashes or hangs during request handling.

**MCP works locally but fails when deployed.** The local-only stdio transport doesn't work for remote MCP. Make sure the deployed code uses HTTP/SSE or Streamable HTTP.

**WebSocket-based MCP transports don't connect.** Brimble's edge supports WebSockets, but make sure your server actually upgrades the connection — some MCP transports need both `Upgrade: websocket` headers passed through. Brimble passes them.

## Next steps

- [Networking](../concepts/networking.md) — how the edge handles HTTP, WebSockets, and authentication headers.
- [Custom domains](../getting-started/custom-domains.md) — give your MCP server a stable URL.

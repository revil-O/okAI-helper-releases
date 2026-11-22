# Using okAI-helper over MCP

[English](MCP.md) · [Deutsch](MCP.de.md)

MCP (the [Model Context Protocol](https://modelcontextprotocol.io)) lets assistant and agent programs use tools. With the
MCP server switched on, such a program uses the language models on your Mac as tools: it lists them, asks one a
question, or turns texts into vectors. Everything stays on your Mac or in your own network.

okAI-helper speaks both MCP generations on the same address: **2025-11-25** (with its handshake, what most programs use
today) and **2026-07-28** (stateless, the current revision).

## Set it up in five steps

1. **Server.** Open **Server** in okAI-helper and switch on **Local server**. The port is shown there (1235 by default).
   For other devices, switch on **Reachable in the network** and, if you like, name the **Allowed devices**.
2. **MCP.** On the same page switch on **MCP server** (off by default). The page now shows the MCP address and a ready
   example.
3. **Token.** Copy the **Access token** (Show, Copy). Every request needs it; keep it like a password.
   **New token** locks out every program that still uses the old one.
4. **Your program.** Add an MCP server of type **HTTP** (also called *streamable HTTP*):
   - on this Mac: `http://127.0.0.1:1235/mcp`
   - from another device: `http://<address of the Mac>:1235/mcp` (the Server page lists it under *In the network*)
   - header: `Authorization: Bearer <token>`

   The exact form depends on the program, see below.
5. **Check it** with the official MCP Inspector (needs Node.js 22.23.4, 24.22.1, 26.12.1 or newer):

   ```bash
   npx @modelcontextprotocol/inspector --cli http://127.0.0.1:1235/mcp --transport http \
     --header "Authorization: Bearer <token>" --method tools/list
   ```

   It should list `list_models`, `ask_model` and `embed`. `--protocol-era modern` checks the 2026-07-28 generation.

## What a program gets

| Tool | What it does | Fields |
| --- | --- | --- |
| `list_models` | the models of the library: id, kind (chat or embedding), format, size, whether loaded, and the prompt profiles that fit | none |
| `ask_model` | asks a chat model and returns its answer as text; a model that is not loaded is loaded first | `prompt` (required), `model` (an id from `list_models`; without it the loaded model answers), `profile` (needs `model`), `system`, `max_tokens` |
| `embed` | turns texts into vectors with an embedding model | `model` (required), `input` (a list of texts) |

The **prompt profiles** of the app come as MCP *prompts* too: a program can pick one and gets its system prompt (as
the first message) and its examples.

Every tool goes through the app's own server: loading on demand, the profiles, the model's sampling values, the guard
monitor and the **server log** apply as for any other program. Each question shows up in the log twice: as `/mcp` from
the program's device and as `/v1/…` from 127.0.0.1 (the tool itself).

## Programs

Tested by us: **MCP Inspector** and **Claude Code**. The others follow their makers' documentation (linked) and were
not tested here; their forms change now and then.

### Claude Code

```bash
claude mcp add --transport http okai http://127.0.0.1:1235/mcp \
  --header "Authorization: Bearer <token>"
```

`--scope user` makes it available in every project, `--scope project` writes a `.mcp.json` for the project. In a
`.mcp.json` the token can come from an environment variable instead of the file:

```json
{ "mcpServers": { "okai": { "type": "http", "url": "http://127.0.0.1:1235/mcp",
  "headers": { "Authorization": "Bearer ${OKAI_TOKEN}" } } } }
```

`"type": "http"` is required. Source: [Claude Code, MCP](https://code.claude.com/docs/en/mcp).

### VS Code (GitHub Copilot)

The current form is a `~/.copilot/mcp-config.json` (for all projects) or a `.mcp.json` in the workspace:

```json
{ "mcpServers": { "okai": { "type": "http", "url": "http://127.0.0.1:1235/mcp",
  "headers": { "Authorization": "Bearer ${OKAI_TOKEN}" }, "tools": ["*"] } } }
```

The older `.vscode/mcp.json` (with `servers` and `inputs`) is deprecated and, according to the documentation, goes away
in February 2027. Source: [VS Code, MCP configuration](https://code.visualstudio.com/docs/agents/reference/mcp-configuration).

### Cursor

`~/.cursor/mcp.json` (all projects) or `.cursor/mcp.json` (one project):

```json
{ "mcpServers": { "okai": { "url": "http://127.0.0.1:1235/mcp",
  "headers": { "Authorization": "Bearer ${env:OKAI_TOKEN}" } } } }
```

Source: [Cursor, MCP](https://cursor.com/docs/context/mcp).

### Continue

A file in `.continue/mcpServers/` of the workspace, e.g. `okai.yaml`:

```yaml
name: okAI-helper
version: 0.0.1
schema: v1
mcpServers:
  - name: okAI-helper
    type: streamable-http
    url: http://127.0.0.1:1235/mcp
    requestOptions:
      headers:
        Authorization: "Bearer <token>"
```

Source: [Continue, MCP](https://docs.continue.dev/customize/deep-dives/mcp).

### LM Studio (0.3.17 or newer)

Program tab → Install → Edit `mcp.json`:

```json
{ "mcpServers": { "okai": { "url": "http://127.0.0.1:1235/mcp",
  "headers": { "Authorization": "Bearer <token>" } } } }
```

Source: [LM Studio, MCP](https://lmstudio.ai/docs/app/mcp).

### Open WebUI (0.6.31 or newer)

As admin: Settings → Integrations → External Tool Servers → **+ Add Connection**, type **MCP (Streamable HTTP)**, the
address, Auth **Bearer** and the token as *Key*. Open WebUI connects from its server: in Docker the address is
`http://host.docker.internal:1235/mcp`. Source: [Open WebUI, MCP](https://docs.openwebui.com/features/extensibility/mcp).

### Claude Desktop

Claude Desktop's *custom connectors* are reached from Anthropic's cloud: they need an address on the public internet
over HTTPS, so they can not reach a server on your Mac or in your network. For a local HTTP server there is the
third-party bridge [mcp-remote](https://github.com/geelen/mcp-remote) (not by Anthropic, not tested here). Sources:
[custom connectors](https://support.claude.com/en/articles/11175166-getting-started-with-custom-connectors-using-remote-mcp),
[local MCP servers](https://support.claude.com/en/articles/10949351-getting-started-with-local-mcp-servers-on-claude-desktop).

## When something does not work

| Answer | Why | What to do |
| --- | --- | --- |
| 401 | missing or wrong token | copy the token again; after **New token** every program needs the new one |
| 403 | the device is not in **Allowed devices**, or the request came from a web page in a browser | add the device's address or network; browsers are always refused |
| 404 | **MCP server** is off, or the address does not end in `/mcp` | switch it on under Server; check the address |
| no connection | the server is off, the port differs, or **Reachable in the network** is off for another device | check the Server page; a firewall on the Mac may block the port |
| the first answer takes long | the model is loaded first | `list_models` shows which models are loaded |
| `a profile needs a model` | `profile` without `model` | give both |

Over the network the server speaks plain HTTP: use it in your own network only (or over a VPN).

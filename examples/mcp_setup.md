# MCP Integration — Claude Code

## Setup

Run the MCP server:

```bash
mempalace-mcp
```

Or add it to Claude Code:

```bash
claude mcp add mempalace -- mempalace-mcp
```

## Available Tools

The server exposes the full MemPalace MCP toolset. Common entry points include:

- **mempalace_status** — palace stats (wings, rooms, drawer counts)
- **mempalace_search** — semantic search across all memories
- **mempalace_list_wings** — list all projects in the palace

## Usage in Claude Code

Once configured, Claude Code can search your memories directly during conversations.

## VS Code Copilot

### Setup

Add MemPalace to `.vscode/mcp.json` in your project (or to VS Code user settings):

```json
{
  "servers": {
    "mempalace": {
      "type": "stdio",
      "command": "mempalace-mcp"
    }
  }
}
```

With a custom palace path:

```json
{
  "servers": {
    "mempalace": {
      "type": "stdio",
      "command": "mempalace-mcp",
      "args": ["--palace", "/path/to/palace"]
    }
  }
}
```

Run `mempalace mcp` to get the exact snippet for your setup, including path expansion if you use `--palace`.

### Verify

After connecting, open GitHub Copilot Chat and ask:

```
what mempalace tools are available?
```

Or call `mempalace_status` directly in a Copilot chat request.


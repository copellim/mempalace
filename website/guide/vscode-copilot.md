# VS Code Copilot

Connect MemPalace to VS Code Copilot Chat through MCP so Copilot can search and write to your palace during any conversation. This guide covers the full shipped workflow: MCP setup, repository instructions, auto-save hooks, and historical backfill.

## Prerequisites

- MemPalace installed (`pip install mempalace`)
- VS Code with the **GitHub Copilot** and **GitHub Copilot Chat** extensions
- VS Code 1.99 or later (MCP support in agent mode)

## MCP Setup

VS Code reads MCP server definitions from a `.vscode/mcp.json` file in your workspace, or from the user-level MCP settings.

### Workspace-level (recommended)

Create `.vscode/mcp.json` in your project root:

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

If your palace lives at a non-default path, pass the `--palace` flag:

```json
{
  "servers": {
    "mempalace": {
      "type": "stdio",
      "command": "mempalace-mcp",
      "args": ["--palace", "/path/to/your/palace"]
    }
  }
}
```

:::tip Setup helper
Run `mempalace mcp` in your terminal. It prints the exact JSON block above, ready to paste.
:::

### User-level

Open the VS Code command palette (`Ctrl+Shift+P` / `Cmd+Shift+P`) and run **MCP: Add Server**. Choose **stdio**, enter `mempalace-mcp` as the command, and name the server `mempalace`.

## Verifying Tool Availability

After saving `.vscode/mcp.json`, open Copilot Chat in **agent mode** (`@agent` or the agent dropdown). Ask:

> "What MemPalace tools are available?"

Or call the status tool directly:

> "Call mempalace_status"

Copilot will list all available tools and return the palace overview. If you see `mempalace_status`, `mempalace_search`, and the other palace tools in the response, the MCP connection is working.

:::warning Agent mode required
MemPalace tools are only available when Copilot Chat is running in **agent mode**. In standard chat mode, MCP tools are not invoked.
:::

## Repository Instructions

For repositories that use MemPalace (such as MemPalace itself), place a `.github/copilot-instructions.md` file at the repository root. This file teaches Copilot when to call `mempalace_search` and how to interpret results.

A minimal instructions file looks like this:

```markdown
# Project Memory

A MemPalace MCP server is active. Use it.

## When to search

Call `mempalace_search` before answering questions about:

- prior decisions and their reasoning
- people and contributors
- project history and past changes

## How to search

mempalace_search("<natural-language query>")

Return verbatim results. Do not paraphrase stored content.
```

VS Code Copilot reads `.github/copilot-instructions.md` automatically for every workspace where it exists. No additional configuration is needed. See the [MCP Integration guide](/guide/mcp-integration) for the full list of available tools.

## Auto-Save Hooks

MemPalace ships a hook template for VS Code Copilot at `.github/hooks/vscode-copilot.json`. Place this file in your workspace's `.github/hooks/` directory and VS Code Copilot picks it up automatically when hooks are enabled in VS Code settings.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "type": "command",
        "command": "mempalace hook run --hook session-start --harness vscode-copilot"
      }
    ],
    "Stop": [
      {
        "type": "command",
        "command": "mempalace hook run --hook stop --harness vscode-copilot"
      }
    ],
    "PreCompact": [
      {
        "type": "command",
        "command": "mempalace hook run --hook precompact --harness vscode-copilot"
      }
    ]
  }
}
```

### What each hook does

| Hook             | When It Fires                     | What Happens                                            |
| ---------------- | --------------------------------- | ------------------------------------------------------- |
| **SessionStart** | When a new Copilot session begins | Registers the session so MemPalace can track context    |
| **Stop**         | After each assistant turn         | Ingests the conversation turn into the palace           |
| **PreCompact**   | Before context compaction         | Emergency ingest — saves context before it is compacted |

### Fail-open behavior

Both Stop and PreCompact hooks are **fail-open**. If a local ingest error occurs (network unavailable, palace locked, process error), the hook exits without blocking the chat. Your conversation continues uninterrupted. Errors are written to the hook log but never surface to the chat window.

:::tip Enabling hooks in VS Code
To activate hooks, open VS Code settings and enable **GitHub Copilot: Hooks** (or the equivalent setting in your VS Code version). VS Code then reads hook definitions from `.github/hooks/` in the workspace root.
:::

## Historical Backfill

VS Code Copilot transcripts are a supported conversation export source. To backfill existing conversations into your palace, point `mempalace mine` at the directory where VS Code Copilot stores its conversation logs on your machine:

```bash
mempalace mine <conversationdir> --mode convos --wing vscode-copilot
```

Replace `<conversationdir>` with the actual path to your VS Code Copilot conversation exports. The `--mode convos` flag tells the miner to treat the directory as conversation transcripts. The `--wing` flag is optional — omit it to let MemPalace detect the wing automatically.

:::info Finding your conversation directory
VS Code Copilot stores conversation transcripts in a platform-specific location inside your VS Code user data folder. Check VS Code documentation or your VS Code installation for the exact path on your system.
:::

## Further Reading

- [MCP Integration](/guide/mcp-integration) — full tool reference and memory protocol
- [Getting Started](/guide/getting-started) — palace setup and first mine
- [Auto-Save Hooks](/guide/hooks) — hook architecture and behavior for all supported surfaces


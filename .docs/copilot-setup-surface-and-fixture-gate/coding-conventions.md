# Codebase Conventions — MemPalace

_Source references verified in session. Phase: Copilot Setup (Surface & Fixture Gate)._

## Python Style — CLI Output

- **Indentation**: CLI output lines use 2-space indentation for subcommands and examples. Example: `mempalace/cli.py` (`print(f"  claude mcp add mempalace -- {server_cmd}")`).
- **Section headers**: Plain text ending with colon, followed by blank line. Example: `"MemPalace MCP quick setup:"`.
- **Path quoting**: All shell-embedded paths wrapped with `shlex.quote()`.
- **Path normalization**: User paths normalized with `Path(path).expanduser()` before use.

## Python Style — General

- **Docstrings**: Single-line docstring format for short functions. Example: `"""Show how to wire MemPalace into MCP-capable hosts."""`.

## Python Testing — pytest Patterns

- **CLI argument mocking**: `monkeypatch.setattr(sys, "argv", ["mempalace", ...])` to set command-line arguments.
- **Output capture**: `capsys.readouterr()` returns object with `.out` and `.err` attributes.
- **CLI invocation**: `main()` function called to execute full CLI flow.
- **Assertions**: Substring presence/absence checks on `captured.out` using `in` and `not in`.
- **Stderr validation**: Explicit assertion `assert captured.err == ""` to verify no stderr output.

## JSON Fixture Conventions — Hook Payloads

Files: `tests/fixtures/vscode_copilot/hooks/*.json`

- **Field naming**: All fields use snake_case (`hook_event_name`, `session_id`, `transcript_path`, `stop_hook_active`, `cwd`).
- **UUID format in tests**: Test session IDs follow pattern `test-session-XXXX-0000-0000-000000000000` (first segment encodes fixture number, rest zeros).
- **Test paths**: Absolute paths use `/tmp/mempalace-test-transcripts/` for transcripts and `/home/user/projects/myapp` for working directories.
- **Timestamp format**: ISO 8601 with milliseconds and Z suffix (`"2026-04-23T04:19:59.674Z"`).
- **Username sanitization**: Real usernames replaced with generic `user`. Example: `/home/user/projects/myapp`.

## JSONL Fixture Conventions — Transcripts

Files: `tests/fixtures/vscode_copilot/transcripts/*.jsonl`

- **Line format**: One JSON object per line, no trailing commas.
- **Event ID format**: Sequentially numbered `event-XXXX` (zero-padded 4 digits). Example: `event-0001`, `event-0002`.
- **Tool call ID format**: Pattern `toolcall-XXXX` (zero-padded 4 digits).
- **Field casing in events**: Event data uses camelCase (`sessionId`, `messageId`, `turnId`, `parentId`) — contrasting with hook payload snake_case.
- **Parent reference**: Root events have `"parentId": null`; child events have `"parentId": "event-XXXX"`.
- **Test session ID**: Matches hook pattern `test-session-XXXX-0000-0000-000000000000`.

## Markdown Documentation — Heading Structure

Files: `website/guide/*.md`, `website/reference/*.md`, `hooks/README.md`

- **H1 (top-level)**: Single H1 per file. Host-specific guides: product/host name (e.g., `# Claude Code Plugin`, `# Gemini CLI`). Reference docs: feature name (e.g., `# CLI Commands`).
- **H2 (sections)**: Major logical sections. Host guides use: `## Installation`, `## How It Works`, `## Hooks`. Reference docs use: `` ## `mempalace <command>` ``. Setup/install docs use: `## Install — {Host Name}` format.
- **No H3+ in simple guides**: Single-level H2 sections for clarity.

## Markdown Documentation — Code Blocks

- **Fenced blocks**: Fenced ` ``` ` with language identifier: ` ```json `, ` ```bash `, ` ```typescript `.
- **Inline code**: Backticks for command names (e.g., `` `mempalace mcp` ``).

## Markdown Documentation — Admonitions and Links

- **VitePress admonitions**: `::: warning` syntax for important callouts.
- **Internal links**: Path-based without `.md` extension. Format: `[link text](/guide/filename)` or `[link text](/reference/filename)`. No trailing slashes.
- **External links**: `[Product Name](https://...)`.

## Markdown Documentation — Tables

- GitHub-flavored Markdown tables with pipes.
- Used for: options reference (in CLI reference docs), hook/event comparison tables.

## VitePress Navigation Config — TypeScript

File: `website/.vitepress/config.mts`

- **Sidebar item format**: `{ text: 'Display Name', link: '/guide/filename-no-ext' }`.
- **Text casing**: `text` values use Title Case. Example: `'Getting Started'`, `'Mining Your Data'`.
- **Link format**: Absolute path starting with `/`, no `.md` extension, no trailing slash. Example: `/guide/getting-started`.
- **Host-specific guide placement**: Host guides (Claude Code, Gemini CLI) appear after foundational guides (Installation, MCP Integration) in the sidebar order.

## File Naming Conventions

- **Markdown files**: Kebab-case filenames. Examples: `getting-started.md`, `mcp-setup.md`, `vscode-copilot.md`.
- **Test fixture files**: `.json` for individual hook events, `.jsonl` for transcript files.
- **Fixture locations**: `tests/fixtures/vscode_copilot/hooks/` for hook payloads, `tests/fixtures/vscode_copilot/transcripts/` for transcripts.


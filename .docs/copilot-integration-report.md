# MemPalace × GitHub Copilot integration report

## Goal

Create a complete, working MemPalace integration for:

- GitHub Copilot in VS Code
- GitHub Copilot CLI

The target is feature parity with the current MemPalace experience wherever the host platform allows it: MCP access, repository instructions, autosave/checkpoint hooks, transcript ingest, project bootstrap, and user-facing documentation.

## Executive summary

The repository does **not** currently offer a first-class GitHub Copilot integration.

What already exists is a strong base:

- MemPalace already exposes an MCP server via `mempalace-mcp`
- the docs already position MemPalace as an MCP-compatible memory layer
- hook infrastructure already exists for `claude-code` and `codex`
- conversation ingest already supports multiple transcript formats and a normalization pipeline

What is missing is Copilot-specific support across the full stack:

- product documentation
- setup helpers
- repository instructions for Copilot
- hook harness support
- transcript normalization for Copilot-generated transcripts
- tests and examples

The biggest technical split is this:

- **VS Code Copilot** looks close to a direct hook migration because its hook model includes `Stop`, `PreCompact`, `sessionId`, `transcript_path`, and `stop_hook_active`
- **Copilot CLI** clearly supports MCP, custom instructions, local session storage, and hooks, but its documented hook lifecycle appears different and will likely need a dedicated strategy rather than a literal port of the current Claude/Codex hook behavior

## What we verified in the repository

### 1. Documentation is not Copilot-first today

Current user-facing docs focus on:

- Claude Code
- Gemini CLI
- generic MCP-compatible tools
- Cursor / ChatGPT mentions in MCP docs

But there is no dedicated GitHub Copilot guide or setup path.

Relevant examples:

- `README.md` points to Claude Code, Gemini CLI, MCP-compatible tools, and local models
- `website/guide/mcp-integration.md` gives `claude mcp add ...` examples only
- `website/guide/hooks.md` documents hooks only for Claude Code and Codex
- `website/guide/getting-started.md` mentions Claude/ChatGPT/Slack imports and generic MCP integration, not Copilot-specific setup
- `mempalace/cli.py` `cmd_mcp()` prints only `claude mcp add ...`

### 2. Hook support is currently limited to Claude/Codex harnesses

`mempalace/hooks_cli.py` currently supports:

```python
SUPPORTED_HARNESSES = {"claude-code", "codex"}
```

The hook parsing layer expects these fields from stdin:

- `session_id`
- `stop_hook_active`
- `transcript_path`

The current hook behavior already implements the core MemPalace lifecycle:

- periodic save logic on `stop`
- emergency save before compaction on `precompact`
- session initialization on `session-start`

The existing save path already includes both:

- **checkpoint/diary writing**
- **transcript ingest**

That means the existing architecture is already close to what we need for Copilot, especially on the VS Code side.

### 3. Conversation ingest is format-aware, but not Copilot-aware

Conversation normalization currently includes support for existing formats such as:

- Claude Code JSONL
- Codex JSONL
- Claude.ai JSON
- ChatGPT JSON
- Slack JSON
- markdown/plain text transcripts

Representative code:

- `mempalace/normalize.py` contains `_try_codex_jsonl()` and `_try_claude_ai_json()`
- docs describe five supported chat formats and do not mention Copilot

This is the main technical gap for Copilot ingest: MemPalace can ingest conversations in principle, but it does not yet recognize or normalize Copilot transcript formats.

### 4. MCP support exists, but setup UX is Claude-oriented

MemPalace already exposes the MCP server via:

- `mempalace-mcp`
- `mempalace.mcp_server:main`

The MCP docs are strong, but examples are currently Claude-first:

- `website/guide/mcp-integration.md`
- `examples/mcp_setup.md`
- `mempalace/mcp_server.py` docstring references Claude Code installation
- `mempalace/cli.py` prints Claude setup commands only

## What we verified outside the repository

## 1. GitHub Copilot CLI

Official Copilot CLI docs confirm:

- custom MCP servers are supported
- repository instructions are supported
- agent instructions such as `AGENTS.md` are supported
- local session data is stored under `~/.copilot/session-state/`
- structured history is also stored in `~/.copilot/session-store.db`

This is important because it gives us:

- a viable MCP surface
- a viable instruction surface
- a local transcript/history surface for ingest and backfill

The **reference** page for Copilot CLI / Copilot agent hooks currently documents these hook types:

- `sessionStart`
- `sessionEnd`
- `userPromptSubmitted`
- `preToolUse`
- `postToolUse`
- `errorOccurred`

The documented input payloads for those hooks include fields such as:

- `timestamp`
- `cwd`
- `source` / `initialPrompt` for `sessionStart`
- `reason` for `sessionEnd`
- `prompt` for `userPromptSubmitted`
- `toolName` / `toolArgs` for `preToolUse`
- `toolResult` for `postToolUse`

Important limitation for MemPalace planning:

- the hook reference does **not** document a `Stop` hook
- it does **not** document a `PreCompact` hook
- it does **not** document `transcript_path`
- it does **not** document `sessionId`
- it does **not** document `stop_hook_active`

There is a separate conceptual hooks page that mentions `agentStop` and `subagentStop`, but the reference page does not document their payloads or example behavior. For implementation planning, we should treat those as **underdocumented** until verified in a real Copilot CLI session.

This means Copilot CLI is still viable, but it does **not** currently present the same documented lifecycle contract as Claude Code or VS Code Copilot.

## 2. GitHub Copilot in VS Code

The VS Code hook documentation is the key discovery.

VS Code Copilot hooks support these events:

- `SessionStart`
- `UserPromptSubmit`
- `PreToolUse`
- `PostToolUse`
- `PreCompact`
- `SubagentStart`
- `SubagentStop`
- `Stop`

The common input fields include:

- `sessionId`
- `transcript_path`
- `cwd`
- `hookEventName`

`Stop` also includes:

- `stop_hook_active`

This matters because it means VS Code Copilot can support the same broad MemPalace lifecycle as Claude Code:

- periodic stop-based checkpoints
- pre-compaction emergency save
- transcript-based ingest
- loop prevention for blocked stop hooks

The VS Code docs also explicitly state:

- VS Code reads `.claude/settings.json` and `.claude/settings.local.json`
- hook configs are largely compatible across Claude Code, Copilot CLI, and VS Code
- when adapting Claude hooks, the main differences are tool names and field naming in tool inputs

That last point is important: it suggests the **Stop/PreCompact** part of the current MemPalace design is likely portable with much less effort than originally expected.

## What "ingest" means and why it matters

Ingest is the pipeline that takes raw source material and turns it into searchable palace memory.

In MemPalace terms, ingest means:

1. reading source material
2. normalizing it into a known internal shape
3. splitting it into useful units
4. attaching metadata such as wing, room, source file, and agent
5. writing it into the palace so search and wake-up can use it

### Ingest is not the same as autosave

- **autosave/checkpoint hooks** decide **when** something should be saved
- **ingest** decides **how raw material becomes memory**

Example:

- a stop hook fires after N user messages
- it decides a checkpoint should happen
- ingest then reads the transcript and files the conversation into the palace

### Why ingest is essential in general

Ingest is required for two different jobs:

#### A. Initial acquisition / bootstrap

This is the first population of the palace:

- project code
- docs
- notes
- existing chat history
- previous CLI/IDE sessions

Without this, MemPalace starts nearly empty and only learns from future saves.

#### B. Ongoing incremental memory

This is the continuous flow after bootstrap:

- new conversations
- new decisions
- new code work
- new checkpoints

Hooks and autosave help trigger this; ingest makes it durable and searchable.

### Why ingest is a central Copilot integration concern

For Copilot, the critical question is not only "can we trigger a save?"

It is also:

- what transcript or session file do we receive?
- what format is it in?
- how do we normalize it?
- which events contain canonical user/assistant turns versus tool noise?

That is why transcript normalization is one of the main implementation tracks.

## Gap analysis by integration area

| Area | Current repo state | VS Code Copilot | Copilot CLI | Notes |
|---|---|---|---|---|
| MCP server | Already present and usable | Likely document-only work plus examples | Likely document-only work plus examples | Core server already exists |
| Setup helper (`mempalace mcp`) | Claude-only output | Needs VS Code examples (`mcp.json`, command palette flow) | Needs Copilot CLI examples (`/mcp add`, `~/.copilot/mcp-config.json`) | UX gap, not backend gap |
| Repo instructions | No `.github/copilot-instructions.md` | Needed | Needed | `AGENTS.md` exists but is Claude-centric |
| Hook harness support | Only `claude-code`, `codex` | Add `vscode-copilot` | Add `copilot-cli` only after validating a workable contract | CLI hook payloads do not currently match existing harness assumptions |
| Stop-based autosave | Implemented for existing harnesses | Strong match to current design | No direct documented equivalent in hook reference | VS Code looks straightforward |
| Precompact emergency save | Implemented for existing harnesses | Strong match to current design | No documented equivalent in hook reference | Main parity risk for CLI |
| Transcript ingest | Existing conversation mining pipeline | Needs Copilot transcript normalizer | Needs Copilot CLI session/event normalizer and session-file discovery | Main technical work |
| Initial project acquisition | Already supported via project miner | No new backend work, only docs/instructions | No new backend work, only docs/instructions | Important to document clearly |
| Backfill of past Copilot sessions | Not supported | Need transcript capture + importer | Likely ingest `~/.copilot/session-state/*` | Valuable for migration story |
| Tests | Existing hooks/normalize tests are Claude/Codex-heavy | Add VS Code hook fixtures | Add Copilot CLI event/session fixtures | Required for safe rollout |
| Documentation | Copilot not first-class | New VS Code guide | New Copilot CLI guide | Also update README and getting-started |

## Implementation target: what "complete integration" should mean

To call this integration complete, MemPalace should support all of the following:

### 1. MCP connectivity

Users should be able to connect MemPalace to:

- Copilot in VS Code
- Copilot CLI

without manual guesswork.

This includes:

- documented setup
- copy-pasteable examples
- helper output from `mempalace mcp`

### 2. Repository-aware Copilot behavior

The repository should expose Copilot-ready instructions through:

- `.github/copilot-instructions.md`
- `AGENTS.md` cleanup where appropriate
- optional path-specific instructions in `.github/instructions/`

The goal is that Copilot:

- searches before answering about past events
- uses MemPalace MCP tools appropriately
- respects the verbatim/local-first design constraints

### 3. Autosave/checkpoint workflow

For supported Copilot surfaces, MemPalace should be able to:

- checkpoint every N user messages or equivalent interaction count
- prevent context loss before compaction where the host supports it
- avoid infinite hook loops
- notify the user clearly when memories were filed

### 4. Transcript ingest

MemPalace should be able to ingest:

- ongoing Copilot sessions
- previously saved Copilot session history

That requires:

- transcript format detection
- normalization
- exchange extraction
- tool noise filtering
- tests with real fixtures

### 5. Initial project bootstrap

The Copilot integration docs should make it clear that a useful setup has two layers:

1. **bootstrap ingest** of project files/docs/history
2. **ongoing autosave + incremental ingest**

Without both, the integration will feel partial.

## Recommended implementation workstreams

## Workstream 1: product surface and instructions

Deliverables:

- add `.github/copilot-instructions.md`
- review `AGENTS.md` to make it less Claude-specific where needed
- optionally add `.github/instructions/*.instructions.md` for path-specific rules

Why this matters:

- Copilot supports these files directly
- this improves behavior before any code changes land

## Workstream 2: MCP setup and docs

Deliverables:

- update `mempalace mcp` so it prints:
  - VS Code MCP setup guidance
  - Copilot CLI MCP setup guidance
  - existing generic direct server command
- add docs pages or sections for:
  - VS Code Copilot
  - Copilot CLI
- update README and getting-started links

Why this matters:

- current setup UX is still Claude-branded
- the backend is already there, but discoverability is poor

## Workstream 3: VS Code Copilot hook harness

Deliverables:

- add `vscode-copilot` harness support in `hooks_cli.py`
- map VS Code input fields to MemPalace internal fields:
  - `sessionId` -> `session_id`
  - `transcript_path` -> `transcript_path`
  - `stop_hook_active` -> `stop_hook_active`
- add example hook configs for `.github/hooks/*.json`
- document compatibility with `.claude/settings.json` loading in VS Code where useful

Why this matters:

- this appears to be the shortest route to strong feature parity

## Workstream 4: Copilot transcript normalization

Deliverables:

- capture sample transcripts from:
  - VS Code Copilot `transcript_path`
  - Copilot CLI session data under `~/.copilot/session-state/*`
- add normalizers for those formats in `normalize.py` or the newer source-adapter path if we choose to align with the RFC direction
- add fixtures and tests

Why this matters:

- this is the main blocker for real ingest and backfill
- hooks without ingest are only partial integration

## Workstream 5: Copilot CLI hook strategy

Deliverables:

- confirm the effective hook lifecycle available in Copilot CLI
- confirm whether undocumented-but-mentioned events such as `agentStop` are actually available and stable in Copilot CLI
- if `Stop` / `PreCompact` equivalents are not available in the CLI itself, design a CLI-specific save strategy using documented events and local session-state access
- add `copilot-cli` harness support if warranted

Possible directions:

- `sessionEnd` final flush
- `userPromptSubmitted` interaction counting
- external resolution of the active session under `~/.copilot/session-state/`
- optional background ingest from the session-state directory
- CLI-specific backfill command or docs flow

Why this matters:

- this is likely the least direct parity path
- the documented hook payload does not expose the same fields MemPalace currently relies on (`transcript_path`, `session_id`, `stop_hook_active`)
- it should be treated as a dedicated implementation track, not just a search/replace from Claude

## Workstream 6: initial acquisition and backfill story

Deliverables:

- document initial project ingest explicitly for Copilot users
- document optional backfill from existing Copilot session history
- ensure examples cover both:
  - `mempalace mine <project-dir>`
  - conversation/session ingest sources

Why this matters:

- "complete integration" must include bootstrap, not only future checkpointing

## Workstream 7: test coverage

Deliverables:

- unit tests for new harness parsing
- normalize tests for Copilot transcript shapes
- hook tests for stop/precompact behavior where applicable
- regression tests for silent save and ingest behavior

Why this matters:

- hooks and transcript parsers are fragile and format-specific

## Suggested implementation order

### Phase 1: documentation and instruction surface

- `.github/copilot-instructions.md`
- README / getting-started / MCP docs updates
- `mempalace mcp` improvements

### Phase 2: VS Code Copilot support

- add `vscode-copilot` harness
- add VS Code hook examples
- add VS Code transcript normalizer
- add tests

### Phase 3: Copilot CLI support

- validate hook/event strategy
- implement CLI-specific harness and/or session-state ingest strategy
- add CLI docs and examples
- add tests

### Phase 4: polish and parity review

- review all MemPalace user journeys from a Copilot user perspective:
  - fresh install
  - initial project acquisition
  - MCP setup
  - autosave
  - backfill
  - search and wake-up
- fill remaining docs and UX gaps

## Important risks and unknowns

### 1. VS Code transcript format is still unverified in-repo

We have strong evidence that VS Code exposes `transcript_path`, but we still need real sample transcripts to build a robust normalizer.

### 2. Copilot CLI lifecycle parity is not yet demonstrated

Official CLI docs clearly support hooks, but the hook reference currently documents only:

- `sessionStart`
- `sessionEnd`
- `userPromptSubmitted`
- `preToolUse`
- `postToolUse`
- `errorOccurred`

and does not document:

- `Stop`
- `PreCompact`
- `transcript_path`
- `sessionId`
- `stop_hook_active`

That means:

- full parity for the CLI may require a different strategy than VS Code
- we should not assume the Claude design ports unchanged to CLI
- if we use `agentStop` / `subagentStop`, we should do so only after runtime verification because the concept page mentions them but the reference page does not specify them

### 3. Tool-level hook differences are real

VS Code documents differences from Claude Code in:

- tool names
- tool input property names
- matcher handling

This probably does **not** block stop/precompact logic, but it will matter if we later want richer PreToolUse/PostToolUse features for Copilot.

### 4. The repository currently signals "Claude-first"

Even if the backend becomes Copilot-compatible, the integration will still feel incomplete until branding, docs, examples, and helper commands reflect Copilot explicitly.

## Definition of done

This project should be considered done only when all of the following are true:

- MemPalace can be connected to VS Code Copilot and Copilot CLI through documented MCP flows
- the repo ships Copilot-specific instructions
- VS Code Copilot supports working autosave/checkpoint hooks and transcript ingest
- Copilot CLI has a documented and tested memory flow, whether via equivalent hooks or a CLI-specific strategy
- project bootstrap and historical backfill are documented for Copilot users
- docs and examples no longer imply that Claude is the only first-class experience
- tests cover the new harnesses and transcript formats

## Key references

### Repository files

- `mempalace/cli.py`
- `mempalace/hooks_cli.py`
- `mempalace/normalize.py`
- `mempalace/mcp_server.py`
- `website/guide/mcp-integration.md`
- `website/guide/hooks.md`
- `website/guide/getting-started.md`
- `README.md`

### External docs

- VS Code Copilot hooks: `https://code.visualstudio.com/docs/copilot/customization/hooks`
- Copilot CLI MCP servers: `https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-mcp-servers`
- Copilot CLI session data: `https://docs.github.com/en/copilot/concepts/agents/copilot-cli/chronicle`
- Copilot custom instruction support: `https://docs.github.com/en/copilot/reference/custom-instructions-support`

## Practical next step

Start implementation from the **VS Code Copilot path first**.

Reason:

- it already exposes the lifecycle events MemPalace wants (`Stop`, `PreCompact`)
- it passes `sessionId`, `transcript_path`, and `stop_hook_active`
- it is closest to the current hook architecture

That gives the fastest route to a real, testable Copilot integration while the more host-specific Copilot CLI path is designed in parallel.

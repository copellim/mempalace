# Gap Analysis: VS Code Copilot Integration

## Reference Patterns

- `mempalace/hooks_cli.py` — existing `session-start` / `stop` / `precompact` hook flow and harness parsing pattern to extend with `vscode-copilot`.
- `tests/test_hooks_cli.py` — existing hook tests show the mocking and stdout-capture pattern to mirror for VS Code fixtures.
- `mempalace/normalize.py#_try_claude_code_jsonl` — event-stream parser pattern with turn merging and noise stripping.
- `mempalace/normalize.py#_try_codex_jsonl` — JSONL detector pattern for a new transcript source without introducing a new mining path.
- `mempalace/convo_miner.py` — proves `.jsonl` discovery and `mempalace mine <dir> --mode convos` already support backfill once normalization recognizes the source.
- `website/guide/claude-code.md` and `website/guide/gemini-cli.md` — host-specific guide pattern for a new VS Code Copilot guide.
- `#skill:agent-customization` — relevant for planning `.github/copilot-instructions.md` and any optional `.github/instructions/**` follow-up.

## Areas

### Area: Hook lifecycle and capture

**Current state**: `mempalace/hooks_cli.py` implements the shared hook lifecycle for `claude-code` and `codex` and currently exposes only `session-start`, `stop`, and `precompact` behavior; there is no subagent lifecycle handling yet. The raw VS Code capture set outside the repo now includes real `PreCompact`, `SubagentStart`, and `SubagentStop` payloads, and `.github/hooks/capture.json` has been extended to capture those events. The committed VS Code fixture subtree under `tests/fixtures/vscode_copilot/hooks/` still contains only `session_start.json` and `stop.json`, so the remaining phase-1 gap is sanitizing and checking in the newer raw captures rather than recollecting them.

**Target state**: `mempalace/hooks_cli.py` should recognize `vscode-copilot` as a first-class harness, map real VS Code hook payloads onto the existing internal hook shape, and keep fail-open behavior intact across `SessionStart`, `Stop`, `PreCompact`, `SubagentStart`, and `SubagentStop`. The repo should ship a reusable `.github/hooks` template for VS Code Copilot hook setup, and `tests/test_hooks_cli.py` should validate the supported lifecycle behavior, including the explicit policy for subagent lifecycle events.

**Delta**:

- **Create**: `.github/hooks/vscode-copilot.json` (primary user-facing VS Code Copilot hook template for SessionStart, Stop, and PreCompact)
- **Create**: `tests/fixtures/vscode_copilot/hooks/precompact.json` (sanitized real VS Code PreCompact payload required by R-8)
- **Create**: `tests/fixtures/vscode_copilot/hooks/subagent_start.json` (sanitized real VS Code SubagentStart payload required to validate subagent-creation interception)
- **Create**: `tests/fixtures/vscode_copilot/hooks/subagent_stop.json` (sanitized real VS Code SubagentStop payload required to validate subagent lifecycle completion policy)
- **Modify**: `mempalace/hooks_cli.py` (add `vscode-copilot` to supported harnesses and extend harness input parsing for VS Code payload shape)
- **Modify**: `tests/test_hooks_cli.py` (add VS Code fixture-driven coverage for session-start, stop, precompact, and subagent lifecycle handling while preserving Claude/Codex coverage)

**Hotspots**:

- `mempalace/hooks_cli.py` — modified by: harness registration, payload parsing, fail-open compatibility preservation
- `tests/test_hooks_cli.py` — modified by: VS Code fixture loading, multi-harness assertions

**Blockers**:

- The raw `PreCompact`, `SubagentStart`, and `SubagentStop` captures now exist, but any runtime change that claims validated VS Code lifecycle support is still blocked until sanitized repo fixtures exist, because current committed repo evidence only covers SessionStart and Stop.
- The shared template file `.github/hooks/vscode-copilot.json` is a cross-area dependency: hook behavior and user-facing docs both need the same final event wiring.

### Area: Transcript normalization and backfill

**Current state**: `mempalace/normalize.py` detects and normalizes Claude Code JSONL, Codex JSONL, Claude.ai JSON, ChatGPT JSON, and Slack JSON, but no VS Code Copilot transcript shape. `mempalace/convo_miner.py` already scans `.jsonl` files through the existing `mine --mode convos` flow, so backfill plumbing already exists once the normalizer recognizes the format. The raw capture set outside the repo now includes transcript snapshots with subagent activity and precompact context, but the committed VS Code transcript fixture subtree still contains only `simple_with_tools.jsonl`.

**Target state**: `mempalace/normalize.py` should detect VS Code Copilot JSONL transcripts using the captured schema, preserve canonical user/assistant turns under the existing project transcript standard, merge assistant multi-part turns, and drop tool chrome/noise. The existing `mempalace mine <dir> --mode convos` path should backfill historical VS Code transcripts without introducing a new command, and `tests/test_normalize.py` should cover the VS Code-specific edge cases with real fixtures including tool activity, precompact context, and subagent activity.

**Delta**:

- **Create**: `tests/fixtures/vscode_copilot/transcripts/with_subagent.jsonl` (sanitized real transcript containing SubagentStart/SubagentStop activity required by R-8)
- **Create**: `tests/fixtures/vscode_copilot/transcripts/with_precompact_context.jsonl` (sanitized long-session transcript exercising compaction-related context required by R-8)
- **Modify**: `mempalace/normalize.py` (add VS Code Copilot JSONL detection/parsing and any small helper needed for assistant content or `toolRequests` handling)
- **Modify**: `tests/test_normalize.py` (add fixture-driven VS Code parser coverage for detection, turn merging, tool-noise filtering, and backfill-safe normalization)
- **Modify**: `mempalace/README.md` (update internal module summary so supported transcript formats include VS Code Copilot)

**Hotspots**:

- `mempalace/normalize.py` — modified by: new detector ordering, VS Code parser logic, assistant-content handling under existing transcript rules
- `tests/test_normalize.py` — modified by: new VS Code fixtures, regression assertions against existing formats

**Blockers**:

- The raw subagent transcript capture now exists, but any finalized transcript-side design for subagent-event handling is still blocked until that evidence is sanitized and committed into the repo fixture set.
- PreCompact-related transcript handling should not be locked in until the gating fixture set is committed, because the user required those captures before dependent runtime implementation.

### Area: Product surface, instructions, and docs

**Current state**: `mempalace/cli.py` prints Claude-only MCP setup guidance in `cmd_mcp`, and the public docs are still Claude/Codex/Gemini-first: `README.md`, `hooks/README.md`, `website/guide/mcp-integration.md`, `website/guide/hooks.md`, `website/guide/getting-started.md`, and `website/reference/cli.md` do not present VS Code Copilot as a first-class path. `.github/copilot-instructions.md` does not exist, and website navigation in `website/.vitepress/config.mts` has no VS Code Copilot guide entry.

**Target state**: The repository should present a complete VS Code Copilot setup path across CLI helper output, root docs, website guides, hooks docs, and Copilot-specific repository instructions. `.github/copilot-instructions.md` should define Copilot-facing behavior for MemPalace, `.github/hooks/vscode-copilot.json` should be the primary documented hook path, and the docs should explain MCP setup, hook installation, live ingest, and historical backfill without implying Copilot CLI scope.

**Delta**:

- **Create**: `.github/copilot-instructions.md` (Copilot-specific repository instructions for VS Code per R-2)
- **Create**: `website/guide/vscode-copilot.md` (dedicated end-user guide covering setup, hooks, ingest, and backfill)
- **Modify**: `mempalace/cli.py` (extend `cmd_mcp` output beyond Claude-only guidance)
- **Modify**: `README.md` (surface VS Code Copilot as a first-class integration path)
- **Modify**: `hooks/README.md` (add VS Code Copilot install and backfill guidance using `.github/hooks/*.json` as the primary path)
- **Modify**: `website/guide/mcp-integration.md` (add VS Code Copilot setup guidance and compatible-tool positioning)
- **Modify**: `website/guide/hooks.md` (add VS Code Copilot hook installation path and align wording with current silent-save + ingest behavior)
- **Modify**: `website/guide/getting-started.md` (point users to the VS Code Copilot path)
- **Modify**: `website/reference/cli.md` (document `vscode-copilot` as a hook harness and updated `mempalace mcp` output)
- **Modify**: `website/.vitepress/config.mts` (add navigation entry for the new VS Code Copilot guide)

**Hotspots**:

- `README.md` — modified by: quick-start positioning, integration-path messaging
- `hooks/README.md` — modified by: VS Code install instructions, backfill guidance, hook-behavior explanation
- `mempalace/cli.py` — modified by: MCP helper output and public setup wording

**Blockers**:

- User-facing docs that embed `.github/hooks/vscode-copilot.json` should trail the final template shape from the hook area to avoid shipping instructions that drift from the actual supported event wiring.
- The repo does not currently verify a stable default filesystem location for historical VS Code Copilot transcripts, so docs should avoid hard-coding one unless implementation or external product docs verify it.

## Cross-cutting Hotspots

Files touched by 3+ areas (aggregated across all areas above). These drive the Planner's shared file parallelism check.

- None identified.

## Dead code

- None identified.

## Open questions from discovery

- No planning-blocking open questions remain. The raw capture unknowns required by R-8 are now closed; the remaining phase-1 dependency is sanitizing and committing the captured precompact and subagent fixture set.

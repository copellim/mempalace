# Gap Analysis: VS Code Copilot Integration

## Reference Patterns

- `tests/test_convo_miner.py` — existing end-to-end `mine_convos()` integration-test pattern to extend with VS Code fixtures.
- `tests/test_cli.py` — current CLI output regression pattern for `mempalace mcp` and hook parser choices when user-facing CLI text changes.
- `website/guide/vscode-copilot.md` — canonical VS Code Copilot entry-point guide introduced in phase 1; phase 3 should extend it rather than create a second guide.
- `examples/mcp_setup.md` — existing VS Code Copilot MCP setup example already aligned with the shipped `mempalace mcp` output.
- `.github/hooks/capture.json` — verified VS Code hook event keys (`SessionStart`, `Stop`, `PreCompact`, `SubagentStart`, `SubagentStop`) to convert into a production-safe template.
- `tests/fixtures/vscode_copilot/README.md` — verified VS Code field casing and fixture provenance for docs/examples that mention payload and transcript shapes.

## Areas

### Area: Historical backfill via existing `mine --mode convos` path

**Current state**: `mempalace/normalize.py` already includes `_try_vscode_copilot_jsonl()` in the normalize chain, and `mempalace/convo_miner.py` already calls `normalize()` for every scanned conversation file, so the backfill mechanism exists in the code today. `tests/test_normalize.py` validates VS Code transcript parsing and `tests/fixtures/vscode_copilot/transcripts/` contains realistic fixtures, but `tests/test_convo_miner.py` only has generic text-based mining coverage and no end-to-end VS Code regression proving `mine_convos()` files drawers from those fixtures. User-facing CLI/help text in `mempalace/cli.py` still describes conversation mining as Claude/Claude.ai/ChatGPT/Slack only.

**Target state**: Running `mempalace mine <dir> --mode convos` against VS Code Copilot transcript fixtures is covered by a fixture-backed integration test, and the CLI/help text explicitly lists VS Code Copilot as a supported conversation source without introducing a second backfill path.

**Delta**:

- **Modify**: `tests/test_convo_miner.py` (add a VS Code fixture-backed `mine_convos()` integration test that asserts drawers are created and queryable)
- **Modify**: `mempalace/cli.py` (update the top-level help banner and `mine --mode convos` help text to mention VS Code Copilot)
- **Modify**: `tests/test_cli.py` (add a narrow regression assertion for the updated conversation-mining help text if the planner chooses to lock the CLI wording)

**Hotspots**:

- `tests/test_convo_miner.py` — modified by: new VS Code backfill regression
- `mempalace/cli.py` — modified by: conversation-mining help text parity
- `tests/test_cli.py` — modified by: user-facing CLI text regression coverage

**Blockers**:

- None.

### Area: Website guide and reference parity for VS Code Copilot

**Current state**: `website/guide/getting-started.md` and `website/guide/mcp-integration.md` already route readers to `website/guide/vscode-copilot.md`, and `website/.vitepress/config.mts` already exposes that guide in navigation. But `website/guide/vscode-copilot.md` still states that automatic transcript saving and historical VS Code backfill are not implemented, `website/guide/hooks.md` still says VS Code Copilot hook support is unavailable, `website/guide/mining.md` still says historical VS Code imports are unsupported, and `website/reference/cli.md` documents `mempalace hook` as Claude/Codex-only with only three hook names.

**Target state**: The website surfaces consistently describe the shipped VS Code MCP setup, the checked-in VS Code hook template/install path, live hook-driven ingest, and historical backfill via `mempalace mine <dir> --mode convos`, while keeping Copilot CLI out of scope.

**Delta**:

- **Modify**: `website/guide/vscode-copilot.md` (expand the guide from phase-1-only setup into the full MCP + hooks + live ingest + historical backfill story and remove stale not-yet-implemented claims)
- **Modify**: `website/guide/hooks.md` (document the VS Code Copilot hook template/install path and runtime behavior instead of saying unsupported)
- **Modify**: `website/guide/mining.md` (replace the stale VS Code caveat with historical backfill guidance)
- **Modify**: `website/reference/cli.md` (document the `vscode-copilot` harness, the five supported hook names, and the updated `mine --mode convos` support wording)

**Hotspots**:

- `website/guide/vscode-copilot.md` — modified by: primary guide expansion
- `website/guide/hooks.md` — modified by: VS Code hook guidance
- `website/guide/mining.md` — modified by: historical backfill wording
- `website/reference/cli.md` — modified by: hook and mining reference parity

**Blockers**:

- Docs that show the hook install path should trail the final `.github/hooks/vscode-copilot.json` template so JSON examples do not drift.
- Historical-backfill wording should trail the fixture-backed regression from the backfill area so docs do not outpace verification.

### Area: Repository docs, hook template, and agent guidance parity

**Current state**: `README.md` only points VS Code Copilot users at the external guide, `hooks/README.md` still says VS Code runtime-hook support is unavailable, and `.github/copilot-instructions.md` still forbids describing VS Code hooks, transcript normalization, and historical backfill as available. `.github/hooks/` contains only `capture.json`, a personal capture artifact with hard-coded `/home/michele/tmp/...` paths rather than a user-facing template. `examples/mcp_setup.md` and `mempalace/cli.py#cmd_mcp` already provide the canonical VS Code MCP snippet, and `mempalace/cli.py` already accepts `--harness vscode-copilot` with five hook names.

**Target state**: Root docs and agent-facing repo instructions reflect the shipped phase-2/3 behavior, with a checked-in `.github/hooks/vscode-copilot.json` as the primary VS Code hook configuration path and no stale capture artifact presented as guidance.

**Delta**:

- **Create**: `.github/hooks/vscode-copilot.json` (production hook template mapping `SessionStart`, `Stop`, `PreCompact`, `SubagentStart`, and `SubagentStop` to `mempalace hook run --hook ... --harness vscode-copilot`)
- **Modify**: `README.md` (expand the root VS Code entry point to mention MCP, hooks, and historical backfill with links into the canonical docs)
- **Modify**: `hooks/README.md` (replace the phase-1-only caveat with VS Code Copilot guidance that points to the template and explains the shell wrappers remain Claude/Codex-specific)
- **Modify**: `.github/copilot-instructions.md` (remove stale not-implemented bullets and align repository instructions with the shipped VS Code surface)
- **Modify**: `mempalace/instructions/help.md` (align hook/help output with VS Code Copilot runtime availability)
- **Modify**: `mempalace/instructions/mine.md` (align conversation-mining instruction text with VS Code Copilot support)
- **Delete**: `.github/hooks/capture.json` (retire the personal capture artifact after replacing it with a production-safe template)

**Hotspots**:

- `.github/hooks/vscode-copilot.json` — modified by: new primary user-facing hook template
- `hooks/README.md` — modified by: template installation docs and runtime-surface clarification
- `.github/copilot-instructions.md` — modified by: agent-facing availability rules
- `README.md` — modified by: root entry-point parity

**Blockers**:

- The production template must use the verified VS Code event keys from the current capture artifact and fixture set, so template creation should precede docs that quote it.
- `.github/copilot-instructions.md` and the public docs should land together so the repository no longer tells Copilot and humans different stories about VS Code support.

## Cross-cutting Hotspots

Files touched by 3+ areas (aggregated across all areas above). These drive the Planner's shared file parallelism check.

- None identified.

## Dead code

- `.github/hooks/capture.json` — personal capture config with hard-coded local paths; it is not suitable as a user-facing template once `.github/hooks/vscode-copilot.json` exists.

## Open questions from discovery

- None. The remaining scope decisions were resolved in-chat: phase 3 should ship a dedicated `.github/hooks/vscode-copilot.json` template and should update `.github/copilot-instructions.md` to match the shipped VS Code surface.


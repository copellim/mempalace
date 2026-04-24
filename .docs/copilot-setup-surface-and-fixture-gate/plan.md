# Plan: Copilot Setup Surface and Fixture Gate

## Original Request

leggi la cartella .docs e verifica lo stato avanzamento, poi fai piano completo per implementazione integrazione mempalace con vscode copilot

## Approach

The `.docs` state now proves the VS Code Copilot raw capture step is complete: real `SessionStart`, `Stop`, `PreCompact`, `SubagentStart`, and `SubagentStop` payloads have been captured, along with transcript snapshots covering tool activity, precompact context, and subagent activity. Phase 1 should not pretend runtime integration exists yet. Instead, it should make the repository speak VS Code Copilot natively on the setup/documentation surface, and convert that raw capture set into committed sanitized fixtures required before later runtime work can claim support.

The phase therefore stays intentionally narrow:

- convert the now-complete raw VS Code fixture set into committed sanitized fixtures and record its provenance/sanitization rules, including subagent lifecycle payloads and transcript examples
- add Copilot-specific repository instructions
- make the MCP setup path explicit in the CLI helper and setup docs
- add a dedicated VS Code Copilot guide and route users to it
- add explicit boundary notes so no phase-1 surface implies runtime hooks, transcript normalization, or historical VS Code backfill are already shipped

This keeps momentum without violating the blocker the user set: the raw capture prerequisite is now closed, so the remaining fixture-gate blocker is sanitizing and committing those captures before dependent runtime implementation.

## Impacted Areas

- Fixture gate: extend `tests/fixtures/vscode_copilot/` with sanitized forms of the captured raw set and document exactly what was captured and sanitized, including subagent lifecycle hook payloads.
- Repository instruction surface: add `.github/copilot-instructions.md` so Copilot is guided toward MemPalace’s MCP-first, verbatim, local-first behavior.
- MCP setup surface: extend `mempalace mcp`, `tests/test_cli.py`, `website/reference/cli.md`, and `examples/mcp_setup.md` so VS Code Copilot setup is first-class.
- User discovery surface: add `website/guide/vscode-copilot.md` and route `README.md`, `website/guide/getting-started.md`, and `website/guide/mcp-integration.md` toward it.
- Boundary docs: update `hooks/README.md`, `website/guide/hooks.md`, and `website/guide/mining.md` so they do not accidentally advertise unsupported VS Code runtime hooks or backfill in phase 1.

## Architecture to Reuse

- `.docs/vscode-copilot-integration-prep.md` — source of truth for the captured VS Code payload/transcript shape and sanitization workflow.
- `.docs/vscode-copilot-progress.md` — source of truth for current progress, captured raw set, and the remaining sanitization/commit work.
- `mempalace/cli.py#cmd_mcp` — existing setup-helper surface to extend rather than replace.
- `mempalace/mcp_server.py` — authoritative MCP tool names and memory protocol language for Copilot instructions.
- `tests/fixtures/vscode_copilot/` — existing fixture subtree and naming pattern to extend.
- `website/guide/claude-code.md` and `website/guide/gemini-cli.md` — host-specific guide pattern for a new VS Code Copilot guide.
- `website/reference/cli.md` and `examples/mcp_setup.md` — existing MCP setup reference surfaces to extend for VS Code.

## Risks and Dependencies

- The raw capture blocker is closed, but sanitized `PreCompact`, `SubagentStart`/`SubagentStop`, and subagent/compaction fixtures are still absent from the repo, so later runtime claims remain blocked. Mitigation: make conversion of the captured raw set the first UOW and keep the fixture manifest explicit.
- User-facing docs can drift ahead of implementation. Mitigation: make the dedicated VS Code Copilot guide honest about current scope, and add explicit deferral notes to hooks/mining surfaces.
- Existing shell hooks are Claude/Codex-era and are not a safe basis for a user-facing VS Code template yet. Mitigation: do not ship a runnable VS Code hook template in phase 1.
- The repo does not yet verify a stable historical VS Code transcript path. Mitigation: do not document historical VS Code backfill as a runnable workflow in phase 1.

## Decisions

- Phase 1 remains documentation/setup + fixture gate only.
- `mempalace/hooks_cli.py`, `mempalace/normalize.py`, `tests/test_hooks_cli.py`, and `tests/test_normalize.py` are explicitly out of scope for this increment.
- No user-facing `.github/hooks/*.json` runtime template ships in phase 1.
- No user-facing phase-1 path claims live VS Code hook support, transcript normalization, or historical VS Code backfill.
- `website/guide/vscode-copilot.md` is the canonical phase-1 setup guide for VS Code Copilot.
- The phase uses six UOWs: fixture gate, repository instructions, MCP helper/reference, dedicated VS Code guide, entry-point cross-links, and deferred-runtime messaging cleanup.
- The feature extends the existing MemPalace codebase and setup/install surfaces; standalone workaround paths that bypass installing or using MemPalace are out of scope.


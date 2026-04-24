# Plan: VS Code Copilot Integration

## Original Request

leggi la cartella .docs e verifica lo stato avanzamento, poi fai piano completo per implementazione integrazione mempalace con vscode copilot

## Approach

Use a phase-based roadmap that separates already-shipped setup work from the remaining runtime and historical-ingest work, while preserving the committed VS Code fixtures as the handoff boundary between phases. Phase 1 establishes the MCP/setup/documentation surface and commits the sanitized VS Code payload and transcript fixtures; Phase 2 adds the runtime hook harness, transcript normalization, and live ingest on top of those verified inputs; Phase 3 completes historical backfill support and docs/reference parity by reusing the same normalization and `mine --mode convos` path instead of inventing a second integration surface.

## Impacted Areas

- Setup and instruction surface: MCP setup, Copilot instructions, and the durable VS Code entry points already established in Phase 1.
- Hook runtime and CLI surface: `mempalace hook run`, `mempalace/hooks_cli.py`, and their regression tests for harness parsing, stop/precompact behavior, and subagent lifecycle handling.
- Transcript normalization and mining reuse: `mempalace/normalize.py` and the existing `mempalace mine <dir> --mode convos` path reused for both live ingest and historical backfill.
- Fixture and regression-test surface: `tests/fixtures/vscode_copilot/`, `tests/test_cli.py`, `tests/test_hooks_cli.py`, and `tests/test_normalize.py`.
- Final parity surface: README, website guides, hooks docs, CLI reference, repository instructions, and the primary VS Code hook template for phase 3 parity.

## Architecture to Reuse

- `.github/copilot-instructions.md` — the repository-level Copilot guidance established in Phase 1.
- `website/guide/vscode-copilot.md` — the canonical user-facing VS Code Copilot entry point.
- `mempalace/cli.py#cmd_mcp` and `mempalace/cli.py#cmd_hook` — the public CLI entry points for MCP setup and hook execution.
- `mempalace/hooks_cli.py` — the current stop/precompact hook runtime, diary checkpointing, and transcript-ingest entry point.
- `mempalace/normalize.py` — the shared transcript normalization chain reused by live ingest and backfill.
- `mempalace/convo_miner.py` — the existing `mine --mode convos` path that should remain the reuse point for historical backfill.
- `tests/fixtures/vscode_copilot/` — the committed VS Code payload and transcript fixtures that gate downstream runtime implementation.

## Risks and Dependencies

- Phase 2 depends on the committed sanitized VS Code fixtures from Phase 1; no new raw-capture work should reopen that dependency.
- `mempalace/hooks_cli.py` and `tests/test_hooks_cli.py` are the main shared hotspots, so runtime work should be sliced to minimize symbol overlap and regression risk.
- Historical backfill should keep reusing `mempalace mine <dir> --mode convos`; introducing a second backfill path would fragment the integration.
- Docs must stay aligned with shipped behavior so phase 3 closes the remaining parity gap without diverging from the runtime already implemented in phase 2.
- Phase 3 depends on the normalizer and harness work from Phase 2 but should remain a parity pass, not a redesign of the runtime path.

## Decisions

- Scope stays limited to VS Code Copilot, not Copilot CLI.
- The canonical durable plan folder is derived from the plan title: `vs-code-copilot-integration`.
- Phase 1 and Phase 2 are done, and Phase 3 is planned.
- The session-memory requirements brief and gap analysis are persisted verbatim in this folder so future chats can resume planning without rerunning full discovery.
- Historical backfill remains coupled to the existing `mempalace mine <dir> --mode convos` flow.


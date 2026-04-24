# Requirements Brief: VS Code Copilot Integration

## Original Request

leggi la cartella .docs e verifica lo stato avanzamento, poi fai piano completo per implementazione integrazione mempalace con vscode copilot

## Functional Requirements

- **R-1**: A user can configure MemPalace as an MCP server for VS Code Copilot using repository-provided setup guidance, and verify that MemPalace tools load successfully in Copilot.
- **R-2**: The repository provides Copilot-specific instructions for VS Code so Copilot is guided to use MemPalace consistently with MemPalace product constraints.
- **R-3**: MemPalace hook runtime accepts VS Code Copilot hook payloads and supports the VS Code lifecycle needed for the integration roadmap, including `SessionStart`, `Stop`, `PreCompact`, `SubagentStart`, and `SubagentStop`, with `vscode-copilot` as a first-class harness.
- **R-4**: VS Code Copilot transcripts are normalized into MemPalace's standard conversation transcript format by preserving canonical user/assistant turns, merging assistant multi-part turns, and removing tool chrome/noise according to current project transcript rules.
- **R-5**: Live VS Code Copilot sessions are ingested into the palace from `transcript_path` during hook-driven saves/checkpoints without blocking the chat on ingest errors.
- **R-6**: Historical VS Code Copilot sessions can be backfilled through the existing `mempalace mine <dir> --mode convos` flow once VS Code transcript support is implemented.
- **R-7**: The integration ships with fixture-driven automated tests and end-user documentation covering setup, hooks, transcript ingest, and historical backfill for VS Code Copilot.
- **R-8**: The real-world VS Code `PreCompact` and subagent hook/transcript raw captures are available, and they are sanitized and committed before any runtime implementation phase that depends on those shapes proceeds.

## Authorization Rules

For each write operation:

- **Operation**: Write live checkpoints, transcript ingest output, and state files during VS Code hook execution
- **Who**: The current local OS user running MemPalace and VS Code Copilot
- **Ownership check**: Input transcript and target palace/state paths must be local paths accessible to the current user; no cross-user or remote authorization model is introduced
- **On failure**: Local pass-through behavior for hooks with local logging/error reporting; no application-level 403/404 contract
- **Operation**: Import historical VS Code Copilot sessions into the palace
- **Who**: The current local OS user invoking `mempalace mine`
- **Ownership check**: The source transcript directory must be provided/readable by the current user and the target palace path must be writable by the current user
- **On failure**: Local CLI/log error handling only; no application-level 403/404 contract

## Scope Boundaries

- **In scope**: VS Code Copilot only; MCP setup guidance; Copilot instructions; VS Code hook harness support; transcript normalization; live ingest via hooks; historical backfill via existing `mine --mode convos`; docs/examples; fixtures and regression tests; sanitization/commit of the captured VS Code `PreCompact` and subagent raw fixture set as a gating phase
- **Out of scope**: Copilot CLI support; new remote/cloud authorization features; a new dedicated backfill command unless discovery proves the existing mining flow insufficient; changes unrelated to VS Code Copilot integration

## Naming Conventions

Feature-specific naming decisions (new slice names, new type suffixes introduced by this feature, new route names). General codebase conventions (e.g. "all repositories use `*Repository` suffix") are NOT captured here — they belong to the Codebase Scout output during implementation.

- Harness name: `vscode-copilot`
- Fixture subtree: `tests/fixtures/vscode_copilot/`
- User-facing docs and plan language should say `VS Code Copilot`, not generic `Copilot`, unless referring to existing non-VS Code surfaces explicitly

## Non-Functional Constraints

- VS Code Copilot is the only target surface for this plan
- The plan must start from the current `.docs` status and treat the existing captured schema/fixtures as already completed work
- The raw `PreCompact` and subagent captures now exist, but committed sanitized fixtures remain the blocker for dependent runtime work, and the roadmap should still be planned end-to-end now
- Primary hook configuration path should be `.github/hooks/*.json`; Claude-compatible config can be documented only as compatibility context if needed
- Historical backfill should reuse `mempalace mine <dir> --mode convos` unless discovery proves that insufficient
- Hook behavior must remain fail-open on local ingest/runtime errors
- Existing Claude/Codex behavior must not regress
- Hook latency budget remains under 500ms; startup injection budget remains under 100ms
- The integration must remain local-first, zero-API, and consistent with MemPalace's transcript normalization rules
- The feature extends the existing MemPalace codebase and installation/setup surfaces; standalone workarounds that bypass installing or using MemPalace are out of scope.

## Open Assumptions

- The existing `mempalace mine <dir> --mode convos` flow can cover VS Code historical backfill once normalization support lands, because the user chose reuse over introducing a new command.
- The current project transcript standard should be applied to VS Code transcripts, because the user asked to follow project standards rather than define VS Code-specific transcript semantics.
- No additional authorization model is required beyond the current local process/user boundary, because the integration is a local developer workflow rather than a multi-user application feature.

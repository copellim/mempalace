# VS Code Copilot integration plan

## Problem statement

MemPalace already has the core building blocks for a strong GitHub Copilot integration:

- MCP server
- hook runtime
- transcript ingest pipeline
- project bootstrap/mining
- repository-level agent instructions

But the current implementation is still Claude/Codex-first. For **VS Code Copilot**, the goal is to deliver a complete integration that feels first-class across the full MemPalace workflow:

1. connect MemPalace as an MCP server in VS Code
2. teach Copilot how to use MemPalace correctly
3. autosave/checkpoint ongoing work through hooks
4. ingest current session transcripts into the palace
5. backfill/import historical VS Code Copilot sessions
6. document bootstrap + ongoing usage clearly
7. cover the new behavior with fixtures and tests

This plan is intentionally scoped to **VS Code Copilot only**. Copilot CLI is out of scope for this plan.

## Assumptions

- Include **historical backfill/import** of existing VS Code Copilot sessions.
- Use **native VS Code Copilot hooks** as the primary target, even if some Claude-format compatibility remains useful.
- Treat **complete integration** as including:
  - MCP setup
  - Copilot-specific instructions
  - stop/precompact autosave
  - transcript ingest
  - historical backfill
  - docs/examples
  - tests

## Proposed approach

Deliver the integration in four layers:

### Layer 1: Copilot-facing product surface

Make the repository speak Copilot natively:

- add `.github/copilot-instructions.md`
- make setup docs and helper output mention VS Code Copilot explicitly
- add VS Code-specific examples for MCP and hooks

### Layer 2: Hook lifecycle support

Add a `vscode-copilot` harness to the existing MemPalace hook engine and map VS Code hook payloads onto MemPalace’s internal shape:

- `sessionId` -> `session_id`
- `transcript_path` -> `transcript_path`
- `stop_hook_active` -> `stop_hook_active`

Use VS Code `Stop` and `PreCompact` to preserve the current MemPalace autosave model.

### Layer 3: Transcript ingest and backfill

Build first-class ingestion for VS Code Copilot transcripts:

- detect transcript format
- normalize user/assistant turns
- ignore tool/system noise
- support both ongoing hook-driven ingest and historical backfill

### Layer 4: Validation and docs

Add realistic fixtures, regression tests, docs, and examples so the integration is stable and usable by contributors and end users.

## Detailed work plan

### 1. Capture and characterize real VS Code Copilot data

**Why first:** transcript ingest and some hook details should be driven by real fixtures, not assumptions.

Tasks:

- collect real VS Code Copilot hook payloads for:
  - `SessionStart`
  - `Stop`
  - `PreCompact`
- capture real files pointed to by `transcript_path`
- identify:
  - message event shapes
  - tool result noise
  - system message noise
  - where canonical user/assistant text lives
- write down the stable assumptions we can code against

Deliverables:

- fixture samples for hook payloads
- fixture samples for transcript files
- short developer note summarizing transcript shape and noise patterns

Dependencies:

- blocks the transcript normalizer
- informs test coverage and backfill design

### 2. Add repository-level Copilot instructions

Tasks:

- create `.github/copilot-instructions.md`
- make instructions explicitly tell Copilot to:
  - consult MemPalace before answering about prior work
  - preserve verbatim behavior expectations
  - use MCP tools where available
  - avoid guessing when memory is relevant
- review `AGENTS.md` so repo-wide guidance is not unnecessarily Claude-branded where generic agent guidance belongs
- decide whether to add path-specific instructions under `.github/instructions/` for docs, hooks, or transcript parsing areas

Deliverables:

- `.github/copilot-instructions.md`
- any required edits to `AGENTS.md`
- optional path-specific instruction files

Dependencies:

- independent

### 3. Make MCP setup first-class for VS Code Copilot

Tasks:

- update `mempalace mcp` output to include:
  - VS Code MCP configuration guidance
  - direct server command examples
  - pointer to workspace/user-level config locations where appropriate
- add VS Code setup documentation:
  - install/configure MCP server
  - verify tools are loaded
  - expected Copilot behavior once connected
- update existing generic MCP docs so VS Code is not implied rather than explicit
- ensure README/getting-started point users to the new VS Code path

Deliverables:

- improved `mempalace mcp`
- updated docs for VS Code MCP setup
- refreshed README/getting-started references

Dependencies:

- independent of transcript work

### 4. Add `vscode-copilot` hook harness support

Tasks:

- extend `SUPPORTED_HARNESSES` in `mempalace/hooks_cli.py`
- implement VS Code payload parsing:
  - `sessionId`
  - `transcript_path`
  - `stop_hook_active`
- confirm hook output shape expected by VS Code for:
  - user-facing `systemMessage`
  - blocking stop behavior
  - precompact pass-through after ingest/save
- keep the internal hook flow aligned with current logic:
  - periodic checkpoint on stop
  - loop prevention
  - emergency save on precompact
- decide whether VS Code should use the same shell wrappers or rely directly on `mempalace hook run`

Deliverables:

- working `vscode-copilot` harness
- example hook config under `.github/hooks/*.json`
- docs showing exact setup

Dependencies:

- mostly independent
- benefits from real hook payload capture

### 5. Implement VS Code transcript normalization

Tasks:

- add VS Code transcript detection to `normalize.py` (or the chosen source-adapter path if aligning with the RFC)
- parse canonical user/assistant turns
- drop or merge tool noise appropriately
- preserve verbatim conversation content
- ensure output shape matches the existing conversation mining pipeline
- decide how to handle:
  - tool-only assistant messages
  - multi-part assistant turns
  - system reminders/hook chrome
  - subagent content if present

Deliverables:

- VS Code transcript normalizer
- transcript fixtures
- unit tests for normalization and edge cases

Dependencies:

- depends on real transcript samples

### 6. Wire ongoing ingest for active VS Code sessions

Tasks:

- ensure stop-based checkpoints can ingest the current transcript via `transcript_path`
- ensure precompact ingest runs before context loss
- verify existing diary/checkpoint logic works with the VS Code harness
- confirm the output/notification experience is appropriate in VS Code chat
- decide whether any VS Code-specific throttling or save-interval adjustments are needed

Deliverables:

- working ongoing autosave + ingest path for active VS Code sessions
- tests for stop/precompact behavior with VS Code-shaped payloads

Dependencies:

- depends on harness support
- depends on transcript normalizer

### 7. Add historical backfill/import for VS Code Copilot sessions

Tasks:

- define how users point MemPalace at historical VS Code Copilot session files
- decide whether backfill should be:
  - automatic from a known directory
  - documented manual `mempalace mine ... --mode convos` flow
  - a dedicated helper flow
- make the VS Code transcript normalizer work both for live hook-driven ingest and offline backfill
- document the one-time bootstrap/backfill workflow clearly

Deliverables:

- documented historical import path
- any required code to make offline backfill practical
- tests for fixture-based historical imports

Dependencies:

- depends on transcript normalizer

### 8. Add VS Code-native docs and examples

Tasks:

- create or update a dedicated VS Code Copilot guide
- document:
  - bootstrap ingest of project files/docs/history
  - MCP configuration
  - hook installation
  - autosave behavior
  - precompact behavior
  - historical backfill
  - verification/troubleshooting flow
- update hooks docs to include VS Code examples rather than only Claude/Codex
- update examples to show `.github/hooks/*.json` usage

Deliverables:

- VS Code Copilot guide
- updated hooks guide
- updated MCP guide
- example configuration files/snippets

Dependencies:

- should land after the code paths are known enough to document accurately

### 9. Add test coverage and regression protection

Tasks:

- add fixture-driven tests for:
  - VS Code hook payload parsing
  - stop hook behavior
  - precompact hook behavior
  - transcript normalization
  - ongoing ingest
  - historical backfill
- add regression tests around:
  - loop prevention
  - silent save mode
  - partial/invalid transcript handling
  - tool-noise filtering

Deliverables:

- expanded automated test coverage
- representative fixtures checked into the test suite

Dependencies:

- depends on implementation details from workstreams 4–7

### 10. Run end-to-end parity review for the VS Code user journey

Tasks:

- verify the full flow from a fresh repo:
  - install MemPalace
  - bootstrap project ingest
  - connect MCP in VS Code
  - add hooks
  - run a live session
  - observe checkpoint save
  - observe precompact save
  - search recalled data
  - import past sessions
- identify any last-mile UX gaps:
  - naming
  - documentation ambiguity
  - missing examples
  - brittle assumptions

Deliverables:

- final issue list or polish pass
- confidence that the VS Code integration is first-class, not partial

Dependencies:

- last step after all major code and docs changes

## Proposed execution order

### Phase A: foundation

1. Capture real VS Code data
2. Add Copilot instructions
3. Improve VS Code MCP setup/docs

### Phase B: runtime support

4. Add `vscode-copilot` harness
5. Implement transcript normalizer
6. Wire ongoing ingest for live sessions

### Phase C: historical coverage

7. Implement/document backfill of existing VS Code sessions

### Phase D: quality and polish

8. Finalize VS Code docs/examples
9. Add/finish tests
10. Run end-to-end parity review

## Notes and open questions

- The most important unknown is the exact **VS Code transcript file shape** behind `transcript_path`.
- If VS Code already reads `.claude/settings.json`, we still need to decide whether the recommended MemPalace path should be:
  - native `.github/hooks/*.json`
  - Claude-compatible config
  - both, with one documented as preferred
- If subagent content appears in transcripts, decide whether it should be:
  - folded into the main conversation
  - ignored
  - stored separately
- We should prefer using **real captured fixtures** over inferred schemas whenever possible.

## Success criteria

The VS Code Copilot integration should be considered complete when:

- MemPalace is easy to connect as an MCP server in VS Code
- Copilot receives correct repo-level memory instructions
- stop/precompact hooks work reliably in VS Code
- active session transcripts are ingested into MemPalace
- historical VS Code Copilot sessions can be backfilled/imported
- docs explain both initial bootstrap and ongoing memory flows
- tests cover the new behavior end to end

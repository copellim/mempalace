# Increment 1: Copilot Setup Surface and Fixture Gate

**Depends on**: none
**Acceptance Verification**:

1. `python -m pytest tests/test_cli.py -k mcp -v`
2. Validate the new VS Code fixtures and manifest: parse `tests/fixtures/vscode_copilot/hooks/precompact.json`, `tests/fixtures/vscode_copilot/hooks/subagent_start.json`, and `tests/fixtures/vscode_copilot/hooks/subagent_stop.json` as JSON; parse each line of `tests/fixtures/vscode_copilot/transcripts/with_subagent.jsonl` and `tests/fixtures/vscode_copilot/transcripts/with_precompact_context.jsonl` as JSON; confirm the manifest records the exact subagent hook/event names and precompact correlation metadata.
3. Audit user-facing docs with `rg` to confirm no phase-1 VS Code setup path instructs users to install `.github/hooks/*.json`, run `hooks/mempal_save_hook.sh`, run `hooks/mempal_precompact_hook.sh`, or import historical VS Code transcripts as a supported workflow.
4. Build the website docs if the environment supports it: `cd website && npm run docs:build`.
5. Manually confirm `README.md`, `website/guide/getting-started.md`, `website/guide/mcp-integration.md`, and `website/guide/vscode-copilot.md` point to the same honest setup story: MCP setup and tool verification are available now; runtime hooks/normalization/backfill are deferred.

## Requirements Matrix (UOW-level)

| Req ID | Requirement (summary)                                                                                                                            | UOW                   |
| ------ | ------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------- |
| R-8    | The complete real-world VS Code PreCompact and subagent hook/transcript raw fixture set is sanitized and committed before dependent runtime work | UOW-1                 |
| R-2    | Repository provides Copilot-specific instructions for VS Code consistent with MemPalace constraints                                              | UOW-2                 |
| R-1    | User can configure MemPalace as an MCP server for VS Code Copilot with repository-provided guidance                                              | UOW-3                 |
| R-1    | User can configure MemPalace as an MCP server for VS Code Copilot with repository-provided guidance                                              | UOW-4 (primary owner) |
| R-1    | User can configure MemPalace as an MCP server for VS Code Copilot with repository-provided guidance                                              | UOW-5                 |
| R-1    | User can configure MemPalace as an MCP server for VS Code Copilot with repository-provided guidance                                              | UOW-6                 |

---

## UOW-1: Finalize VS Code Fixture Gate From Captured Raw Set

**Objective**: Convert the now-complete real VS Code Copilot raw capture set, including subagent lifecycle payloads, into committed sanitized fixtures and record exactly what was captured and sanitized so later runtime work can rely on verified inputs.
**covers**: [R-8]
**depends_on**: none
**parallel_group**: A

**Current status**: The raw prerequisite is already satisfied outside the repo: real `PreCompact`, `SubagentStart`, `SubagentStop`, subagent transcript, and precompact-correlated transcript captures now exist under `~/tmp/mempal-vscode-capture/`.

### Scope

- **Create**: `tests/fixtures/vscode_copilot/README.md` — fixture manifest describing provenance, sanitization, exact subagent event names, and precompact payload/transcript correlation rules.
- **Create**: `tests/fixtures/vscode_copilot/hooks/precompact.json` — sanitized real VS Code `PreCompact` hook payload.
- **Create**: `tests/fixtures/vscode_copilot/hooks/subagent_start.json` — sanitized real VS Code `SubagentStart` hook payload.
- **Create**: `tests/fixtures/vscode_copilot/hooks/subagent_stop.json` — sanitized real VS Code `SubagentStop` hook payload.
- **Create**: `tests/fixtures/vscode_copilot/transcripts/with_subagent.jsonl` — sanitized real transcript containing subagent activity.
- **Create**: `tests/fixtures/vscode_copilot/transcripts/with_precompact_context.jsonl` — sanitized real transcript correlated with the `PreCompact` payload.
- **Modify**: `.docs/vscode-copilot-integration-prep.md` — align the documented committed-fixture list and raw-capture status with the committed manifest.
- **Do not touch**: `mempalace/hooks_cli.py` — runtime harness work belongs to a later increment.
- **Do not touch**: `mempalace/normalize.py` — runtime normalizer work belongs to a later increment.
- **Do not touch**: `tests/test_hooks_cli.py` — runtime hook tests belong to a later increment.
- **Do not touch**: `tests/test_normalize.py` — runtime normalization tests belong to a later increment.

### Contracts

- No code/public-member contracts are introduced. The deliverables are static fixture files and a fixture manifest whose contract is structural fidelity to the real captures plus explicit documentation of sanitization, subagent hook naming, and correlation rules.

### Negative Constraints

- Do NOT invent subagent hook names, transcript event names, or payload fields.
- Do NOT flatten, summarize, or clean away realistic tool noise from the transcripts.
- Do NOT alter the existing authoritative fixtures `session_start.json`, `stop.json`, or `simple_with_tools.jsonl`.
- Do NOT add runtime code, runtime tests, or user-facing setup docs in this UOW.

### Operational Sequence

1. Sanitize the captured real VS Code `PreCompact` payload.
2. Sanitize the captured real VS Code `SubagentStart` and `SubagentStop` payloads.
3. Sanitize the captured real VS Code transcript with subagent activity.
4. Sanitize the captured real VS Code transcript correlated with a `PreCompact` event.
5. Create `tests/fixtures/vscode_copilot/README.md` documenting the exact hook/event names, field-casing notes, raw-source provenance, and precompact correlation metadata.
6. Update `.docs/vscode-copilot-integration-prep.md` so it matches the committed fixture set, manifest, and closed raw-capture status.

### Acceptance Criteria

- [ ] `tests/fixtures/vscode_copilot/README.md` lists every committed VS Code fixture and marks which were newly added in this increment.
- [ ] `tests/fixtures/vscode_copilot/hooks/subagent_start.json` and `tests/fixtures/vscode_copilot/hooks/subagent_stop.json` both parse as valid JSON and preserve the real sanitized hook-event names.
- [ ] The manifest contains a `Subagent hook payloads` section naming the exact hook-event values observed in the raw subagent captures.
- [ ] The manifest contains a `Subagent transcript event types` section naming the exact event type strings observed in the raw subagent transcript capture.
- [ ] `tests/fixtures/vscode_copilot/transcripts/with_subagent.jsonl` contains every event type named in the manifest’s `Subagent transcript event types` section at least once.
- [ ] The manifest contains a `PreCompact correlation` section recording the payload filename, transcript filename or `transcript_path` basename, observed session-id field/value, observed hook-event field/value, and timestamp matching rule used during sanitization.
- [ ] `tests/fixtures/vscode_copilot/hooks/precompact.json` parses as valid JSON, represents a real sanitized `PreCompact` payload, and includes `trigger: "auto"`.
- [ ] `tests/fixtures/vscode_copilot/transcripts/with_precompact_context.jsonl` parses as JSONL, matches the manifest’s precompact-correlation record, and carries the same sanitized session identifier as `precompact.json`.
- [ ] `.docs/vscode-copilot-integration-prep.md` matches the committed manifest and fixture set.

### Verification

- `python3 -c "import json; json.load(open('tests/fixtures/vscode_copilot/hooks/precompact.json'))"`
- `python3 -c "import json; json.load(open('tests/fixtures/vscode_copilot/hooks/subagent_start.json')); json.load(open('tests/fixtures/vscode_copilot/hooks/subagent_stop.json'))"`
- `python3 -c "import json; [json.loads(line) for line in open('tests/fixtures/vscode_copilot/transcripts/with_subagent.jsonl', encoding='utf-8')]"`
- `python3 -c "import json; [json.loads(line) for line in open('tests/fixtures/vscode_copilot/transcripts/with_precompact_context.jsonl', encoding='utf-8')]"`
- Manual comparison of `tests/fixtures/vscode_copilot/README.md` against the committed fixtures and `.docs/vscode-copilot-integration-prep.md`.

## UOW-2: Add Repository-Level Copilot Instructions

**Objective**: Create repository-scoped Copilot instructions that teach VS Code Copilot how to use MemPalace correctly in this repository.
**covers**: [R-2]
**depends_on**: none
**parallel_group**: A

### Scope

- **Create**: `.github/copilot-instructions.md` — repository-level instructions for GitHub Copilot in this repo.
- **Do not touch**: `.github/instructions/` — path-specific instruction files are unnecessary in this phase.
- **Do not touch**: `mempalace/hooks_cli.py` — runtime hook work belongs to a later increment.
- **Do not touch**: `mempalace/normalize.py` — runtime normalization work belongs to a later increment.

### Contracts

- No code/public-member contracts are introduced. The artifact contract is behavioral guidance only.

### Negative Constraints

- Do NOT claim that VS Code runtime hooks, transcript normalization, or historical backfill are already available.
- Do NOT reference MCP tool names that are not verified in `mempalace/mcp_server.py`.
- Do NOT duplicate the entire project philosophy from `AGENTS.md`; link or defer where broader repository context is already documented.

### Operational Sequence

1. Read `mempalace/mcp_server.py` to verify the MCP tool names and the existing Memory Protocol language.
2. Create `.github/copilot-instructions.md` explaining when Copilot must search memory before answering.
3. Encode MemPalace’s verbatim and local-first constraints in Copilot-facing language.
4. Ensure the instructions stay within phase-1 scope and do not imply unimplemented runtime behavior.

### Acceptance Criteria

- [ ] `.github/copilot-instructions.md` exists.
- [ ] The file instructs Copilot to consult MemPalace before answering about prior work, prior decisions, people, or project history.
- [ ] The file tells Copilot not to guess when memory is relevant.
- [ ] The file preserves MemPalace’s verbatim and local-first constraints.
- [ ] The file contains no claims of shipped VS Code runtime hooks, transcript normalization, or historical backfill.

### Verification

- `test -f .github/copilot-instructions.md`
- `grep -q "mempalace_search" .github/copilot-instructions.md`
- `grep -q "verbatim" .github/copilot-instructions.md`
- `grep -q "local" .github/copilot-instructions.md`
- Manual read to confirm no phase-1 runtime/backfill claim appears.

## UOW-3: Extend the MCP Helper and Reference Surface

**Objective**: Make the `mempalace mcp` helper and its reference surfaces explicitly useful for VS Code Copilot users.
**covers**: [R-1]
**depends_on**: none
**parallel_group**: A

### Scope

- **Modify**: `mempalace/cli.py` — extend `cmd_mcp` output for VS Code Copilot while preserving current behavior.
- **Modify**: `tests/test_cli.py` — add focused coverage for the updated `cmd_mcp` output.
- **Modify**: `website/reference/cli.md` — document the updated `mempalace mcp` output and verification path.
- **Modify**: `examples/mcp_setup.md` — add a VS Code-oriented MCP setup example/note consistent with the CLI helper.
- **Do not touch**: `mempalace/hooks_cli.py` — runtime hook support belongs to a later increment.
- **Do not touch**: `mempalace/normalize.py` — transcript normalization belongs to a later increment.

### Contracts

- `cmd_mcp(args)` — body extended; function signature remains unchanged.

### Behavioral Specification

**Preconditions**:

- MemPalace is installed and the `mcp` subcommand is invoked.
- `args.palace` may be absent or may contain a custom palace path.

**Input validation**:
| Field | Rule | On violation |
|-------|------|--------------|
| `args.palace` | Optional. If present, expand `~` and render the path safely in both shell and JSON examples. | Preserve current best-effort path handling; no new error mode introduced. |

**Behavior**:

- On success without a custom palace path: stdout includes the existing direct-server guidance plus a VS Code Copilot MCP setup example and a concrete tool-verification step.
- On success with a custom palace path: stdout includes the same VS Code setup shape with the custom palace path rendered correctly.
- Existing non-VS Code setup guidance remains intact.
- No runtime hook, transcript-ingest, or backfill claims are added.

**Side effects**:

- Stdout only. No files or runtime state are mutated.

### Negative Constraints

- Do NOT introduce a new CLI subcommand or flag.
- Do NOT print runnable VS Code hook configuration in this increment.
- Do NOT mention historical VS Code transcript import as if it is already supported.

### Operational Sequence

1. Extend `cmd_mcp` to print a VS Code Copilot-oriented MCP setup example.
2. Add a concrete tool-verification step to the helper output.
3. Update `tests/test_cli.py` to lock the new output shape.
4. Update `website/reference/cli.md` and `examples/mcp_setup.md` to match the helper output.

### Acceptance Criteria

- [ ] Running `mempalace mcp` prints a VS Code Copilot-relevant setup path.
- [ ] Running `mempalace mcp --palace /tmp/test-palace` prints the same setup path with the custom palace variant rendered correctly.
- [ ] `tests/test_cli.py` covers the updated `cmd_mcp()` output.
- [ ] `website/reference/cli.md` and `examples/mcp_setup.md` match the new helper output.
- [ ] None of these surfaces claim runtime hooks, transcript normalization, or historical VS Code backfill support.

### Verification

- `python -m pytest tests/test_cli.py -k mcp -v`
- `python3 -c "from argparse import Namespace; from mempalace.cli import cmd_mcp; cmd_mcp(Namespace(palace=None))"`
- `python3 -c "from argparse import Namespace; from mempalace.cli import cmd_mcp; cmd_mcp(Namespace(palace='/tmp/test-palace'))"`

## UOW-4: Create the Dedicated VS Code Copilot Guide

**Objective**: Publish a canonical phase-1 guide for VS Code Copilot that covers MCP setup, tool verification, and repository instructions without promising unimplemented runtime features.
**covers**: [R-1]
**depends_on**: [UOW-2, UOW-3]
**parallel_group**: B

### Scope

- **Create**: `website/guide/vscode-copilot.md` — canonical phase-1 guide for VS Code Copilot.
- **Modify**: `website/.vitepress/config.mts` — add the new guide to website navigation.
- **Do not touch**: `.github/hooks/` — no runnable VS Code runtime hook template ships in this phase.
- **Do not touch**: `mempalace/hooks_cli.py` — runtime hook support belongs to a later increment.
- **Do not touch**: `mempalace/normalize.py` — transcript normalization belongs to a later increment.

### Contracts

- No code/public-member contracts are introduced. The artifact contract is a canonical documentation page plus a navigation entry.

### Negative Constraints

- Do NOT include `.github/hooks/*.json` install steps as a supported MemPalace runtime workflow.
- Do NOT tell users to run `hooks/mempal_save_hook.sh` or `hooks/mempal_precompact_hook.sh` for VS Code Copilot.
- Do NOT present live transcript ingest or historical VS Code backfill as already available.

### Operational Sequence

1. Create `website/guide/vscode-copilot.md` covering MCP setup, tool verification, repository instructions, and explicit status boundaries.
2. Link users to the existing generic docs where needed instead of duplicating every setup detail.
3. Add the guide to `website/.vitepress/config.mts` navigation.
4. Ensure the guide is the canonical owner of the phase-1 VS Code setup story.

### Acceptance Criteria

- [ ] `website/guide/vscode-copilot.md` exists.
- [ ] The guide explains how to connect MemPalace to VS Code Copilot through MCP.
- [ ] The guide explains how to verify that MemPalace tools are available in Copilot.
- [ ] The guide references repository instructions and explains their role.
- [ ] The guide contains no runnable VS Code runtime-hook installation path and no claim of shipped historical VS Code backfill.
- [ ] `website/.vitepress/config.mts` links to the guide.

### Verification

- `grep -n "vscode-copilot" website/.vitepress/config.mts`
- `grep -n "MCP" website/guide/vscode-copilot.md`
- Manual content audit of `website/guide/vscode-copilot.md` for phase-1 scope honesty.

## UOW-5: Route Entry-Point Docs to the Dedicated Guide

**Objective**: Make the repository and onboarding entry points send VS Code Copilot users to the canonical guide instead of leaving them on generic surfaces.
**covers**: [R-1]
**depends_on**: [UOW-4]
**parallel_group**: C

### Scope

- **Modify**: `README.md` — explicitly route GitHub Copilot users to the dedicated VS Code guide.
- **Modify**: `website/guide/getting-started.md` — add the VS Code Copilot path to the onboarding flow.
- **Modify**: `website/guide/mcp-integration.md` — add VS Code Copilot as an explicit compatible setup path and route to the dedicated guide.
- **Do not touch**: `mempalace/hooks_cli.py` — runtime hook support belongs to a later increment.
- **Do not touch**: `mempalace/normalize.py` — transcript normalization belongs to a later increment.

### Contracts

- No code/public-member contracts are introduced. The artifact contract is documentation routing only.

### Negative Constraints

- Do NOT turn `README.md` into a full setup manual.
- Do NOT add runtime-hook or backfill claims to any of these entry-point surfaces.
- Do NOT create a second competing VS Code setup path outside the dedicated guide.

### Operational Sequence

1. Update `README.md` to call out VS Code Copilot as a first-class path and route readers to the dedicated guide.
2. Update `website/guide/getting-started.md` so Copilot users encounter the dedicated guide early.
3. Update `website/guide/mcp-integration.md` to reference VS Code Copilot explicitly and route to the canonical page.
4. Keep all three surfaces aligned to the same phase-1 scope.

### Acceptance Criteria

- [ ] `README.md` mentions VS Code Copilot and points users to the dedicated guide.
- [ ] `website/guide/getting-started.md` references the dedicated guide alongside the general MCP path.
- [ ] `website/guide/mcp-integration.md` contains a VS Code Copilot path that points to the dedicated guide.
- [ ] None of these files imply that VS Code runtime hooks, transcript normalization, or historical backfill are part of phase-1 setup.

### Verification

- `grep -n "VS Code Copilot" README.md`
- `grep -n "vscode-copilot" website/guide/getting-started.md`
- `grep -n "/guide/vscode-copilot" website/guide/mcp-integration.md`

## UOW-6: Add Deferred-Runtime Boundary Notes to Hook and Mining Docs

**Objective**: Prevent existing hook and mining docs from being misread as a supported VS Code Copilot runtime flow in phase 1.
**covers**: [R-1]
**depends_on**: [UOW-4]
**parallel_group**: C

### Scope

- **Modify**: `hooks/README.md` — add an explicit boundary note that current shell-hook guidance is for Claude Code/Codex, not a supported VS Code Copilot runtime path yet.
- **Modify**: `website/guide/hooks.md` — add the same explicit boundary note for website users.
- **Modify**: `website/guide/mining.md` — avoid presenting historical VS Code transcript import as currently supported; mark it deferred if mentioned.
- **Do not touch**: `hooks/mempal_save_hook.sh` — current shell hooks stay unchanged in this phase.
- **Do not touch**: `hooks/mempal_precompact_hook.sh` — current shell hooks stay unchanged in this phase.
- **Do not touch**: `mempalace/hooks_cli.py` — runtime hook support belongs to a later increment.
- **Do not touch**: `mempalace/normalize.py` — transcript normalization belongs to a later increment.

### Contracts

- No code/public-member contracts are introduced. The artifact contract is explicit scope-boundary messaging in existing docs.

### Negative Constraints

- Do NOT add runnable VS Code hook configuration to these docs.
- Do NOT tell users to import historical VS Code sessions as a supported current workflow.
- Do NOT weaken or remove valid Claude Code or Codex guidance that already reflects shipped behavior.

### Operational Sequence

1. Add an explicit phase-1 boundary note to `hooks/README.md`.
2. Add the equivalent boundary note to `website/guide/hooks.md`.
3. Update `website/guide/mining.md` so any future VS Code import mention is clearly deferred, not advertised as shipped.
4. Cross-check the wording against the dedicated VS Code guide so the repo tells one consistent story.

### Acceptance Criteria

- [ ] `hooks/README.md` explicitly states that the current shell-hook flow is for Claude Code and Codex, and that VS Code Copilot runtime-hook support is deferred.
- [ ] `website/guide/hooks.md` contains the same boundary note.
- [ ] `website/guide/mining.md` does not present historical VS Code transcript import as a currently supported workflow.
- [ ] No user-facing phase-1 doc instructs VS Code users to install `.github/hooks/*.json` or run the current shell hooks for MemPalace.

### Verification

- `rg -n "VS Code|Copilot|mempal_save_hook.sh|mempal_precompact_hook.sh|\.github/hooks|PreCompact" hooks/README.md website/guide`
- Manual review of matching lines to confirm every VS Code runtime mention is explicitly deferred.

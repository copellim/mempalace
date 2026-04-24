# Increment 3: Historical Backfill and Docs Parity

**Depends on**: increment 2 completed
**Acceptance Verification**:

1. `python -m pytest tests/test_convo_miner.py -v`
2. `python -m pytest tests/test_cli.py -v`
3. `python -m pytest tests/test_instructions_cli.py -v`
4. `python -m mempalace --palace <tmp-palace> mine tests/fixtures/vscode_copilot/transcripts --mode convos --wing vscode-fixture-test`
5. Parse `.github/hooks/vscode-copilot.json` as JSON and confirm `.github/hooks/capture.json` is absent.
6. Build the website docs if the environment supports it: `cd website && npm run docs:build`.
7. Manually confirm `README.md`, `.github/copilot-instructions.md`, `hooks/README.md`, `website/guide/vscode-copilot.md`, `website/guide/hooks.md`, `website/guide/mining.md`, `website/reference/cli.md`, `mempalace/instructions/help.md`, and `mempalace/instructions/mine.md` all describe the same `VS Code Copilot` MCP + hooks + live-ingest + historical-backfill flow, with no `Copilot CLI` reference.

## Requirements Matrix (UOW-level)

| Req ID | Requirement (summary)                                                                                                                                                      | UOW                    |
| ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- |
| R-6    | Historical VS Code Copilot sessions can be backfilled through the existing `mempalace mine <dir> --mode convos` flow                                                       | UOW-1 (primary owner)  |
| R-7    | The integration ships with fixture-driven automated tests and end-user documentation covering setup, hooks, transcript ingest, and historical backfill for VS Code Copilot | UOW-1                  |
| R-7    | The integration ships with fixture-driven automated tests and end-user documentation covering setup, hooks, transcript ingest, and historical backfill for VS Code Copilot | UOW-2                  |
| R-7    | The integration ships with fixture-driven automated tests and end-user documentation covering setup, hooks, transcript ingest, and historical backfill for VS Code Copilot | UOW-3                  |
| R-7    | The integration ships with fixture-driven automated tests and end-user documentation covering setup, hooks, transcript ingest, and historical backfill for VS Code Copilot | UOW-4                  |
| R-7    | The integration ships with fixture-driven automated tests and end-user documentation covering setup, hooks, transcript ingest, and historical backfill for VS Code Copilot | UOW-5                  |
| R-7    | The integration ships with fixture-driven automated tests and end-user documentation covering setup, hooks, transcript ingest, and historical backfill for VS Code Copilot | UOW-6                  |
| R-7    | The integration ships with fixture-driven automated tests and end-user documentation covering setup, hooks, transcript ingest, and historical backfill for VS Code Copilot | UOW-7                  |
| R-7    | The integration ships with fixture-driven automated tests and end-user documentation covering setup, hooks, transcript ingest, and historical backfill for VS Code Copilot | UOW-8                  |
| R-7    | The integration ships with fixture-driven automated tests and end-user documentation covering setup, hooks, transcript ingest, and historical backfill for VS Code Copilot | UOW-9                  |
| R-7    | The integration ships with fixture-driven automated tests and end-user documentation covering setup, hooks, transcript ingest, and historical backfill for VS Code Copilot | UOW-10 (primary owner) |

---

## UOW-1: Lock historical VS Code backfill with a fixture-backed convo-miner regression

**Objective**: Prove that the already-shipped VS Code normalizer and conversation miner support historical backfill end to end by mining copied VS Code transcript fixtures through the existing `mine_convos()` path.
**covers**: [R-6, R-7]
**depends_on**: none
**parallel_group**: A

### Scope

- **Modify**: `tests/test_convo_miner.py` — add a VS Code Copilot regression that copies committed transcript fixtures into a temporary source directory and mines them into a temporary palace.
- **Do not touch**: `mempalace/convo_miner.py` — historical backfill must keep reusing the existing miner.
- **Do not touch**: `mempalace/normalize.py` — VS Code transcript recognition landed in increment 2 and is reused here.
- **Do not touch**: `tests/fixtures/vscode_copilot/transcripts/` — consume the committed fixtures as-is.

### Contracts

- `mine_convos(convo_dir: str, palace_path: str, wing: str = None, agent: str = "mempalace", limit: int = 0, dry_run: bool = False, extract_mode: str = "exchange")` remains signature-stable and is exercised as the historical backfill entry point.
- The regression proves the same code path used by `mempalace mine <dir> --mode convos` can mine VS Code Copilot transcripts without introducing a second command or helper.

### DI Registrations

- None.

### Behavioral Specification

**Preconditions**:

- Committed VS Code transcript fixtures exist under `tests/fixtures/vscode_copilot/transcripts/`.
- Increment 2 has already landed the VS Code normalizer and live-ingest runtime support.

**Input validation**:

| Field                    | Rule                                                                                                 | On violation         |
| ------------------------ | ---------------------------------------------------------------------------------------------------- | -------------------- |
| Copied fixture directory | Must contain the committed VS Code `.jsonl` transcript fixtures copied from the repo fixture subtree | Test fails           |
| Temporary palace path    | Must be writable and isolated from the repo tree                                                     | Test fails           |
| Wing override            | Use a stable explicit wing name for assertions                                                       | Test assertion fails |

**Behavior**:

- On success, the regression copies the committed VS Code transcript fixtures into a temporary source directory before mining.
- The regression calls `mine_convos()` against the copied directory with `wing="vscode-fixture-test"`.
- The regression asserts that mining produces stored drawers or registry entries and that their `source_file` metadata points to the copied fixture paths rather than the checked-in fixture tree.
- No runtime behavior changes; this UOW only locks the already-shipped historical backfill path.

**Side effects**:

- Temporary source and palace directories are created and cleaned up within the test.
- Chroma collections are created only under the temporary palace path.

### Negative Constraints

- Do NOT add a second historical backfill command.
- Do NOT special-case VS Code Copilot in `mempalace/convo_miner.py`.
- Do NOT mine directly against the checked-in fixture directory in the regression.
- Do NOT duplicate or rewrite the committed fixture transcripts.

### Operational Sequence

1. Create a temporary source directory and copy the committed VS Code transcript fixtures into it.
2. Create a temporary palace directory.
3. Run `mine_convos()` against the copied fixture directory with a stable explicit wing.
4. Assert that the mine produces stored output for the copied VS Code fixtures.
5. Assert that filed metadata references the copied paths, proving end-to-end reuse of the existing convo-miner path.

### Acceptance Criteria

- [ ] `tests/test_convo_miner.py` contains a VS Code Copilot regression that copies the committed transcript fixtures into a temporary source directory before mining.
- [ ] The regression invokes `mine_convos()` directly and does not introduce a new backfill helper or command.
- [ ] Mining the copied VS Code fixtures under wing `vscode-fixture-test` produces stored drawers or registry entries in a temporary palace.
- [ ] Filed metadata references the copied fixture paths, not the checked-in `tests/fixtures/` paths.
- [ ] The regression passes without modifying any checked-in fixture files.

### Verification

- `python -m pytest tests/test_convo_miner.py -v`
- `python -m mempalace --palace <tmp-palace> mine tests/fixtures/vscode_copilot/transcripts --mode convos --wing vscode-fixture-test`

---

## UOW-2: Align CLI help text with shipped VS Code Copilot conversation mining

**Objective**: Update the public CLI help surface so `mempalace --help` and `mempalace mine --help` explicitly describe VS Code Copilot conversation mining as a supported `--mode convos` source.
**covers**: [R-7]
**depends_on**: none
**parallel_group**: A

### Scope

- **Modify**: `mempalace/cli.py` — update the top-level help epilog, examples, and `mine` help wording to include `VS Code Copilot` as a supported conversation-export source.
- **Modify**: `tests/test_cli.py` — add focused assertions on the updated help text.
- **Do not touch**: `website/reference/cli.md` — website CLI reference parity belongs to UOW-7.
- **Do not touch**: `mempalace/instructions/help.md`, `mempalace/instructions/mine.md` — built-in instruction parity belongs to UOW-8.

### Contracts

- The CLI surface remains signature-stable: no new subcommand, option, or flag is introduced.
- `mempalace mine <dir> --mode convos` remains the single documented historical backfill path.
- The top-level help epilog and `--mode` help text are updated to mention `VS Code Copilot` alongside the already-supported conversation sources.

### DI Registrations

- None.

### Behavioral Specification

**Preconditions**:

- The CLI parser in `mempalace/cli.py` remains the authoritative public help surface.

**Input validation**:

| Field                          | Rule                                                                         | On violation                      |
| ------------------------------ | ---------------------------------------------------------------------------- | --------------------------------- |
| Top-level help epilog/examples | Must name `VS Code Copilot` as a supported conversation-export source        | Help-output regression test fails |
| `mine --mode` help text        | Must describe `convos` mode as covering VS Code Copilot conversation exports | Help-output regression test fails |

**Behavior**:

- `mempalace --help` describes conversation mining as including VS Code Copilot exports.
- `mempalace mine --help` describes `--mode convos` as the conversation-mining path for VS Code Copilot exports.
- Existing help for Claude Code, Claude.ai, ChatGPT, Slack, and the rest of the CLI remains intact.
- No parser behavior changes; only public help wording changes.

**Side effects**:

- Stdout help output only. No runtime or filesystem behavior changes.

### Negative Constraints

- Do NOT add a second backfill command.
- Do NOT rename `--mode convos` or add a VS Code-only flag.
- Do NOT change `cmd_mine()` behavior.
- Do NOT mention `Copilot CLI`.

### Operational Sequence

1. Update the top-level help epilog so the conversation-mining banner includes VS Code Copilot.
2. Update the `mine` subcommand help text so `--mode convos` explicitly covers VS Code Copilot conversation exports.
3. Update examples or surrounding help strings as needed so the wording is internally consistent.
4. Add focused tests in `tests/test_cli.py` that lock the updated help output.

### Acceptance Criteria

- [ ] The top-level CLI help names `VS Code Copilot` as a supported conversation-export source.
- [ ] `mempalace mine --help` describes `--mode convos` as covering VS Code Copilot conversation exports.
- [ ] `tests/test_cli.py` contains focused help-output assertions for both surfaces.
- [ ] No parser flag, command name, or second backfill command is introduced.
- [ ] No help text mentions `Copilot CLI`.

### Verification

- `python -m pytest tests/test_cli.py -v`

---

## UOW-3: Expand the dedicated VS Code Copilot guide into the full shipped workflow

**Objective**: Turn `website/guide/vscode-copilot.md` from a phase-1 setup page into the canonical shipped guide covering MCP setup, repository instructions, hook template install, live ingest behavior, and historical backfill.
**covers**: [R-7]
**depends_on**: [UOW-1, UOW-2, UOW-4]
**parallel_group**: B

### Scope

- **Modify**: `website/guide/vscode-copilot.md` — expand the guide into the full shipped VS Code Copilot workflow.
- **Do not touch**: `.github/hooks/vscode-copilot.json` — the hook template is created in UOW-4 and must be referenced, not redefined here.
- **Do not touch**: `website/guide/hooks.md` — website hooks-guide parity belongs to UOW-5.
- **Do not touch**: `website/guide/mining.md` — mining-guide parity belongs to UOW-6.
- **Do not touch**: `website/reference/cli.md` — CLI reference parity belongs to UOW-7.
- **Do not touch**: `README.md`, `.github/copilot-instructions.md` — terminal repository-surface convergence belongs to UOW-10.

### Contracts

- No code/public-member contracts are introduced. The artifact contract is a canonical website guide for the shipped VS Code Copilot path.
- The guide must preserve the already-shipped MCP setup and tool-verification flow while appending the shipped hook/live-ingest/backfill story.

### Negative Constraints

- Do NOT redefine the `.github/hooks/vscode-copilot.json` event wiring independently of UOW-4.
- Do NOT invent a second historical backfill command.
- Do NOT hard-code an unverified default filesystem location for VS Code transcripts.
- Do NOT mention `Copilot CLI`.
- Do NOT regress the existing agent-mode MCP setup guidance.

### Operational Sequence

1. Preserve the existing MCP setup and Copilot tool-verification guidance already shipped in phase 1.
2. Replace the old phase-1-only boundary language with the shipped phase-3 story.
3. Add the repository-surface guidance for `.github/copilot-instructions.md` and `.github/hooks/vscode-copilot.json`.
4. Document the shipped live-ingest behavior for stop/precompact hooks, including fail-open local-error behavior.
5. Document historical backfill with the existing `mempalace mine <dir> --mode convos` flow and no second command.
6. Cross-check language against UOW-2 and UOW-4 so this page becomes the canonical end-user guide.

### Acceptance Criteria

- [ ] The existing MCP setup, agent-mode note, and tool-verification guidance remain present.
- [ ] The guide documents `.github/copilot-instructions.md` and `.github/hooks/vscode-copilot.json` as the repository guidance/template surfaces for VS Code Copilot.
- [ ] The guide explains shipped stop/precompact live ingest and explicitly preserves fail-open local-error language.
- [ ] The guide documents historical backfill with `mempalace mine <dir> --mode convos` and does not invent a second command.
- [ ] The guide uses `VS Code Copilot` consistently and contains no `Copilot CLI` reference.

### Verification

- `rg -n "VS Code Copilot|\.github/hooks/vscode-copilot\.json|--mode convos|fail-open" website/guide/vscode-copilot.md`
- `cd website && npm run docs:build`

---

## UOW-4: Ship the dedicated VS Code Copilot hook template and retire `capture.json`

**Objective**: Create the durable user-facing VS Code Copilot hook template at `.github/hooks/vscode-copilot.json` and remove the raw-capture artifact `.github/hooks/capture.json` from the supported repository surface.
**covers**: [R-7]
**depends_on**: none
**parallel_group**: A

### Scope

- **Create**: `.github/hooks/vscode-copilot.json` — primary user-facing VS Code Copilot hook template.
- **Delete**: `.github/hooks/capture.json` — obsolete raw-capture artifact that must not remain a supported template.
- **Do not touch**: `hooks/mempal_save_hook.sh`, `hooks/mempal_precompact_hook.sh` — existing shell hooks remain Claude/Codex compatibility artifacts.
- **Do not touch**: `website/guide/vscode-copilot.md`, `website/guide/hooks.md`, `hooks/README.md` — downstream docs must depend on this template rather than define it.

### Contracts

- `.github/hooks/vscode-copilot.json` uses the existing VS Code hook-file root shape: `{ "hooks": { ... } }`.
- The new template wires `SessionStart`, `Stop`, and `PreCompact` to `mempalace hook run --harness vscode-copilot` with the matching `--hook` values.
- `.github/hooks/capture.json` is removed and no longer part of the supported repository surface.

### Negative Constraints

- Do NOT preserve `.github/hooks/capture.json` as a second supported template.
- Do NOT point users at raw capture scripts or temp-directory paths.
- Do NOT tell VS Code Copilot users to install `hooks/mempal_save_hook.sh` or `hooks/mempal_precompact_hook.sh` as their primary path.
- Do NOT let downstream docs define a different template event set than the file created here.
- Do NOT mention `Copilot CLI`.

### Operational Sequence

1. Create `.github/hooks/vscode-copilot.json` with a top-level `hooks` object.
2. Wire `SessionStart`, `Stop`, and `PreCompact` to `mempalace hook run --harness vscode-copilot` with matching hook names.
3. Remove `.github/hooks/capture.json` from the repo.
4. Treat `.github/hooks/vscode-copilot.json` as the authoritative hook template for all downstream docs.

### Acceptance Criteria

- [ ] `.github/hooks/vscode-copilot.json` exists and parses as valid JSON.
- [ ] The file wires `SessionStart`, `Stop`, and `PreCompact` to `mempalace hook run` with `--harness vscode-copilot` and the matching `--hook` values.
- [ ] The file contains no raw-capture script paths, temp-directory paths, or shell-hook script commands.
- [ ] `.github/hooks/capture.json` is deleted.
- [ ] Downstream docs can reference `.github/hooks/vscode-copilot.json` as the authoritative VS Code Copilot hook template without redefining a different event set.

### Verification

- `python3 -c "import json; json.load(open('.github/hooks/vscode-copilot.json', encoding='utf-8'))"`
- `test ! -f .github/hooks/capture.json`

---

## UOW-5: Bring the website hooks guide to shipped VS Code Copilot parity

**Objective**: Update `website/guide/hooks.md` so it documents the shipped VS Code Copilot hook path, live-ingest behavior, and compatibility boundaries without leaving the old “not yet available” story in place.
**covers**: [R-7]
**depends_on**: [UOW-3, UOW-4]
**parallel_group**: C

### Scope

- **Modify**: `website/guide/hooks.md` — document the shipped VS Code Copilot hook path and keep Claude/Codex shell-hook guidance as compatibility context.
- **Do not touch**: `.github/hooks/vscode-copilot.json` — template content is owned by UOW-4.
- **Do not touch**: `hooks/README.md` — repo-local hooks README parity belongs to UOW-9.
- **Do not touch**: `hooks/mempal_save_hook.sh`, `hooks/mempal_precompact_hook.sh` — shell-hook implementation remains unchanged.

### Contracts

- No code/public-member contracts are introduced. The artifact contract is website hooks documentation aligned to the shipped VS Code Copilot flow and the authoritative template from UOW-4.

### Negative Constraints

- Do NOT keep stating that VS Code Copilot hook support is unavailable.
- Do NOT present the shell-hook scripts as the primary VS Code Copilot path.
- Do NOT contradict the event wiring defined in `.github/hooks/vscode-copilot.json`.
- Do NOT mention `Copilot CLI`.

### Operational Sequence

1. Replace the old VS Code deferment note with shipped VS Code Copilot hook guidance.
2. Document `.github/hooks/vscode-copilot.json` as the primary hook template for VS Code Copilot.
3. Explain the shipped stop/precompact live-ingest behavior and its fail-open local-error behavior.
4. Preserve the Claude Code and Codex shell-hook sections as compatibility guidance rather than the VS Code primary path.
5. Link to the canonical VS Code guide for the broader setup/backfill workflow.

### Acceptance Criteria

- [ ] The opening VS Code Copilot note no longer says hook support is unavailable.
- [ ] The page identifies `.github/hooks/vscode-copilot.json` as the primary VS Code Copilot hook install surface.
- [ ] The page explains shipped stop/precompact live ingest and fail-open local-error behavior in VS Code terms.
- [ ] Claude Code and Codex shell-hook instructions remain present as compatibility sections and are not presented as the primary VS Code path.
- [ ] No `.github/hooks/capture.json` or `Copilot CLI` reference remains.

### Verification

- `rg -n "VS Code Copilot|\.github/hooks/vscode-copilot\.json|capture\.json|Copilot CLI|not yet available" website/guide/hooks.md`
- `cd website && npm run docs:build`

---

## UOW-6: Replace the mining-guide deferment with shipped historical backfill guidance

**Objective**: Update `website/guide/mining.md` so VS Code Copilot historical backfill is documented as a shipped `mempalace mine <dir> --mode convos` workflow instead of a deferred feature.
**covers**: [R-7]
**depends_on**: [UOW-1, UOW-3]
**parallel_group**: C

### Scope

- **Modify**: `website/guide/mining.md` — replace the old deferred VS Code note with shipped historical backfill guidance.
- **Do not touch**: `mempalace/convo_miner.py` — the miner behavior is already shipped and locked by UOW-1.
- **Do not touch**: `website/guide/vscode-copilot.md` — the canonical end-user guide is owned by UOW-3.
- **Do not touch**: `mempalace/instructions/mine.md` — built-in instruction parity belongs to UOW-8.

### Contracts

- No code/public-member contracts are introduced. The artifact contract is accurate mining documentation for the existing `mempalace mine <dir> --mode convos` flow.

### Negative Constraints

- Do NOT invent a second historical backfill command.
- Do NOT hard-code an unverified default filesystem location for VS Code transcripts.
- Do NOT keep saying that historical VS Code Copilot backfill is unsupported.
- Do NOT mention `Copilot CLI`.

### Operational Sequence

1. Replace the old VS Code Copilot deferment note with shipped backfill guidance.
2. Add VS Code Copilot to the supported conversation-export list for `--mode convos`.
3. Document historical backfill through `mempalace mine <dir> --mode convos`, optionally with a `--wing` example.
4. Link readers back to the canonical VS Code guide for the full MCP + hooks + live-ingest story.

### Acceptance Criteria

- [ ] `website/guide/mining.md` no longer says historical VS Code Copilot backfill is unsupported.
- [ ] The Conversations Mode section lists VS Code Copilot as a supported conversation-export source.
- [ ] The page documents historical backfill with `mempalace mine <dir> --mode convos` and does not invent a second command.
- [ ] The page does not hard-code an unverified default transcript location.
- [ ] No `Copilot CLI` reference appears.

### Verification

- `rg -n "VS Code Copilot|--mode convos|Copilot CLI|not yet supported" website/guide/mining.md`
- `cd website && npm run docs:build`

---

## UOW-7: Update the CLI reference for VS Code Copilot backfill and hook parity

**Objective**: Bring `website/reference/cli.md` into line with the shipped VS Code Copilot CLI surface for `mempalace mcp`, `mempalace hook run`, and historical backfill through `mempalace mine <dir> --mode convos`.
**covers**: [R-7]
**depends_on**: [UOW-1, UOW-2, UOW-4]
**parallel_group**: B

### Scope

- **Modify**: `website/reference/cli.md` — update `mcp`, `mine`, and `hook` reference content for the shipped VS Code Copilot surface.
- **Do not touch**: `mempalace/cli.py` — CLI behavior/help wording belongs to UOW-2 and is already authoritative.
- **Do not touch**: `tests/test_cli.py` — CLI regression coverage belongs to UOW-2.
- **Do not touch**: `.github/hooks/vscode-copilot.json` — template ownership belongs to UOW-4.

### Contracts

- No code/public-member contracts change. The documentation must reflect the existing CLI contracts:
  - `mempalace mine <dir> --mode convos`
  - `mempalace mcp`
  - `mempalace hook run --hook {session-start, stop, precompact, subagent-start, subagent-stop} --harness {claude-code, codex, vscode-copilot}`

### Negative Constraints

- Do NOT document a second backfill command.
- Do NOT document unsupported hook names or harness values.
- Do NOT contradict the hook template created in UOW-4.
- Do NOT mention `Copilot CLI`.

### Operational Sequence

1. Update the `mempalace mine` reference content so conversation mining explicitly includes VS Code Copilot.
2. Update the `mempalace mcp` reference content so the VS Code Copilot MCP setup and verification path remain accurate.
3. Update the `mempalace hook run` reference examples and option table to include `vscode-copilot` and all five supported hook names.
4. Cross-check the reference text against `mempalace/cli.py` so the documentation matches the actual CLI surface.

### Acceptance Criteria

- [ ] The `mempalace mine` reference names VS Code Copilot as a supported conversation-export source.
- [ ] The `mempalace hook run` reference includes `vscode-copilot` and the five supported hook names: `session-start`, `stop`, `precompact`, `subagent-start`, and `subagent-stop`.
- [ ] The `mempalace mcp` reference preserves the VS Code Copilot MCP setup and tool-verification guidance.
- [ ] No `Copilot CLI` reference or second backfill command appears.

### Verification

- `rg -n "vscode-copilot|subagent-start|subagent-stop|VS Code Copilot|--mode convos|Copilot CLI" website/reference/cli.md`
- `cd website && npm run docs:build`

---

## UOW-8: Align built-in help and mine instructions with the shipped VS Code path

**Objective**: Update the package-shipped `help` and `mine` instruction markdown so `mempalace instructions help` and `mempalace instructions mine` reflect the shipped VS Code Copilot MCP, hooks, and historical backfill story.
**covers**: [R-7]
**depends_on**: [UOW-1, UOW-2]
**parallel_group**: B

### Scope

- **Modify**: `mempalace/instructions/help.md` — update the built-in help text for the shipped VS Code Copilot path.
- **Modify**: `mempalace/instructions/mine.md` — update the mining instructions to treat VS Code Copilot transcripts as supported conversation input.
- **Do not touch**: `mempalace/instructions_cli.py` — the instruction loader remains unchanged.
- **Do not touch**: `README.md`, `.github/copilot-instructions.md` — terminal repository-surface convergence belongs to UOW-10.
- **Do not touch**: `website/guide/*` — website parity is owned by UOW-3, UOW-5, and UOW-6.

### Contracts

- `run_instructions(name: str)` remains unchanged.
- `AVAILABLE` remains unchanged; only the markdown payloads for `help` and `mine` are updated.
- The built-in instruction content must present `mempalace mine <dir> --mode convos` as the single VS Code Copilot historical-backfill path.

### Negative Constraints

- Do NOT modify `mempalace/instructions_cli.py` or the `AVAILABLE` list.
- Do NOT invent a second backfill command.
- Do NOT mention `Copilot CLI`.
- Do NOT contradict the shipped CLI help wording from UOW-2.

### Operational Sequence

1. Update `mempalace/instructions/help.md` so it accurately describes the shipped VS Code Copilot MCP, hook, and historical-backfill surfaces.
2. Update `mempalace/instructions/mine.md` so it treats VS Code Copilot conversation exports as supported input for `--mode convos`.
3. Keep the loader untouched so the existing `run_instructions()` path continues to print the files verbatim.
4. Cross-check wording against UOW-2 so CLI help and built-in instructions stay aligned.

### Acceptance Criteria

- [ ] `mempalace/instructions/help.md` accurately describes the shipped VS Code Copilot MCP, hook, and historical-backfill surfaces.
- [ ] `mempalace/instructions/mine.md` treats VS Code Copilot conversation exports as supported input for `mempalace mine <dir> --mode convos`.
- [ ] `python -m mempalace instructions help` and `python -m mempalace instructions mine` remain loadable through the existing `run_instructions()` path.
- [ ] No `Copilot CLI` reference or second backfill command appears.

### Verification

- `python -m pytest tests/test_instructions_cli.py -v`
- `python -m mempalace instructions help`
- `python -m mempalace instructions mine`

---

## UOW-9: Bring `hooks/README.md` to shipped VS Code Copilot parity

**Objective**: Update the repo-local hooks README so it matches the shipped VS Code Copilot hook template, live-ingest behavior, and historical-backfill story instead of the old “not yet available” boundary note.
**covers**: [R-7]
**depends_on**: [UOW-3, UOW-4]
**parallel_group**: C

### Scope

- **Modify**: `hooks/README.md` — document the shipped VS Code Copilot hook path and historical backfill.
- **Do not touch**: `.github/hooks/vscode-copilot.json` — template ownership belongs to UOW-4.
- **Do not touch**: `hooks/mempal_save_hook.sh`, `hooks/mempal_precompact_hook.sh` — existing shell-hook scripts remain unchanged.
- **Do not touch**: `website/guide/hooks.md` — website hooks-guide parity belongs to UOW-5.

### Contracts

- No code/public-member contracts are introduced. The artifact contract is accurate repo-local documentation for the shipped VS Code Copilot hook path.
- The README must treat `.github/hooks/vscode-copilot.json` as the primary VS Code Copilot hook path while preserving Claude/Codex shell-hook guidance for those surfaces.

### Negative Constraints

- Do NOT keep claiming that VS Code Copilot runtime-hook support is unavailable.
- Do NOT present the shell-hook scripts as the primary VS Code Copilot path.
- Do NOT reference `.github/hooks/capture.json`.
- Do NOT invent a second backfill command.
- Do NOT mention `Copilot CLI`.

### Operational Sequence

1. Replace the old VS Code deferment note with shipped VS Code Copilot hook guidance.
2. Document `.github/hooks/vscode-copilot.json` as the primary VS Code Copilot hook path.
3. Explain shipped stop/precompact live-ingest behavior and historical backfill through `mempalace mine <dir> --mode convos`.
4. Preserve the existing Claude Code and Codex shell-hook installation sections for those surfaces.

### Acceptance Criteria

- [ ] `hooks/README.md` no longer claims VS Code Copilot runtime-hook support is unavailable.
- [ ] The README documents `.github/hooks/vscode-copilot.json` as the primary VS Code Copilot hook path.
- [ ] The README explains shipped stop/precompact live ingest and historical backfill through the existing `mempalace mine <dir> --mode convos` flow.
- [ ] Claude Code and Codex shell-hook installation instructions remain intact for those surfaces.
- [ ] No `.github/hooks/capture.json` or `Copilot CLI` reference appears.

### Verification

- `rg -n "VS Code Copilot|\.github/hooks/vscode-copilot\.json|capture\.json|Copilot CLI|not yet available" hooks/README.md`

---

## UOW-10: Terminal convergence on `README.md` and repository Copilot instructions

**Objective**: Reconcile the highest-visibility repository surfaces so `README.md` and `.github/copilot-instructions.md` accurately describe the shipped VS Code Copilot integration and no longer carry stale phase-1 or phase-2 deferment language.
**covers**: [R-7]
**depends_on**: [UOW-1, UOW-3, UOW-5, UOW-6, UOW-7, UOW-8, UOW-9]
**parallel_group**: D

### Scope

- **Modify**: `README.md` — present VS Code Copilot as a first-class path that now includes MCP, hooks, and historical backfill through the canonical guide.
- **Modify**: `.github/copilot-instructions.md` — remove stale “not yet implemented” claims for shipped VS Code Copilot surfaces while preserving MemPalace’s search-first, verbatim, and local-first constraints.
- **Do not touch**: `website/guide/vscode-copilot.md`, `website/guide/hooks.md`, `website/guide/mining.md`, `website/reference/cli.md` — those surfaces must already be converged before this terminal UOW.
- **Do not touch**: `.github/hooks/vscode-copilot.json` — template ownership belongs to UOW-4.
- **Do not touch**: `mempalace/instructions/help.md`, `mempalace/instructions/mine.md` — built-in instruction parity belongs to UOW-8.
- **Do not touch**: `hooks/README.md` — repo-local hooks README parity belongs to UOW-9.

### Contracts

- No code/public-member contracts are introduced. The artifact contracts are:
  - `README.md` remains the high-level repository entry point and routes readers to the canonical guide rather than duplicating the whole setup.
  - `.github/copilot-instructions.md` remains repository-level Copilot guidance consumed automatically by VS Code Copilot and must continue to preserve memory-first behavior.

### Negative Constraints

- Do NOT turn `README.md` into a full installation or hook-template manual.
- Do NOT weaken the requirement that Copilot search memory before answering when memory is relevant.
- Do NOT reintroduce stale phase-based “not yet implemented” claims for shipped VS Code Copilot hooks, normalization, or historical backfill.
- Do NOT mention `Copilot CLI`.
- Do NOT contradict the canonical website guide or hook template.

### Operational Sequence

1. Update `README.md` so the VS Code Copilot entry point reflects the shipped MCP + hooks + historical-backfill story and routes readers to the canonical guide.
2. Remove stale phase-1/phase-2 deferment language from `.github/copilot-instructions.md`.
3. Preserve the instruction file’s memory-first behavior, verbatim constraint, and local-first constraint while making the availability statements accurate.
4. Cross-check both files against the website guides, hook template, and built-in instructions so the repository entry points converge last.

### Acceptance Criteria

- [ ] `README.md` presents VS Code Copilot as a first-class path and points readers to the canonical guide without contradicting shipped hooks or historical backfill.
- [ ] `.github/copilot-instructions.md` no longer says VS Code Copilot hooks, transcript normalization, or historical backfill are not implemented.
- [ ] `.github/copilot-instructions.md` continues to require memory-first search behavior and preserves verbatim/local-first constraints.
- [ ] Both files use `VS Code Copilot` naming and contain no `Copilot CLI` reference.
- [ ] This UOW is the terminal primary owner of `R-7` by reconciling the highest-visibility repository surfaces after all other docs, help, and template updates land.

### Verification

- `rg -n "VS Code Copilot|Copilot CLI|not yet implemented|historical backfill|vscode-copilot" README.md .github/copilot-instructions.md`
- Manual read of `README.md` and `.github/copilot-instructions.md` against the website guides and `.github/hooks/vscode-copilot.json`

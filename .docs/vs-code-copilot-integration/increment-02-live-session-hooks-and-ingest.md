# Increment 2: Live Session Hooks and Ingest

**Depends on**: increment 1 completed
**Acceptance Verification**:

1. `python -m pytest tests/test_cli.py -v`
2. `python -m pytest tests/test_normalize.py -v`
3. `python -m pytest tests/test_hooks_cli.py -v`
4. Fixture-backed smoke invocations of `python -m mempalace hook run --harness vscode-copilot` accept `session-start`, `stop`, `precompact`, `subagent-start`, and `subagent-stop` without regressing Claude/Codex paths.

## Requirements Matrix (UOW-level)

| Req ID | Requirement (summary)                                                                                                    | UOW                   |
| ------ | ------------------------------------------------------------------------------------------------------------------------ | --------------------- |
| R-3    | Support VS Code Copilot hook payloads and lifecycle with `vscode-copilot` as a first-class harness                       | UOW-1                 |
| R-3    | Support VS Code Copilot hook payloads and lifecycle with `vscode-copilot` as a first-class harness                       | UOW-3 (primary owner) |
| R-4    | Normalize VS Code Copilot transcripts into standard MemPalace conversation text                                          | UOW-2                 |
| R-5    | Ingest live VS Code sessions from `transcript_path` during hook-driven saves/checkpoints without blocking chat on errors | UOW-4 (primary owner) |
| R-5    | Ingest live VS Code sessions from `transcript_path` during hook-driven saves/checkpoints without blocking chat on errors | UOW-5                 |

---

## UOW-1: Expand public hook CLI choices for VS Code Copilot

**Objective**: Extend the public `mempalace hook run` parser so phase-2 VS Code hook names and harness values are accepted without changing runtime behavior.
**covers**: [R-3]
**depends_on**: none
**parallel_group**: A

### Scope

- **Modify**: `mempalace/cli.py` — expand `--hook` and `--harness` choices.
- **Modify**: `tests/test_cli.py` — add parser and dispatch coverage for `vscode-copilot`, `subagent-start`, and `subagent-stop`.
- **Do not touch**: `mempalace/hooks_cli.py` — runtime handling is a later slice.
- **Do not touch**: `hooks/mempal_save_hook.sh`, `hooks/mempal_precompact_hook.sh`, `README.md`, `website/`.

### Contracts

- `cmd_hook(args)` remains `run_hook(hook_name=args.hook, harness=args.harness)`.
- `--hook` choices expand to `session-start`, `stop`, `precompact`, `subagent-start`, `subagent-stop`.
- `--harness` choices expand to `claude-code`, `codex`, `vscode-copilot`.

### DI Registrations

- None.

### Behavioral Specification

**Preconditions**:

- `mempalace` is invocable as a CLI command.

**Input validation**:
| Field | Rule | On violation |
|-------|------|--------------|
| `--hook` | Must be one of the five supported hook names | argparse usage error, exit code 2 |
| `--harness` | Must be one of `claude-code`, `codex`, `vscode-copilot` | argparse usage error, exit code 2 |

**Behavior**:

- Valid hook/harness combinations parse and dispatch to `cmd_hook`.
- Invalid hook or harness values still fail at argparse before runtime dispatch.

**Side effects**:

- None beyond the existing `cmd_hook` call.

### Negative Constraints

- Do NOT change the body or signature of `cmd_hook`.
- Do NOT add new hook CLI flags.
- Do NOT change runtime dispatch in `mempalace/hooks_cli.py`.
- Do NOT update docs in this UOW.

### Operational Sequence

1. Expand the `--hook` choices in `mempalace/cli.py`.
2. Expand the `--harness` choices in `mempalace/cli.py`.
3. Add `cmd_hook` and `main()` parser tests in `tests/test_cli.py` for the new valid and invalid combinations.

### Acceptance Criteria

- [ ] `mempalace hook run --hook subagent-start --harness vscode-copilot` is accepted by argparse.
- [ ] `mempalace hook run --hook subagent-stop --harness vscode-copilot` is accepted by argparse.
- [ ] Existing `claude-code` and `codex` parser paths still parse.
- [ ] Invalid hook and harness values still fail with argparse exit code 2.

### Verification

- `python -m pytest tests/test_cli.py -v`

---

## UOW-2: Add VS Code Copilot transcript normalization

**Objective**: Make `normalize()` recognize VS Code Copilot JSONL transcripts and emit standard MemPalace conversation text reused by both live ingest and historical backfill.
**covers**: [R-4]
**depends_on**: none
**parallel_group**: A

### Scope

- **Modify**: `mempalace/normalize.py` — add a VS Code JSONL detector/parser and insert it into the detector chain.
- **Modify**: `tests/test_normalize.py` — add fixture-driven VS Code normalization coverage.
- **Do not touch**: `mempalace/convo_miner.py` — reuse only.
- **Do not touch**: `mempalace/hooks_cli.py` — runtime behavior belongs to later UOWs.
- **Do not touch**: `tests/fixtures/vscode_copilot/transcripts/` — consume committed fixtures as-is.

### Contracts

- Add `_try_vscode_copilot_jsonl(content: str) -> Optional[str]` in `mempalace/normalize.py`.
- Modify `_try_normalize_json(content: str) -> Optional[str]` to call the VS Code parser after `_try_codex_jsonl()` and before the structured JSON parsers.

### DI Registrations

- None.

### Behavioral Specification

**Preconditions**:

- The input file is non-empty JSONL.
- The transcript contains a `session.start` record with `data.producer == copilot-agent`.

**Input validation**:
| Field | Rule | On violation |
|-------|------|--------------|
| JSONL line | Must parse as a JSON object | Skip the line |
| VS Code sentinel | Must include `session.start` with `producer: copilot-agent` | Return `None` |
| Conversation length | Must yield at least two canonical turns after filtering | Return `None` |

**Behavior**:

- Canonical `user.message` content becomes user turns.
- Canonical non-empty `assistant.message` content becomes assistant turns.
- Consecutive assistant messages merge into a single assistant turn.
- Tool noise, tool-only shells, reasoning text, tool requests, and subagent-internal chatter are excluded.
- Successful parse returns `_messages_to_transcript(messages)`.

**Side effects**:

- None.

### Negative Constraints

- Do NOT create a second backfill path.
- Do NOT include `reasoningText`, `toolRequests`, or tool call ids in normalized output.
- Do NOT modify fixture files.
- Do NOT change Claude Code or Codex normalization behavior.

### Operational Sequence

1. Add `_try_vscode_copilot_jsonl()` in `mempalace/normalize.py`.
2. Insert it into `_try_normalize_json()`.
3. Add fixture-driven tests in `tests/test_normalize.py` for `simple_with_tools.jsonl`, `with_precompact_context.jsonl`, and `with_subagent.jsonl`.
4. Add detector-isolation assertions so Claude/Codex parsers do not misidentify VS Code transcripts.

### Acceptance Criteria

- [ ] `simple_with_tools.jsonl` normalizes into canonical user/assistant text without tool chrome.
- [ ] `with_precompact_context.jsonl` preserves the real user turns and merged assistant output.
- [ ] `with_subagent.jsonl` drops subagent-internal prompt chatter but keeps the surrounding canonical conversation.
- [ ] `normalize()` on the three VS Code fixtures produces standard transcript text with `>` user markers.

### Verification

- `python -m pytest tests/test_normalize.py -v`

---

## UOW-3: Register the VS Code Copilot hook harness and payload shape

**Objective**: Extend the hook runtime to accept `vscode-copilot` payloads and all phase-2 hook names, including explicit fail-open subagent lifecycle handlers, without changing existing Claude/Codex contracts.
**covers**: [R-3]
**depends_on**: [UOW-1]
**parallel_group**: B

### Scope

- **Modify**: `mempalace/hooks_cli.py` — add `vscode-copilot`, broaden `_parse_harness_input`, add subagent handlers, extend `run_hook`.
- **Modify**: `tests/test_hooks_cli.py` — add fixture-driven VS Code hook coverage and backward-compatibility assertions.
- **Do not touch**: `mempalace/normalize.py`.
- **Do not touch**: `hooks/mempal_save_hook.sh`, `hooks/mempal_precompact_hook.sh`, `README.md`, `website/`.

### Contracts

- `SUPPORTED_HARNESSES` expands to include `vscode-copilot`.
- `_parse_harness_input(data: dict, harness: str) -> dict` returns additive VS Code fields `cwd`, `agent_id`, `agent_type`, and `trigger`.
- Add `hook_subagent_start(data: dict, harness: str) -> None`.
- Add `hook_subagent_stop(data: dict, harness: str) -> None`.
- Extend `run_hook(hook_name: str, harness: str) -> None` with `subagent-start` and `subagent-stop`.

### DI Registrations

- None.

### Behavioral Specification

**Preconditions**:

- `harness` is one of `claude-code`, `codex`, or `vscode-copilot`.
- `hook_name` is one of the five supported hook names.

**Input validation**:
| Field | Rule | On violation |
|-------|------|--------------|
| `harness` | Must be in `SUPPORTED_HARNESSES` | `sys.exit(1)` |
| `hook_name` | Must be in the dispatch table | `sys.exit(1)` |
| `session_id` | Sanitized through `_sanitize_session_id` | Falls back to `unknown` |
| additive VS Code fields | Optional string payload fields | Default to empty string |

**Behavior**:

- VS Code `session-start`, `stop`, and `precompact` payloads dispatch through the existing handlers.
- VS Code `subagent-start` and `subagent-stop` dispatch through explicit pass-through handlers.
- Subagent handlers log the lifecycle event and emit `{}`.
- Claude/Codex parsing remains unchanged except for additive empty-string keys in the parsed dict.

**Side effects**:

- Subagent handlers append to the hook log.
- No blocking response is introduced for subagent lifecycle events.

### Negative Constraints

- Do NOT move or restructure the existing dispatch model.
- Do NOT add live ingest behavior to subagent handlers.
- Do NOT add docs or config surfaces.
- Do NOT regress unknown-harness or unknown-hook failure behavior.

### Operational Sequence

1. Add `vscode-copilot` to `SUPPORTED_HARNESSES`.
2. Extend `_parse_harness_input()` with additive VS Code-only fields.
3. Add `hook_subagent_start()` and `hook_subagent_stop()`.
4. Extend `run_hook()` with the two new hook names.
5. Add fixture-backed tests in `tests/test_hooks_cli.py` for all five VS Code hook events plus Claude/Codex regression assertions.

### Acceptance Criteria

- [ ] `_parse_harness_input(..., vscode-copilot)` retains `cwd`, `agent_id`, `agent_type`, and `trigger`.
- [ ] `run_hook(session-start, vscode-copilot)` accepts the committed session-start fixture and emits `{}`.
- [ ] `run_hook(stop, vscode-copilot)` accepts the committed stop fixture and emits a fail-open response.
- [ ] `run_hook(precompact, vscode-copilot)` accepts the committed precompact fixture and emits `{}`.
- [ ] `run_hook(subagent-start, vscode-copilot)` and `run_hook(subagent-stop, vscode-copilot)` both emit `{}`.
- [ ] Existing `claude-code` and `codex` paths remain green.

### Verification

- `python -m pytest tests/test_hooks_cli.py -v`

---

## UOW-4: Make stop and precompact extraction understand VS Code transcripts

**Objective**: Adapt the hook runtime so exchange counting and recent-message extraction work for VS Code Copilot event-tree transcripts without regressing Claude/Codex behavior. This is the primary observable owner of `R-5` because it makes the existing `transcript_path`-driven stop/precompact checkpoint ingest path work on real VS Code transcripts.
**covers**: [R-5]
**depends_on**: [UOW-2, UOW-3]
**parallel_group**: C

### Scope

- **Modify**: `mempalace/hooks_cli.py` — teach `_count_human_messages()` and `_extract_recent_messages()` to recognize canonical VS Code `user.message` events and ignore internal subagent/tool chatter without changing existing Claude/Codex behavior.
- **Modify**: `tests/test_hooks_cli.py` — add VS Code transcript-aware count/extract coverage plus stop/precompact regression tests that exercise the real VS Code fixtures.
- **Do not touch**: `mempalace/normalize.py`.
- **Do not touch**: `mempalace/cli.py`.
- **Do not touch**: `tests/test_normalize.py`.
- **Do not touch**: `hooks/mempal_save_hook.sh`, `hooks/mempal_precompact_hook.sh`.

### Contracts

- `_count_human_messages(transcript_path: str) -> int` counts canonical VS Code `user.message` turns in event-tree transcripts while continuing to support Claude Code and Codex transcript shapes.
- `_extract_recent_messages(transcript_path: str, count: int = _RECENT_MSG_COUNT) -> list[str]` extracts canonical VS Code user turns while filtering internal subagent chatter and preserving existing Claude/Codex filtering rules.
- `hook_stop(data: dict, harness: str)` remains signature-stable while now operating correctly on VS Code transcripts.
- `hook_precompact(data: dict, harness: str)` remains signature-stable while now operating correctly on VS Code transcripts.

### DI Registrations

- None.

### Behavioral Specification

**Preconditions**:

- UOW-2 and UOW-3 have landed, so the hook runtime can receive VS Code payloads and the downstream ingest path can mine VS Code transcripts from `transcript_path`.
- Hooks remain fail-open on local parse or subprocess errors.

**Input validation**:
| Field | Rule | On violation |
|-------|------|--------------|
| `transcript_path` | Existing validator rules remain unchanged | Ignore the transcript, log a warning when applicable, and continue without blocking chat |
| VS Code transcript records | Only canonical `user.message` records count as user turns | Skip the record and continue processing |

**Authorization**:

- The current local OS user running MemPalace and VS Code Copilot performs all file-system operations.
- `transcript_path` and hook state paths must resolve to local files accessible to that user.
- On access or parse failure, preserve current pass-through behavior with local logging only.

**Behavior**:

- On success, `_count_human_messages()` counts canonical VS Code user turns so the existing stop-hook save threshold can fire on real VS Code sessions.
- On success, `_extract_recent_messages()` returns canonical VS Code user messages and excludes internal subagent prompts, assistant tool chatter, and existing command/system reminder noise.
- This UOW is the final observable owner of `R-5`, because once stop/precompact exchange counting and recent-message extraction understand real VS Code transcripts, the existing checkpoint save plus `_ingest_transcript(transcript_path)` path becomes observably functional for hook-driven VS Code sessions.
- On malformed lines or unsupported event shapes, skip the bad record and continue without blocking the hook.
- `hook_precompact()` continues to ingest from `transcript_path` and allow compaction to proceed even if transcript processing is partial or noisy.

**Side effects**:

- Existing stop-hook checkpoint behavior can now call `_save_diary_direct(transcript_path, session_id, toast=toast)` on real VS Code transcripts.
- Existing stop and precompact flows can now call `_ingest_transcript(transcript_path)` against real VS Code transcript files.
- Existing local hook logging and state-file updates remain unchanged.

### Negative Constraints

- Do NOT modify `mempalace/normalize.py` in this UOW.
- Do NOT regress existing Claude Code or Codex transcript counting and extraction behavior.
- Do NOT change save-marker semantics in `hook_stop`.
- Do NOT make hook execution blocking on malformed transcript records or ingest noise.
- Do NOT introduce new config surface or shell-wrapper behavior.

### Operational Sequence

1. Update `_count_human_messages()` so VS Code `user.message` event-tree records count as canonical user turns while existing Claude/Codex rules stay intact.
2. Update `_extract_recent_messages()` so VS Code canonical user turns are extracted and subagent/tool chatter is excluded.
3. Add regression tests in `tests/test_hooks_cli.py` that use the committed VS Code transcript fixtures for count/extract behavior.
4. Add stop/precompact regression tests in `tests/test_hooks_cli.py` that prove the existing checkpoint and ingest path now fires correctly on real VS Code transcripts.

### Acceptance Criteria

- [ ] A committed VS Code transcript fixture increments the stop-hook exchange counter using canonical `user.message` turns without changing current Claude/Codex counts.
- [ ] A VS Code transcript fixture containing subagent chatter does not count or extract subagent-injected prompts as user turns.
- [ ] `hook_stop` on a VS Code transcript that reaches the save interval still follows the existing checkpoint plus `transcript_path` ingest path and remains non-blocking.
- [ ] `hook_precompact` on a VS Code transcript still ingests and allows compaction to proceed when transcript parsing is noisy or partially malformed.

### Verification

- `python -m pytest tests/test_hooks_cli.py -k "count or extract or stop or precompact"`

---

## UOW-5: Thread VS Code workspace context through fail-open live ingest

**Objective**: Thread the retained VS Code `cwd` field through `_get_mine_dir()`, `_maybe_auto_ingest()`, and `_mine_sync()` so the workspace-scoped follow-on mining associated with the already-working `transcript_path` ingest path resolves against the real project directory instead of the transcript cache location. This UOW contributes to `R-5` only as a secondary owner after UOW-4 establishes the primary observable transcript-path ingest behavior.
**covers**: [R-5]
**depends_on**: [UOW-3, UOW-4]
**parallel_group**: D

### Scope

- **Modify**: `mempalace/hooks_cli.py` — add `cwd` parameters to `_get_mine_dir()`, `_maybe_auto_ingest()`, and `_mine_sync()`, and thread parsed `cwd` from `hook_stop()` and `hook_precompact()`.
- **Modify**: `tests/test_hooks_cli.py` — add `cwd`-aware mine-dir resolution tests, runtime propagation tests, and regressions proving `MEMPAL_DIR` override and fail-open fallback still behave correctly.
- **Do not touch**: `mempalace/normalize.py`.
- **Do not touch**: `mempalace/convo_miner.py`.
- **Do not touch**: `mempalace/cli.py`.
- **Do not touch**: `README.md`, `website/`, `hooks/mempal_save_hook.sh`, `hooks/mempal_precompact_hook.sh`.

### Contracts

- `_get_mine_dir(transcript_path: str = "", cwd: str = "") -> str`
- `_maybe_auto_ingest(transcript_path: str = "", cwd: str = "")`
- `_mine_sync(transcript_path: str = "", cwd: str = "")`
- `hook_stop(data: dict, harness: str)` consumes parsed `cwd` and forwards it to `_maybe_auto_ingest(...)`.
- `hook_precompact(data: dict, harness: str)` consumes parsed `cwd` and forwards it to `_mine_sync(...)`.

### DI Registrations

- None.

### Behavioral Specification

**Preconditions**:

- UOW-4 has already made the existing `transcript_path`-based stop/precompact ingest path work on real VS Code transcripts.
- Earlier hook-plumbing UOWs already allow VS Code payloads carrying `cwd` to reach this runtime surface.

**Input validation**:
| Field | Rule | On violation |
|-------|------|--------------|
| `cwd` | Used only if it points to a real local directory | Silently ignored; resolution falls through |
| `transcript_path` | Existing validator rules stay unchanged | Existing fail-open handling stays unchanged |
| `MEMPAL_DIR` | Existing explicit override remains highest priority when set to a valid directory | Ignore invalid override and continue with normal fallback behavior |

**Authorization**:

- The current local OS user running MemPalace and VS Code Copilot performs all file-system operations.
- `cwd`, `transcript_path`, and hook state paths must be local paths accessible to that user.
- On access failure or subprocess failure, hooks continue fail-open with local logging only.

**Behavior**:

- On success, `_get_mine_dir()` resolves in this order: `MEMPAL_DIR`, then valid `cwd`, then transcript parent, then empty string.
- `hook_stop()` and `hook_precompact()` continue to ingest the live session from `transcript_path`; this UOW only changes the directory used for the follow-on project mine step.
- This UOW is not the final observable owner of `R-5`; it closes the remaining workspace-resolution gap after UOW-4 by making follow-on auto-ingest and sync-mine behavior technically coherent for real VS Code payloads.
- On invalid or missing `cwd`, missing transcript files, or local subprocess failures, preserve current fail-open behavior and do not block chat or compaction.

**Side effects**:

- Background `mempalace mine <dir>` calls launched by `_maybe_auto_ingest()` may target VS Code `cwd` instead of transcript cache parent.
- Synchronous `mempalace mine <dir>` calls launched by `_mine_sync()` may target VS Code `cwd` instead of transcript cache parent.
- Existing `_ingest_transcript(transcript_path)` behavior remains unchanged and still mines the transcript parent with `--mode convos --wing sessions`.

### Negative Constraints

- Do NOT reroute session ingest away from `transcript_path`.
- Do NOT change `_ingest_transcript()` to use `cwd`.
- Do NOT modify `mempalace/normalize.py` or `mempalace/convo_miner.py` in this UOW.
- Do NOT regress Claude/Codex behavior when `cwd` is absent.
- Do NOT add new config surface, docs changes, or shell-wrapper logic.
- Do NOT make invalid `cwd` values fatal.

### Operational Sequence

1. Add `cwd` parameters to `_get_mine_dir()`, `_maybe_auto_ingest()`, and `_mine_sync()`.
2. Insert the `cwd` directory check between `MEMPAL_DIR` and transcript-parent resolution in `_get_mine_dir()`.
3. Update `hook_stop()` to forward `cwd` into `_maybe_auto_ingest(...)`.
4. Update `hook_precompact()` to forward `cwd` into `_mine_sync(...)`.
5. Add tests proving `cwd` resolution, `MEMPAL_DIR` precedence, fallback behavior, and fail-open behavior.

### Acceptance Criteria

- [ ] `_get_mine_dir("", valid_cwd)` returns the workspace `cwd` when `MEMPAL_DIR` is unset.
- [ ] `_get_mine_dir(transcript_path, valid_cwd)` prefers `cwd` over transcript parent when `MEMPAL_DIR` is unset.
- [ ] `_get_mine_dir(...)` still prefers `MEMPAL_DIR` over `cwd`.
- [ ] A VS Code stop-hook payload uses `cwd` as the auto-ingest mine directory.
- [ ] A VS Code precompact payload uses `cwd` as the sync-mine directory.
- [ ] Invalid `cwd` values fail open and do not crash the hook runtime.
- [ ] Claude/Codex behavior remains unchanged when `cwd` is absent.

### Verification

- `python -m pytest tests/test_hooks_cli.py -k "mine_dir or auto_ingest or precompact or parse_harness_input"`

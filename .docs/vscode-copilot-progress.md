# VS Code Copilot integration — progress

## Status: schema captured, raw fixture capture complete, sanitization pending

---

## Completed

### 2026-04-23 — Schema capture

- Set up capture infrastructure (`~/tmp/mempal-vscode-capture/`)
- Captured real `SessionStart` and `Stop` hook payloads from VS Code Copilot agent
- Captured real `PreCompact`, `SubagentStart`, and `SubagentStop` hook payloads from VS Code Copilot agent
- Captured real transcript JSONL from live sessions with tool activity, precompact context, and subagent activity
- Fixed capture script: VS Code sends `hook_event_name` / `session_id` (snake_case), not `hookEventName` / `sessionId` as the official docs say
- Extended `.github/hooks/capture.json` so the temporary capture flow covers `SubagentStart` and `SubagentStop`
- Baseline sanitized fixtures remain committed under `tests/fixtures/vscode_copilot/`; the newer precompact/subagent raw captures are still outside the repo waiting for sanitization
- Documented the real schema in `vscode-copilot-integration-prep.md`

---

## Pending

### Immediate next step — finalize the fixture gate

**Goal**: sanitize and commit the complete raw capture set gathered on 2026-04-23.

Files to add in the repo:

- `tests/fixtures/vscode_copilot/hooks/precompact.json`
- `tests/fixtures/vscode_copilot/hooks/subagent_start.json`
- `tests/fixtures/vscode_copilot/hooks/subagent_stop.json`
- `tests/fixtures/vscode_copilot/transcripts/with_subagent.jsonl`
- `tests/fixtures/vscode_copilot/transcripts/with_precompact_context.jsonl`
- `tests/fixtures/vscode_copilot/README.md`

Docs to align after commit:

- `.docs/vscode-copilot-integration-prep.md`
- `.docs/copilot-setup-surface-and-fixture-gate/*`

### Phase 1 — hooks_cli.py adaptation

**Goal**: recognize VS Code Copilot payloads and map them onto the internal session shape.

Fields to map:

| VS Code field      | Internal field       |
| ------------------ | -------------------- |
| `hook_event_name`  | event type detection |
| `session_id`       | `session_id`         |
| `transcript_path`  | `transcript_path`    |
| `stop_hook_active` | `stop_hook_active`   |
| `cwd`              | workspace context    |

Harness to add: `vscode-copilot`

Files to change:

- `mempalace/hooks_cli.py`
- `tests/test_hooks_cli.py` (fixture-driven tests)

### Phase 2 — normalize.py adaptation

**Goal**: detect and parse the VS Code Copilot transcript JSONL format.

Detection heuristic: first line has `"type": "session.start"` with `"producer": "copilot-agent"`.

Parsing rules:

- Extract `user.message` events → user turns (`data.content`)
- Extract `assistant.message` events → assistant turns (`data.content`)
- Merge multi-part assistant turns within the same `turnId`
- Filter out `tool.execution_start/complete` events (noise)
- Optionally include `data.toolRequests` in assistant turns for context
- Handle parallel tool calls (multiple `tool.execution_start` with same `parentId`)

Files to change:

- `mempalace/normalize.py`
- `tests/test_normalize.py` (fixture-driven tests)

### Fixture gate note

The raw captures that were previously missing are now available outside the repo. What is still missing is the sanitized committed fixture set that phase-1 and phase-2 code will test against.

---

## Key findings

### Hook payload

```json
{
  "timestamp": "2026-04-23T04:19:59.674Z",
  "hook_event_name": "SessionStart",
  "session_id": "b90d268f-...",
  "transcript_path": "/home/.../.jsonl",
  "source": "new",
  "cwd": "/path/to/workspace"
}
```

Docs say `hookEventName`/`sessionId`. Reality: `hook_event_name`/`session_id`.

### Transcript JSONL structure

```
session.start       → session metadata
user.message        → user turn (data.content)
assistant.turn_start / assistant.turn_end  → turn boundaries (data.turnId)
assistant.message   → assistant content (data.content + data.toolRequests)
tool.execution_start  → tool call (data.toolName, data.toolCallId, data.arguments)
tool.execution_complete → tool result (data.success)
```

Parallel tool calls: multiple `tool.execution_start` can share the same `parentId`.

### Fixture paths

- `tests/fixtures/vscode_copilot/hooks/session_start.json`
- `tests/fixtures/vscode_copilot/hooks/stop.json`
- `tests/fixtures/vscode_copilot/transcripts/simple_with_tools.jsonl`

### Raw capture paths now available outside the repo

- `~/tmp/mempal-vscode-capture/payloads/*-PreCompact.json`
- `~/tmp/mempal-vscode-capture/payloads/*-SubagentStart.json`
- `~/tmp/mempal-vscode-capture/payloads/*-SubagentStop.json`
- `~/tmp/mempal-vscode-capture/transcripts/*-PreCompact-*.jsonl`
- `~/tmp/mempal-vscode-capture/transcripts/*-SubagentStart-*.jsonl`
- `~/tmp/mempal-vscode-capture/transcripts/*-SubagentStop-*.jsonl`


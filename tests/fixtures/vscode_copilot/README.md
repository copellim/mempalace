# VS Code Copilot Fixtures

Sanitized real VS Code Copilot hook payloads and transcript files for testing MemPalace's VS Code integration.

## Fixtures

### Hook payloads (`hooks/`)

| File                  | hook_event_name | Session           | Notes                                           |
| --------------------- | --------------- | ----------------- | ----------------------------------------------- |
| `session_start.json`  | `SessionStart`  | test-session-0001 | Baseline fixture (pre-existing)                 |
| `stop.json`           | `Stop`          | test-session-0001 | Baseline fixture (pre-existing)                 |
| `precompact.json`     | `PreCompact`    | test-session-0002 | **New in increment 1** — real capture sanitized |
| `subagent_start.json` | `SubagentStart` | test-session-0003 | **New in increment 1** — real capture sanitized |
| `subagent_stop.json`  | `SubagentStop`  | test-session-0003 | **New in increment 1** — real capture sanitized |

### Transcripts (`transcripts/`)

| File                            | Session           | Notes                                                               |
| ------------------------------- | ----------------- | ------------------------------------------------------------------- |
| `simple_with_tools.jsonl`       | test-session-0001 | Baseline fixture (pre-existing)                                     |
| `with_precompact_context.jsonl` | test-session-0002 | **New in increment 1** — real long-session transcript sanitized     |
| `with_subagent.jsonl`           | test-session-0003 | **New in increment 1** — real subagent session transcript sanitized |

## Field Casing Notes

VS Code Copilot hook payloads use **snake_case** for all fields:

- `hook_event_name` (not `hookEventName` as the official docs imply)
- `session_id` (not `sessionId`)
- `transcript_path` (snake_case — matches both docs and reality)

VS Code Copilot transcript events use **camelCase** in data objects:

- `sessionId`, `messageId`, `turnId`, `parentId` (camelCase)

## Subagent Hook Payloads

The following hook-event name values were observed in the real raw subagent captures:

- `SubagentStart` — fires when a subagent is created; includes `agent_id` and `agent_type` fields
- `SubagentStop` — fires when a subagent completes; includes `agent_id`, `agent_type`, and `stop_hook_active` fields

The `agent_id` in SubagentStart/SubagentStop corresponds to the `toolCallId` of the `runSubagent` tool call in the parent session transcript.

## Subagent Transcript Event Types

The following event type strings were observed in the raw subagent transcript capture (`with_subagent.jsonl`):

- `session.start`
- `user.message`
- `assistant.turn_start`
- `assistant.message`
- `assistant.turn_end`
- `tool.execution_start`
- `tool.execution_complete`

Tool names observed in subagent transcript: `runSubagent`, `read_file`, `vscode_askQuestions`

## PreCompact Correlation

The PreCompact hook payload and the correlated transcript are linked by their shared session identifier:

| Property                     | Value                                                                                                                                                                                                                                                                                      |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Payload filename             | `precompact.json`                                                                                                                                                                                                                                                                          |
| Transcript filename          | `with_precompact_context.jsonl`                                                                                                                                                                                                                                                            |
| Session-id field             | `session_id` (in payload), `sessionId` (in transcript session.start event data)                                                                                                                                                                                                            |
| Session-id value (sanitized) | `test-session-0002-0000-0000-000000000000`                                                                                                                                                                                                                                                 |
| Hook-event field             | `hook_event_name`                                                                                                                                                                                                                                                                          |
| Hook-event value             | `PreCompact`                                                                                                                                                                                                                                                                               |
| Correlation rule             | The `session_id` in `precompact.json` matches the `data.sessionId` in the `session.start` event of `with_precompact_context.jsonl`. Timestamp in payload (2026-04-23T05:14:36.734Z) is later than any event in the transcript, confirming the transcript was active when PreCompact fired. |

## Provenance

All fixtures are sanitized from real VS Code Copilot captures made on 2026-04-23.

Raw source (outside repo): `~/tmp/mempal-vscode-capture/`

Sanitization applied:

- Real session UUIDs replaced with `test-session-XXXX-0000-0000-000000000000`
- Real event UUIDs replaced with `event-XXXX` sequential IDs
- Real tool call IDs replaced with `toolcall-XXXX` sequential IDs
- Real user paths replaced with `/home/user/projects/myapp`
- Real VS Code config paths replaced with `/tmp/mempalace-test-transcripts/`
- Real username replaced with `user`
- Non-English message content replaced with English equivalents
- All JSON structure, event types, field names, and field values preserved verbatim


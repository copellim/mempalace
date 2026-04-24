# VS Code Copilot integration prep

## REAL SCHEMA — confirmed 2026-04-23

### Hook payload fields (actual, snake_case — **differs from official docs**)

```json
{
  "timestamp": "2026-04-23T04:19:59.674Z",
  "hook_event_name": "SessionStart",
  "session_id": "b90d268f-f7ac-459c-86c2-a3a54ddf3277",
  "transcript_path": "/home/.../.jsonl",
  "source": "new",
  "cwd": "/path/to/workspace"
}
```

Docs say `hookEventName` / `sessionId` (camelCase). Reality sends `hook_event_name` / `session_id` (snake_case).
`transcript_path` is snake_case in both docs and reality.

Stop-specific extra fields: `stop_hook_active: false`
PreCompact-specific extra fields: `trigger: "auto"`

### Transcript format (JSONL, event tree)

Each line is a JSON event with `type`, `data`, `id`, `timestamp`, `parentId`. Event types:

- `session.start` — session metadata (`data.sessionId`, `data.producer`, versions)
- `user.message` — user turn (`data.content`, `data.attachments`)
- `assistant.turn_start` / `assistant.turn_end` — turn boundaries (`data.turnId`)
- `assistant.message` — assistant content: `data.content` (text) + `data.toolRequests` (array) + `data.reasoningText`
- `tool.execution_start` — tool invocation: `data.toolName`, `data.toolCallId`, `data.arguments` (object)
- `tool.execution_complete` — tool result: `data.toolCallId`, `data.success`

Tool names are VS Code camelCase: `read_file`, `replace_string_in_file`, `vscode_askQuestions`, etc.
Multiple tool calls can be parallel (same `parentId`, overlapping `tool.execution_start` events).

### Sanitized fixtures committed

- `tests/fixtures/vscode_copilot/hooks/session_start.json`
- `tests/fixtures/vscode_copilot/hooks/stop.json`
- `tests/fixtures/vscode_copilot/hooks/precompact.json` _(added increment 1)_
- `tests/fixtures/vscode_copilot/hooks/subagent_start.json` _(added increment 1)_
- `tests/fixtures/vscode_copilot/hooks/subagent_stop.json` _(added increment 1)_
- `tests/fixtures/vscode_copilot/transcripts/simple_with_tools.jsonl`
- `tests/fixtures/vscode_copilot/transcripts/with_precompact_context.jsonl` _(added increment 1)_
- `tests/fixtures/vscode_copilot/transcripts/with_subagent.jsonl` _(added increment 1)_
- `tests/fixtures/vscode_copilot/README.md` — fixture manifest _(added increment 1)_

### Raw capture status

**COMPLETE** — all captures sanitized and committed as of 2026-04-23.

Raw source location (outside repo, kept for reference): `~/tmp/mempal-vscode-capture/`

---

## Goal

Prepare the minimum real-world material needed to start the VS Code Copilot integration work safely, especially the data required to adapt transcript ingest and normalization.

The main rule is simple:

- do **not** start by guessing the VS Code transcript schema
- collect **real hook payloads** and **real transcript files** first
- only then adapt `hooks_cli.py` and `normalize.py`

## Why this comes first

The current repository is already close on the hook side:

- `mempalace/hooks_cli.py` already handles `session_id`, `transcript_path`, and `stop_hook_active`
- `mempalace/normalize.py` already supports multiple transcript formats
- the missing piece is first-class recognition of **VS Code Copilot transcript shape**

That made fixture capture the first practical step, and that raw-capture step is now complete.

## Files that matter most

These are the main files that will be involved once implementation starts:

- `mempalace/hooks_cli.py`
- `mempalace/normalize.py`
- `tests/test_hooks_cli.py`
- `tests/test_normalize.py`

For bootstrap project ingest, the existing command is already enough:

```bash
mempalace mine ~/projects/myapp
```

The new work is about **conversation ingest from VS Code Copilot**, not project mining.

## Recommended first setup

Install the project in editable mode and run the current tests for the relevant surfaces:

```bash
pip install -e ".[dev]"
python -m pytest tests/test_normalize.py tests/test_hooks_cli.py -v
```

## Raw capture status before coding

The small but real VS Code dataset is now available outside the repo.

Captured raw set:

1. hook payload for `SessionStart`
2. hook payload for `Stop`
3. hook payload for `PreCompact`
4. hook payloads for `SubagentStart` and `SubagentStop`
5. transcript files referenced by `transcript_path`
6. a transcript containing real tool noise
7. a transcript snapshot correlated with `PreCompact`
8. a transcript snapshot containing subagent activity

## Capture strategy

Do not hunt for hidden VS Code storage manually first.

Use VS Code hooks to dump:

- the full hook payload JSON
- the file pointed to by `transcript_path`

This is the cleanest way to get authoritative fixtures.

## Local capture workspace

Create a local folder outside the repository:

```bash
mkdir -p ~/tmp/mempal-vscode-capture/{payloads,transcripts}
```

## Capture script

Create:

`~/tmp/mempal-vscode-capture/capture_vscode_hook.py`

```python
#!/usr/bin/env python3
import json
import shutil
import sys
from datetime import datetime
from pathlib import Path

out = Path.home() / "tmp" / "mempal-vscode-capture"
(out / "payloads").mkdir(parents=True, exist_ok=True)
(out / "transcripts").mkdir(parents=True, exist_ok=True)

data = json.load(sys.stdin)
stamp = datetime.now().strftime("%Y%m%d-%H%M%S-%f")
event = data.get("hookEventName", "unknown")

(out / "payloads" / f"{stamp}-{event}.json").write_text(
    json.dumps(data, indent=2),
    encoding="utf-8",
)

transcript = data.get("transcript_path")
if transcript:
    src = Path(transcript).expanduser()
    if src.is_file():
        shutil.copy2(src, out / "transcripts" / f"{stamp}-{event}-{src.name}")

print(json.dumps({"continue": True}))
```

Make it executable:

```bash
chmod +x ~/tmp/mempal-vscode-capture/capture_vscode_hook.py
```

## VS Code hook config for capture

Create:

`.github/hooks/capture.json`

```json
{
  "hooks": {
    "SessionStart": [
      {
        "type": "command",
        "command": "/home/michele/tmp/mempal-vscode-capture/capture_vscode_hook.py"
      }
    ],
    "SubagentStart": [
      {
        "type": "command",
        "command": "/home/michele/tmp/mempal-vscode-capture/capture_vscode_hook.py"
      }
    ],
    "SubagentStop": [
      {
        "type": "command",
        "command": "/home/michele/tmp/mempal-vscode-capture/capture_vscode_hook.py"
      }
    ],
    "Stop": [
      {
        "type": "command",
        "command": "/home/michele/tmp/mempal-vscode-capture/capture_vscode_hook.py"
      }
    ],
    "PreCompact": [
      {
        "type": "command",
        "command": "/home/michele/tmp/mempal-vscode-capture/capture_vscode_hook.py"
      }
    ]
  }
}
```

## What to inspect in the captured files

For hook payloads, verify:

- exact casing of fields
- presence of `sessionId`
- presence of `transcript_path`
- presence of `stop_hook_active`
- any extra fields that may be useful later

For transcript files, verify:

- whether the file is JSON or JSONL
- where canonical user turns live
- where canonical assistant turns live
- how tool calls and tool results are represented
- what should be treated as system noise
- whether multi-part assistant turns need merging
- whether subagent content appears in the transcript

## Sanitization rules

Keep raw captures **outside the repository**.

Only copy sanitized fixtures into the repo.

When sanitizing:

- remove secrets
- replace private names and paths
- preserve the original JSON structure
- preserve tool noise and event layout

Do **not** over-clean the transcript. The noise is exactly what the normalizer needs to learn to ignore or merge correctly.

## Suggested fixture layout in the repo

Once sanitized, store fixtures under:

- `tests/fixtures/vscode_copilot/hooks/`
- `tests/fixtures/vscode_copilot/transcripts/`

## Implementation order after capture

Now that the raw capture set exists, proceed in this order:

1. sanitize and commit the captured raw fixture set under `tests/fixtures/vscode_copilot/`
2. add `vscode-copilot` harness support in `mempalace/hooks_cli.py`
3. map VS Code fields onto the internal shape:
   - `sessionId` -> `session_id`
   - `transcript_path` -> `transcript_path`
   - `stop_hook_active` -> `stop_hook_active`
4. add VS Code transcript detection and parsing in `mempalace/normalize.py`
5. add fixture-driven tests in:
   - `tests/test_hooks_cli.py`
   - `tests/test_normalize.py`

## Practical note on ingest

The hook side is already structurally close to what VS Code provides.

The main uncertainty is the transcript format behind `transcript_path`.

So the fastest safe path is:

1. sanitize the captured real hook payloads and transcript files into fixtures
2. commit the fixture manifest and aligned docs
3. only then write the VS Code normalizer and harness support

That avoids building the ingest layer on assumptions.


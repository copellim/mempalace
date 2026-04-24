# Codebase Conventions — MemPalace

_Source references verified in session. Last updated: phase 2._

## Naming

- **Module-level constants**: UPPER_SNAKE_CASE. Example: `SAVE_INTERVAL`, `STATE_DIR`, `SUPPORTED_HARNESSES` in `mempalace/hooks_cli.py`.
- **Functions and variables**: snake_case. Example: `_parse_harness_input`, `hook_stop`, `transcript_path`, `messages` in `mempalace/hooks_cli.py` and `mempalace/normalize.py`.
- **Private helper functions**: underscore prefix `_` (e.g., `_parse_harness_input`, `_count_human_messages`, `_extract_recent_messages`). Public functions have no prefix (e.g., `hook_stop`, `hook_session_start`, `run_hook`).

## Fail-Open Pattern

- **Graceful error handling**: Catch specific exceptions (`json.JSONDecodeError`, `OSError`, `AttributeError`, `subprocess.TimeoutExpired`) and return sensible defaults (empty dict `{}`, empty list `[]`, `0`, `None`) without raising to caller. Example: `_count_human_messages` returns `0` on file errors; `_extract_recent_messages` returns `[]` on OSError.
- **stdin parsing resilience**: Wrap `json.load(sys.stdin)` in try/except for `json.JSONDecodeError` and `EOFError`. Log warning and proceed with empty dict. Example: `run_hook()` in `hooks_cli.py`.
- **Pass-through on invalid input**: Validate input and skip invalid entries without error (e.g., skip lines that don't parse as JSON, skip entries with wrong structure). Pattern: `try/except json.JSONDecodeError: continue` in JSONL parsers.

## Dispatcher Pattern

- **Dict-based dispatchers**: Map string keys to handler functions in a dict. Look up handler with `.get()`. On unknown key, print to `sys.stderr` and call `sys.exit(1)`. Example in `run_hook()`:
  ```python
  hooks = {
      "session-start": hook_session_start,
      "stop": hook_stop,
      "precompact": hook_precompact,
  }
  handler = hooks.get(hook_name)
  if handler is None:
      print(f"Unknown hook: {hook_name}", file=sys.stderr)
      sys.exit(1)
  ```
- **argparse choices**: Register valid string choices as lists (e.g., `choices=["session-start", "stop", "precompact"]`).

## Harness and Format Extension Pattern

- **Supported harnesses set**: Maintain `SUPPORTED_HARNESSES` as a set of strings (e.g., `{"claude-code", "codex"}`).
- **Harness validation in parser**: Check `if harness not in SUPPORTED_HARNESSES: sys.exit(1)`.
- **Normalized return dict**: `_parse_harness_input` accepts `data: dict` and `harness: str`, returns normalized dict with extracted keys (`session_id`, `stop_hook_active`, `transcript_path`). Keys are additive for new harnesses.

## JSONL Parser Pattern

- **Line iteration and JSON parsing**: Strip lines, split by `\n`, skip empty lines. Wrap `json.loads(line)` in try/except catching `json.JSONDecodeError`, skip invalid lines.
- **Type validation**: After parsing, check structure with `isinstance()` (e.g., `isinstance(entry, dict)`, `isinstance(payload, dict)`). Skip if type is wrong.
- **Sentinel field detection**: Identify message format by checking for specific fields:
  - Codex: `entry_type == "session_meta"` flag, `entry_type == "event_msg"`, `payload.get("type") == "user_message"`.
  - Claude Code: `msg.get("role") == "user"`, `msg_type in ("human", "user")`.
- **Return type**: Parser functions return `Optional[str]` (normalized transcript string or `None`).
- **Extraction helpers**: Use helper functions to extract and clean message text (e.g., `_extract_content()`, `strip_noise()`).

## Chain-of-Parsers Pattern

- **Sequential format detection**: Main function (e.g., `_try_normalize_json`) calls format-specific parsers in sequence. Each returns `Optional[str]`. Return first non-None result with early return (`if normalized: return normalized`).
- **Order matters**: Try more specific/likely formats first, fall back to general JSON parsers last.

## Test Conventions

- **Test file naming**: `test_<module>.py` files.
- **Function imports**: Import functions directly from module under test for clear dependency and easy mocking:
  ```python
  from mempalace.hooks_cli import (
      _count_human_messages,
      _parse_harness_input,
      hook_stop,
      run_hook,
  )
  ```
- **Temp files**: Use pytest `tmp_path` fixture.
- **Helper functions**: Extract test setup into module-level helper functions with `_` prefix (e.g., `_write_transcript(path, entries)`):
  ```python
  def _write_transcript(path: Path, entries: list[dict]):
      with open(path, "w", encoding="utf-8") as f:
          for entry in entries:
              f.write(json.dumps(entry) + "\n")
  ```
- **Mocking**: Use `unittest.mock.patch` and `MagicMock` for subprocess and stdout mocking.
- **Assertions**: Simple `assert` statements with expected value. Example: `assert _count_human_messages(str(transcript)) == 2`.

## Import Style

- **Stdlib imports**: Import at module level before local imports.
- **Local/deferred imports**: Import inside function when avoiding circular dependencies or deferring expensive imports:
  ```python
  def cmd_hook(args):
      from .hooks_cli import run_hook
      run_hook(...)
  ```

## Function Signatures

- **Optional parameters with defaults**: Use default values (e.g., `count: int = _RECENT_MSG_COUNT`).
- **Path handling**: Accept paths as `str` parameter and convert with `Path()` inside function.
- **Return type annotations**: Use for public and format-detection functions. Example: `Optional[str]`, `list[str]`, `int`, `dict`.

## File Organization

- **Hook handler structure**: Define internal helpers first, then public hook handlers (e.g., `hook_stop`, `hook_session_start`, `hook_precompact`), then main entry point (`run_hook`).
- **Normalization module structure**: Define chain-of-parsers entry point first, then format-specific parsers in sequence, then helper functions.


# hooks

**Role:** Claude hook dispatcher. Receives hook events from the Claude CLI (`PreToolUse`, `PostToolUse`, `Stop`, `Notification`, `CompactStart`) via stdin, routes them to configured handlers (shell commands or Python callables), and logs results.

**Source:** `src/aipass/hooks/apps/`

---

## How Hooks Work

Claude CLI calls hooks by:
1. Writing JSON to the hook script's stdin
2. Running the script
3. Reading stdout back

AIPass hooks scripts live in `.claude/hooks/` and call `drone @hooks dispatch <event_type>`.

---

## Commands

### engine (dispatch)
**File:** `apps/modules/engine.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "dispatch"
  # args: [event_type]   e.g. "PreToolUse"
  # reads stdin, calls dispatch(event_type, stdin, config)

dispatch(event_type: str, stdin_data: str, config: dict) -> str
  # routes event to all matching hooks in config[event_type]
  # returns merged stdout from all hooks
  # matching: hook_def.get("matcher") checked against tool_name/compact_type/type
  # skips disabled hooks
  # returns "" if hooks_enabled=False or no hooks for event
```

**Hook def shape in config:**
```python
{
  "handler": "module.path.function",   # Python callable (direct import)
  "command": "shell command string",   # or shell command
  "matcher": "tool_name1|tool_name2",  # empty = match all
  "enabled": True,
  "timeout": 30,                       # seconds, for shell commands
}
```

**Internal dispatch helpers:**
```python
_run_hook(hook_cmd: str, stdin_data: str, timeout_s: int = 30) -> dict
  # subprocess, shell=True
  # returns {exit_code, stdout, stderr, elapsed_ms}

_run_handler(handler_path: str, hook_data: dict) -> dict
  # importlib.import_module + getattr
  # handler_fn(hook_data: dict) -> {exit_code, stdout, stderr}
  # returns same shape

_matches(matcher: str, value: str) -> bool
  # value in matcher.split("|")
```

---

### hookstatus
**File:** `apps/modules/hookstatus.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "show", "reset", "tail [N]"

# Manages hook event state persistence
# Reads/writes: hooks/config/hook_state.json
```

---

### hooksound
**File:** `apps/modules/hooksound.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "play <sound_name>", "list", "enable", "disable"

# Plays .wav/.mp3 files on hook events
# Sound files: .claude/sounds/
# Config: hooks/config/sound_config.json
```

---

## Config Schema (`hooks/config/hooks_config.json`)

```json
{
  "hooks_enabled": true,
  "PreToolUse": {
    "<hook_name>": {
      "handler": "aipass.memory.apps.modules.symbolic.process_hook",
      "matcher": "",
      "enabled": true
    }
  },
  "PostToolUse": {
    "<hook_name>": {
      "command": "drone @seedgo hook-check",
      "matcher": "Write|Edit",
      "enabled": true,
      "timeout": 60
    }
  },
  "Stop": {},
  "Notification": {},
  "CompactStart": {}
}
```

---

## Hook stdin payload (from Claude CLI)

```json
{
  "type": "PreToolUse | PostToolUse | Stop | Notification | CompactStart",
  "tool_name": "string",
  "tool_input": {},
  "tool_result": {},
  "compact_type": "string | null"
}
```

---

## Diagnostics Log
**File:** `apps/handlers/config/diagnostics.py`

```python
log_entry(entry: dict) -> None
  # appends to hooks/config/hook_diag.jsonl
  # schema: {ts, event, hook, exit_code, elapsed_ms, stdout_len, stderr_preview, cwd}

tail_log(n: int = 20) -> List[dict]
  # reads last n entries from hook_diag.jsonl
```

---

## Dependencies

| Depends on | Why |
|-----------|-----|
| `prax` | logger |
| `cli` | err_console output |
| `memory` (symbolic) | `process_hook` is the main PreToolUse handler |

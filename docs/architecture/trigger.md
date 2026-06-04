# trigger

**Role:** Event bus. Branches fire named events; registered handlers react. Also watches `.prax.log` for new log lines and dispatches log-based events. Includes a `medic` module for self-healing automation.

**Source:** `src/aipass/trigger/apps/`

---

## Core: Trigger class
**File:** `apps/modules/core.py`

```python
class Trigger:
    # Class-level state (singleton pattern)
    _handlers: Dict[str, List[Callable]]  # event_name → [handler_fn, ...]
    _history: List[Dict]
    _initialized: bool
    _firing: bool                           # recursion guard
    _deferred_queue: List[Dict]             # events fired during handling
    _draining_deferred: bool
    _log_watcher_started: bool              # lazy-start flag (DISABLED — see note)
    _handler_failures: Dict[tuple, int]     # (handler_fn, branch) → failure count
    _disabled_handlers: Set[tuple]          # auto-disabled after 5 consecutive failures
    _HANDLER_FAILURE_THRESHOLD: int = 5

    @classmethod
    def register(cls, event_name: str, handler: Callable) -> None
      # adds handler to event_name bucket

    @classmethod
    def fire(cls, event_name: str, **kwargs) -> List[Any]
      # calls all handlers for event_name
      # defers re-entrant fires to _deferred_queue
      # drains queue after top-level call completes
      # auto-disables handlers after 5 consecutive failures

    @classmethod
    def fire_for_branch(cls, event_name: str, branch: str, **kwargs) -> List[Any]
      # fires event scoped to a specific branch

    @classmethod
    def list_events(cls) -> List[str]
    @classmethod
    def list_handlers(cls, event_name: str) -> List[Callable]
    @classmethod
    def reset(cls) -> None
      # clears all handlers; used in tests
```

**Note:** Log watcher lazy-start is **DISABLED** (inotify exhaustion issue — see `_ensure_log_watcher`). Log watching is done by `prax monitor` as a dedicated process.

---

## Commands

### core module (handle_command)
**File:** `apps/modules/core.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # No drone commands — Trigger is used as a library.
  # Returns False always; command routing unused.
```

---

### log_events
**File:** `apps/modules/log_events.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "watch [--tail N]", "status", "history"
  # registers log-line patterns → event fire mappings
```

---

### branch_log_events
**File:** `apps/modules/branch_log_events.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "watch @branch", "list-watched"
  # watches a specific branch's log file
```

---

### medic
**File:** `apps/modules/medic.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "run [--dry-run]", "status", "rules"

# Medic: self-healing event handler
# Registered against trigger events; fires when anomaly detected
# Rules: {event_pattern: action_fn}
# Actions: restart agent, send ai_mail, write to log
```

---

### errors
**File:** `apps/modules/errors.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "list", "clear", "summary"
  # reads error events from trigger history
```

---

## Event Handler Registration

Handlers auto-register via `apps/handlers/events/registry.py`:

```python
def setup_handlers() -> None
  # called once on first Trigger use (_ensure_initialized)
  # imports handler modules, calls Trigger.register() for each
```

---

## Built-in Events

| Event name | Fired by | Payload |
|-----------|---------|---------|
| `mail_dispatch` | `ai_mail.email_send._fire_dispatch_trigger` | `{branch, subject}` |
| `log_error` | log watcher | `{line, level, branch, timestamp}` |
| `agent_crashed` | dispatch_monitor | `{branch, pid, exit_code}` |
| `plan_closed` | `flow.close_plan` | `{plan_key, title}` |
| `rollover_complete` | `memory.rollover` | `{branch, entries_archived}` |

---

## Handler Signature

All registered event handlers follow:

```python
def my_handler(event_name: str, **kwargs) -> Any
  # kwargs = event payload fields
  # return value is collected but not currently used
```

---

## Log Watcher Service
**File:** `log_watcher_service.py` (top-level, not in apps/)

```python
# Standalone script run by prax monitor or as daemon
# Uses watchdog/inotify to tail .prax.log
# For each new line: parse log entry, fire matching Trigger event
```

---

## Dependencies

| Depends on | Why |
|-----------|-----|
| `prax` | logger; watches prax log file |
| `ai_mail` | medic sends mail on certain events |
| `cli` | display |

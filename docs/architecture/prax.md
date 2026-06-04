# prax

**Role:** Observability layer. Every other agent imports `from aipass.prax import logger` — this is the single shared logging sink. Also provides real-time file + log watching, dashboard state, and log auditing.

**Source:** `src/aipass/prax/apps/`

---

## Commands

### logger
**File:** `apps/modules/logger.py`

```python
# Public API — imported everywhere as:
#   from aipass.prax import logger
#   from aipass.prax.apps.modules.logger import system_logger as logger

system_logger: SystemLogger  # singleton, auto-routing

get_system_logger() -> SystemLogger
  # creates/returns singleton
  # auto-detects calling module via stack walk

# SystemLogger wraps stdlib logging.Logger
# Routes each call to per-module log file: .prax/logs/<branch>/<module>.log
# Also writes to .prax.log (shared, structured JSON lines)

class SystemLogger:
    def debug(self, msg, *args, **kwargs) -> None
    def info(self, msg, *args, **kwargs) -> None
    def warning(self, msg, *args, **kwargs) -> None
    def error(self, msg, *args, **kwargs) -> None
    def critical(self, msg, *args, **kwargs) -> None

class DirectLogger:
    # Bypasses the event pipeline — for infrastructure use
    # (hook callbacks, dispatch monitor) to avoid recursion

def get_direct_logger(name: str) -> DirectLogger
def direct_log(name: str, level: str, msg: str, **kwargs) -> None

def initialize_logging_system() -> None
def shutdown_logging_system() -> None
def get_system_status() -> Dict[str, Any]
def enable_terminal_output() -> None   # show logs in terminal
def disable_terminal_output() -> None

handle_command(command: str, args: List[str]) -> bool
  # commands: "status", "tail [N]", "enable-terminal", "disable-terminal"
```

**Writes:** `.prax/logs/<branch>/<module>.log`, `.prax.log`

---

### monitor
**File:** `apps/modules/monitor.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "run [--filter <level>]", "status"

_run_monitor(args: List[str]) -> bool
  # starts two watchers in background threads:
  #   1. File watcher (watchdog inotify) on repo directories
  #   2. Log watcher on .prax.log
  # interactive loop: user types "quit", "status", "filter <level>"

_get_watch_directories(repo_root: Path) -> list[tuple[Path, bool]]
  # returns (path, recursive) pairs
  # watches: src/, .aipass/, .ai_mail.local/, .flow-registry.json

_start_observer_with_fallback(handler, watch_dirs) -> Observer
  # tries inotify, falls back to polling if exhausted

_start_log_watcher_with_fallback(event_queue) -> bool
_get_pid_for_branch(branch: str) -> int | None
  # reads branch .dispatch_lock to get active PID
```

**Reads:** `.prax.log`, `AIPASS_REGISTRY.json`

---

### dashboard
**File:** `apps/modules/dashboard.py`

```python
update_section(branch_path: Path, section_name: str, section_data: Dict) -> bool
  # writes one section of DASHBOARD.local.json
  # sections: "status", "active_plans", "recent_mail", "memory_stats"

print_status() -> None
  # reads DASHBOARD.local.json, renders Rich table

handle_command(command: str, args: List[str]) -> bool
  # commands: "refresh [@branch]", "show [@branch]", "update <section> <json>"
```

**Reads/Writes:** `<branch>/DASHBOARD.local.json`

---

### status
**File:** `apps/modules/status.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "show", "agents", "system"
  # aggregates: active agents, PID states, log tail
```

---

### log_audit
**File:** `apps/modules/log_audit.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "run [--enforce]", "report"
  # checks log files for size limits, rotation, stale entries
  # enforce: rotates oversized files
```

---

## Log File Schemas

### `.prax.log` (append-only, one JSON object per line)
```json
{"ts": 1234567890.123, "level": "INFO", "module": "router", "branch": "flow", "msg": "..."}
```

### `DASHBOARD.local.json`
```json
{
  "branch": "string",
  "last_updated": "ISO-8601",
  "status": {
    "state": "active | idle | busy",
    "current_task": "string | null"
  },
  "active_plans": [{"key": "...", "title": "..."}],
  "recent_mail": [{"from": "@...", "subject": "...", "ts": "..."}],
  "memory_stats": {
    "local_json_lines": 0,
    "observations_lines": 0,
    "chroma_fragments": 0
  }
}
```

---

## Key Handler Modules

| Path | Purpose |
|------|---------|
| `handlers/logging/setup.py` | `setup_individual_logger(name, path) -> Logger` |
| `handlers/logging/override.py` | `is_override_active() -> bool` — env-var log level override |
| `handlers/logging/direct.py` | `DirectLogger` class — bypass pipeline |
| `handlers/logging/introspection.py` | `get_caller_info() -> Dict` — stack-walk module detection |
| `handlers/registry/load.py` | `load_module_registry() -> Dict` — which modules log where |
| `handlers/config/load.py` | `get_system_logs_dir()`, `get_module_logs_dir()`, `PRAX_JSON_DIR` |
| `handlers/discovery/watcher.py` | `start_file_watcher(path) -> bool`, `is_file_watcher_active() -> bool` |

---

## Dependencies

**Prax has no runtime dependencies on other AIPass agents.** It is the foundation layer — everything imports prax, prax imports nothing from other agents.

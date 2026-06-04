# ai_mail

**Role:** Agent-to-agent message queue. Handles composing, delivering, and reading emails between agents. Optionally wakes sleeping agents via the dispatch daemon.

**Source:** `src/aipass/ai_mail/apps/`

---

## Commands

### email (inbox / read / send)
**File:** `apps/modules/email.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # sub-commands: "inbox", "view <id>", "close <id>", "reply <id>",
  #               "sent", "contacts", "register"

handle_inbox(args: List[str]) -> bool
  # reads .ai_mail.local/<branch>/inbox/, displays new mail

handle_view(args: List[str]) -> bool
  # args: [id]   id = email filename stem or numeric index

handle_close(args: List[str]) -> bool
  # marks email status = "closed"

handle_reply(args: List[str]) -> bool
  # args: [id, "Subject", "Body"]

handle_sent(args: List[str]) -> bool
  # reads .ai_mail.local/<branch>/sent/

handle_contacts(args: List[str]) -> bool
  # reads AIPASS_REGISTRY for all branch emails

handle_register(args: List[str]) -> bool
  # adds a branch to contacts
```

**Reads:** `.ai_mail.local/<self>/inbox/*.json`, `.ai_mail.local/<self>/sent/*.json`  
**Writes:** `.ai_mail.local/<recipient>/inbox/<timestamp>_<id>.json`, `.ai_mail.local/<self>/sent/<id>.json`

---

### email_send
**File:** `apps/modules/email_send.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "send", "broadcast"

handle_send(args: List[str]) -> bool
  # args: [@to, "Subject", "Body"] or interactive prompt

_send_direct(
    to_branch: str,
    subject: str,
    message: str,
    user_info: Dict,
    auto_execute: bool = False,
    no_memory_save: bool = False,
    reply_to: str | None = None,
    dispatched_to: str | None = None,
) -> bool
  # writes email JSON to recipient inbox
  # calls _fire_dispatch_trigger if auto_execute=True

_send_broadcast(
    subject: str,
    message: str,
    user_info: Dict,
    auto_execute: bool,
    no_memory_save: bool,
    reply_to: str | None,
    dispatched_to: str | None,
) -> bool
  # sends to all active branches except self

_fire_dispatch_trigger(to_branch: str, subject: str) -> None
  # fires trigger event "mail_dispatch" for the target branch
```

**Writes:** `.ai_mail.local/<recipient>/inbox/<id>.json`

---

### dispatch
**File:** `apps/modules/dispatch.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "status", "wake @branch", "send @branch Subject Body",
  #           "daemon", "watchdog"

_orchestrate_status() -> bool
  # reads dispatch log, renders table

_orchestrate_wake(args: List[str]) -> bool
  # args: ["@branch"]  — calls wake.wake_branch()

_orchestrate_dispatch_send(args: List[str]) -> bool
  # args: ["@branch", "Subject", "Body"]
  # sends email then wakes branch

_orchestrate_daemon() -> bool
  # starts daemon.run_daemon() in foreground (blocks)

_spawn_watchdog(target: str) -> None
  # runs prax watchdog for a branch
```

---

## Key Handlers

### dispatch/wake.py

```python
wake_branch(
    target: str,                 # "@branchname"
    prompt: str = DEFAULT_PROMPT,
    model: str = DEFAULT_MODEL,  # "opus" | "sonnet" | "haiku"
    no_memory_save: bool = False,
) -> bool
  # spawns: claude -p "<prompt>" --permission-mode bypassPermissions
  #         in branch CWD
  # returns False if: wake blocked, lock held, branch occupied

is_wake_blocked(target: str) -> bool
  # WAKE_BLOCKLIST = {"@devpulse"}

resolve_branch(branch_email: str) -> Tuple[Path, str] | None
  # email "@name" → (branch_path, branch_name)

_acquire_lock(branch_path: Path, pid: int) -> Tuple[bool, str]
_check_lock(branch_path: Path) -> dict | None
  # lock file: branch_path/.ai_mail.local/.dispatch_lock
```

**Subprocess:** `claude -p <prompt> --permission-mode bypassPermissions`  
**Reads:** `AIPASS_REGISTRY.json`, `ai_mail/safety_config.json`  
**Writes:** `.ai_mail.local/.dispatch_lock`  

---

### dispatch/daemon.py

```python
run_daemon() -> None
  # main loop: poll_cycle() every config["poll_interval_seconds"] (default 30s)
  # polls each registered branch's inbox for auto_execute=True emails
  # spawns agent via spawn_agent() when unread dispatch mail found

poll_cycle(config: Dict, state: Dict) -> int
  # returns count of dispatches started this cycle

spawn_agent(
    branch_path: Path,
    branch_email: str,
    email: Dict,
    config: Dict,
) -> bool
  # calls dispatch_monitor.py subprocess; logs to dispatch log

check_inbox_for_dispatch(branch_path: Path) -> Dict[str, Any] | None
  # reads inbox/, returns first email with status="new" and auto_execute=True

load_config() -> Dict[str, Any]
  # reads ai_mail/safety_config.json or returns defaults
  # key fields: poll_interval_seconds, max_concurrent_agents, kill_switch
```

**Subprocess:** `python dispatch_monitor.py`  
**Reads:** `.ai_mail.local/<branch>/inbox/*.json`, `.aipass/autonomous_pause`  
**Writes:** `.aipass/dispatch_log.json`, `.ai_mail.local/.daemon_state.json`

---

### dispatch/status.py

```python
load_dispatch_log() -> List[Dict[str, Any]]
save_dispatch_log(dispatches: List[Dict]) -> bool
log_dispatch(branch: str, pid: int | None, status: str, error_msg: str | None) -> bool
check_pid_status(pid: int) -> str   # "running" | "stopped" | "unknown"
```

**Reads/Writes:** `.aipass/dispatch_log.json`

---

### users/user.py

```python
get_current_user() -> Dict
  # detects caller from PWD (walks up to find .trinity/passport.json)
  # returns {email_address, display_name, mailbox_path, timestamp_format}

get_user_by_email(email: str) -> Dict | None
get_all_users() -> Dict[str, Dict]
```

---

### users/branch_detection.py

```python
detect_branch_from_pwd() -> Dict | None
find_branch_root(start_path: Path) -> Path | None
get_branch_info_from_registry(branch_path: Path) -> Dict | None
```

---

## Email JSON Schema

```json
{
  "id": "uuid4",
  "from": "@sender",
  "to": "@recipient",
  "subject": "string",
  "body": "string",
  "timestamp": "YYYY-MM-DD HH:MM:SS",
  "status": "new | read | closed",
  "auto_execute": false,
  "reply_to": "@branch | null",
  "dispatched_to": "@branch | null"
}
```

---

## File Layout

```
<branch>/
  .ai_mail.local/
    inbox/
      <timestamp>_<id>.json   ← incoming
    sent/
      <id>.json               ← outgoing copies
    .dispatch_lock            ← PID lock while agent running

.aipass/
  dispatch_log.json           ← history of all dispatches
  autonomous_pause            ← presence of file = kill switch
```

---

## Dependencies

| Depends on | Why |
|-----------|-----|
| `prax` | `system_logger` |
| `drone` (resolver) | `@name` → path resolution for recipient lookup |
| `trigger` | fires "mail_dispatch" event on send with `auto_execute=True` |

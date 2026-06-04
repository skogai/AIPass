# devpulse

**Role:** System orchestrator. The agent you talk to — it coordinates all other agents. Has no unique domain logic; its value is context (identity, memory, relationships) stored in `.trinity/`. Provides two thin utility modules: `feedback` and `watchdog`.

**Source:** `src/aipass/devpulse/apps/`

---

## Architecture Note

`devpulse` is the only agent that a human interacts with directly (by running `claude` in its CWD). All other agents receive work via `ai_mail` dispatch or direct `drone @agent` calls. Devpulse orchestrates by:

1. Reading its inbox (ai_mail)
2. Deciding which agent to route work to
3. Sending tasks via `drone @ai_mail dispatch @agent "subject" "body"`
4. Monitoring via `drone @prax dashboard refresh @devpulse`

No unique commands to expose — devpulse is agent-as-coordinator, not agent-as-service.

---

## Entry Point
**File:** `apps/devpulse.py`

Same auto-discovery pattern as all other agents: globs `modules/*.py`, routes to first `handle_command` that returns True.

---

## Modules

### watchdog
**File:** `apps/modules/watchdog.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "watchdog"
  # sub-commands:
  #   agent <@branch> [--timeout SECONDS]
  #     — blocks until the dispatched agent process exits (polls PID from dispatch lock)
  #   timer <duration>           e.g. "5m", "1h30m", "30s"
  #     — wakes devpulse after duration (SIGALRM / threading.Timer)
  #   timer start <name>         named timer
  #   timer stop <name>
  #   timer list
  #   timer report
  #   schedule <HH:MM | +N>  [command]
  #     — wakes at wall-clock time, optionally runs a drone command
  #   status / list
  #     — reads watchdog_active.json, renders active watches table
  #   cancel <handle> | --all
  #     — SIGTERMs a watch subprocess, removes from registry
```

**Reads/Writes:** `devpulse/watchdog_active.json`

---

### feedback
**File:** `apps/modules/feedback.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "feedback"
  # opens browser or prompts for GitHub issue URL
  # writes feedback entry to devpulse/feedback_log.json
```

---

## Identity Files (`.trinity/`)

Devpulse's trinity files carry the orchestrator's operational context:

| File | Contents (typical) |
|------|-------------------|
| `passport.json` | role: "System Orchestrator", traits: "strategic, collaborative", principles: [...] |
| `local.json` | session history — who was dispatched, what was built, decisions made |
| `observations.json` | patterns noticed across agents, system-level learnings |

---

## Dependencies

| Depends on | Why |
|-----------|-----|
| `prax` | logger |
| `cli` | display |
| `ai_mail` | sends/receives task mail |
| `drone` | routes commands to all other agents |
| `flow` | creates/tracks plans |
| `prax` (dashboard) | status refresh |

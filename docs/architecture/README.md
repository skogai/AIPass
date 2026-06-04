# AIPass Architecture — Interface Reference

Prepared for paradigm rewrite. Focus: function signatures, data types, IO contracts, and inter-agent dependencies.

---

## Rewrite Notes

Two key translation mappings to carry forward:

**Command dispatch → command queue:**  
Every module exposes `handle_command(command: str, args: List[str]) -> bool`. In the rewrite this becomes a **JSONL queue**: each line is a command object, and `handle_command` → `command.invoke()`. The `bool` return becomes a typed `Result`. The agent entry-point's module-discovery loop becomes a **command router** that pops from the queue and dispatches.

Sketch of the queue entry shape:
```jsonl
{"id": "uuid", "command": "create", "args": [".", "Add user auth"], "source": "@devpulse", "ts": "ISO-8601"}
```

**Memory / Trinity → event log + vector DB:**  
`.trinity/` is powered by the [`trinity-pattern`](https://pypi.org/project/trinity-pattern/) pip plugin — three JSON files acting as a structured event log. `@memory` (the agent) is the **boundary service** between that log and ChromaDB: it reads the files, archives old entries as vectors, and exposes search. In the rewrite, model this as:
- **Logger side:** append-only event log (each command invocation → log entry)
- **DB connector side:** background job that flushes old log entries into the vector index

---

## System Diagram

```
CLI User
  │
  ▼
aipass (CLI: `aipass init|doctor|...`)
  │
  ▼
drone (CLI: `drone @branch command [args]`)
  ├── resolver    — @name → path (reads AIPASS_REGISTRY.json)
  ├── router      — path + command → subprocess → CommandResult
  └── modules: commands, config, discovery, git, scan, registry

                         dispatches to →
                                        ╔═══════════╗
                                        ║  any agent ║
                                        ║  CWD + CLI ║
                                        ╚═══════════╝

Cross-agent services (all agents import these):
  prax    — structured logger (→ .prax.log)
  cli     — Rich console helpers (stdout)
  api     — LLM + bridge (in-process contract registry)

Coordination layer:
  ai_mail — message queue (.ai_mail.local/) + dispatch daemon
  trigger — event bus (log file watcher → handler callbacks)
  hooks   — Claude hook dispatcher (PreToolUse / PostToolUse / ...)
  flow    — plan lifecycle (.flow-registry.json)
  memory  — Trinity archival (JSON → ChromaDB)

Agent management:
  spawn   — create/delete/repair agents from templates
  seedgo  — 36 quality-standard checkers
  devpulse — orchestrator agent (no unique code; the "conductor")
```

---

## Universal Module Contract

Every `apps/modules/<name>.py` must export:

```python
def handle_command(command: str, args: List[str]) -> bool
def print_introspection() -> None   # machine-readable self-description
def print_help() -> None            # human-readable help
```

Entry points (`apps/<agent>.py`) auto-discover modules by globbing `modules/*.py` for files that have `handle_command`. First module that returns `True` wins.

---

## Shared Data Structures

### AIPASS_REGISTRY.json
```json
{
  "project": "string",
  "created": "ISO-8601",
  "branches": {
    "<name>": {
      "name": "string",
      "path": "string (relative to repo root)",
      "type": "agent | service | tool",
      "status": "active | inactive | archived",
      "email": "@<name>",
      "citizen_number": "integer"
    }
  }
}
```

### .trinity/ (per-agent, Trinity plugin)
| File | Purpose | Key fields |
|------|---------|------------|
| `passport.json` | Identity — who this agent is | `name`, `role`, `traits`, `purpose`, `principles` |
| `local.json` | Session log — command history | `sessions: [{date, summary, work_done, key_decisions}]` |
| `observations.json` | Learnings accumulation | `observations: [{date, insight, category}]` |

### .ai_mail.local/\<branch\>/inbox/\<id\>.json
```json
{
  "id": "uuid",
  "from": "@sender",
  "to": "@recipient",
  "subject": "string",
  "body": "string",
  "timestamp": "ISO-8601",
  "status": "new | read | closed",
  "auto_execute": "boolean",
  "reply_to": "@branch | null"
}
```

### CommandResult (drone/handlers/executor.py)
```python
@dataclass
class CommandResult:
    stdout: str
    stderr: str
    exit_code: int   # 0 = success
    branch: str
    command: str
```

---

## Agent Index

| Agent | CLI Entry | Role |
|-------|-----------|------|
| [drone](drone.md) | `drone @branch cmd` | Command router — resolves names, dispatches to subprocesses |
| [aipass](aipass.md) | `aipass` | Concierge — project init, doctor, profile |
| [ai_mail](ai_mail.md) | `drone @ai_mail ...` | Message queue + dispatch daemon |
| [flow](flow.md) | `drone @flow ...` | Plan lifecycle management |
| [memory](memory.md) | `drone @memory ...` | Trinity archival + vector search |
| [spawn](spawn.md) | `drone @spawn ...` | Agent creation/deletion from templates |
| [trigger](trigger.md) | `drone @trigger ...` | Event bus (log → handlers) |
| [hooks](hooks.md) | `drone @hooks ...` | Claude hook event dispatcher |
| [prax](prax.md) | `drone @prax ...` | Structured logger + real-time monitor |
| [seedgo](seedgo.md) | `drone @seedgo ...` | 36-point quality audit engine |
| [api](api.md) | `drone @api ...` | LLM bridge + provider clients |
| [cli](cli.md) | (library only) | Rich console output helpers |
| [devpulse](devpulse.md) | (orchestrator agent) | System orchestrator — no unique commands |

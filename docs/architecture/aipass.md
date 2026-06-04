# aipass (agent)

**Role:** Concierge. Handles project scaffolding (`aipass init`), system health checks (`aipass doctor`), and agent identity/profile management. This is the `aipass` CLI command — separate from the package name.

**CLI entry point:** `aipass.aipass.apps.aipass:main`  
**Source:** `src/aipass/aipass/apps/`

---

## Commands

### init_flow
**File:** `apps/modules/init_flow.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "init"
  # args: ["run"] [--non-interactive] [--name <name>] [--cli <claude|codex>]
  #        [--dry-run]

# 12-stage guided setup, resumable via .aipass/init_progress.json:
#   Stage 1:  Detect system (OS, shell, RAM, CPU, tmux, git, Python)
#   Stage 2:  Confirm project directory
#   Stage 3:  Ask project name
#   Stage 4:  Ask CLI (claude / codex)
#   Stage 5:  Create .aipass/ scaffold
#   Stage 6:  Write CLAUDE.md / AGENTS.md
#   Stage 7:  Detect existing git repo or init new
#   Stage 8:  drone @spawn create (creates first agent)
#   Stage 9:  Wire hooks for chosen CLI
#   Stage 10: Create AIPASS_REGISTRY.json
#   Stage 11: Launch terminal handoff (tmux new-window or Windows Terminal)
#   Stage 12: Write init_progress.json = complete
```

**Subprocess:** `drone @spawn create`, tmux, wt (Windows Terminal)  
**Reads:** `.aipass/init_progress.json`  
**Writes:** `.aipass/`, `CLAUDE.md`, `AGENTS.md`, `AIPASS_REGISTRY.json`, `.aipass/init_progress.json`

---

### doctor
**File:** `apps/modules/doctor.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "doctor"
  # args: ["--json"] ["--fix"] ["--wire <provider>"]

# 15+ health checks organized by category:
#   System:   Python version, git, shell, OS
#   Project:  .aipass/ present, AIPASS_REGISTRY.json readable, pyproject.toml
#   Agents:   each registered branch has passport.json, trinity files
#   Hooks:    .claude/hooks/ wired, hook files executable
#   CLI:      claude/codex binaries found on PATH
#   Registry: branch count, stale entries, path validity

class HealthCheck(NamedTuple):
    name: str
    status: str   # "pass" | "warn" | "fail"
    detail: str
    fix: str | None
```

**Subprocess:** `claude --version`, `codex --version`, `git --version`  
**Reads:** `AIPASS_REGISTRY.json`, `pyproject.toml`, branch directories

---

### doctor_fix
**File:** `apps/modules/doctor_fix.py`

```python
print_json_report(checks: List[HealthCheck]) -> None
  # JSON output for machine consumption

print_remediation_report(checks: List[HealthCheck]) -> None
  # human-readable fix instructions for failed checks
```

---

### doctor_wire
**File:** `apps/modules/doctor_wire.py`

```python
prompt_auto_wire(provider: str) -> bool
  # interactive: asks user before wiring

_auto_wire_provider(provider: str) -> bool
  # writes/updates .claude/hooks/ or .codex/hooks/
  # links drone commands to Claude hook event types
```

**Writes:** `.claude/hooks/`, `.codex/hooks/`

---

### profile
**File:** `apps/modules/profile.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "show [@branch]", "edit [@branch]", "list"
  # reads/updates passport.json for an agent
```

**Reads/Writes:** `<branch>/.trinity/passport.json`

---

### handoff
**File:** `apps/modules/handoff.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "handoff"
  # args: [@branch] [--model opus|sonnet|haiku]
  # opens a new terminal pane at the branch CWD
  # tmux (Linux/macOS) or wt (Windows)
```

**Subprocess:** `tmux new-window -c <path>` or `wt -d <path>`

---

### help_chat
**File:** `apps/modules/help_chat.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "help"
  # interactive help mode — pipes user questions to LLM via api.openrouter_client
```

---

## System Detection (`handlers/system_detect/system_detector.py`)

```python
detect_os() -> Dict[str, str]        # {name, version, platform}
detect_python() -> Dict[str, str]    # {version, executable, venv}
detect_shell() -> str                # "bash" | "zsh" | "fish" | "unknown"
detect_git() -> Dict[str, Any]       # {installed, version, repo_root}
detect_cpu() -> Dict[str, Any]       # {count, model}
detect_ram() -> Dict[str, Any]       # {total_gb, available_gb}
detect_tmux() -> bool
detect_wt() -> bool                  # Windows Terminal
detect_docker() -> bool
detect_install_method() -> str       # "pip" | "uv" | "dev"
```

---

## Dependencies

| Depends on | Why |
|-----------|-----|
| `prax` | logger |
| `cli` | display output |
| `drone` (spawn) | `aipass init run` calls `drone @spawn create` |
| `api` (openrouter) | `help` command uses LLM |

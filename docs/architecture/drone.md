# drone

**Role:** Command router. Translates `drone @branch command [args]` into a subprocess invocation at the branch's directory, or routes to an internal module.

**CLI entry point:** `aipass.drone.cli:main`  
**Source:** `src/aipass/drone/apps/`

---

## Commands

```
drone @<branch> <command> [args...]   — route to registered branch
drone <module> <command> [args...]    — route to internal module
drone --help                          — list all branches + modules
```

Special flags on branch commands:
- `--model <sonnet|opus|haiku>` — override model when waking an agent
- `--interactive` — bypass output capture (for `monitor`, `audit`, `watchdog`)

---

## Modules

### resolver
**File:** `apps/modules/resolver.py`

```python
handle_command(command: str | None, args: List[str] | None) -> bool
  # commands: "resolve", "exists", "info", "list"

normalize_branch_name(symbolic_name: str) -> str
  # strips "@" prefix

normalize_branch_arg(target: str) -> str
  # strips "@", lowercases

resolve_branch(symbolic_name: str) -> str
  # "@name" → absolute path string
  # raises BranchNotFoundError if not in registry
  # raises RegistryNotFoundError if registry file missing
  # security: blocks path-traversal (branch path must stay within project root)

branch_exists(symbolic_name: str) -> bool

get_branch_info(symbolic_name: str) -> Dict[str, Any]
  # returns branch entry from registry
  # raises BranchNotFoundError

list_branches(branch_type: str | None = None, status: str = "active") -> List[str]
  # returns ["@name1", "@name2", ...]
```

**Reads:** `AIPASS_REGISTRY.json` (via registry_handler)

---

### router
**File:** `apps/modules/router.py`

```python
handle_command(command: str | None, args: List[str] | None) -> bool
  # commands: "route", "route_all"

route_command(
    target: str,
    command: str | None = None,
    args: List[str] | None = None,
    timeout: int = 30,
    interactive: bool = False,
) -> CommandResult
  # resolves @target → path, calls execute_branch_command
  # command=None → introspection (branch runs with no args)

route_all(
    command: str,
    args: List[str] | None = None,
    timeout: int = 30,
) -> Dict[str, CommandResult]
  # broadcasts to all active branches; never throws on individual failures
```

**Subprocess:** invokes `python -m aipass.<branch>.apps.<branch>` (or detected entry point) via `executor.execute_command`

---

### registry
**File:** `apps/modules/registry.py`

```python
handle_command(command: str | None, args: List[str] | None) -> bool
  # commands: "load", "branches [type]", "lookup <name>"

# Re-exports from registry_handler:
load_registry() -> Dict[str, Any]
get_all_branches(branch_type: str | None = None, status: str = "active") -> List[Dict]
get_branch_by_name(name: str) -> Dict[str, Any] | None
```

**Reads:** `AIPASS_REGISTRY.json`

---

### commands
**File:** `apps/modules/commands.py`

```python
handle_command(command: str | None, args: list[str] | None) -> bool
  # commands: "add", "remove <name>", "list", "lookup <name>", "match <args>"

add(name: str, target: str, description: str = "") -> bool
remove(name: str) -> bool
list_all() -> list[dict[str, Any]]
lookup(name: str) -> dict[str, Any] | None
match(args: list[str]) -> tuple[dict[str, Any], list[str]] | None
  # finds a registered shortcut matching the args prefix
```

**Reads/Writes:** `.aipass/drone_commands.json`

---

### discovery
**File:** `apps/modules/discovery.py`

```python
handle_command(command: str | None, args: List[str] | None) -> bool
  # commands: "help [target]", "introspect", "modules <target>"

discover_modules(target: str) -> List[str]
  # returns module names for a given branch

get_help(target: str, command: str | None = None) -> HelpResult
get_system_help() -> Dict[str, HelpResult]
```

**Subprocess:** calls `drone @<target>` to retrieve introspection output

---

### scan
**File:** `apps/modules/scan.py`

```python
handle_command(command: str | None, args: list[str] | None) -> bool
  # commands: "scan [target]"

scan(target: str) -> list[dict] | None
  # returns list of {name, path, type, status} for all agents or a specific one
```

---

### config
**File:** `apps/modules/config.py`

```python
handle_command(command: str | None, args: List[str] | None) -> bool
  # commands: "get <key>", "set <key> <value>", "list"
```

**Reads/Writes:** `.aipass/config.json`

---

### git_module
**File:** `apps/modules/git_module.py`

```python
handle_command(command: str | None, args: list[str] | None) -> dict
  # commands: "branches", "pr", "dev-pr", "delete-branch", "close-pr",
  #           "merge", "smart-sync", "fix", "status", "diff", "log",
  #           "commit", "checkout", "sync", "lock", "unlock"
  # returns: {"success": bool, "output": str, "error": str | None}
```

**Subprocess:** shells out to `git` and `gh` CLI

---

## Key Handler Types

**`CommandResult`** (`apps/handlers/executor.py`)
```python
@dataclass
class CommandResult:
    stdout: str
    stderr: str
    exit_code: int
    branch: str
    command: str
```

**Exceptions** (`apps/handlers/exceptions.py`)
- `BranchNotFoundError(ValueError)`
- `CommandExecutionError(RuntimeError)`
- `RegistryError(RuntimeError)`

---

## Dependencies

| Depends on | Why |
|-----------|-----|
| `prax` | `system_logger` for all log calls |
| `cli` | `console`, `err_console` for output |
| `AIPASS_REGISTRY.json` | branch name → path resolution |

# flow

**Role:** Plan lifecycle management. Creates, closes, lists, and restores work plans. Maintains a central registry and supports 6 template types.

**Source:** `src/aipass/flow/apps/`

---

## Commands

### create_plan
**File:** `apps/modules/create_plan.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "create"
  # args: [target_path, "Title"] [--template <type>] [--dry-run]

create_plan(
    target_path: str,
    title: str,
    template_type: str = "default",
    dry_run: bool = False,
) -> bool
  # 1. resolves template (template_manager)
  # 2. writes plan file to target_path
  # 3. registers in .flow-registry.json
  # returns True on success
```

**Writes:** `<target_path>/<slug>.md`, `.flow-registry.json`

---

### close_plan
**File:** `apps/modules/close_plan.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "close"
  # args: [plan_id_or_path] [--confirm] [--dry-run]
  # also: "close-all" [--confirm] [--dry-run]

close_plan(
    plan_key: str,
    confirm: bool = False,
    dry_run: bool = False,
) -> bool
  # marks plan as closed in registry
  # moves plan file to .archive/
  # triggers post_close_runner

close_all_plans(confirm: bool = False, dry_run: bool = False) -> bool
```

**Reads:** `.flow-registry.json`  
**Writes:** `.flow-registry.json`, moves `.md` → `.archive/<name>.md`

---

### list_plans
**File:** `apps/modules/list_plans.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "list"
  # args: ["open" | "closed" | "all"]

list_plans(filter_type: str = "open") -> bool
  # reads .flow-registry.json, renders table via cli
```

**Reads:** `.flow-registry.json`

---

### restore_plan
**File:** `apps/modules/restore_plan.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "restore"
  # args: [plan_id | plan_number]

recover_plan_from_backup(plan_key: str) -> tuple[bool, str]
  # copies plan back from .archive/ to original path
  # re-registers in .flow-registry.json

restore_plan(plan_num: str | None) -> bool
  # interactive if plan_num is None
```

**Reads:** `.archive/<name>.md`, `.flow-registry.json`  
**Writes:** `<original_path>/<name>.md`, `.flow-registry.json`

---

### template_manager
**File:** `apps/modules/template_manager.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "list", "register <type> <path>", "unregister <type>",
  #           "info <type>", "suggest <dir_name>"

_suggest_prefix(dir_name: str) -> str
  # heuristic: maps dir name to template type prefix
```

**Reads/Writes:** `.flow-template-registry.json`  
Template types: `default`, `feature`, `bug`, `refactor`, `research`, `ops`

---

### aggregate_central
**File:** `apps/modules/aggregate_central.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "aggregate"

aggregate_central(heal: bool = True) -> bool
  # walks all branches in registry
  # collects their local .flow-registry.json entries
  # merges into root-level FLOW_CENTRAL.json
  # heal=True: removes entries for missing plan files
```

**Reads:** each branch's `.flow-registry.json`  
**Writes:** `<repo_root>/FLOW_CENTRAL.json`

---

### registry_monitor
**File:** `apps/modules/registry_monitor.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "status", "scan"

scan_plan_files() -> Dict[str, Any]
  # walks repo for *.md plan files, checks registry consistency

get_status() -> Dict[str, Any]
  # returns {total_plans, open, closed, orphaned}
```

---

### post_close_runner
**File:** `apps/modules/post_close_runner.py`

```python
handle_command(command: str, args: list) -> bool
  # command: "run"
  # runs post-close hooks: memory verification, dashboard refresh
```

---

## Plan Registry Schema (`.flow-registry.json`)

```json
{
  "plans": {
    "<plan_key>": {
      "key": "string",
      "title": "string",
      "path": "string (relative)",
      "template_type": "default | feature | bug | refactor | research | ops",
      "status": "open | closed",
      "created": "ISO-8601",
      "closed_at": "ISO-8601 | null",
      "archive_path": "string | null"
    }
  }
}
```

---

## Handlers (internal)

Key handler modules under `apps/handlers/plan/`:

| Handler | Key functions |
|---------|--------------|
| `create_ops.py` | `create_plan_file(path, title, template) -> bool` |
| `close_ops.py` | `close_plan_file(plan_key, registry) -> bool` |
| `list_ops.py` | `get_open_plans(registry) -> List[Dict]` |
| `resolve_location.py` | `resolve_plan_path(target_path, title) -> Path` |
| `build_registry_entry.py` | `build_entry(path, title, template_type) -> Dict` |
| `aggregate_ops.py` | `merge_registries(registries: List[Dict]) -> Dict` |
| `auto_cleanup.py` | `cleanup_orphaned_entries(registry) -> int` |
| `append_closed_plan.py` | `append_to_closed(plan, registry) -> bool` |
| `get_closed_plans.py` | `get_closed_plans(registry) -> List[Dict]` |

---

## Dependencies

| Depends on | Why |
|-----------|-----|
| `prax` | logger |
| `memory` (verify) | vectorization check after close |
| `cli` | display output |

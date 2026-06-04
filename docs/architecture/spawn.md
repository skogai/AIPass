# spawn

**Role:** Agent factory. Creates new AIPass agents from templates (filling `{{PLACEHOLDER}}` patterns), registers them in `AIPASS_REGISTRY.json`, and issues a passport. Also handles delete, repair, update, and registry sync.

**Source:** `src/aipass/spawn/apps/`

---

## Commands

### core (create)
**File:** `apps/modules/core.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "create"
  # args: [target_path]
  #       [--role "string"]
  #       [--traits "string"]
  #       [--purpose "string"]
  #       [--template <class_name | path>]
  #       [--registry <path>]

_spawn_agent(
    target_path: str,
    role: str = "",
    traits: str = "",
    purpose: str = "",
    profile: str | None = None,
    template_dir: Path | None = None,
    registry_path: str | None = None,
    citizen_class: str = "builder",
) -> Dict[str, Any]
  # returns: {
  #   "success": bool,
  #   "branch_path": str,
  #   "branch_name": str,
  #   "citizen_number": int,
  #   "error": str | None
  # }
```

Full workflow (7 steps):
1. Validate `target_path` does not exist
2. `copy_template(template_dir, target_path)`
3. `rename_placeholder_paths(target_path, branch_name)`
4. `build_replacements_dict(branch_name, role, traits, purpose, ...)` → fill all `{{PLACEHOLDER}}`
5. `regenerate_template_registry(target_path)`
6. `add_to_registry(registry_path, branch_meta)` → `AIPASS_REGISTRY.json`
7. `validate_no_placeholders(target_path)` — raises if any `{{...}}` remain

**Writes:** entire agent directory at `target_path`, `AIPASS_REGISTRY.json`

---

### delete
**File:** `apps/modules/delete.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "delete"
  # args: [@branch_name] [--confirm]
  # 1. removes from AIPASS_REGISTRY.json
  # 2. archives directory to .archive/<branch_name>_<timestamp>/
```

**Reads/Writes:** `AIPASS_REGISTRY.json`  
**Moves:** `src/<agent>/` → `.archive/<agent>_<ts>/`

---

### update
**File:** `apps/modules/update.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "update"
  # args: [@branch] [--role ...] [--traits ...] [--purpose ...]
  # updates passport.json fields; does NOT recreate directory
```

**Reads/Writes:** `<branch>/.trinity/passport.json`

---

### repair
**File:** `apps/modules/repair.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "repair"
  # args: [@branch] [--dry-run]
  # checks for missing files (passport, trinity files, hooks)
  # recreates from template if absent
```

---

### passport
**File:** `apps/modules/passport.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "show [@branch]", "validate [@branch]"
  # reads and validates passport.json against expected schema
```

---

### sync_registry
**File:** `apps/modules/sync_registry.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "sync-registry" [--dry-run]
  # scans all branch dirs, reconciles with AIPASS_REGISTRY.json
  # adds missing entries, marks deleted dirs as inactive
```

---

### regenerate_registry
**File:** `apps/modules/regenerate_registry.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "regen-registry" [--dry-run]
  # rebuilds AIPASS_REGISTRY.json from scratch by scanning dirs
```

---

### sync_templates
**File:** `apps/modules/sync_templates.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "sync-templates" [--dry-run]
  # pushes template file updates to all registered branches
```

---

## Key Handler Modules

| Handler | Key functions |
|---------|--------------|
| `metadata.py` | `get_branch_name(target_path) -> str`, `normalize_branch_name(name) -> str`, `detect_profile() -> str` |
| `placeholders.py` | `build_replacements_dict(name, role, ...) -> Dict[str,str]`, `validate_no_placeholders(path) -> bool` |
| `file_ops.py` | `copy_template(src, dst)`, `rename_placeholder_paths(path, name)`, `regenerate_template_registry(path)`, `ensure_directory(path)` |
| `meta_ops.py` | `load_template_registry(path) -> Dict`, `generate_branch_meta(name, role, ...) -> Dict`, `save_branch_meta(path, meta)` |
| `registry.py` | `find_registry(start: Path) -> Path`, `add_to_registry(path, meta) -> bool`, `get_next_citizen_number(registry) -> int`, `fix_passport_registry_id(branch_path, registry)`, `ensure_project_has_owner(registry)` |
| `class_registry.py` | `validate_class(name) -> bool`, `get_default_class() -> str`, `get_available_classes() -> List[str]`, `get_template_dir(class_name) -> Path` |

---

## Passport Schema (`.trinity/passport.json`)

```json
{
  "name": "string",
  "email": "@name",
  "role": "string",
  "traits": "string",
  "purpose": "string",
  "principles": ["string"],
  "citizen_number": 42,
  "registry_id": "uuid4",
  "created": "ISO-8601"
}
```

---

## Placeholder Patterns

Template files use `{{BRANCH_NAME}}`, `{{ROLE}}`, `{{TRAITS}}`, `{{PURPOSE}}`, `{{EMAIL}}`, `{{CITIZEN_NUMBER}}` etc. `validate_no_placeholders` scans all files in the created directory and raises if any `{{...}}` remain after substitution.

---

## Dependencies

| Depends on | Why |
|-----------|-----|
| `prax` | logger |
| `cli` | display output |
| `drone` (registry) | read AIPASS_REGISTRY for citizen number allocation |

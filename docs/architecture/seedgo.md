# seedgo

**Role:** Quality enforcement engine. Runs 36 automated checks across all agents covering code standards, test coverage, inbox hygiene, hook integrity, README accuracy, and permissions. Used in CI and manually.

**Source:** `src/aipass/seedgo/apps/`

---

## Commands

### standards_audit
**File:** `apps/modules/standards_audit.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "audit [@branch]", "audit-all", "list-standards"
  # runs all 36 standard checks for a branch or all branches
  # outputs: pass/fail table + summary count

# 36 standards cover:
#   - File structure (passport.json present, trinity files, CLAUDE.md, README.md)
#   - Code quality (no bare except, type hints on public fns, max line length)
#   - Test presence (each module has test file)
#   - Seedgo coverage (checklist entries match implemented checks)
#   - Hook wiring (hooks present, enabled, correct paths)
#   - Inbox hygiene (no stale unread mail)
#   - Memory (local.json not exceeding line limit)
```

---

### diagnostics_audit
**File:** `apps/modules/diagnostics_audit.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "run [@branch]", "summary"
  # deeper diagnostics: import errors, circular deps, subprocess availability
```

---

### checklist
**File:** `apps/modules/checklist.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "show", "update [@branch]", "status"
  # manages SEEDGO_CHECKLIST.json per branch
  # links each standard to its pass/fail state
```

**Reads/Writes:** `<branch>/SEEDGO_CHECKLIST.json`

---

### hook_bridge
**File:** `apps/modules/hook_bridge.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # command: "hook-check"
  # called by hooks engine PostToolUse when Write/Edit tool fires
  # runs subset of standards on just-edited file
```

---

### hooks, hooks_ext, hooks_probe
**Files:** `apps/modules/hooks.py`, `hooks_ext.py`, `hooks_probe.py`

```python
# hooks.py: standard hook runners
handle_command(command: str, args: List[str]) -> bool
  # commands: "run <hook_name>", "list"

# hooks_ext.py: extended / custom checks
# hooks_probe.py: lightweight probe checks (fast subset for CI)
```

---

### inbox_audit
**File:** `apps/modules/inbox_audit.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "audit [@branch]", "audit-all"
  # counts unread mail per branch
  # flags branches with >N unread as failing standard
```

---

### permissions
**File:** `apps/modules/permissions.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "check [@branch]", "fix [@branch]"
  # verifies .claude/settings.json allowlist is correct
  # fix: updates allowlist to match expected entries
```

---

### proof_query, seedgo_proof
**Files:** `apps/modules/proof_query.py`, `seedgo_proof.py`

```python
# proof_query: queries which branches pass/fail a specific standard
proof_query(standard_id: int, branches: List[str] | None = None) -> Dict[str, bool]

# seedgo_proof: generates proof artifact (SEEDGO_PROOF.json)
# used in CI to demonstrate all standards pass
```

---

### readme_update
**File:** `apps/modules/readme_update.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "check [@branch]", "update [@branch]"
  # checks README.md has required sections
  # update: rewrites stale sections from passport + observations
```

---

### standards_query, test_map
**Files:** `apps/modules/standards_query.py`, `test_map.py`

```python
# standards_query: query standard definitions
get_standard(id: int) -> Dict | None
list_standards() -> List[Dict]

# test_map: map modules → their test files
build_test_map(branch_path: Path) -> Dict[str, str | None]
  # {module_path: test_file_path | None}
```

---

## Standard Definition Schema

```json
{
  "id": 1,
  "name": "string",
  "description": "string",
  "category": "structure | code | tests | hooks | inbox | memory | permissions",
  "severity": "error | warning",
  "auto_fix": false
}
```

---

## Dependencies

| Depends on | Why |
|-----------|-----|
| `prax` | logger |
| `drone` (router) | route_all to broadcast audit to all branches |
| `cli` | Rich table output |

# cli

**Role:** Shared UI library. Provides pre-configured Rich console instances, formatted output helpers, and table/panel templates used by every other agent. No CLI commands of its own — import-only.

**Source:** `src/aipass/cli/apps/`

---

## Public API (`apps/modules/__init__.py`)

```python
from aipass.cli.apps.modules import (
    console,        # rich.Console — main stdout
    err_console,    # rich.Console — stderr
    header,         # (text: str) -> None
    error,          # (text: str) -> None
    warning,        # (text: str) -> None
    success,        # (text: str) -> None
    info,           # (text: str) -> None
)
```

All agents import from here. No other import path is expected.

---

## Modules

### display
**File:** `apps/modules/display.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "demo", "colors", "styles"
  # demonstrates available output styles

# Exports:
console: rich.Console        # width=120, highlight=False
err_console: rich.Console    # stderr=True

def header(text: str) -> None
  # prints: [bold cyan]── text ──[/]

def error(text: str) -> None
  # prints: [bold red]✗ text[/]

def warning(text: str) -> None
  # prints: [bold yellow]⚠ text[/]

def success(text: str) -> None
  # prints: [bold green]✓ text[/]

def info(text: str) -> None
  # prints: [dim]ℹ text[/]
```

---

### templates
**File:** `apps/modules/templates.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "table", "panel", "status"

def make_table(
    columns: List[str],
    rows: List[List[str]],
    title: str | None = None,
    style: str = "cyan",
) -> rich.table.Table

def make_panel(
    content: str,
    title: str | None = None,
    style: str = "blue",
) -> rich.panel.Panel

def status_row(label: str, value: str, ok: bool) -> str
  # colored label=value string for status displays
```

---

## No File IO

`cli` reads and writes nothing. It has no file dependencies, no subprocess calls, no network calls.

---

## Dependencies

| Depends on | Why |
|-----------|-----|
| `rich` | all output formatting |
| `prax` | logger (standard import) |

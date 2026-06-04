# memory

**Role:** Trinity archival — when `.trinity/` JSON files exceed line limits, rolls them over into ChromaDB for long-term semantic search. Also provides fragment-level vector storage and symbolic dimension extraction from conversation history.

**Rewrite note:** Trinity is a [`trinity-pattern`](https://pypi.org/project/trinity-pattern/) pip plugin — three JSON files as a structured event log. `@memory` is the boundary between that log and the vector store. In the rewrite, split this into two concerns:
- **Logger side:** every command invocation appends an entry to the structured log (replaces trinity's `local.json` / `observations.json`)
- **DB connector side:** a background flush job reads the log, generates embeddings, writes to the vector index — exactly what `rollover` does today

**Source:** `src/aipass/memory/apps/`

---

## Commands

### rollover
**File:** `apps/modules/rollover.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "run", "status", "check", "sync-lines"

run_rollover() -> bool
  # full rollover cycle:
  # 1. detector.check_triggers() → which files exceed limit
  # 2. extractor.extract_oldest(file) → List[Dict] entries
  # 3. embedder.generate(entries) → List[float] vectors
  # 4. chroma.store(entries, vectors)
  # 5. writes updated file back (oldest entries removed)

sync_line_counts() -> None
  # updates metadata: {file_path: line_count} in .memory_meta.json

show_status() -> None
  # reads .memory_meta.json, renders table

check_triggers() -> None
  # dry-run: lists files that would roll over
```

**Reads:** `.trinity/local.json`, `.trinity/observations.json` (of any branch)  
**Writes:** same files (truncated), ChromaDB at `.chroma/` (optional dep)  
**Writes:** `.memory_meta.json`

---

### search
**File:** `apps/modules/search.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "query <text>", "semantic <text>", "fragment <id>"

show_search_results(
    query: str,
    n_results: int = 5,
    search_type: str = "semantic",  # "semantic" | "keyword" | "dimension"
) -> bool
  # delegates to symbolic.search_fragments_by_vector() or _by_dimensions()
```

---

### symbolic
**File:** `apps/modules/symbolic.py`  
The densest module in the system — full vector fragment lifecycle.

```python
# === Extraction ===

analyze_conversation(chat_history: List[Dict[str, Any]]) -> Dict[str, Any]
  # runs all extractors below, returns composite

extract_technical_flow(chat_history: List[Dict]) -> Dict
extract_emotional_journey(chat_history: List[Dict]) -> Dict
extract_collaboration_patterns(chat_history: List[Dict]) -> Dict
extract_key_learnings(chat_history: List[Dict]) -> Dict
extract_context_triggers(chat_history: List[Dict]) -> Dict
extract_symbolic_dimensions(chat_history: List[Dict]) -> Dict

# === Fragment creation ===

create_fragment(
    content: str,
    source_branch: str,
    dimensions: Dict[str, Any] | None = None,
    metadata: Dict[str, Any] | None = None,
) -> Dict[str, Any]
  # returns fragment dict ready for storage

flatten_dimensions(fragment: Dict) -> Dict
  # collapses nested dimension keys for ChromaDB metadata

# === Storage ===

store_fragment(fragment: Dict, db_path: Path | None = None) -> Dict[str, Any]
  # stores in ChromaDB; returns {success, id, error?}

store_fragments_batch(fragments: List[Dict], db_path: Path | None = None) -> Dict[str, Any]
  # returns {stored: int, failed: int, errors: List[str]}

store_llm_fragment(
    content: str,
    source_branch: str,
    dimensions: Dict | None = None,
    metadata: Dict | None = None,
    db_path: Path | None = None,
) -> Dict[str, Any]

store_llm_fragments_batch(fragments: List[Dict], db_path: Path | None = None) -> Dict

deduplicate_fragment(new_fragment: Dict, existing_fragments: List[Dict]) -> Dict
  # merges or skips if content-identical

# === Retrieval ===

retrieve_fragments(
    query: str | None = None,
    n_results: int = 10,
    db_path: Path | None = None,
) -> Dict[str, Any]
  # returns {fragments: List[Dict], total: int}

search_fragments_by_vector(query: str, n_results: int = 5, db_path: Path | None = None) -> Dict
search_fragments_by_dimensions(
    dimensions: Dict[str, Any],
    n_results: int = 10,
    db_path: Path | None = None,
) -> Dict
search_fragments_by_triggers(
    triggers: List[str],
    n_results: int = 5,
    db_path: Path | None = None,
) -> Dict

# === Hook integration ===

process_hook(hook_data: Dict[str, Any]) -> Dict[str, Any]
  # called by hooks engine on PreToolUse/PostToolUse
  # if context is rich enough, extracts + stores fragment
  # returns {exit_code: 0, stdout: str}

load_hook_config(config_path: Path | None = None) -> Dict[str, Any]
reset_hook_session() -> None
get_hook_session_state() -> Dict[str, Any]

# === Context helpers ===

extract_conversation_context(messages: List[Dict], max_messages: int = 5) -> Dict
find_relevant_fragments(context: Dict, n_results: int = 3, db_path: Path | None = None) -> Dict
format_fragment_recall(fragment: Dict) -> str
should_surface_fragment(fragment: Dict | None, config: Dict | None) -> tuple[bool, str]

# === Bootstrap ===

bootstrap_from_jsonl(max_sessions: int = 8) -> None
  # parses Claude JSONL session files from ~/.claude/projects/
  # extracts fragments from past conversations, stores in ChromaDB
```

**Reads:** ChromaDB at `.chroma/` (or `db_path`), JSONL session files  
**Writes:** ChromaDB

---

### templates
**File:** `apps/modules/templates.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "push [branch]", "push-spawn [branch]", "diff [branch]", "status"
  # manages memory template files (trinity schema) across branches
```

---

### verify
**File:** `apps/modules/verify.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "check <plan_label>", "discover"

is_plan_vectorized(plan_label: str) -> dict
  # returns {vectorized: bool, found_in: str | None}

_discover_handlers() -> dict[str, list[str]]
  # auto-discovers ChromaDB handlers
```

---

## Fragment Schema

```json
{
  "id": "uuid4",
  "content": "string",
  "source_branch": "string",
  "created_at": "ISO-8601",
  "dimensions": {
    "technical": {...},
    "emotional": {...},
    "collaboration": {...},
    "learnings": [...],
    "triggers": [...]
  },
  "metadata": {
    "session_id": "string | null",
    "tool_call": "string | null"
  }
}
```

---

## Key Handler Modules

| Path | Purpose |
|------|---------|
| `handlers/monitor/detector.py` | `check_triggers() -> List[Path]` — which trinity files need rollover |
| `handlers/rollover/orchestrator.py` | `execute_rollover(file_path) -> bool`, `sync_line_counts() -> None` |
| `handlers/rollover/extractor.py` | `extract_oldest(file_path, n) -> List[Dict]` |
| `handlers/vector/embedder.py` | `generate(texts: List[str]) -> List[List[float]]` (fastembed) |
| `handlers/storage/chroma.py` | `store(entries, vectors, collection_name) -> bool` |
| `handlers/json/memory_files.py` | `read_memory_file(path) -> Dict`, `write_memory_file(path, data) -> Dict` |
| `handlers/schema/normalize.py` | `normalize_memory_file(path, dry_run) -> Dict` |

---

## Dependencies

| Depends on | Why |
|-----------|-----|
| `prax` | logger |
| `cli` | display output |
| ChromaDB (optional) | vector storage (`memory` extra: `pip install aipass[memory]`) |
| fastembed (optional) | embedding generation |

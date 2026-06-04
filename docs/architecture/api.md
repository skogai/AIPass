# api

**Role:** LLM access layer. Provides multi-provider LLM calls (primarily OpenRouter), a generic contract bridge for in-process driver registration, Google API auth, API key management, and usage tracking.

**Source:** `src/aipass/api/apps/`

---

## Commands

### bridge (contract registry)
**File:** `apps/modules/bridge.py`

```python
# In-process registry: str → Callable
# No file IO, no subprocess — pure in-memory map.

register(contract_name: str, driver_fn: Callable) -> None
  # e.g. register("memory", some_fn)

resolve(contract_name: str) -> Callable | None
  # returns driver_fn or None if not registered

list_contracts() -> list[str]
  # sorted list of registered names

clear() -> None
  # test teardown only

handle_command(command: str, args: list) -> bool
  # always returns False — bridge is a library, not a command handler
```

---

### openrouter_client
**File:** `apps/modules/openrouter_client.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "call <prompt>", "models", "status"

get_response(
    prompt: str,
    caller: str | None = None,
    model: str | None = None,
    **kwargs,
) -> Dict[str, Any]
  # POSTs to https://openrouter.ai/api/v1/chat/completions
  # reads OPENROUTER_API_KEY from env
  # returns {content: str, model: str, usage: {prompt_tokens, completion_tokens}}
  # kwargs: temperature, max_tokens, system_prompt, stream

extract_response(response: Dict) -> str
  # returns response["choices"][0]["message"]["content"]

list_models(args: List[str] | None = None) -> None
  # GET /api/v1/models, renders table

check_status() -> None
  # GET /api/v1/auth/key, validates key

make_call(args: List[str]) -> None
  # CLI wrapper: args = [prompt, --model m, --temp t, --max-tokens n]
```

**Network:** `POST https://openrouter.ai/api/v1/chat/completions`  
**Env:** `OPENROUTER_API_KEY`

---

### api_key
**File:** `apps/modules/api_key.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "get [provider]", "validate [provider]", "list", "init", "fetch [provider]"

fetch_api_key(provider: str = "openrouter") -> str | None
  # reads from env: OPENROUTER_API_KEY, OPENAI_API_KEY, GOOGLE_API_KEY, etc.

fetch_validate_key(key: str, provider: str = "openrouter") -> bool
  # makes test API call

get_validation_rules(provider: str) -> dict
  # returns {min_length, prefix, test_endpoint} for each provider
```

**Reads:** environment variables, `.env` file

---

### google_client
**File:** `apps/modules/google_client.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "validate", "reauth"

get_drive_service(thread_safe: bool = False) -> object
  # returns googleapiclient Resource for Drive v3

get_google_service(
    service_name: str,
    version: str,
    scopes: list | None = None,
    thread_safe: bool = False,
) -> object
  # generic service builder

authenticate_google(scopes: list | None = None) -> bool
validate_google(scopes: list | None = None) -> bool
reauth_google(scopes: list | None = None) -> bool

api_call_with_retry(*args, **kwargs)
  # wraps API call with exponential backoff on 429/503

is_ssl_error(error) -> bool
```

**Reads:** `~/.credentials/google_oauth.json`  
**Network:** Google OAuth2 + API endpoints

---

### registry
**File:** `apps/modules/registry.py`

```python
load_drivers(integrations_dir: Path | None = None) -> int
  # auto-discovers Python files in integrations_dir
  # imports each as a driver module
  # registers via bridge.register()
  # returns count of loaded drivers

handle_command(command: str, args: list) -> bool
  # commands: "load [dir]", "list"
```

---

### integrations_manager
**File:** `apps/modules/integrations_manager.py`

```python
# Thin wrapper over bridge + registry
# Ensures drivers loaded before resolve calls
```

---

### usage_tracker
**File:** `apps/modules/usage_tracker.py`

```python
handle_command(command: str, args: List[str]) -> bool
  # commands: "track <prompt_tokens> <completion_tokens> <model> [caller]",
  #           "stats", "session", "caller <name>", "cleanup [--days N]"

track_usage(args: List[str]) -> None
  # appends usage record to usage_log.json

show_stats() -> None
  # aggregate: total tokens, cost estimate by model

show_session() -> None
  # current session totals

show_caller_usage(args: List[str]) -> None
  # filter by caller branch

cleanup_data(args: List[str]) -> None
  # removes entries older than N days
```

**Reads/Writes:** `api/usage_log.json`

---

## Usage Log Schema

```json
{
  "entries": [
    {
      "ts": "ISO-8601",
      "model": "string",
      "prompt_tokens": 0,
      "completion_tokens": 0,
      "caller": "string | null",
      "cost_usd": 0.0
    }
  ]
}
```

---

## Dependencies

| Depends on | Why |
|-----------|-----|
| `prax` | logger |
| `cli` | display output |
| `requests` | HTTP calls to OpenRouter / validation endpoints |

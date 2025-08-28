# Toolbox Cache / Utils / Rules

`async with toolbox()` spin up new connection, create polling task, storage and jar. And never closes it automatically.

That's because each "key": `(["US", "DE"], "list", "random")` is cached. If you ever create toolbox with same values it will be loaded from cache

- **Safe:** Calling `toolbox()` many times **with the same key** (e.g., inside handlers) — you'll get the cached instance.
- **Safe** Don't close toolbox in one-time scripts, it will close automatically
- **Risky:** Spinning up many toolboxes **with different keys** — each creates its own polling/storage tasks and can exhaust resources.

> warning "Closing behavior"
    The cached toolbox **is not auto-closed** by the context manager if it was already created before.  
    If you open multiple toolboxes (especially with different keys), **manually clean them up** on shutdown.

> In jupyter notebooks, toolbox can hang-up infinitely in jupyter, so remember to close it 

> It is safe to don't close toolbox for one-time scripts, it will closes automatically, but better to have graceful shutdown in any case

### Recommended patterns

**a) Single key across your app (most common)**

```python
from unified_scraping import toolbox

TOOLBOX_ARGS = dict(country_codes=("DE",), storage_type="list", jar_type="random")
```

```python
# module: scraping.py
from unified_scraping import request
from deps import TOOLBOX_ARGS, toolbox

async def handle():
    async with toolbox(**TOOLBOX_ARGS) as tb:  # returns cached instance
        r = await request("http://ip-api.com/json/", "GET", enable_proxy=True, proxy_jar=tb.jar)
        return r.json()
```

**b) Cleanup on shutdown**

```python
import asyncio
import signal
from unified_scraping.toolbox_runner import cleanup_all_toolboxes

async def _graceful_shutdown(*_):
    await cleanup_all_toolboxes()

def install_signals(loop: asyncio.AbstractEventLoop):
    for sig in (signal.SIGINT, signal.SIGTERM):
        loop.add_signal_handler(sig, lambda s=sig: asyncio.create_task(_graceful_shutdown(s)))
```

**c) Tests / scripts (explicit cleanup)**

```python
from unified_scraping import toolbox
from unified_scraping.toolbox_runner import cleanup_toolbox

async def run_once():
    tb = await toolbox(country_codes=("DE",), storage_type="queue", jar_type="exhausting").__aenter__()
    try:
        # ... use tb.jar / tb.storage
        ...
    finally:
        await cleanup_toolbox(tb)
```

## Long-running or scraping-API case:
If you want use toolbox in developing scraping-API and curious how to do it - limit yourself with one key, it won't take to much RAM or CPU, and you will always have clean and up-to-date proxies in any part of code

## Manual cleanup API

These helpers let you manage cached toolboxes and their tasks explicitly.

```python
from unified_scraping.toolbox_runner import (
    cleanup_toolbox,
    cleanup_all_toolboxes,
    get_cached_toolbox_keys,
    get_toolbox,
    get_key_by_toolbox,
    get_toolbox_tasks,
)
```

### `await cleanup_toolbox(tb: ToolBox) -> None`

Closes and removes a **specific** toolbox from the cache.

```python
await cleanup_toolbox(tb)
```
- Stops `tb.proxy_storage` and `tb.integration`.
- Raises `KeyError` if the toolbox is not found in the cache.

### `await cleanup_all_toolboxes() -> None`

Closes **all** cached toolboxes. Useful in app shutdown hooks.

```python
await cleanup_all_toolboxes()
```

### `get_cached_toolbox_keys() -> list[_ToolboxKey]`

Returns all cache keys currently present.

```python
keys = get_cached_toolbox_keys()
# Example item: (("DE", "US"), "queue", "exhausting")
```

### `get_toolbox(key: _ToolboxKey) -> ToolBox | None`

Fetches a toolbox by key (without creating a new one).

```python
key = (("DE",), "list", "random")
tb = get_toolbox(key)
```

### `get_key_by_toolbox(tb: ToolBox) -> _ToolboxKey | None`

Returns a toolbox's cache key if it exists in the cache.

```python
key = get_key_by_toolbox(tb)
```

### `get_toolbox_tasks(key: _ToolboxKey) -> dict[str, asyncio.Task]`

Returns **running** background tasks for a toolbox (namespaced as `integration:*` or `storage:*`).

```python
tasks = get_toolbox_tasks((("DE",), "queue", "exhausting"))
for name, task in tasks.items():
    print(name, task, task.done())
```

---
# Toolbox Cache / Utils / Rules

`async with toolbox()` spins up a new connection, creates polling task, storage and jar. And never closes it automatically.

That's because each "key": `(("US", "DE"), "list", "random", "webshare")` is cached. If you ever create toolbox with the same values it will be loaded from cache.

- **Safe:** Calling `toolbox()` many times **with the same key** (e.g., inside handlers) — you'll get the cached instance.
- **Safe:** Don't close toolbox in one-time scripts, it will close automatically.
- **Risky:** Spinning up many toolboxes **with different keys** — each creates its own polling/storage tasks and can exhaust resources (note: residential proxy services don't create background polling tasks).

> **Warning: Closing behavior**  
> The cached toolbox **is not auto-closed** by the context manager if it was already created before.  
> If you open multiple toolboxes (especially with different keys), **manually clean them up** on shutdown.

> **Note:** In jupyter notebooks, toolbox can hang up infinitely, so remember to close it.

> **Note:** It is safe to not close toolbox for one-time scripts - it will close automatically, but it's better to have graceful shutdown in any case.

---

## Cache key structure

A toolbox cache key consists of **four components**:

```python
(country_codes, storage_type, jar_type, service)
```

**Examples:**
- `(("DE",), "list", "random", "webshare")` — German datacenter proxies
- `(("US", "GB"), "queue", "exhausting", "webshare")` — US+GB datacenter proxies with exhausting jar
- `(("DE",), "list", "random", "evomi")` — German residential proxies
- `((), "list", "roundrobin", "webshare_high_rotating")` — All countries, residential rotating proxies

> **Important:** Changing **any** of these four parameters creates a **new** cache entry with separate polling/storage tasks.

---

## Recommended patterns

### a) Single key across your app (most common)

```python
# module: scraping.py
from unified_scraping import request, toolbox

async def handle():
    async with toolbox(country_codes=("DE",), storage_type="list", jar_type="random") as tb:  # returns cached instance
        r = await request(
            "http://ip-api.com/json/",
            "GET",
            enable_proxy=True,
            proxy_jar=tb.jar
        )
        return r.json()
```

### b) Cleanup on shutdown

```python
import asyncio
import signal
from unified_scraping.toolbox_runner import cleanup_all_toolboxes

async def _graceful_shutdown(*_):
    await cleanup_all_toolboxes()

def install_signals(loop: asyncio.AbstractEventLoop):
    for sig in (signal.SIGINT, signal.SIGTERM):
        loop.add_signal_handler(
            sig,
            lambda s=sig: asyncio.create_task(_graceful_shutdown(s))
        )
```

### c) Tests / scripts (explicit cleanup)

```python
from unified_scraping import toolbox
from unified_scraping.toolbox_runner import cleanup_toolbox

async def run_once():
    async with toolbox(
        country_codes=("DE",),
        storage_type="queue",
        jar_type="exhausting",
        service="webshare",
    ) as tb:
        try:
            # ... use tb.jar / tb.storage
            ...
        finally:
            await cleanup_toolbox(tb)
```

---

## Long-running or scraping-API case

If you want to use toolbox in developing scraping-API and are curious how to do it - limit yourself with one key, it won't take too much RAM or CPU, and you will always have clean and up-to-date proxies in any part of code.

---

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

**Example:**
```python
tb = await toolbox(country_codes=("DE",), service="webshare").__aenter__()
try:
    # ... use tb
    pass
finally:
    await cleanup_toolbox(tb)
```

---

### `await cleanup_all_toolboxes() -> None`

Closes **all** cached toolboxes. Useful in app shutdown hooks.

```python
await cleanup_all_toolboxes()
```

**Example:**
```python
async def shutdown():
    print("Cleaning up all toolboxes...")
    await cleanup_all_toolboxes()
    print("Shutdown complete")
```

---

### `get_cached_toolbox_keys() -> list[tuple[tuple[str, ...], str, str, str]]`

Returns all cache keys currently present. Each key is a 4-tuple:

```python
(country_codes, storage_type, jar_type, service)
```

**Example:**
```python
keys = get_cached_toolbox_keys()
for key in keys:
    country_codes, storage_type, jar_type, service = key
    print(f"Service: {service}, Countries: {country_codes}, "
          f"Storage: {storage_type}, Jar: {jar_type}")

# Example output:
# Service: webshare, Countries: ('DE', 'US'), Storage: queue, Jar: exhausting
# Service: evomi, Countries: ('US',), Storage: list, Jar: random
```

---

### `get_toolbox(key: tuple[tuple[str, ...], str, str, str]) -> ToolBox | None`

Fetches a toolbox by key (without creating a new one).

```python
key = (("DE",), "list", "random", "webshare")
tb = get_toolbox(key)
if tb:
    print("Toolbox found in cache")
else:
    print("Toolbox not in cache")
```

---

### `get_key_by_toolbox(tb: ToolBox) -> tuple[tuple[str, ...], str, str, str] | None`

Returns a toolbox's cache key if it exists in the cache.

```python
async with toolbox(country_codes=("US",), service="evomi") as tb:
    key = get_key_by_toolbox(tb)
    print(key)  # (('US',), 'list', 'roundrobin', 'evomi')
```

---

### `get_toolbox_tasks(key: tuple[tuple[str, ...], str, str, str]) -> dict[str, asyncio.Task]`

Returns **running** background tasks for a toolbox (namespaced as `integration:*` or `storage:*`).

```python
key = (("DE",), "queue", "exhausting", "webshare")
tasks = get_toolbox_tasks(key)

for name, task in tasks.items():
    print(f"Task: {name}, Done: {task.done()}, Cancelled: {task.cancelled()}")

# Example output:
# Task: integration:poll, Done: False, Cancelled: False
# Task: storage:refill, Done: False, Cancelled: False
```

**Use case:** Debugging or monitoring background task health in production.

---

## Cache management tips

### Inspect current cache state

```python
from unified_scraping.toolbox_runner import get_cached_toolbox_keys, get_toolbox_tasks

# See what's cached
keys = get_cached_toolbox_keys()
print(f"Active toolbox configurations: {len(keys)}")

for key in keys:
    tasks = get_toolbox_tasks(key)
    print(f"\nKey: {key}")
    print(f"  Active tasks: {len(tasks)}")
    for task_name, task in tasks.items():
        print(f"    {task_name}: {'running' if not task.done() else 'finished'}")
```

### Prevent cache bloat

**Anti-pattern (creates many cache entries):**
```python
# DON'T: Different country codes each time
for country in ["US", "DE", "FR", "GB", "ES"]:
    async with toolbox(country_codes=(country,), service="webshare") as tb:
        ...  # Creates 5 separate cache entries!
```

**Better pattern:**
```python
# DO: Use one broad configuration
async with toolbox(
    country_codes=("US", "DE", "FR", "GB", "ES"),
    service="webshare"
) as tb:
    ...  # Single cache entry
```

Or use the toolbox's proxy filtering capabilities after retrieval if you need country-specific behavior.

---

## Service-specific considerations

### Datacenter proxies (webshare)
- Support all storage and jar types
- Background polling refreshes proxy list periodically

### Residential proxies (evomi, webshare_high_rotating)
- Only support `ListProxyStorage` + `RandomProxyJar`/`RoundRobinProxyJar`
- Proxy rotation handled at provider level
- Do not create background polling tasks

---

## Summary

- **Cache key** = `(country_codes, storage_type, jar_type, service)`
- **Same key** = reuses cached instance (efficient)
- **Different key** = new instance with separate background tasks (resource cost)
- **Best practice**: Use 1-2 keys for most applications
- **Always**: Call `cleanup_all_toolboxes()` on shutdown for clean exit

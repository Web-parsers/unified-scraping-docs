# Toolbox

Toolbox is the main entry point for interacting with proxies in **unified-scraping**.

---

## Default usage

```python
import asyncio
from unified_scraping import toolbox, request

async def main():
    async with toolbox() as tb:
        response = await request(
            url="http://ip-api.com/json/",
            method="GET",
            enable_proxy=True,
            proxy_jar=tb.jar,   # use the jar provided by toolbox
        )
        print(response.json())

if __name__ == "__main__":
    asyncio.run(main())
```
---

## What `toolbox()` provides

`toolbox()` is a context manager that wires together three building blocks:

- **ProxyIntegration** — background poller that refreshes proxies every `PROXY_INTEGRATION_POLLING_INTERVAL` seconds.
- **ProxyStorage** — where proxies are stored/cached.
  - `ListProxyStorage` — simple in‑memory list
  - `QueueProxyStorage` — queue semantics (FIFO) suited for sticky/one‑shot use
- **ProxyJar** — the *strategy* for choosing a proxy for a request.
  - `RandomProxyJar` — random selection each time
  - `RoundRobinProxyJar` — cycles through proxies evenly
  - `ExhaustingProxyJar` — sticky/one‑time use; removes a proxy until it is refilled

---

## Jars × Storage compatibility

Some strategies fit certain storages better. Use this matrix as a quick guide:

| Storage ↓ / Jar →      | RandomProxyJar | RoundRobinProxyJar | ExhaustingProxyJar |
|------------------------|----------------|--------------------|--------------------|
| **ListProxyStorage**   | ✅ Supported   | ✅ Supported       | ❌ Not designed    |
| **QueueProxyStorage**  | ✅ Supported   | ✅ Supported       | ✅ **Best fit**    |

> **Rule of thumb:** `ListProxyStorage` is a great default for ~80% of use cases. Prefer `QueueProxyStorage` when you need sticky/one‑shot semantics with `ExhaustingProxyJar`.

---

## How to choose a jar (strategy)

There is no single “best” strategy. Pick based on the target site’s defenses and your traffic pattern:

- **RandomProxyJar** — good default when you have many independent requests and want simple load spreading.
- **RoundRobinProxyJar** — evens out usage across your pool; helpful when providers rate‑limit per IP.
- **ExhaustingProxyJar** — avoid reusing the same proxy within a short window; ideal for flows where you want “consume and move on.”

> **Tip:** run short A/B tests (same workload, different jars) and compare success rate, average latency, and ban rate.

---

## Custom toolbox

You can customize country filtering, storage type, and jar type. Example:

```python
from unified_scraping import toolbox

async with toolbox(
    country_codes=("DE",),   # e.g. ("DE", "US"); case-insensitive
    storage_type="queue",    # "list" or "queue"
    jar_type="exhausting",   # "random", "roundrobin", or "exhausting" ("exhausting" requires "queue")
) as tb:
    ...
```

### Parameters

| Name            | Type              | Default  | Notes                                                                 |
|-----------------|-------------------|----------|-----------------------------------------------------------------------|
| `country_codes` | tuple[str \| None] | `()`     | Filter proxies by ISO country code. Case-insensitive.                 |
| `storage_type`  | `"list" \| "queue"` | `"list"` | Choose proxy storage implementation.                                  |
| `jar_type`      | `"random" \| "roundrobin" \| "exhausting"` | `"random"` | Strategy for selecting proxies. `exhausting` pairs with `"queue"`. |

---

## FAQ
**Q: What happens if proxy polling fails temporarily?**  
A: Existing proxies remain usable; the next successful poll will refresh the pool.

**Q: Can I mix multiple country codes?**  
A: Yes. Example: `country_codes=("DE", "US")`.

---

## Minimal example without context manager

If you want full control over lifecycle, you can instantiate and close components yourself (advanced):

```python
from unified_scraping import toolbox

tb = await toolbox().__aenter__()
try:
    # use tb.jar / tb.storage
    ...
finally:
    await toolbox().__aexit__(None, None, None)
```

> For most cases, prefer the `async with toolbox():` pattern.

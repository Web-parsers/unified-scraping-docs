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

## Proxy services

Toolbox supports multiple proxy services via the `service` parameter:

- **webshare** (default) — datacenter proxies with full feature support
- **evomi** — residential proxies (rotating)
- **webshare_high_rotating** — residential proxies from Webshare (rotating)

> **Note:** Rotating residential proxy services (`evomi`, `webshare_high_rotating`) only support `ListProxyStorage` with `RandomProxyJar` or `RoundRobinProxyJar`. The `ExhaustingProxyJar` and `QueueProxyStorage` are not compatible with rotating services.

---

## Jars × Storage × Service compatibility

Some strategies fit certain storages and services better. Use this matrix as a quick guide:

### Webshare (datacenter proxies)

| Storage ↓ / Jar →      | RandomProxyJar | RoundRobinProxyJar | ExhaustingProxyJar |
|------------------------|----------------|--------------------|--------------------|
| **ListProxyStorage**   | ✅ Supported   | ✅ Supported       | ❌ Not designed    |
| **QueueProxyStorage**  | ✅ Supported   | ✅ Supported       | ✅ **Best fit**    |

### Evomi & Webshare High Rotating (residential proxies)

| Storage ↓ / Jar →      | RandomProxyJar | RoundRobinProxyJar | ExhaustingProxyJar |
|------------------------|----------------|--------------------|--------------------|
| **ListProxyStorage**   | ✅ Supported   | ✅ Supported       | ❌ Not supported   |
| **QueueProxyStorage**  | ❌ Not supported | ❌ Not supported | ❌ Not supported   |

> **Rule of thumb:** `ListProxyStorage` is a great default for ~80% of use cases. Prefer `QueueProxyStorage` when you need sticky/one‑shot semantics with `ExhaustingProxyJar` (only available with `webshare` service).

---

## How to choose a jar (strategy)

There is no single "best" strategy. Pick based on the target site's defenses and your traffic pattern:

- **RandomProxyJar** — good default when you have many independent requests and want simple load spreading.
- **RoundRobinProxyJar** — evens out usage across your pool; helpful when providers rate‑limit per IP.
- **ExhaustingProxyJar** — avoid reusing the same proxy within a short window; ideal for flows where you want "consume and move on." *(Only available with `webshare` service and `QueueProxyStorage`)*

> **Tip:** run short A/B tests (same workload, different jars) and compare success rate, average latency, and ban rate.

---

## Custom toolbox

You can customize country filtering, storage type, jar type, and proxy service. Example:

```python
from unified_scraping import toolbox

# Datacenter proxies with full features (default)
async with toolbox(
    country_codes=("DE",),   # e.g. ("DE", "US"); case-insensitive
    storage_type="queue",    # "list" or "queue"
    jar_type="exhausting",   # "random", "roundrobin", or "exhausting"
    service="webshare",      # default service
) as tb:
    ...

# Residential proxies (rotating)
async with toolbox(
    country_codes=("DE", "US"),
    service="webshare_high_rotating",  # or "evomi"
    jar_type="random",                  # only "random" or "roundrobin" supported
) as tb:
    ...
```

### Parameters

| Name            | Type              | Default      | Notes                                                                 |
|-----------------|-------------------|--------------|-----------------------------------------------------------------------|
| `country_codes` | tuple[str, ...] | `()`         | Filter proxies by ISO country code. Case-insensitive.                 |
| `storage_type`  | `"list"` \| `"queue"` | `"list"` | Choose proxy storage implementation. Rotating services only support `"list"`. |
| `jar_type`      | `"random"` \| `"roundrobin"` \| `"exhausting"` | `"roundrobin"` | Strategy for selecting proxies. `"exhausting"` requires `"queue"` storage and `"webshare"` service. |
| `service`       | `"webshare"` \| `"evomi"` \| `"webshare_high_rotating"` | `"webshare"` | Proxy service provider. `"webshare"` provides datacenter proxies with full features. `"evomi"` and `"webshare_high_rotating"` provide residential rotating proxies with limited feature support. |

---

## Service-specific limitations

### Webshare (datacenter)
- ✅ All storage types supported
- ✅ All jar types supported
- ✅ Full feature set

### Evomi & Webshare High Rotating (residential)
- ✅ Only `ListProxyStorage` supported
- ✅ Only `RandomProxyJar` and `RoundRobinProxyJar` supported
- ❌ `QueueProxyStorage` not supported
- ❌ `ExhaustingProxyJar` not supported

Attempting to use unsupported combinations will raise a `ValueError`:
```python
# This will raise ValueError
async with toolbox(
    service="evomi",
    storage_type="queue"  # Not supported for residential services
):
    ...

# This will also raise ValueError
async with toolbox(
    service="webshare_high_rotating",
    jar_type="exhausting"  # Not supported for residential services
):
    ...
```

---

## FAQ

**Q: What happens if proxy polling fails temporarily?**  
A: Existing proxies remain usable; the next successful poll will refresh the pool.

**Q: Can I mix multiple country codes?**  
A: Yes. Example: `country_codes=("DE", "US")`.

**Q: When should I use residential proxies vs datacenter proxies?**  
A: Residential proxies (`evomi`, `webshare_high_rotating`) are better for sites with strict anti-bot protection, while datacenter proxies (`webshare`) offer more features and flexibility for general scraping tasks.

**Q: Why can't I use ExhaustingProxyJar with residential services?**  
A: Residential rotating proxies already rotate automatically at the provider level, so there are no option to "bake" some proxy in jar and use it

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

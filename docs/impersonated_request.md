# Impersonated Request

Utilities for making HTTP requests with **browser impersonation**, **proxy rotation**, and **fine-grained retry control**. Built on `curl_cffi.AsyncSession`.

---

- Retries are split into **per-URL** and **per-proxy** budgets.
- On each URL attempt, a **browser type** is chosen (from `BROWSER_TYPES`).
- If proxies are enabled, a proxy is drawn from your **ProxyJar** strategy.
- **Checkers** (response & error) decide what to do next: continue, retry, change proxy, or drop.

---

## Pseudocode

```python
from unified_scraping import request

async def request(kwargs):
    while url_attempts < URL_MAX_RETRIES:
        bt = random(BROWSER_TYPES)
        proxy = proxy_jar.get_proxy() if enable_proxy else None

        for proxy_attempt in range(PROXY_MAX_RETRIES):
            resp_or_exc = session.request(bt, proxy, **kwargs)

            action = run_checkers(resp_or_exc)
            if action == "continue":
                return response
            if action == "drop":
                raise SkippedRequest
            if action in {"change_proxy"} or timeout_hit:
                # change proxy; break inner loop
                break
            # else: retry with same proxy

    raise TooMuchRetries
```
You can configure:

- **URL retries**: `IMPERSONATED_REQUEST_URL_MAX_RETRIES`
- **Proxy retries**: `IMPERSONATED_REQUEST_PROXY_MAX_RETRIES`

---

## Quick start

```python
import asyncio
from curl_cffi import Response
from unified_scraping import request

# Example checkers
@request.response_checker(on_check="continue")
async def log_url(resp: Response):
    print(f"Url and Status: {resp.status_code} {resp.url}")

@request.response_checker(on_check="continue")
async def log_country(resp: Response):
    print(f"Country: {resp.json()['country']}")

@request.response_checker(on_check="change_proxy")
async def ensure_ukraine(resp: Response):
    is_ukraine = resp.json().get("country") == "Ukraine"
    print(f"[is_ukraine]: {is_ukraine}")
    return not is_ukraine  # True -> trigger change_proxy

async def main():
    fixture_jar = FixtureJar()  # returns DE, DE, then UA proxy
    response = await request(
        "http://ip-api.com/json/",
        method="GET",
        enable_proxy=True,
        proxy_jar=fixture_jar,
    )
    print(response.text)

if __name__ == "__main__":
    asyncio.run(main())
```

**Expected output:**
```text
Url and Status: 200 http://ip-api.com/json/
Country: Germany
[is_ukraine]: False

Url and Status: 200 http://ip-api.com/json/
Country: Germany
[is_ukraine]: False

Url and Status: 200 http://ip-api.com/json/
Country: Ukraine
[is_ukraine]: True
```

---

## Request checkers

Checkers let you *program the control flow* for any URL (optionally matching by pattern).

### Actions & priority

The highest-priority action observed across matching checkers is chosen.

| Action         | Priority | Meaning                                                                 |
|----------------|---------:|-------------------------------------------------------------------------|
| `continue`     | 0        | Accept and return the response                                          |
| `retry`        | 1        | Retry with the **same proxy**                                           |
| `change_proxy` | 2        | Retry but **switch proxy** (or on timeout)                              |
| `drop`         | 3        | Skip this URL entirely (raises `SkippedRequest`)                        |

> Multiple checkers can run; the action with the highest priority wins.

### Response checker

Decorator for inspecting a successful `Response`.

```python
from curl_cffi import Response

@request.response_checker(pattern=r"https://example\.com/.*", on_check="retry")
async def must_have_ok(resp: Response) -> bool:
    data = resp.json()
    return data.get("ok") is False  # True -> trigger retry
```

### Error checker

Decorator for inspecting exceptions (and optional `response` if present).

```python
from typing import Optional
from curl_cffi import Response

@request.error_checker(on_check="drop")
async def drop_on_404(exc: Exception, resp: Optional[Response]) -> bool:
    return bool(resp and resp.status_code == 404)
```

> Both checkers **must be `async`** and return `True` to trigger their `on_check` action.

### Pattern matching

- `pattern` accepts a raw string or precompiled `re.Pattern`.
- If omitted, the checker applies to **all URLs**.
- Pattern is tested against the **full URL**.

---

## Advanced control: subclassing

For multi-domain projects or custom behavior, subclass `ImpersonatedRequest`:

```python
from unified_scraping import ImpersonatedRequest

class CustomImpersonatedRequest(ImpersonatedRequest):
    # override defaults, add/register project-specific checkers, etc.
    ...

custom_request = CustomImpersonatedRequest(
    url_max_retries=50,
    proxy_max_retries=5,
)

resp = await custom_request.get(
    "https://target.example/",
    enable_proxy=True,
    proxy_jar=my_jar,
)
```

---

## Browser impersonation & session pool

- A session pool is maintained per `BrowserType` (`curl_cffi.BrowserType`).
- A browser type is picked per URL attempt (or you can pass `impersonate=...` in kwargs).
- Some browser types (e.g., specific Safari/Tor variants) are excluded by default.

> You can override the chosen browser type per call using `impersonate=<BrowserType>`.

---

## API reference (high-level)

```python
from unified_scraping import request, ImpersonatedRequest

# callable shortcut:
await request(url, method, enable_proxy=True, proxy_jar=jar, **kwargs)

# explicit:
await request.request(url, method, enable_proxy=True, proxy_jar=jar, **kwargs)

# convenience verbs:
await request.get(url, enable_proxy=True, proxy_jar=jar, **kwargs)
await request.post(url, enable_proxy=True, proxy_jar=jar, **kwargs)
await request.put(url, enable_proxy=True, proxy_jar=jar, **kwargs)
await request.patch(url, enable_proxy=True, proxy_jar=jar, **kwargs)
await request.delete(url, enable_proxy=True, proxy_jar=jar, **kwargs)
await request.options(url, enable_proxy=True, proxy_jar=jar, **kwargs)
await request.head(url, enable_proxy=True, proxy_jar=jar, **kwargs)
```

**Parameters**
- `url`: str — target URL
- `method`: one of `HttpMethod` (case-insensitive string is fine)
- `enable_proxy`: bool — toggle proxy usage (default from env)
- `proxy_jar`: `ProxyJar` — required if `enable_proxy=True`
- `**kwargs`: forwarded to `AsyncSession.request` (e.g., headers, data, timeout, impersonate, proxies, etc.)

**Exceptions**
- `TooMuchRetries` — URL budget exhausted
- `SkippedRequest` — checker requested to drop this URL

---

## Environment variables

```dotenv
IMPERSONATED_REQUEST_DEFAULT_ENABLE_PROXY=False
IMPERSONATED_REQUEST_PROXY_MAX_RETRIES=10
IMPERSONATED_REQUEST_URL_MAX_RETRIES=100
```

> You can still override `enable_proxy`, `proxy_jar`, and retry counts per call or via subclassing.

---

## Integration with Toolbox & ProxyJar

- Pass `proxy_jar=tb.jar` from `async with toolbox():` for best results.
- Use `RandomProxyJar` / `RoundRobinProxyJar` / `ExhaustingProxyJar` depending on the target’s defenses.
- Combine with **response/error checkers** to adapt to CAPTCHAs, login walls, or region gating.

---

## Notes on timeouts & proxy switching

- A `Timeout` triggers the same behavior as `change_proxy` (switches proxy).
- When a proxy consistently fails (max retries reached), it’s marked as **blocked** via the jar before switching.
- The original `kwargs` are preserved in logs to help debugging failed attempts.

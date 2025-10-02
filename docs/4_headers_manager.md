# HeadersManager

A smart cookie monitoring system focused on providing fresh cookies, headers, and proxies for web requests.

## Overview

HeadersManager consists of two components:

- **Server Side**: Provides API and monitoring for automatic data maintenance
- **Client Side**: Python async API with configurable behavior via the `use_sem` parameter

## Client API

### Default Mode (Simple Get)

```python
from unified_scraping import headers_manager

async with headers_manager("kaufland.de") as hm:
    print(hm.proxy)
    print(hm.cookies)
    try:
        await request(...)
    except:
        await hm.mark_failure()
```

**How it works:**
```python
async headers_manager(...) as hm:
    # GET /{hm}/get - retrieves available headers/cookies
    request(hm.headers)
```

### Semaphore Mode (Acquire/Release)

```python
from unified_scraping import headers_manager

async with headers_manager("kaufland.de", use_sem=True) as hm:
    print(hm.proxy)
    print(hm.cookies)
    try:
        await request(...)
    except:
        await hm.mark_failure()
```

**How it works:**
```python
async headers_manager(..., use_sem=True) as hm:
    # POST /{hm}/acquire - acquires exclusive access to headers/cookies
    request(hm.headers)
    # POST /{hm}/release/{uuid} - releases acquired cookies by UUID
```

## Server Configuration

### Adding a New HeadersManager

To add a new HeadersManager configuration:

1. **Create a browser bypass script** in `service/headers_manager/bypasses/{script_name}.py`
2. **Write a configuration** that references your script

### Example Configuration

[Headers-manager server repository](https://github.com/Web-parsers/proxy-service/tree/headers-manager)

**Example bypass script:** [bypass_cf_rozetka.py](https://github.com/Web-parsers/proxy-service/blob/headers-manager/service/headers_manager/bypasses/bypass_cf_rozetka.py)

**Configuration file:**
```json
{
    "name": "rozetka_podushka",
    "script_name": "bypass_cf_rozetka",
    "url": "any",
    "max_cache": 2,
    "idle_time": 150,
    "updates_semaphore": 2,
    "meta": {
        "proxy": {
            "country_codes": ["ua"],
            "service": "webshare"
        }
    }
}
```

### Configuration Parameters

> **Note:** If a parameter is not specified in the config, the DEFAULT value is used.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | *required* | Unique identifier used in client: `headers_manager(name)` |
| `script_name` | `str` | *required* | Script filename (without path) located in `bypasses/` directory |
| `url` | `str` | `None` | Optional URL if one script handles multiple domains |
| `max_cache` | `int` | `10` | Maximum number of cached headers/cookies at one time |
| `max_failures` | `int` | `10` | Number of failures before cache is removed |
| `updates_semaphore` | `int` | `5` | Maximum simultaneous browser updates (e.g., 5 = 5 browsers) |
| `idle_time` | `int` | `1800` (30 min) | Seconds of inactivity before HeadersManager pauses monitoring |
| `ttl` | `int` | `DEFAULT_CACHE_TTL` | Time-to-live before cache is automatically removed |
| `cache_concurrency` | `int \| None` | `None` | Concurrent requests per cache. `None` = unlimited. If set to `3`, only 3 requests allowed simultaneously per cache |
| `meta` | `dict \| None` | `None` | Additional data passed to browser script (proxy settings, URLs, etc.) |

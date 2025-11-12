# Impersonate Request

This is module for doing requests

Pseudo-code behavior:

```python
from unified_scraping import request
async def request(kwargs):
    while not URL_MAX_RETRIES:
        bt = random(BROWSER_TYPES)
        proxy = proxy_jar.get_proxy()

        while not PROXY_MAX_RETRIES:
            session.request(**kwargs)
    raise TooMuchRetries
```
So we can setup behavior:
* retries per url
* retries per proxy

## Request Statistics

The `ImpersonatedRequest` class now tracks request statistics automatically:

- **`success_count`** — number of successful requests (HTTP 2xx/3xx that passed all checkers without failure actions)
- **`failure_count`** — number of failed attempts (exceptions or checker failures)
- **`total_count`** — total number of request attempts
- **`bussines_failure_count`** — number of URLs that exhausted all retries (raised `TooMuchRetries`)

```python
from unified_scraping import request

# Make some requests
await request("https://example.com", "GET", enable_proxy=False)
await request("https://example.com", "GET", enable_proxy=False)

# Check statistics
print(f"Success: {request.success_count}")
print(f"Failures: {request.failure_count}")
print(f"Total attempts: {request.total_count}")
print(f"Business failures: {request.bussines_failure_count}")
```

These counters are useful for monitoring scraping health and debugging retry patterns.

### How success/failure is counted

The success/failure counting is determined by **response checkers**:

- **Success** is counted when the request succeeds AND all response checkers either return `False` or have `on_check="continue"`
- **Failure** is counted when at least one response checker returns `True` with `on_check` set to `"retry"`, `"change_proxy"`, or `"drop"`

**Important:** Multiple checkers with failure actions (`retry`, `change_proxy`, `drop`) will only increment the failure counter **once per request**, not once per checker. This means it's safe to use multiple checkers without inflating your failure statistics.

```python
@request.response_checker(on_check="retry")
async def check_captcha(resp):
    return "captcha" in resp.text

@request.response_checker(on_check="retry")
async def check_rate_limit(resp):
    return resp.status_code == 429

@request.response_checker(on_check="change_proxy")
async def check_blocked(resp):
    return "blocked" in resp.text

# If request triggers check_captcha=True and check_blocked=True:
# - failure_count increases by 1 (not 2!)
# - The action with highest priority (change_proxy) is executed
```

Exception-based failures (network errors, timeouts, HTTP errors) also increment `failure_count` once per attempt.

---

## Own Realisation
If you want more control for request you can import class
`from unified_scraping import ImpersonatedRequest`
It can be useful for project which scrape multiple domains, or for adding own control points, remake impersonation and other changes 

```python
class CustomImpersonatedRequest(ImpersonatedRequest):
    ... # make changes you need
custom_request = CustomImpersonatedRequest(url_max_retries, proxy_max_retries)

await custom_request(...)
```

---

## Request Checkers

Each site have own pipeline how to scrape, own handlers and edge-cases, `request()` provide you all possibilities to control flow

Let's view easy example (you can find it in `impersonated_request_change_proxy_example.py`)

```python
@request.response_checker(on_check="continue")
async def log_url(resp):
    print(f"Url and Status: {resp.status_code} {resp.url}")

@request.response_checker(on_check="continue")
async def log_country(resp: Response):
    print(f"Country: {resp.json()['country']}")

@request.response_checker(on_check="change_proxy")
async def check_ukraine(resp: Response):
    is_ukraine = True if resp.json()["country"] == "Ukraine" else False
    print(f"[is_ukraine]: {is_ukraine}")
    return not is_ukraine

if __name__ == '__main__':
    import asyncio
    fixture_jar = FixtureJar() # Jar which return DE, DE, then UA proxy
    async def main():
        try:
            response = await request(
                "http://ip-api.com/json/",
                method="GET",
                enable_proxy=True,
                proxy_jar=fixture_jar,
            )
            print(response.text)
        except SkippedRequest as e:
            print("Request was skipped")
    asyncio.run(main())
```

let's see what we have. There are decorator @request.response_checker
it gives us possibility to make dynamic checkers. It has 4 cases:
`_PRIORITY = {"continue": 0, "retry": 1, "change_proxy": 2, "drop": 3}`
as you see each have own priority
continue - generally log
retry - does imidiate retry, but you can add time.sleep or else
change_proxy - case when you know, you need retry request, but change proxy also
drop - there are no sense to waste traffic, this url not processable


So when we have this knowledge, lets look again at code:
we have checker "change_proxy", if current proxy is not Ukraine, so it will do 2 requests with DE, and final with UA, which will be valid request, let's look at logs:

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


there are also @request.error_checker, which have same logic with prioritates, but works after Excpetion or raise_for_status

both error_checker, and response_checker have field parttern, that means you can have different checkers for different domain, routes and etc.

f.e. we have site1.com, which can return 404 page - we should imidietly exit, and can have 200 status, but login protected

```python
@request.error_checker(on_check="drop")
def check_404(e, resp):
    if resp.status_code == 404:
        return True
    return False

@request.response_checker(on_check="change_proxy")
def check_login(resp: Response):
    tree = html.fromstring(resp.text)
    if tree.xpath('//div[@class="login_form"]'):
        return True
    return False
```

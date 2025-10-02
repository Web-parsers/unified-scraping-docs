# Welcome

[**Toolbox**](https://github.com/Web-parsers/unified-scraping) to provide easy-to-use requests with:

- Impersonation
- Proxy-Service integration
- Proxy strategies

---

## 🚀 How to add in your project

### 1. Add as a submodule
```bash
git submodule add https://github.com/Web-parsers/unified-scraping.git unified_scraping
```

```dotenv
# Proxy integration
PROXY_INTEGRATION_API_KEY=<api key to proxy-service>
PROXY_INTEGRATION_URL=<proxy-service url>
PROXY_INTEGRATION_POLLING_INTERVAL=600
HEADERS_MANAGER_URL=<headers-manager url>

# Project settings
PROJECT_NAME=my-project

# Impersonated request defaults
IMPERSONATED_REQUEST_DEFAULT_ENABLE_PROXY=False
IMPERSONATED_REQUEST_PROXY_MAX_RETRIES=10
IMPERSONATED_REQUEST_URL_MAX_RETRIES=100
```

### Configuration Reference

| Variable | Description | Default/Notes |
|----------|-------------|---------------|
| `PROXY_INTEGRATION_API_KEY` | API key for proxy service | Find in Slack #dev-links |
| `PROXY_INTEGRATION_URL` | Proxy service endpoint | Find in Slack #dev-links |
| `PROXY_INTEGRATION_POLLING_INTERVAL` | Polling interval in seconds | `600` recommended |
| `HEADERS_MANAGER_URL` | Headers manager endpoint | Find in Slack #dev-links |
| `PROJECT_NAME` | Project identifier for logs | Any string |
| `IMPERSONATED_REQUEST_DEFAULT_ENABLE_PROXY` | Enable proxies by default | `False` (use `True` in prod) |
| `IMPERSONATED_REQUEST_PROXY_MAX_RETRIES` | Retries per proxy per URL | `10` |
| `IMPERSONATED_REQUEST_URL_MAX_RETRIES` | Total retries per URL | `100` |

### 3. Try it out
Run:
```bash
python -m unified_scraping.examples.impersonated_request_base_examples
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

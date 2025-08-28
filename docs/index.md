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

### 2. Configure `.env` file
```dotenv
# Proxy integration
PROXY_INTEGRATION_API_KEY=<api key to proxy-service>    # see Slack #dev-links channel
PROXY_INTEGRATION_URL=<proxy-service url>
PROXY_INTEGRATION_POLLING_INTERVAL=600                  # polling interval (600s = good default)

# Project settings
PROJECT_NAME=my-project                                 # project name (used in logs)

# Impersonated request defaults
IMPERSONATED_REQUEST_DEFAULT_ENABLE_PROXY=False         # use proxies by default
IMPERSONATED_REQUEST_PROXY_MAX_RETRIES=10               # retries per proxy for a single URL
IMPERSONATED_REQUEST_URL_MAX_RETRIES=100                # total retries per URL
```

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

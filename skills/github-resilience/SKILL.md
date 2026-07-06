---
name: github-resilience
description: Enforce resilient integration with the GitHub API, handling rate limits using exponential backoff, random jitter, and header parsing.
tags:
  - github
  - api
  - resilience
  - rate-limiting
---
# GitHub API Resilience & Rate Limit Handling

This skill outlines strategies and provides code patterns for robustly interacting with the GitHub API, ensuring operations survive rate limits and temporary API abuse blocks.

## 1. GitHub Rate Limits & Specific Headers

GitHub API enforces two types of rate limits:
1. **Primary Rate Limits:** Standard hourly quotas (typically 5,000 requests/hour for authenticated users).
2. **Secondary Rate Limits (Abuse Detection):** Thresholds guarding against spikes in concurrent requests.

### Key Headers to Monitor:
* `x-ratelimit-remaining`: The number of requests remaining in the current rate limit window.
* `x-ratelimit-reset`: The time (in UTC epoch seconds) when the current rate limit window resets.
* `retry-after`: The number of seconds to wait before retrying (usually returned with HTTP 429 or 403 secondary limits).

> [!NOTE]
> GitHub occasionally returns HTTP 403 with `x-ratelimit-remaining: "0"` instead of HTTP 429 when the primary rate limit is exhausted. Treat this scenario identically to an HTTP 429.

---

## 2. Resilient Request Workflow

GitHub API clients should wrap requests in a handler that intercepts throttling responses and delegates to general retry logic.

For the mathematical models governing sleep durations, including handling Unix reset epochs, exponential backoff, and the mitigation of "thundering herd" issues using random jitter, refer to the [rate-limit-resilience](../rate-limit-resilience/SKILL.md) skill.

### Implementation Checklist for GitHub:
1. **Intercept HTTP 429** or **HTTP 403 with `x-ratelimit-remaining: "0"`**.
2. **Resolve Sleep Duration:**
   * Extract `retry-after` header if present (standard secondary limits).
   * Otherwise, extract `x-ratelimit-reset` epoch seconds.
   * Otherwise, fall back to computed exponential backoff with jitter.
3. **Wait & Retry:** Wait the resolved duration and attempt the request again.

---

## 3. Implementation Reference (Python)

```python
import urllib.request
import urllib.error
import json
import time
import random
import sys

def github_request_with_retry(url, headers, data=None, method='GET', max_attempts=5, base_backoff=2.0):
    payload = json.dumps(data).encode('utf-8') if data else None
    req = urllib.request.Request(url, data=payload, headers=headers, method=method)
    
    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(req) as response:
                return json.loads(response.read().decode('utf-8'))
        except urllib.error.HTTPError as e:
            status = e.code
            res_headers = {k.lower(): v for k, v in e.headers.items()}
            
            # Identify primary and secondary rate limits
            is_rate_limited = (status == 429)
            if status == 403 and res_headers.get('x-ratelimit-remaining') == '0':
                is_rate_limited = True
                
            if is_rate_limited and attempt < max_attempts - 1:
                # 1. Respect Retry-After
                retry_after = res_headers.get('retry-after')
                if retry_after:
                    try:
                        wait_time = float(retry_after) + 0.5
                    except ValueError:
                        wait_time = base_backoff * (2 ** attempt) + random.uniform(0, 1)
                # 2. Respect Reset Epoch
                else:
                    reset_time = res_headers.get('x-ratelimit-reset')
                    if reset_time:
                        try:
                            wait_time = max(0.0, float(reset_time) - time.time()) + 1.0
                        except ValueError:
                            wait_time = base_backoff * (2 ** attempt) + random.uniform(0, 1)
                    # 3. Fallback to Exponential Backoff with Jitter
                    else:
                        wait_time = base_backoff * (2 ** attempt) + random.uniform(0, 1)
                
                print(f"[Rate Limit] Status {status}. Retrying in {wait_time:.2f}s...", file=sys.stderr)
                time.sleep(wait_time)
            else:
                raise e
```

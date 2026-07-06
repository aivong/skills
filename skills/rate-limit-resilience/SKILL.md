---
name: rate-limit-resilience
description: General client-side rate limit handling strategies, including server-sent hints, exponential backoff, and random jitter to avoid thundering herd issues.
tags:
  - api
  - resilience
  - rate-limiting
  - backoff
  - jitter
---
# API Rate Limit Resilience & Retry Strategies

This skill outlines API-agnostic strategies and mathematical models for robustly handling rate limits (throttling) and transient faults on the client side.

## 1. Rate Limit Types & Server-Sent Hints

APIs typically enforce limits to protect their infrastructure. Clients should prioritize explicit instructions sent by the server before falling back to programmatic algorithms.

### Deterministic Sleep Durations (Server-Sent Hints)
When a server returns a standard rate limit code (typically HTTP 429 Too Many Requests or HTTP 403 Forbidden with specific context), it usually indicates when the client is allowed to retry via HTTP headers:

* **Relative Reset Time (`Retry-After`):** Specifies the number of seconds to wait before retrying.
* **Absolute Reset Time (`X-RateLimit-Reset`):** Specifies the Unix epoch timestamp (in seconds or milliseconds) when the rate limit window resets.

#### Header Evaluation Strategy:
1. Parse the `Retry-After` header. If present, sleep for that duration plus a small safety buffer (e.g., `+ 0.5s`).
2. Parse the reset epoch timestamp. If present, sleep for `max(0, reset_timestamp - current_timestamp) + safety_buffer`.
3. If neither header is present or readable, fall back to **Probabilistic Sleep**.

---

## 2. Probabilistic Sleep (Exponential Backoff & Jitter)

When the server does not provide instructions (e.g., during secondary rate limits, severe service overloads, or connection drops), the client must incrementally slow down its retry rate.

### Exponential Backoff
Exponential backoff increases the wait time exponentially with each consecutive failure, giving the server time to recover.

$$\text{wait} = \text{base backoff} \times 2^{\text{attempt}}$$

### The Thundering Herd Problem & Jitter
If multiple client instances hit a rate limit or a server outage at the same time, they will all fail at roughly the same time. If they all use a simple exponential backoff, they will sleep for the exact same duration and wake up to retry at the exact same millisecond. This creates massive traffic spikes, keeping the server overloaded.

To prevent this, clients introduce **random jitter** (a randomized offset) to desynchronize their retries.

$$\text{wait} = \text{base backoff} \times 2^{\text{attempt}} + \text{random jitter}$$

* **Full Jitter:** Typically, `random_jitter` is selected uniformly between `0` and a maximum value (e.g., `random.uniform(0, 1)` or scaling dynamically with the backoff).

---

## 3. Generic Retry Logic (Pseudocode)

```python
import time
import random

def request_with_resilience(api_call, max_attempts=5, base_backoff=2.0):
    for attempt in range(max_attempts):
        try:
            return api_call()
        except RateLimitException as e:
            if attempt == max_attempts - 1:
                raise e
            
            # 1. Check for deterministic server hints
            if e.retry_after is not None:
                wait_time = e.retry_after + 0.5
            elif e.reset_time is not None:
                wait_time = max(0.0, e.reset_time - time.time()) + 1.0
            
            # 2. Fall back to probabilistic backoff with jitter
            else:
                wait_time = base_backoff * (2 ** attempt) + random.uniform(0.0, 1.0)
                
            time.sleep(wait_time)
```

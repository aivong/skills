# Custom agent skills

This repository contains agent skills that can be imported to extend agent capabilities:

* **[rate-limit-resilience](skills/rate-limit-resilience/SKILL.md):** General client-side rate limit handling strategies, including server-sent hints, exponential backoff, and random jitter to avoid thundering herd issues.
* **[github-resilience](skills/github-resilience/SKILL.md):** Enforce resilient integration with the GitHub API, handling rate limits using exponential backoff, random jitter, and header parsing.

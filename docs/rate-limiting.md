# Rate Limiting

The NLnest API enforces per-key daily request quotas to ensure fair usage and platform stability. Limits are generous for typical ATS integrations and can be raised on request.

---

## Default Limits

| Key type | Daily limit |
|----------|------------|
| Production (`nlnest_live_*`) | **1,000 requests / day** |
| Test (`nlnest_test_*`) | **200 requests / day** |

The counter resets at **midnight UTC** each day.

If your integration requires higher limits (e.g. bulk job imports or high-frequency polling), contact [NLnest](https://nlnest.com) to request a raised quota.

---

## Rate Limit Headers

Every API response includes three headers indicating your current quota status:

| Header | Description |
|--------|-------------|
| `X-RateLimit-Limit` | Your total daily request allowance |
| `X-RateLimit-Remaining` | Requests remaining today |
| `X-RateLimit-Reset` | Unix timestamp (UTC) when the counter resets |

### Example Response Headers

```
HTTP/1.1 200 OK
Content-Type: application/json
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1710374400
```

---

## When the Limit Is Exceeded

Once `X-RateLimit-Remaining` reaches `0`, further requests return HTTP `429`:

```json
{
  "success": false,
  "error": {
    "code": "rate_limit_exceeded",
    "message": "Daily API quota exceeded. Your limit resets at midnight UTC.",
    "details": {
      "limit": 1000,
      "remaining": 0,
      "reset_at": "2026-03-14T00:00:00Z"
    }
  }
}
```

The `reset_at` field is also available in ISO 8601 format in the error body for convenience.

---

## Best Practices

### Check remaining quota before bulk operations

```bash
# A quick HEAD or GET to inspect headers before a batch run
curl -I https://nlnest.com/api/v1/jobs \
  -H "X-API-Key: nlnest_live_your_key_here"
```

### Implement exponential backoff on 429

If you receive a `429`, wait until `X-RateLimit-Reset` before retrying. Do not poll continuously.

```python
import time

def api_request_with_backoff(url, headers):
    response = requests.get(url, headers=headers)
    if response.status_code == 429:
        reset_at = int(response.headers.get('X-RateLimit-Reset', time.time() + 60))
        wait_seconds = max(reset_at - int(time.time()), 1)
        print(f"Rate limited. Waiting {wait_seconds}s until reset.")
        time.sleep(wait_seconds)
        return api_request_with_backoff(url, headers)  # retry once
    return response
```

### Use webhooks instead of polling

For real-time application notifications, configure a [webhook](webhooks.md) rather than polling `GET /api/v1/applications` every few minutes. Webhooks do not count against your rate limit and are delivered within seconds.

### Batch job imports efficiently

When importing many jobs at once, spread requests over time. At 1,000 requests/day, you can safely create up to **~40 jobs per hour** (leaving headroom for reads and updates) without approaching the daily limit.

---

## Counting Requests

Each HTTP request to any `/api/v1/` endpoint counts as **one request** against your daily quota, regardless of:

- The HTTP method (GET, POST, PATCH, DELETE all count equally)
- Whether the request succeeds or fails
- Response payload size

`429` responses themselves do **not** count against your quota.

---

## See Also

- [Authentication →](authentication.md)
- [Errors — 429 rate_limit_exceeded →](errors.md)
- [Webhooks (no quota cost) →](webhooks.md)

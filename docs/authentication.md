# Authentication

All requests to the NLnest Employer API must include a valid API key. Keys are tied to a single company account and carry the permissions of the company owner.

---

## Getting an API Key

1. Log in to your company account at [nlnest.com](https://nlnest.com)
2. Navigate to **Dashboard → Settings → API**
3. Click **Generate New Key**
4. Copy the key immediately — it is shown only once

---

## Key Types

| Type | Prefix | Behaviour |
|------|--------|-----------|
| **Production** | `nlnest_live_` | Full functionality; jobs can be auto-published based on your company setting |
| **Test** | `nlnest_test_` | Safe for development; jobs are always created as `draft` regardless of settings |

Use test keys during development and CI/CD pipelines. Switch to a production key when going live.

---

## Base URL

```
https://nlnest.com/api/v1/
```

All endpoints in this documentation are relative to this base URL. Always use HTTPS — HTTP requests will be rejected.

---

## Sending the API Key

Pass your key in the `X-API-Key` request header:

```
X-API-Key: nlnest_live_your_key_here
```

### Example — Verify Authentication

```bash
curl https://nlnest.com/api/v1/jobs \
  -H "X-API-Key: nlnest_live_your_key_here"
```

A successful response returns HTTP `200` with a JSON body. An invalid or missing key returns HTTP `401`:

```json
{
  "success": false,
  "error": {
    "code": "unauthorized",
    "message": "Invalid or missing API key."
  }
}
```

---

## Security Best Practices

- **Never expose keys in client-side code** (JavaScript bundles, mobile apps). The API is designed for server-to-server use.
- **Store keys in environment variables**, not in source code or configuration files committed to version control.
- **Rotate keys regularly** — you can regenerate a key from the dashboard at any time. The old key is immediately invalidated.
- **Use one key per integration** — create separate keys for different environments (staging vs. production) or different services.

---

## Key Permissions

API keys inherit the permissions of the company account. They can:

- Create, read, update, and close all jobs belonging to the company
- Read all applications submitted to company jobs
- Update application pipeline status
- Create and manage webhooks
- Query analytics for company jobs

Keys cannot access data belonging to other companies, modify platform settings, or perform administrative actions.

---

## Next Steps

- [Create your first job →](jobs.md)
- [Set up a webhook →](webhooks.md)
- [Understand rate limits →](rate-limiting.md)

# Webhooks

Receive real-time push notifications when events happen in your [NLnest](https://nlnest.com) company account. Instead of polling the API, configure a webhook and NLnest will HTTP POST the event payload to your server within seconds.

---

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/webhooks` | Create a new webhook |
| `GET` | `/api/v1/webhooks` | List all webhooks |
| `PATCH` | `/api/v1/webhooks/{id}` | Update a webhook |
| `DELETE` | `/api/v1/webhooks/{id}` | Delete a webhook |

---

## POST /api/v1/webhooks — Create Webhook

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | Yes | Your HTTPS endpoint. Must use `https://`. |
| `events` | array | Yes | Array of event names to subscribe to. Use `["*"]` for all events. |
| `description` | string | No | Human-readable label for your own reference |

### Supported Events

| Event | Triggered when |
|-------|---------------|
| `application.created` | A candidate submits a new application to one of your jobs |
| `application.status_changed` | An application moves to a new pipeline status |
| `job.expired` | A job listing reaches its `expires_at` date and is automatically closed |
| `job.approved` | A pending job is approved by the platform and goes live |
| `message.received` | A candidate sends you a message via the platform messaging system |
| `*` | All events (wildcard — subscribe to everything) |

### Example Request

```bash
curl -X POST https://nlnest.com/api/v1/webhooks \
  -H "X-API-Key: nlnest_live_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ats.example.com/nlnest/webhook",
    "events": ["application.created", "application.status_changed"],
    "description": "ATS sync — new applications and status changes"
  }'
```

### Example Response

```json
{
  "success": true,
  "data": {
    "id": 55,
    "url": "https://your-ats.example.com/nlnest/webhook",
    "events": ["application.created", "application.status_changed"],
    "description": "ATS sync — new applications and status changes",
    "status": "active",
    "secret": "whsec_a3f8b2c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1",
    "created_at": "2026-03-13T10:00:00Z"
  }
}
```

> **Important**: The `secret` is shown **only once** at creation time. Store it securely — you will need it to verify incoming webhook signatures. It cannot be retrieved again.

---

## GET /api/v1/webhooks — List Webhooks

```bash
curl https://nlnest.com/api/v1/webhooks \
  -H "X-API-Key: nlnest_live_your_key_here"
```

```json
{
  "success": true,
  "data": [
    {
      "id": 55,
      "url": "https://your-ats.example.com/nlnest/webhook",
      "events": ["application.created", "application.status_changed"],
      "status": "active",
      "created_at": "2026-03-13T10:00:00Z",
      "last_triggered_at": "2026-03-13T14:37:00Z",
      "consecutive_failures": 0
    }
  ]
}
```

The `secret` is never returned in list or get responses — only at creation.

---

## PATCH /api/v1/webhooks/{id} — Update Webhook

Update the URL, event subscriptions, or enable/disable the webhook.

```bash
curl -X PATCH https://nlnest.com/api/v1/webhooks/55 \
  -H "X-API-Key: nlnest_live_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "events": ["*"],
    "status": "active"
  }'
```

---

## DELETE /api/v1/webhooks/{id} — Delete Webhook

```bash
curl -X DELETE https://nlnest.com/api/v1/webhooks/55 \
  -H "X-API-Key: nlnest_live_your_key_here"
```

```json
{
  "success": true,
  "data": {
    "id": 55,
    "deleted": true
  }
}
```

---

## Webhook Delivery

### Request Format

NLnest sends an HTTP `POST` to your endpoint with:

- `Content-Type: application/json`
- `X-Webhook-Signature: sha256=<hex>`
- `X-NLnest-Event: application.created`
- `X-NLnest-Delivery: <uuid>`

Your endpoint must return HTTP `2xx` within **10 seconds**. Any non-2xx response or timeout is treated as a failure.

### Example Payload — application.created

```json
{
  "event": "application.created",
  "delivery_id": "d3a4b5c6-e7f8-9012-abcd-ef1234567890",
  "timestamp": "2026-03-13T14:37:22Z",
  "data": {
    "application": {
      "id": 9341,
      "status": "new",
      "created_at": "2026-03-13T14:37:22Z"
    },
    "job": {
      "id": 18472,
      "title_en": "Warehouse Operative",
      "external_id": "WH-2026-042"
    },
    "candidate": {
      "id": 4812,
      "name": "Andrei Popescu",
      "email": "andrei.popescu@example.com",
      "nationality": "Romanian",
      "country_code": "ro",
      "cv_available": true
    }
  }
}
```

### Example Payload — application.status_changed

```json
{
  "event": "application.status_changed",
  "delivery_id": "e4b5c6d7-f8a9-0123-bcde-f12345678901",
  "timestamp": "2026-03-14T09:15:00Z",
  "data": {
    "application": {
      "id": 9341,
      "previous_status": "shortlisted",
      "status": "interview",
      "changed_by": "api"
    },
    "job": {
      "id": 18472,
      "title_en": "Warehouse Operative"
    },
    "candidate": {
      "id": 4812,
      "name": "Andrei Popescu"
    }
  }
}
```

---

## Signature Verification

Every webhook request includes an `X-Webhook-Signature` header containing an HMAC-SHA256 signature of the raw request body, using your webhook secret.

**Always verify the signature** before processing the payload to ensure the request is genuinely from NLnest.

### Verification — PHP

```php
function verify_nlnest_webhook(string $payload, string $signature_header, string $secret): bool {
    $expected = 'sha256=' . hash_hmac('sha256', $payload, $secret);
    return hash_equals($expected, $signature_header);
}

$payload   = file_get_contents('php://input');
$signature = $_SERVER['HTTP_X_WEBHOOK_SIGNATURE'] ?? '';
$secret    = 'whsec_a3f8b2c1d4e5f6a7b8c9...'; // your stored secret

if (!verify_nlnest_webhook($payload, $signature, $secret)) {
    http_response_code(401);
    exit('Invalid signature');
}

$event = json_decode($payload, true);
// process $event...
```

### Verification — Python

```python
import hmac
import hashlib

def verify_nlnest_webhook(payload: bytes, signature_header: str, secret: str) -> bool:
    expected = 'sha256=' + hmac.new(
        secret.encode(), payload, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature_header)
```

### Verification — Node.js

```javascript
const crypto = require('crypto');

function verifyNlnestWebhook(payload, signatureHeader, secret) {
  const expected = 'sha256=' + crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(expected),
    Buffer.from(signatureHeader)
  );
}
```

---

## Retry Policy

If your endpoint fails (non-2xx or timeout), NLnest retries with exponential backoff:

| Attempt | Delay after previous failure |
|---------|------------------------------|
| 1st retry | 1 minute |
| 2nd retry | 5 minutes |
| 3rd retry (final) | 30 minutes |

After 3 consecutive failures for a single delivery, the event is dropped.

**Auto-disable**: If a webhook accumulates **10 consecutive delivery failures** across any events, it is automatically set to `status: disabled`. You will receive an email notification. Re-enable it from the dashboard or via `PATCH /api/v1/webhooks/{id}` with `{"status": "active"}` once your endpoint is fixed.

---

## Best Practices

- Respond with `200 OK` immediately and process the payload asynchronously (e.g. push to a queue). This prevents timeouts.
- Use the `X-NLnest-Delivery` UUID for idempotency — your endpoint may receive duplicate deliveries during retries.
- Check the `timestamp` field to detect and discard replayed or delayed events.
- Store the webhook secret in an environment variable, never in source code.

---

## See Also

- [Authentication →](authentication.md)
- [Applications →](applications.md)
- [Errors →](errors.md)

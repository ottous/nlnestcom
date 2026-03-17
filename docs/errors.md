# Errors

The NLnest API uses standard HTTP status codes and returns a consistent JSON error structure. All error responses have `"success": false`.

---

## Error Response Format

```json
{
  "success": false,
  "error": {
    "code": "validation_error",
    "message": "The request contains invalid data.",
    "details": {
      "title_en": "This field is required.",
      "salary_min": "Must be a positive number."
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `code` | Machine-readable error identifier (see table below) |
| `message` | Human-readable explanation |
| `details` | Optional object with per-field validation messages (present on `validation_error` and `invalid_city`) |

---

## Error Code Reference

| HTTP Status | Code | Description |
|-------------|------|-------------|
| `400` | `invalid_json` | Request body is not valid JSON or `Content-Type` is missing |
| `401` | `unauthorized` | API key is missing, malformed, or does not exist |
| `403` | `forbidden` | The API key exists but does not have permission to access this resource (e.g. another company's data) |
| `404` | `not_found` | The requested resource does not exist |
| `405` | `method_not_allowed` | HTTP method is not supported on this endpoint |
| `422` | `validation_error` | Request body failed validation - check `details` for field-level messages |
| `422` | `invalid_city` | `city_name` was provided but did not match any Dutch city in the platform database |
| `429` | `rate_limit_exceeded` | You have exceeded your daily API quota - see [rate-limiting.md](rate-limiting.md) |
| `500` | `server_error` | Unexpected server-side error - please retry and contact support if persistent |

---

## Detailed Examples

### 401 - Unauthorized

```json
{
  "success": false,
  "error": {
    "code": "unauthorized",
    "message": "Invalid or missing API key. Pass your key in the X-API-Key header."
  }
}
```

**Common causes:**
- Key not included in the request header
- Key copied with extra whitespace
- Key was regenerated and the old value is still in use

### 403 - Forbidden

```json
{
  "success": false,
  "error": {
    "code": "forbidden",
    "message": "You do not have permission to access this resource."
  }
}
```

**Common causes:**
- Attempting to read or modify a job or application belonging to a different company
- Attempting to access an admin-only endpoint

### 404 - Not Found

```json
{
  "success": false,
  "error": {
    "code": "not_found",
    "message": "Job with ID 99999 does not exist."
  }
}
```

### 422 - Validation Error

```json
{
  "success": false,
  "error": {
    "code": "validation_error",
    "message": "The request contains invalid data.",
    "details": {
      "title_en": "This field is required.",
      "salary_min": "Must be a positive number.",
      "job_type": "Must be one of: full-time, part-time, contract, temporary, internship."
    }
  }
}
```

### 422 - Invalid City

```json
{
  "success": false,
  "error": {
    "code": "invalid_city",
    "message": "The city name could not be matched to a known Dutch city.",
    "details": {
      "city_name": "No match found for 'Amsterd'. Did you mean 'Amsterdam'?"
    }
  }
}
```

**Tip:** City matching is flexible (case-insensitive, accent-tolerant) but requires recognisable Dutch city names. Use the full official name (e.g. `"'s-Hertogenbosch"` not `"Den Bosch"`).

### 429 - Rate Limit Exceeded

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

See [rate-limiting.md](rate-limiting.md) for details on quota headers and limits.

### 500 - Server Error

```json
{
  "success": false,
  "error": {
    "code": "server_error",
    "message": "An unexpected error occurred. Please try again."
  }
}
```

If a `server_error` persists, contact support via [nlnest.com](https://nlnest.com) and include the request timestamp and endpoint.

---

## Troubleshooting Tips

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `401` on every request | Missing or wrong header name | Use `X-API-Key`, not `Authorization` or `Bearer` |
| `401` with correct key | Key was regenerated | Copy the latest key from the dashboard |
| `403` on a specific resource | Wrong company's data | Verify the job/application ID belongs to your account |
| `422` on city | Abbreviated city name | Use the full official Dutch city name |
| `429` unexpectedly | Burst of automated requests | Add pacing; check [rate-limiting.md](rate-limiting.md) |
| `500` intermittently | Transient server issue | Retry with exponential backoff; alert support if sustained |

---

## See Also

- [Authentication →](authentication.md)
- [Rate Limiting →](rate-limiting.md)

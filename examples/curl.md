# curl Examples

Complete curl command examples for every [NLnest](https://nlnest.com) API endpoint. Replace `nlnest_live_your_key_here` with your actual API key from the company dashboard.

---

## Setup

```bash
# Store your key in an environment variable to avoid repetition
export NLNEST_KEY="nlnest_live_your_key_here"
export NLNEST_API="https://nlnest.com/api/v1"
```

---

## Authentication Test

```bash
# Verify your key works - expect 200 with a list of your jobs
curl "$NLNEST_API/jobs" \
  -H "X-API-Key: $NLNEST_KEY"
```

---

## Jobs

### Create a Job (minimal)

```bash
curl -X POST "$NLNEST_API/jobs" \
  -H "X-API-Key: $NLNEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title_en": "Production Operative",
    "description_en": "We are looking for production operatives to work at our food processing facility in Rotterdam."
  }'
```

### Create a Job (full, with auto-translate)

```bash
curl -X POST "$NLNEST_API/jobs" \
  -H "X-API-Key: $NLNEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title_en": "Warehouse Operative",
    "description_en": "<p>We are looking for a reliable warehouse operative to join our team in Amsterdam.</p>",
    "requirements_en": "<ul><li>EU citizenship or work permit</li><li>Able to work shifts</li></ul>",
    "benefits_en": "<ul><li>Housing assistance available</li><li>Transport from Rotterdam</li></ul>",
    "sector_id": 4,
    "city_name": "Amsterdam",
    "job_type": "full-time",
    "experience_level": "entry",
    "salary_min": 2550,
    "salary_max": 3100,
    "salary_period": "month",
    "salary_visible": true,
    "positions": 3,
    "housing_provided": true,
    "transport_provided": true,
    "remote_option": "onsite",
    "expires_at": "2026-06-30",
    "external_id": "WH-2026-042",
    "auto_translate": true
  }'
```

### List Jobs

```bash
# All active jobs, page 1
curl "$NLNEST_API/jobs?status=active" \
  -H "X-API-Key: $NLNEST_KEY"
```

```bash
# With pagination
curl "$NLNEST_API/jobs?status=active&page=2&per_page=50" \
  -H "X-API-Key: $NLNEST_KEY"
```

```bash
# Filter by sector (sector 4 = Logistics & Warehousing)
curl "$NLNEST_API/jobs?sector_id=4" \
  -H "X-API-Key: $NLNEST_KEY"
```

### Get a Single Job

```bash
curl "$NLNEST_API/jobs/18472" \
  -H "X-API-Key: $NLNEST_KEY"
```

### Update a Job

```bash
# Update salary range
curl -X PATCH "$NLNEST_API/jobs/18472" \
  -H "X-API-Key: $NLNEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "salary_min": 2700,
    "salary_max": 3300
  }'
```

```bash
# Extend expiry date
curl -X PATCH "$NLNEST_API/jobs/18472" \
  -H "X-API-Key: $NLNEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "expires_at": "2026-09-30"
  }'
```

```bash
# Increase open positions
curl -X PATCH "$NLNEST_API/jobs/18472" \
  -H "X-API-Key: $NLNEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"positions": 5}'
```

### Close a Job

```bash
curl -X DELETE "$NLNEST_API/jobs/18472" \
  -H "X-API-Key: $NLNEST_KEY"
```

---

## Applications

### List All Applications

```bash
curl "$NLNEST_API/applications" \
  -H "X-API-Key: $NLNEST_KEY"
```

### List Applications for a Specific Job

```bash
curl "$NLNEST_API/applications?job_id=18472" \
  -H "X-API-Key: $NLNEST_KEY"
```

### Filter by Status and Date Range

```bash
curl "$NLNEST_API/applications?status=shortlisted&date_from=2026-03-01&date_to=2026-03-13" \
  -H "X-API-Key: $NLNEST_KEY"
```

### Get a Single Application

```bash
curl "$NLNEST_API/applications/9341" \
  -H "X-API-Key: $NLNEST_KEY"
```

### Update Application Status

```bash
# Move to interview stage
curl -X PATCH "$NLNEST_API/applications/9341" \
  -H "X-API-Key: $NLNEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "interview",
    "note": "Strong profile. Scheduling video interview."
  }'
```

```bash
# Mark as hired
curl -X PATCH "$NLNEST_API/applications/9341" \
  -H "X-API-Key: $NLNEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status": "hired", "note": "Offer accepted. Starting 1 April."}'
```

```bash
# Reject with note
curl -X PATCH "$NLNEST_API/applications/9341" \
  -H "X-API-Key: $NLNEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status": "rejected", "note": "Position filled internally."}'
```

### Download CV

```bash
# Save to file
curl "$NLNEST_API/applications/9341/cv" \
  -H "X-API-Key: $NLNEST_KEY" \
  --output candidate_9341_cv.pdf
```

---

## Webhooks

### Create a Webhook

```bash
curl -X POST "$NLNEST_API/webhooks" \
  -H "X-API-Key: $NLNEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ats.example.com/nlnest/webhook",
    "events": ["application.created", "application.status_changed"],
    "description": "ATS sync webhook"
  }'
```

```bash
# Subscribe to all events
curl -X POST "$NLNEST_API/webhooks" \
  -H "X-API-Key: $NLNEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-ats.example.com/nlnest/webhook",
    "events": ["*"]
  }'
```

### List Webhooks

```bash
curl "$NLNEST_API/webhooks" \
  -H "X-API-Key: $NLNEST_KEY"
```

### Update a Webhook

```bash
curl -X PATCH "$NLNEST_API/webhooks/55" \
  -H "X-API-Key: $NLNEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"events": ["*"], "status": "active"}'
```

### Delete a Webhook

```bash
curl -X DELETE "$NLNEST_API/webhooks/55" \
  -H "X-API-Key: $NLNEST_KEY"
```

---

## Analytics

### Job Analytics

```bash
curl "$NLNEST_API/analytics/jobs/18472" \
  -H "X-API-Key: $NLNEST_KEY"
```

```bash
# With custom date range
curl "$NLNEST_API/analytics/jobs/18472?date_from=2026-03-01&date_to=2026-03-13" \
  -H "X-API-Key: $NLNEST_KEY"
```

### Company Overview

```bash
curl "$NLNEST_API/analytics/overview" \
  -H "X-API-Key: $NLNEST_KEY"
```

```bash
# Q1 2026
curl "$NLNEST_API/analytics/overview?date_from=2026-01-01&date_to=2026-03-31" \
  -H "X-API-Key: $NLNEST_KEY"
```

---

## Inspecting Rate Limit Headers

```bash
# Use -I (HEAD equivalent) or -i (include headers in output) to inspect quota
curl -i "$NLNEST_API/jobs" \
  -H "X-API-Key: $NLNEST_KEY" | grep -i "X-RateLimit"
```

---

## See Also

- [PHP example →](php.md)
- [Python example →](python.md)
- [Node.js example →](nodejs.md)
- [Full documentation →](../docs/authentication.md)

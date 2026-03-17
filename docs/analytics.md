# Analytics

Query job performance data and company-wide hiring metrics from [NLnest](https://nlnest.com). Use these endpoints to build dashboards, track sourcing effectiveness, and measure recruiting KPIs in your own BI tools.

---

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/analytics/jobs/{id}` | Performance data for a single job |
| `GET` | `/api/v1/analytics/overview` | Company-wide hiring overview |

---

## GET /api/v1/analytics/jobs/{id} - Job Analytics

Returns view counts, application volume, and conversion rate for a specific job listing.

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `date_from` | string | 30 days ago | ISO 8601 date (e.g. `2026-01-01`) - start of reporting window |
| `date_to` | string | today | ISO 8601 date - end of reporting window |

### Example Request

```bash
curl "https://nlnest.com/api/v1/analytics/jobs/18472?date_from=2026-03-01&date_to=2026-03-13" \
  -H "X-API-Key: nlnest_live_your_key_here"
```

### Example Response

```json
{
  "success": true,
  "data": {
    "job_id": 18472,
    "job_title_en": "Warehouse Operative",
    "date_from": "2026-03-01",
    "date_to": "2026-03-13",
    "views": 238,
    "unique_views": 194,
    "applications": 14,
    "conversion_rate": 7.2,
    "applications_by_status": {
      "new": 3,
      "reviewed": 4,
      "shortlisted": 3,
      "interview": 2,
      "offered": 1,
      "hired": 1,
      "rejected": 0
    },
    "applications_by_source": {
      "organic": 9,
      "alert": 3,
      "referral": 1,
      "feed": 1
    },
    "applications_by_country": {
      "ro": 6,
      "pl": 3,
      "hu": 2,
      "bg": 2,
      "nl": 1
    },
    "daily_views": [
      { "date": "2026-03-01", "views": 12 },
      { "date": "2026-03-02", "views": 19 },
      { "date": "2026-03-03", "views": 22 }
    ],
    "daily_applications": [
      { "date": "2026-03-01", "applications": 1 },
      { "date": "2026-03-02", "applications": 2 }
    ]
  }
}
```

### Field Definitions

| Field | Description |
|-------|-------------|
| `views` | Total page views (includes repeat visits) |
| `unique_views` | Unique visitors (deduplicated by IP + session) |
| `applications` | Total applications submitted in the date range |
| `conversion_rate` | `(applications / unique_views) × 100`, as a percentage |
| `applications_by_status` | Breakdown of all current application statuses |
| `applications_by_source` | How candidates discovered the job |
| `applications_by_country` | Country codes of applying candidates |
| `daily_views` | Day-by-day view data for charting |
| `daily_applications` | Day-by-day application data for charting |

---

## GET /api/v1/analytics/overview - Company Overview

Returns aggregated hiring metrics across your entire company account.

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `date_from` | string | 30 days ago | Start of reporting window |
| `date_to` | string | today | End of reporting window |

### Example Request

```bash
curl "https://nlnest.com/api/v1/analytics/overview?date_from=2026-01-01&date_to=2026-03-13" \
  -H "X-API-Key: nlnest_live_your_key_here"
```

### Example Response

```json
{
  "success": true,
  "data": {
    "date_from": "2026-01-01",
    "date_to": "2026-03-13",
    "active_jobs": 7,
    "total_jobs": 23,
    "total_views": 4821,
    "total_applications": 186,
    "total_hired": 12,
    "avg_time_to_hire_days": 18.4,
    "avg_applications_per_job": 8.1,
    "overall_conversion_rate": 3.9,
    "applications_by_status": {
      "new": 24,
      "reviewed": 31,
      "shortlisted": 48,
      "interview": 29,
      "offered": 18,
      "hired": 12,
      "rejected": 22,
      "withdrawn": 2
    },
    "top_jobs_by_applications": [
      {
        "job_id": 18472,
        "title_en": "Warehouse Operative",
        "applications": 14,
        "hired": 2
      },
      {
        "job_id": 18105,
        "title_en": "Food Production Operative",
        "applications": 11,
        "hired": 1
      }
    ],
    "applications_by_country": {
      "ro": 72,
      "pl": 41,
      "hu": 28,
      "bg": 19,
      "el": 14,
      "uk": 12
    },
    "applications_by_source": {
      "organic": 118,
      "alert": 42,
      "referral": 15,
      "feed": 11
    }
  }
}
```

### Field Definitions

| Field | Description |
|-------|-------------|
| `active_jobs` | Jobs currently in `active` status |
| `total_jobs` | All jobs (any status) created within the date range |
| `total_hired` | Applications that reached `hired` status |
| `avg_time_to_hire_days` | Average days from application creation to `hired` status |
| `avg_applications_per_job` | Total applications divided by total active jobs |
| `overall_conversion_rate` | `(total_applications / total_views) × 100` |
| `top_jobs_by_applications` | Top 5 jobs ranked by application count |

---

## Notes

- Analytics data is updated every **15 minutes**.
- The `date_from` / `date_to` range is inclusive on both ends.
- `avg_time_to_hire_days` only counts applications that reached `hired` status - it excludes ongoing pipelines.
- Maximum date range for a single request is **365 days**.

---

## See Also

- [Jobs →](jobs.md)
- [Applications →](applications.md)
- [Rate Limiting →](rate-limiting.md)

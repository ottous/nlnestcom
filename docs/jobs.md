# Jobs

Manage job listings programmatically. Create multilingual postings, update details, and close positions - all without touching the [NLnest](https://nlnest.com) dashboard.

---

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/jobs` | Create a new job |
| `GET` | `/api/v1/jobs` | List all jobs (paginated) |
| `GET` | `/api/v1/jobs/{id}` | Get a single job |
| `PATCH` | `/api/v1/jobs/{id}` | Partially update a job |
| `DELETE` | `/api/v1/jobs/{id}` | Close a job |

---

## POST /api/v1/jobs - Create Job

Creates a new job listing. The job will be published automatically if your company account has auto-publish enabled, or saved as a `draft` for manual review. **Test keys always create drafts.**

### Request Body

**Required fields:**

| Field | Type | Description |
|-------|------|-------------|
| `title_en` | string | Job title in English (max 255 chars) |
| `description_en` | string | Full job description in English (HTML allowed) |

**Multilingual fields (optional):**

Provide translations for any of the 10 additional languages. When `auto_translate` is `true`, NLnest will fill in any missing languages automatically.

| Field | Languages available |
|-------|-------------------|
| `title_{lang}` | `ro`, `nl`, `hu`, `pl`, `es`, `pt`, `lt`, `bg`, `el`, `uk` |
| `description_{lang}` | `ro`, `nl`, `hu`, `pl`, `es`, `pt`, `lt`, `bg`, `el`, `uk` |
| `requirements_{lang}` | `en` + all 10 languages |
| `benefits_{lang}` | `en` + all 10 languages |

**Job details (optional):**

| Field | Type | Description |
|-------|------|-------------|
| `sector_id` | integer | Industry sector ID (1–21). See sector list below. |
| `city_name` | string | Dutch city name (e.g. `"Amsterdam"`, `"Rotterdam"`) |
| `job_type` | string | `full-time` \| `part-time` \| `contract` \| `temporary` \| `internship` |
| `experience_level` | string | `entry` \| `mid` \| `senior` \| `executive` |
| `salary_min` | number | Minimum salary (numeric, in EUR) |
| `salary_max` | number | Maximum salary (numeric, in EUR) |
| `salary_period` | string | `hour` \| `month` \| `year` |
| `salary_visible` | boolean | Show salary publicly (default: `true`) |
| `positions` | integer | Number of open positions (default: `1`) |
| `housing_provided` | boolean | Employer provides housing (default: `false`) |
| `transport_provided` | boolean | Employer provides transport (default: `false`) |
| `remote_option` | string | `onsite` \| `hybrid` \| `remote` (default: `onsite`) |
| `expires_at` | string | ISO 8601 date when the listing closes (e.g. `"2026-06-30"`) |

**Integration fields (optional):**

| Field | Type | Description |
|-------|------|-------------|
| `external_id` | string | Your internal job ID for cross-referencing (max 100 chars) |
| `external_url` | string | Link to the job in your own ATS |
| `auto_translate` | boolean | When `true`, NLnest queues automatic translation to all 11 languages (default: `false`) |

### Sector IDs

| ID | Sector |
|----|--------|
| 1 | Agriculture & Food Production |
| 2 | Construction & Trades |
| 3 | Cleaning & Facilities |
| 4 | Logistics & Warehousing |
| 5 | Transport & Driving |
| 6 | Manufacturing & Production |
| 7 | Healthcare & Nursing |
| 8 | Hospitality & Tourism |
| 9 | Retail & Sales |
| 10 | IT & Technology |
| 11 | Finance & Accounting |
| 12 | Administration & Office |
| 13 | Education & Training |
| 14 | Customer Service |
| 15 | Engineering |
| 16 | Marketing & Communications |
| 17 | Legal & Compliance |
| 18 | HR & Recruitment |
| 19 | Real Estate |
| 20 | Energy & Environment |
| 21 | Other |

### Example Request

```bash
curl -X POST https://nlnest.com/api/v1/jobs \
  -H "X-API-Key: nlnest_live_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "title_en": "Warehouse Operative",
    "description_en": "<p>We are looking for a reliable warehouse operative to join our team in Amsterdam.</p><p>You will be responsible for picking, packing, and dispatching orders in a fast-paced environment.</p>",
    "requirements_en": "<ul><li>EU work permit or citizenship</li><li>Ability to lift up to 20kg</li><li>Forklift licence is a plus</li></ul>",
    "benefits_en": "<ul><li>Housing assistance available</li><li>Transport provided from Rotterdam</li><li>Performance bonuses</li></ul>",
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

### Example Response

```json
{
  "success": true,
  "data": {
    "id": 18472,
    "external_id": "WH-2026-042",
    "status": "active",
    "title_en": "Warehouse Operative",
    "city_name": "Amsterdam",
    "job_type": "full-time",
    "salary_min": 2550,
    "salary_max": 3100,
    "salary_period": "month",
    "auto_translate": true,
    "translate_status": "queued",
    "created_at": "2026-03-13T10:45:00Z",
    "expires_at": "2026-06-30T23:59:59Z",
    "url": "https://nlnest.com/en/job/warehouse-operative-amsterdam-a1b2c3"
  }
}
```

> **Note on auto_translate**: When `auto_translate` is `true`, `translate_status` will be `queued` initially. Translation typically completes within a few minutes. Translated slugs and titles become available in subsequent GET requests.

---

## GET /api/v1/jobs - List Jobs

Returns a paginated list of all jobs belonging to your company.

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | integer | `1` | Page number |
| `per_page` | integer | `20` | Results per page (max `100`) |
| `status` | string | - | Filter by status: `active`, `draft`, `closed`, `expired` |
| `sector_id` | integer | - | Filter by sector |

### Example Request

```bash
curl "https://nlnest.com/api/v1/jobs?status=active&page=1&per_page=20" \
  -H "X-API-Key: nlnest_live_your_key_here"
```

### Example Response

```json
{
  "success": true,
  "data": [
    {
      "id": 18472,
      "title_en": "Warehouse Operative",
      "status": "active",
      "city_name": "Amsterdam",
      "job_type": "full-time",
      "applications_count": 14,
      "created_at": "2026-03-13T10:45:00Z",
      "expires_at": "2026-06-30T23:59:59Z",
      "url": "https://nlnest.com/en/job/warehouse-operative-amsterdam-a1b2c3"
    }
  ],
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 1,
    "total_pages": 1
  }
}
```

---

## GET /api/v1/jobs/{id} - Get Job

Returns full details for a single job, including all available translations.

### Example Request

```bash
curl https://nlnest.com/api/v1/jobs/18472 \
  -H "X-API-Key: nlnest_live_your_key_here"
```

### Example Response

```json
{
  "success": true,
  "data": {
    "id": 18472,
    "external_id": "WH-2026-042",
    "status": "active",
    "title_en": "Warehouse Operative",
    "title_ro": "Operator depozit",
    "title_nl": "Magazijnmedewerker",
    "title_pl": "Pracownik magazynu",
    "description_en": "<p>We are looking for...</p>",
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
    "auto_translate": true,
    "translate_status": "done",
    "applications_count": 14,
    "views_count": 238,
    "created_at": "2026-03-13T10:45:00Z",
    "expires_at": "2026-06-30T23:59:59Z",
    "url": "https://nlnest.com/en/job/warehouse-operative-amsterdam-a1b2c3"
  }
}
```

---

## PATCH /api/v1/jobs/{id} - Update Job

Partially update a job. Send only the fields you want to change - all other fields remain unchanged.

### Example Request - Update Salary

```bash
curl -X PATCH https://nlnest.com/api/v1/jobs/18472 \
  -H "X-API-Key: nlnest_live_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "salary_min": 2700,
    "salary_max": 3300
  }'
```

### Example Response

```json
{
  "success": true,
  "data": {
    "id": 18472,
    "salary_min": 2700,
    "salary_max": 3300,
    "updated_at": "2026-03-14T08:30:00Z"
  }
}
```

---

## DELETE /api/v1/jobs/{id} - Close Job

Closes a job listing. This is a soft delete - the record is preserved for analytics and application history. Closed jobs no longer appear in public search results.

### Example Request

```bash
curl -X DELETE https://nlnest.com/api/v1/jobs/18472 \
  -H "X-API-Key: nlnest_live_your_key_here"
```

### Example Response

```json
{
  "success": true,
  "data": {
    "id": 18472,
    "status": "closed",
    "closed_at": "2026-03-14T09:00:00Z"
  }
}
```

---

## Job Status Reference

| Status | Description |
|--------|-------------|
| `draft` | Not visible publicly; awaiting manual publish or approval |
| `pending_approval` | Submitted for admin review (if company requires approval) |
| `active` | Live on [nlnest.com](https://nlnest.com), visible to candidates |
| `closed` | Manually closed via API or dashboard |
| `expired` | Automatically closed after `expires_at` date |

---

## See Also

- [Applications →](applications.md)
- [Analytics →](analytics.md)
- [Webhooks →](webhooks.md)

# Applications

Read and manage candidate applications submitted to your job listings on [NLnest](https://nlnest.com). Move candidates through your hiring pipeline, download CVs, and sync status changes back to your own ATS.

---

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/applications` | List applications (paginated) |
| `GET` | `/api/v1/applications/{id}` | Get application details |
| `PATCH` | `/api/v1/applications/{id}` | Update application status |
| `GET` | `/api/v1/applications/{id}/cv` | Download candidate CV |

---

## GET /api/v1/applications — List Applications

Returns a paginated list of applications across all your company's jobs, or filtered by job.

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | integer | `1` | Page number |
| `per_page` | integer | `20` | Results per page (max `100`) |
| `job_id` | integer | — | Filter by specific job ID |
| `status` | string | — | Filter by pipeline status (see status reference below) |
| `date_from` | string | — | ISO 8601 date — only return applications submitted on or after this date |
| `date_to` | string | — | ISO 8601 date — only return applications submitted on or before this date |

### Example Request

```bash
curl "https://nlnest.com/api/v1/applications?job_id=18472&status=shortlisted" \
  -H "X-API-Key: nlnest_live_your_key_here"
```

### Example Response

```json
{
  "success": true,
  "data": [
    {
      "id": 9341,
      "job_id": 18472,
      "job_title_en": "Warehouse Operative",
      "status": "shortlisted",
      "candidate": {
        "id": 4812,
        "name": "Andrei Popescu",
        "email": "andrei.popescu@example.com",
        "phone": "+40712345678",
        "nationality": "Romanian",
        "country_code": "ro",
        "city": "Cluj-Napoca",
        "languages": ["ro", "en"],
        "cv_available": true
      },
      "match_score": 87,
      "source": "organic",
      "created_at": "2026-03-10T14:22:00Z",
      "updated_at": "2026-03-12T09:15:00Z"
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

## GET /api/v1/applications/{id} — Get Application

Returns full details for a single application, including candidate profile, cover letter, screening answers, and CV availability.

### Example Request

```bash
curl https://nlnest.com/api/v1/applications/9341 \
  -H "X-API-Key: nlnest_live_your_key_here"
```

### Example Response

```json
{
  "success": true,
  "data": {
    "id": 9341,
    "job_id": 18472,
    "job_title_en": "Warehouse Operative",
    "status": "shortlisted",
    "priority": "high",
    "pipeline_stage": "Phone Screen",
    "candidate": {
      "id": 4812,
      "name": "Andrei Popescu",
      "email": "andrei.popescu@example.com",
      "phone": "+40712345678",
      "nationality": "Romanian",
      "country_code": "ro",
      "city": "Cluj-Napoca",
      "date_of_birth": "1994-07-15",
      "languages": ["ro", "en", "nl"],
      "education": "Vocational Certificate",
      "experience_years": 4,
      "linkedin_url": "https://linkedin.com/in/andrei-popescu",
      "cv_available": true,
      "cv_url": "https://nlnest.com/api/v1/applications/9341/cv"
    },
    "cover_letter": "I am very interested in this position and have 4 years of experience in warehouse logistics...",
    "screening_answers": [
      {
        "question": "Do you have a valid EU driving licence?",
        "answer": "Yes, category B and C"
      },
      {
        "question": "Are you available to start within 4 weeks?",
        "answer": "Yes"
      }
    ],
    "match_score": 87,
    "source": "organic",
    "tags": ["experienced", "dutch-speaker"],
    "avg_rating": 4.5,
    "interviews": [
      {
        "id": 201,
        "type": "video",
        "scheduled_at": "2026-03-15T11:00:00Z",
        "status": "scheduled"
      }
    ],
    "created_at": "2026-03-10T14:22:00Z",
    "updated_at": "2026-03-12T09:15:00Z"
  }
}
```

---

## PATCH /api/v1/applications/{id} — Update Status

Move a candidate to a different stage in the hiring pipeline. This is the primary way to sync your external ATS decisions back to NLnest.

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `status` | string | Yes | New pipeline status (see reference below) |
| `note` | string | No | Internal note recorded in the candidate's activity log (not visible to candidate) |

### Pipeline Status Values

| Status | Description |
|--------|-------------|
| `new` | Just applied, not yet reviewed |
| `reviewed` | Recruiter has read the application |
| `shortlisted` | Candidate moved to shortlist |
| `interview` | Interview scheduled or in progress |
| `offered` | Offer extended to candidate |
| `hired` | Candidate accepted offer |
| `rejected` | Application declined |
| `withdrawn` | Candidate withdrew their application |

### Example Request

```bash
curl -X PATCH https://nlnest.com/api/v1/applications/9341 \
  -H "X-API-Key: nlnest_live_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "interview",
    "note": "Strong CV. Phone screen went well. Scheduling video interview for next week."
  }'
```

### Example Response

```json
{
  "success": true,
  "data": {
    "id": 9341,
    "status": "interview",
    "previous_status": "shortlisted",
    "note": "Strong CV. Phone screen went well. Scheduling video interview for next week.",
    "updated_at": "2026-03-13T16:00:00Z"
  }
}
```

> **Webhook tip**: Configure the `application.status_changed` webhook event to receive real-time notifications whenever a status changes — either via API or from within the NLnest dashboard. See [webhooks.md](webhooks.md).

---

## GET /api/v1/applications/{id}/cv — Download CV

Returns the candidate's CV file. The response is the raw file with the appropriate `Content-Type` header (`application/pdf`, `application/msword`, etc.).

### Example Request

```bash
curl https://nlnest.com/api/v1/applications/9341/cv \
  -H "X-API-Key: nlnest_live_your_key_here" \
  --output candidate_cv.pdf
```

### Notes

- Returns HTTP `404` with `{"success": false, "error": {"code": "not_found", "message": "No CV on file for this candidate."}}` if no CV was uploaded.
- CV files are stored securely and are only accessible by your company's API key.
- Each CV download is logged in the candidate's activity timeline on [nlnest.com](https://nlnest.com).

---

## Application Source Reference

The `source` field indicates how the candidate discovered the job:

| Source | Description |
|--------|-------------|
| `organic` | Found the job directly on nlnest.com |
| `alert` | Matched via a job alert subscription |
| `referral` | Referred by another candidate |
| `feed` | Imported from an external job feed |
| `api` | Applied via a company's custom application integration |
| `linkedin` | Via LinkedIn job slot |
| `direct` | Deep link / direct URL share |

---

## See Also

- [Jobs →](jobs.md)
- [Webhooks — application events →](webhooks.md)
- [Analytics →](analytics.md)

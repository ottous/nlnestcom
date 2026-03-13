# Python Integration Example

A ready-to-use Python client for the [NLnest](https://nlnest.com) Employer API. Requires Python 3.8+ and the `requests` library (`pip install requests`).

---

## NLnestClient Class

```python
"""
NLnest Employer API Client — Python
https://nlnest.com

A clean requests-based wrapper for the NLnest API v1.

Install dependency:
    pip install requests

Usage:
    from nlnest import NLnestClient
    client = NLnestClient(api_key=os.environ['NLNEST_API_KEY'])
    jobs = client.list_jobs(status='active')
"""

import hashlib
import hmac
import os
from typing import Any, Dict, List, Optional
from urllib.parse import urljoin

import requests


class NLnestAPIError(Exception):
    """Raised when the NLnest API returns an error response."""

    def __init__(self, code: str, message: str, details: Optional[dict] = None):
        super().__init__(f"[{code}] {message}")
        self.code = code
        self.details = details or {}


class NLnestClient:
    """
    Client for the NLnest Employer API v1.

    Args:
        api_key: Your NLnest API key (production: nlnest_live_*, test: nlnest_test_*)
        timeout:  Request timeout in seconds (default: 30)
    """

    BASE_URL = "https://nlnest.com/api/v1/"

    def __init__(self, api_key: str, timeout: int = 30):
        if not api_key:
            raise ValueError("NLnest API key is required.")
        self.api_key = api_key
        self.timeout = timeout
        self.session = requests.Session()
        self.session.headers.update({
            "X-API-Key": self.api_key,
            "Accept": "application/json",
            "Content-Type": "application/json",
        })

    # -------------------------------------------------------------------------
    # JOBS
    # -------------------------------------------------------------------------

    def create_job(self, **kwargs) -> Dict[str, Any]:
        """
        Create a new job listing.

        Keyword args map directly to the API request body fields.
        See docs/jobs.md for the full field reference.

        Example:
            job = client.create_job(
                title_en="Warehouse Operative",
                description_en="<p>Join our team in Amsterdam.</p>",
                sector_id=4,
                city_name="Amsterdam",
                job_type="full-time",
                salary_min=2550,
                salary_max=3100,
                salary_period="month",
                positions=3,
                housing_provided=True,
                auto_translate=True,
            )
        """
        return self._request("POST", "jobs", json=kwargs)

    def list_jobs(
        self,
        status: Optional[str] = None,
        sector_id: Optional[int] = None,
        page: int = 1,
        per_page: int = 20,
    ) -> Dict[str, Any]:
        """
        List all company jobs with optional filters.

        Args:
            status:    Filter by status: 'active', 'draft', 'closed', 'expired'
            sector_id: Filter by sector ID (1–21)
            page:      Page number (default 1)
            per_page:  Results per page, max 100 (default 20)

        Returns:
            Dict with 'data' (list) and 'meta' (pagination info)
        """
        params = {"page": page, "per_page": per_page}
        if status:
            params["status"] = status
        if sector_id is not None:
            params["sector_id"] = sector_id
        return self._request("GET", "jobs", params=params)

    def get_job(self, job_id: int) -> Dict[str, Any]:
        """Get a single job by ID."""
        return self._request("GET", f"jobs/{job_id}")

    def update_job(self, job_id: int, **kwargs) -> Dict[str, Any]:
        """
        Partially update a job. Pass only the fields you want to change.

        Example:
            client.update_job(18472, salary_min=2700, salary_max=3300)
        """
        return self._request("PATCH", f"jobs/{job_id}", json=kwargs)

    def close_job(self, job_id: int) -> Dict[str, Any]:
        """Close (soft-delete) a job listing."""
        return self._request("DELETE", f"jobs/{job_id}")

    # -------------------------------------------------------------------------
    # APPLICATIONS
    # -------------------------------------------------------------------------

    def list_applications(
        self,
        job_id: Optional[int] = None,
        status: Optional[str] = None,
        date_from: Optional[str] = None,
        date_to: Optional[str] = None,
        page: int = 1,
        per_page: int = 20,
    ) -> Dict[str, Any]:
        """
        List applications with optional filters.

        Args:
            job_id:    Filter by job ID
            status:    Filter by pipeline status
            date_from: ISO 8601 date string (e.g. '2026-03-01')
            date_to:   ISO 8601 date string
        """
        params: Dict[str, Any] = {"page": page, "per_page": per_page}
        if job_id is not None:
            params["job_id"] = job_id
        if status:
            params["status"] = status
        if date_from:
            params["date_from"] = date_from
        if date_to:
            params["date_to"] = date_to
        return self._request("GET", "applications", params=params)

    def get_application(self, application_id: int) -> Dict[str, Any]:
        """Get full details for a single application."""
        return self._request("GET", f"applications/{application_id}")

    def update_application_status(
        self,
        application_id: int,
        status: str,
        note: Optional[str] = None,
    ) -> Dict[str, Any]:
        """
        Update the pipeline status of an application.

        Args:
            application_id: The application ID
            status:         One of: new, reviewed, shortlisted, interview,
                            offered, hired, rejected, withdrawn
            note:           Optional internal note (not visible to candidate)
        """
        body: Dict[str, Any] = {"status": status}
        if note:
            body["note"] = note
        return self._request("PATCH", f"applications/{application_id}", json=body)

    def download_cv(self, application_id: int, save_path: str) -> str:
        """
        Download a candidate CV and save it to a local file.

        Args:
            application_id: The application ID
            save_path:      Local file path to write the CV to

        Returns:
            The save_path string

        Raises:
            NLnestAPIError: If no CV is available
        """
        url = urljoin(self.BASE_URL, f"applications/{application_id}/cv")
        response = self.session.get(url, timeout=self.timeout, stream=True)

        if response.status_code == 404:
            raise NLnestAPIError("not_found", f"No CV on file for application {application_id}.")
        response.raise_for_status()

        with open(save_path, "wb") as fh:
            for chunk in response.iter_content(chunk_size=8192):
                fh.write(chunk)

        return save_path

    # -------------------------------------------------------------------------
    # WEBHOOKS
    # -------------------------------------------------------------------------

    def create_webhook(
        self,
        url: str,
        events: List[str],
        description: str = "",
    ) -> Dict[str, Any]:
        """
        Create a new webhook subscription.

        Args:
            url:         Your HTTPS endpoint
            events:      List of event names, or ['*'] for all events
            description: Optional human-readable label

        Returns:
            Response including 'secret' — store it securely, shown only once!
        """
        return self._request("POST", "webhooks", json={
            "url": url,
            "events": events,
            "description": description,
        })

    def list_webhooks(self) -> Dict[str, Any]:
        """List all webhooks."""
        return self._request("GET", "webhooks")

    def update_webhook(self, webhook_id: int, **kwargs) -> Dict[str, Any]:
        """Update a webhook."""
        return self._request("PATCH", f"webhooks/{webhook_id}", json=kwargs)

    def delete_webhook(self, webhook_id: int) -> Dict[str, Any]:
        """Delete a webhook."""
        return self._request("DELETE", f"webhooks/{webhook_id}")

    # -------------------------------------------------------------------------
    # ANALYTICS
    # -------------------------------------------------------------------------

    def get_job_analytics(
        self,
        job_id: int,
        date_from: Optional[str] = None,
        date_to: Optional[str] = None,
    ) -> Dict[str, Any]:
        """Get performance analytics for a single job."""
        params = {}
        if date_from:
            params["date_from"] = date_from
        if date_to:
            params["date_to"] = date_to
        return self._request("GET", f"analytics/jobs/{job_id}", params=params)

    def get_overview(
        self,
        date_from: Optional[str] = None,
        date_to: Optional[str] = None,
    ) -> Dict[str, Any]:
        """Get company-wide hiring overview."""
        params = {}
        if date_from:
            params["date_from"] = date_from
        if date_to:
            params["date_to"] = date_to
        return self._request("GET", "analytics/overview", params=params)

    # -------------------------------------------------------------------------
    # WEBHOOK SIGNATURE VERIFICATION
    # -------------------------------------------------------------------------

    @staticmethod
    def verify_webhook_signature(payload: bytes, signature_header: str, secret: str) -> bool:
        """
        Verify an incoming webhook signature using HMAC-SHA256.

        Use this in your webhook endpoint before processing the payload.

        Args:
            payload:          Raw request body bytes
            signature_header: Value of the X-Webhook-Signature header
            secret:           Your webhook secret (stored at creation time)

        Returns:
            True if the signature is valid, False otherwise

        Example:
            from flask import Flask, request
            app = Flask(__name__)

            @app.route('/webhook', methods=['POST'])
            def webhook():
                payload   = request.get_data()
                signature = request.headers.get('X-Webhook-Signature', '')
                if not NLnestClient.verify_webhook_signature(payload, signature, WEBHOOK_SECRET):
                    return 'Unauthorized', 401
                event = request.get_json()
                # process event...
                return '', 200
        """
        expected = "sha256=" + hmac.new(
            secret.encode(), payload, hashlib.sha256
        ).hexdigest()
        return hmac.compare_digest(expected, signature_header)

    # -------------------------------------------------------------------------
    # HTTP TRANSPORT
    # -------------------------------------------------------------------------

    def _request(
        self,
        method: str,
        path: str,
        json: Optional[dict] = None,
        params: Optional[dict] = None,
    ) -> Dict[str, Any]:
        """Internal method to make authenticated API requests."""
        url = urljoin(self.BASE_URL, path)

        response = self.session.request(
            method=method,
            url=url,
            json=json,
            params=params,
            timeout=self.timeout,
        )

        try:
            data = response.json()
        except ValueError:
            response.raise_for_status()
            raise NLnestAPIError("parse_error", f"Invalid JSON response (HTTP {response.status_code})")

        if not data.get("success", True):
            error = data.get("error", {})
            raise NLnestAPIError(
                code=error.get("code", "unknown"),
                message=error.get("message", "Unknown error"),
                details=error.get("details"),
            )

        return data
```

---

## Usage Examples

```python
import os
from nlnest import NLnestClient, NLnestAPIError

client = NLnestClient(api_key=os.environ["NLNEST_API_KEY"])

# --- Create a job ---
try:
    job = client.create_job(
        title_en="Food Production Operative",
        description_en="<p>Join our food processing team in Utrecht.</p>",
        sector_id=1,
        city_name="Utrecht",
        job_type="full-time",
        experience_level="entry",
        salary_min=2550,
        salary_max=2900,
        salary_period="month",
        positions=5,
        housing_provided=True,
        transport_provided=True,
        expires_at="2026-08-31",
        external_id="FP-2026-019",
        auto_translate=True,
    )
    print(f"Job created: {job['data']['id']} → {job['data']['url']}")
except NLnestAPIError as e:
    print(f"Failed to create job: {e}")

# --- List active jobs ---
result = client.list_jobs(status="active", per_page=50)
for j in result["data"]:
    print(f"[{j['id']}] {j['title_en']} — {j['applications_count']} applications")

# --- Process new applications ---
apps = client.list_applications(job_id=job["data"]["id"], status="new")
for app in apps["data"]:
    candidate = app["candidate"]
    print(f"New: {candidate['name']} from {candidate['nationality']}")
    client.update_application_status(app["id"], "reviewed", "Initial review done.")

# --- Download a CV ---
saved = client.download_cv(9341, "/tmp/cv_9341.pdf")
print(f"CV saved to {saved}")

# --- Analytics ---
stats = client.get_job_analytics(job["data"]["id"], "2026-03-01", "2026-03-31")
d = stats["data"]
print(f"Views: {d['views']}, Applications: {d['applications']}, "
      f"Conversion: {d['conversion_rate']}%")

# --- Overview ---
overview = client.get_overview("2026-01-01", "2026-03-31")
d = overview["data"]
print(f"Active jobs: {d['active_jobs']}, Total hired: {d['total_hired']}, "
      f"Avg time to hire: {d['avg_time_to_hire_days']} days")
```

---

## Flask Webhook Handler Example

```python
import os
import json
from flask import Flask, request, abort
from nlnest import NLnestClient

app = Flask(__name__)
WEBHOOK_SECRET = os.environ["NLNEST_WEBHOOK_SECRET"]


@app.route("/webhooks/nlnest", methods=["POST"])
def nlnest_webhook():
    payload   = request.get_data()
    signature = request.headers.get("X-Webhook-Signature", "")

    if not NLnestClient.verify_webhook_signature(payload, signature, WEBHOOK_SECRET):
        abort(401)

    event = request.get_json()

    if event["event"] == "application.created":
        candidate = event["data"]["candidate"]
        job       = event["data"]["job"]
        print(f"New application: {candidate['name']} → {job['title_en']}")
        # sync to your ATS, send Slack notification, etc.

    elif event["event"] == "application.status_changed":
        app_data = event["data"]["application"]
        print(f"Status change: {app_data['previous_status']} → {app_data['status']}")

    return "", 200
```

---

## See Also

- [PHP example →](php.md)
- [Node.js example →](nodejs.md)
- [curl examples →](curl.md)
- [Full documentation →](../docs/authentication.md)
- [NLnest platform →](https://nlnest.com)

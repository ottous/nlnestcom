# Node.js Integration Example

A ready-to-use Node.js client for the [NLnest](https://nlnest.com) Employer API. Uses the built-in `fetch` API (Node.js 18+) and the `crypto` module - no external dependencies required.

For Node.js 16 or earlier, replace `fetch` with the `node-fetch` package (`npm install node-fetch`).

---

## NLnestClient Class

```javascript
/**
 * NLnest Employer API Client - Node.js
 * https://nlnest.com
 *
 * Uses built-in fetch (Node.js 18+) and crypto - zero dependencies.
 *
 * Usage:
 *   import { NLnestClient } from './nlnest.js';
 *   const client = new NLnestClient(process.env.NLNEST_API_KEY);
 */

import crypto from 'crypto';
import { createWriteStream } from 'fs';
import { pipeline } from 'stream/promises';
import { Readable } from 'stream';

const BASE_URL = 'https://nlnest.com/api/v1';

export class NLnestAPIError extends Error {
  /**
   * @param {string} code
   * @param {string} message
   * @param {object} [details]
   */
  constructor(code, message, details = {}) {
    super(`[${code}] ${message}`);
    this.name = 'NLnestAPIError';
    this.code = code;
    this.details = details;
  }
}

export class NLnestClient {
  /**
   * @param {string} apiKey   Your NLnest API key
   * @param {number} [timeout] Request timeout in milliseconds (default: 30000)
   */
  constructor(apiKey, timeout = 30_000) {
    if (!apiKey) throw new TypeError('NLnest API key is required.');
    this.apiKey  = apiKey;
    this.timeout = timeout;
  }

  // ---------------------------------------------------------------------------
  // JOBS
  // ---------------------------------------------------------------------------

  /**
   * Create a new job listing.
   *
   * @param {object} data  Job fields - see docs/jobs.md for the full reference
   * @returns {Promise<object>}
   *
   * @example
   * const job = await client.createJob({
   *   title_en:        'Warehouse Operative',
   *   description_en:  '<p>Join our team in Amsterdam.</p>',
   *   sector_id:       4,
   *   city_name:       'Amsterdam',
   *   job_type:        'full-time',
   *   salary_min:      2550,
   *   salary_max:      3100,
   *   salary_period:   'month',
   *   positions:       3,
   *   housing_provided: true,
   *   auto_translate:   true,
   * });
   */
  async createJob(data) {
    return this.#request('POST', '/jobs', data);
  }

  /**
   * List all company jobs with optional filters.
   *
   * @param {{ status?: string, sector_id?: number, page?: number, per_page?: number }} [params]
   * @returns {Promise<{ data: object[], meta: object }>}
   */
  async listJobs(params = {}) {
    return this.#request('GET', '/jobs', null, params);
  }

  /**
   * Get a single job by ID.
   *
   * @param {number} jobId
   * @returns {Promise<object>}
   */
  async getJob(jobId) {
    return this.#request('GET', `/jobs/${jobId}`);
  }

  /**
   * Partially update a job - send only the fields you want to change.
   *
   * @param {number} jobId
   * @param {object} data
   * @returns {Promise<object>}
   */
  async updateJob(jobId, data) {
    return this.#request('PATCH', `/jobs/${jobId}`, data);
  }

  /**
   * Close (soft-delete) a job listing.
   *
   * @param {number} jobId
   * @returns {Promise<object>}
   */
  async closeJob(jobId) {
    return this.#request('DELETE', `/jobs/${jobId}`);
  }

  // ---------------------------------------------------------------------------
  // APPLICATIONS
  // ---------------------------------------------------------------------------

  /**
   * List applications with optional filters.
   *
   * @param {{ job_id?: number, status?: string, date_from?: string, date_to?: string, page?: number, per_page?: number }} [params]
   * @returns {Promise<{ data: object[], meta: object }>}
   */
  async listApplications(params = {}) {
    return this.#request('GET', '/applications', null, params);
  }

  /**
   * Get full details for a single application.
   *
   * @param {number} applicationId
   * @returns {Promise<object>}
   */
  async getApplication(applicationId) {
    return this.#request('GET', `/applications/${applicationId}`);
  }

  /**
   * Update the pipeline status of an application.
   *
   * @param {number} applicationId
   * @param {string} status   One of: new, reviewed, shortlisted, interview, offered, hired, rejected, withdrawn
   * @param {string} [note]   Optional internal note (not visible to candidate)
   * @returns {Promise<object>}
   */
  async updateApplicationStatus(applicationId, status, note) {
    const body = { status };
    if (note) body.note = note;
    return this.#request('PATCH', `/applications/${applicationId}`, body);
  }

  /**
   * Download a candidate CV and save it to a local file.
   *
   * @param {number} applicationId
   * @param {string} savePath  Local file path to write the CV to
   * @returns {Promise<string>}  The savePath
   */
  async downloadCV(applicationId, savePath) {
    const url = `${BASE_URL}/applications/${applicationId}/cv`;
    const response = await this.#fetch(url);

    if (response.status === 404) {
      throw new NLnestAPIError('not_found', `No CV on file for application ${applicationId}.`);
    }
    if (!response.ok) {
      throw new NLnestAPIError('server_error', `CV download failed: HTTP ${response.status}`);
    }

    const dest = createWriteStream(savePath);
    await pipeline(Readable.fromWeb(response.body), dest);
    return savePath;
  }

  // ---------------------------------------------------------------------------
  // WEBHOOKS
  // ---------------------------------------------------------------------------

  /**
   * Create a new webhook subscription.
   *
   * @param {string}   url         Your HTTPS endpoint
   * @param {string[]} events      Event names - use ['*'] for all events
   * @param {string}   [description]
   * @returns {Promise<object>}    Includes 'secret' - store it securely, shown only once!
   */
  async createWebhook(url, events, description = '') {
    return this.#request('POST', '/webhooks', { url, events, description });
  }

  /**
   * List all webhooks.
   * @returns {Promise<object>}
   */
  async listWebhooks() {
    return this.#request('GET', '/webhooks');
  }

  /**
   * Update a webhook.
   *
   * @param {number} webhookId
   * @param {object} data
   * @returns {Promise<object>}
   */
  async updateWebhook(webhookId, data) {
    return this.#request('PATCH', `/webhooks/${webhookId}`, data);
  }

  /**
   * Delete a webhook.
   *
   * @param {number} webhookId
   * @returns {Promise<object>}
   */
  async deleteWebhook(webhookId) {
    return this.#request('DELETE', `/webhooks/${webhookId}`);
  }

  // ---------------------------------------------------------------------------
  // ANALYTICS
  // ---------------------------------------------------------------------------

  /**
   * Get analytics for a single job.
   *
   * @param {number}  jobId
   * @param {string}  [dateFrom]  ISO 8601 date string
   * @param {string}  [dateTo]    ISO 8601 date string
   * @returns {Promise<object>}
   */
  async getJobAnalytics(jobId, dateFrom, dateTo) {
    const params = {};
    if (dateFrom) params.date_from = dateFrom;
    if (dateTo)   params.date_to   = dateTo;
    return this.#request('GET', `/analytics/jobs/${jobId}`, null, params);
  }

  /**
   * Get company-wide hiring overview.
   *
   * @param {string} [dateFrom]
   * @param {string} [dateTo]
   * @returns {Promise<object>}
   */
  async getOverview(dateFrom, dateTo) {
    const params = {};
    if (dateFrom) params.date_from = dateFrom;
    if (dateTo)   params.date_to   = dateTo;
    return this.#request('GET', '/analytics/overview', null, params);
  }

  // ---------------------------------------------------------------------------
  // WEBHOOK SIGNATURE VERIFICATION
  // ---------------------------------------------------------------------------

  /**
   * Verify an incoming webhook signature (HMAC-SHA256).
   *
   * Call this in your webhook handler before trusting the payload.
   *
   * @param {string|Buffer} payload          Raw request body
   * @param {string}        signatureHeader  Value of X-Webhook-Signature header
   * @param {string}        secret           Your webhook secret
   * @returns {boolean}
   *
   * @example
   * // Express.js handler
   * app.post('/webhook', express.raw({ type: '*\/*' }), (req, res) => {
   *   const ok = NLnestClient.verifyWebhookSignature(
   *     req.body,
   *     req.headers['x-webhook-signature'],
   *     process.env.NLNEST_WEBHOOK_SECRET,
   *   );
   *   if (!ok) return res.status(401).send('Unauthorized');
   *   const event = JSON.parse(req.body);
   *   // handle event...
   *   res.sendStatus(200);
   * });
   */
  static verifyWebhookSignature(payload, signatureHeader, secret) {
    const body     = typeof payload === 'string' ? Buffer.from(payload) : payload;
    const expected = 'sha256=' + crypto.createHmac('sha256', secret).update(body).digest('hex');
    try {
      return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signatureHeader));
    } catch {
      return false; // different lengths
    }
  }

  // ---------------------------------------------------------------------------
  // HTTP TRANSPORT (private)
  // ---------------------------------------------------------------------------

  /**
   * @param {string}      method
   * @param {string}      path
   * @param {object|null} [body]
   * @param {object}      [queryParams]
   * @returns {Promise<object>}
   */
  async #request(method, path, body = null, queryParams = {}) {
    let url = `${BASE_URL}${path}`;

    const qs = new URLSearchParams(
      Object.fromEntries(Object.entries(queryParams).filter(([, v]) => v !== undefined && v !== null))
    ).toString();
    if (qs) url += `?${qs}`;

    const response = await this.#fetch(url, {
      method,
      body: body !== null ? JSON.stringify(body) : undefined,
    });

    const data = await response.json().catch(() => {
      throw new NLnestAPIError('parse_error', `Non-JSON response from API (HTTP ${response.status})`);
    });

    if (data.success === false) {
      const { code = 'unknown', message = 'Unknown error', details } = data.error ?? {};
      throw new NLnestAPIError(code, message, details);
    }

    return data;
  }

  /**
   * Wraps fetch with auth headers and timeout.
   *
   * @param {string}      url
   * @param {RequestInit} [init]
   * @returns {Promise<Response>}
   */
  async #fetch(url, init = {}) {
    const controller = new AbortController();
    const timer      = setTimeout(() => controller.abort(), this.timeout);

    try {
      return await fetch(url, {
        ...init,
        headers: {
          'X-API-Key': this.apiKey,
          'Accept':    'application/json',
          ...(init.body ? { 'Content-Type': 'application/json' } : {}),
          ...(init.headers ?? {}),
        },
        signal: controller.signal,
      });
    } finally {
      clearTimeout(timer);
    }
  }
}
```

---

## Usage Examples

```javascript
import { NLnestClient, NLnestAPIError } from './nlnest.js';

const client = new NLnestClient(process.env.NLNEST_API_KEY);

// --- Create a job ---
try {
  const result = await client.createJob({
    title_en:          'Forklift Operator',
    description_en:    '<p>Experienced forklift operator needed in Rotterdam.</p>',
    sector_id:         4,
    city_name:         'Rotterdam',
    job_type:          'full-time',
    experience_level:  'mid',
    salary_min:        2800,
    salary_max:        3400,
    salary_period:     'month',
    positions:         2,
    housing_provided:  true,
    expires_at:        '2026-07-31',
    external_id:       'FO-2026-007',
    auto_translate:    true,
  });
  console.log(`Job created: ${result.data.id} → ${result.data.url}`);
} catch (err) {
  if (err instanceof NLnestAPIError) {
    console.error(`API error (${err.code}): ${err.message}`);
  } else {
    throw err;
  }
}

// --- List active jobs ---
const { data: jobs } = await client.listJobs({ status: 'active' });
for (const job of jobs) {
  console.log(`[${job.id}] ${job.title_en} - ${job.applications_count} applications`);
}

// --- Update application statuses ---
const { data: apps } = await client.listApplications({ status: 'new', per_page: 100 });
await Promise.all(apps.map(app =>
  client.updateApplicationStatus(app.id, 'reviewed', 'Auto-reviewed by integration.')
));
console.log(`Marked ${apps.length} applications as reviewed.`);

// --- Analytics ---
const stats = await client.getJobAnalytics(18472, '2026-03-01', '2026-03-31');
const d = stats.data;
console.log(`Views: ${d.views} | Applications: ${d.applications} | Conversion: ${d.conversion_rate}%`);

// --- Company overview ---
const overview = await client.getOverview('2026-01-01', '2026-03-31');
const o = overview.data;
console.log(`Active jobs: ${o.active_jobs} | Hired: ${o.total_hired} | Avg TTH: ${o.avg_time_to_hire_days} days`);
```

---

## Express.js Webhook Handler

```javascript
import express from 'express';
import { NLnestClient } from './nlnest.js';

const app    = express();
const SECRET = process.env.NLNEST_WEBHOOK_SECRET;

// IMPORTANT: use express.raw() - not express.json() - to get the raw body for signature verification
app.post('/webhooks/nlnest', express.raw({ type: '*/*' }), (req, res) => {
  const signature = req.headers['x-webhook-signature'] ?? '';

  if (!NLnestClient.verifyWebhookSignature(req.body, signature, SECRET)) {
    console.warn('Invalid webhook signature - rejected.');
    return res.status(401).json({ error: 'Invalid signature' });
  }

  const event = JSON.parse(req.body.toString());
  console.log(`NLnest event: ${event.event} (delivery: ${event.delivery_id})`);

  switch (event.event) {
    case 'application.created': {
      const { candidate, job } = event.data;
      console.log(`New application: ${candidate.name} → ${job.title_en}`);
      // Push to your ATS, Slack, database, etc.
      break;
    }
    case 'application.status_changed': {
      const { application } = event.data;
      console.log(`Status: ${application.previous_status} → ${application.status}`);
      break;
    }
    case 'job.expired': {
      const { job } = event.data;
      console.log(`Job expired: [${job.id}] ${job.title_en}`);
      break;
    }
  }

  // Always respond 200 quickly
  res.sendStatus(200);
});

app.listen(3000, () => console.log('Webhook server running on :3000'));
```

---

## See Also

- [PHP example →](php.md)
- [Python example →](python.md)
- [curl examples →](curl.md)
- [Full documentation →](../docs/authentication.md)
- [NLnest platform →](https://nlnest.com)

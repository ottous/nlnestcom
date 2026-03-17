# PHP Integration Example

A ready-to-use PHP class for the [NLnest](https://nlnest.com) Employer API. Requires PHP 7.4+ with the `curl` extension (standard on any modern PHP installation).

---

## NLnestAPI Class

```php
<?php

/**
 * NLnest Employer API Client
 *
 * A minimal cURL-based wrapper for the NLnest API v1.
 * https://nlnest.com
 *
 * Usage:
 *   $api = new NLnestAPI($_ENV['NLNEST_API_KEY']);
 *   $jobs = $api->listJobs(['status' => 'active']);
 *   $job  = $api->createJob([...]);
 */
class NLnestAPI
{
    private const BASE_URL = 'https://nlnest.com/api/v1';
    private const TIMEOUT  = 30;

    private string $apiKey;

    public function __construct(string $apiKey)
    {
        if (empty($apiKey)) {
            throw new InvalidArgumentException('NLnest API key is required.');
        }
        $this->apiKey = $apiKey;
    }

    // -------------------------------------------------------------------------
    // JOBS
    // -------------------------------------------------------------------------

    /**
     * Create a new job listing.
     *
     * @param  array $data  Job fields (see docs/jobs.md for full field list)
     * @return array        Created job data
     */
    public function createJob(array $data): array
    {
        return $this->request('POST', '/jobs', $data);
    }

    /**
     * List all jobs with optional filters.
     *
     * @param  array $params  Query params: status, sector_id, page, per_page
     * @return array          Paginated list with 'data' and 'meta'
     */
    public function listJobs(array $params = []): array
    {
        return $this->request('GET', '/jobs', null, $params);
    }

    /**
     * Get a single job by ID.
     *
     * @param  int $jobId
     * @return array
     */
    public function getJob(int $jobId): array
    {
        return $this->request('GET', "/jobs/{$jobId}");
    }

    /**
     * Partially update a job.
     *
     * @param  int   $jobId
     * @param  array $data   Only the fields to change
     * @return array
     */
    public function updateJob(int $jobId, array $data): array
    {
        return $this->request('PATCH', "/jobs/{$jobId}", $data);
    }

    /**
     * Close (soft-delete) a job.
     *
     * @param  int $jobId
     * @return array
     */
    public function closeJob(int $jobId): array
    {
        return $this->request('DELETE', "/jobs/{$jobId}");
    }

    // -------------------------------------------------------------------------
    // APPLICATIONS
    // -------------------------------------------------------------------------

    /**
     * List applications with optional filters.
     *
     * @param  array $params  job_id, status, date_from, date_to, page, per_page
     * @return array
     */
    public function listApplications(array $params = []): array
    {
        return $this->request('GET', '/applications', null, $params);
    }

    /**
     * Get a single application by ID.
     *
     * @param  int $applicationId
     * @return array
     */
    public function getApplication(int $applicationId): array
    {
        return $this->request('GET', "/applications/{$applicationId}");
    }

    /**
     * Update the pipeline status of an application.
     *
     * @param  int    $applicationId
     * @param  string $status         One of: new, reviewed, shortlisted, interview, offered, hired, rejected, withdrawn
     * @param  string $note           Optional internal note (not visible to candidate)
     * @return array
     */
    public function updateApplicationStatus(int $applicationId, string $status, string $note = ''): array
    {
        $data = ['status' => $status];
        if ($note !== '') {
            $data['note'] = $note;
        }
        return $this->request('PATCH', "/applications/{$applicationId}", $data);
    }

    /**
     * Download a candidate CV and save it to a local path.
     *
     * @param  int    $applicationId
     * @param  string $savePath       Full local file path to write the CV to
     * @return bool                   True on success
     * @throws RuntimeException       If CV not found or download fails
     */
    public function downloadCV(int $applicationId, string $savePath): bool
    {
        $url = self::BASE_URL . "/applications/{$applicationId}/cv";

        $ch = curl_init($url);
        $fh = fopen($savePath, 'wb');

        if ($fh === false) {
            throw new RuntimeException("Cannot write to path: {$savePath}");
        }

        curl_setopt_array($ch, [
            CURLOPT_HTTPHEADER     => ["X-API-Key: {$this->apiKey}"],
            CURLOPT_FILE           => $fh,
            CURLOPT_TIMEOUT        => self::TIMEOUT,
            CURLOPT_FOLLOWLOCATION => true,
            CURLOPT_FAILONERROR    => true,
        ]);

        $ok   = curl_exec($ch);
        $code = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $err  = curl_error($ch);

        curl_close($ch);
        fclose($fh);

        if (!$ok || $code === 404) {
            unlink($savePath);
            throw new RuntimeException("CV not found or download failed (HTTP {$code}): {$err}");
        }

        return true;
    }

    // -------------------------------------------------------------------------
    // WEBHOOKS
    // -------------------------------------------------------------------------

    /**
     * Create a new webhook subscription.
     *
     * @param  string $url     Your HTTPS endpoint
     * @param  array  $events  Event names, e.g. ['application.created'] or ['*']
     * @param  string $description  Optional label
     * @return array           Includes 'secret' - store it securely, shown only once
     */
    public function createWebhook(string $url, array $events, string $description = ''): array
    {
        return $this->request('POST', '/webhooks', [
            'url'         => $url,
            'events'      => $events,
            'description' => $description,
        ]);
    }

    /**
     * List all webhooks.
     *
     * @return array
     */
    public function listWebhooks(): array
    {
        return $this->request('GET', '/webhooks');
    }

    /**
     * Update a webhook.
     *
     * @param  int   $webhookId
     * @param  array $data
     * @return array
     */
    public function updateWebhook(int $webhookId, array $data): array
    {
        return $this->request('PATCH', "/webhooks/{$webhookId}", $data);
    }

    /**
     * Delete a webhook.
     *
     * @param  int $webhookId
     * @return array
     */
    public function deleteWebhook(int $webhookId): array
    {
        return $this->request('DELETE', "/webhooks/{$webhookId}");
    }

    // -------------------------------------------------------------------------
    // ANALYTICS
    // -------------------------------------------------------------------------

    /**
     * Get analytics for a single job.
     *
     * @param  int    $jobId
     * @param  string $dateFrom  ISO 8601 date, e.g. '2026-01-01'
     * @param  string $dateTo    ISO 8601 date
     * @return array
     */
    public function getJobAnalytics(int $jobId, string $dateFrom = '', string $dateTo = ''): array
    {
        $params = array_filter(['date_from' => $dateFrom, 'date_to' => $dateTo]);
        return $this->request('GET', "/analytics/jobs/{$jobId}", null, $params);
    }

    /**
     * Get company-wide hiring overview.
     *
     * @param  string $dateFrom
     * @param  string $dateTo
     * @return array
     */
    public function getOverview(string $dateFrom = '', string $dateTo = ''): array
    {
        $params = array_filter(['date_from' => $dateFrom, 'date_to' => $dateTo]);
        return $this->request('GET', '/analytics/overview', null, $params);
    }

    // -------------------------------------------------------------------------
    // WEBHOOK SIGNATURE VERIFICATION
    // -------------------------------------------------------------------------

    /**
     * Verify an incoming webhook signature.
     *
     * Call this in your webhook handler before processing the payload.
     *
     * @param  string $payload    Raw request body (file_get_contents('php://input'))
     * @param  string $signature  Value of X-Webhook-Signature header
     * @param  string $secret     Your webhook secret (stored at creation time)
     * @return bool
     */
    public static function verifyWebhookSignature(string $payload, string $signature, string $secret): bool
    {
        $expected = 'sha256=' . hash_hmac('sha256', $payload, $secret);
        return hash_equals($expected, $signature);
    }

    // -------------------------------------------------------------------------
    // HTTP TRANSPORT
    // -------------------------------------------------------------------------

    /**
     * Make an authenticated HTTP request to the NLnest API.
     *
     * @param  string      $method   GET, POST, PATCH, DELETE
     * @param  string      $path     Endpoint path, e.g. '/jobs'
     * @param  array|null  $body     JSON request body (for POST/PATCH)
     * @param  array       $query    Query string parameters (for GET)
     * @return array                 Decoded response body
     * @throws RuntimeException      On HTTP error or API error response
     */
    private function request(string $method, string $path, ?array $body = null, array $query = []): array
    {
        $url = self::BASE_URL . $path;
        if (!empty($query)) {
            $url .= '?' . http_build_query($query);
        }

        $headers = [
            "X-API-Key: {$this->apiKey}",
            'Accept: application/json',
        ];

        $ch = curl_init($url);

        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT        => self::TIMEOUT,
            CURLOPT_FOLLOWLOCATION => false,
        ]);

        if ($method === 'POST' || $method === 'PATCH') {
            $json = json_encode($body ?? []);
            $headers[] = 'Content-Type: application/json';
            $headers[] = 'Content-Length: ' . strlen($json);
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
            curl_setopt($ch, CURLOPT_POSTFIELDS, $json);
        } elseif ($method === 'DELETE') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'DELETE');
        }

        curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $curlErr  = curl_error($ch);
        curl_close($ch);

        if ($curlErr) {
            throw new RuntimeException("cURL error: {$curlErr}");
        }

        $decoded = json_decode($response, true);
        if (!is_array($decoded)) {
            throw new RuntimeException("Invalid JSON response from NLnest API (HTTP {$httpCode}).");
        }

        if (isset($decoded['success']) && $decoded['success'] === false) {
            $code    = $decoded['error']['code']    ?? 'unknown';
            $message = $decoded['error']['message'] ?? 'Unknown error';
            throw new RuntimeException("NLnest API error [{$code}]: {$message}");
        }

        return $decoded;
    }
}
```

---

## Usage Examples

```php
<?php

require 'NLnestAPI.php'; // or autoload

$api = new NLnestAPI($_ENV['NLNEST_API_KEY']);

// --- Create a job ---
$job = $api->createJob([
    'title_en'        => 'Forklift Operator',
    'description_en'  => 'Experienced forklift operator needed for our distribution centre in Rotterdam.',
    'sector_id'       => 4,
    'city_name'       => 'Rotterdam',
    'job_type'        => 'full-time',
    'salary_min'      => 2800,
    'salary_max'      => 3400,
    'salary_period'   => 'month',
    'positions'       => 2,
    'housing_provided'=> true,
    'expires_at'      => '2026-07-31',
    'external_id'     => 'FO-2026-007',
    'auto_translate'  => true,
]);
echo "Job created: " . $job['data']['id'] . " → " . $job['data']['url'] . PHP_EOL;

// --- List active jobs ---
$result = $api->listJobs(['status' => 'active']);
foreach ($result['data'] as $j) {
    echo "[{$j['id']}] {$j['title_en']} - {$j['applications_count']} applications" . PHP_EOL;
}

// --- Process new applications ---
$apps = $api->listApplications(['job_id' => $job['data']['id'], 'status' => 'new']);
foreach ($apps['data'] as $app) {
    $candidate = $app['candidate'];
    echo "New application from {$candidate['name']} ({$candidate['nationality']})" . PHP_EOL;

    // Move to reviewed
    $api->updateApplicationStatus($app['id'], 'reviewed', 'Initial review complete.');
}

// --- Download a CV ---
$api->downloadCV(9341, '/tmp/cv_9341.pdf');
echo "CV downloaded." . PHP_EOL;

// --- Analytics ---
$stats = $api->getJobAnalytics($job['data']['id'], '2026-03-01', '2026-03-31');
echo "Views: {$stats['data']['views']}, Applications: {$stats['data']['applications']}" . PHP_EOL;

// --- Webhook handler (in a separate endpoint file) ---
// $payload   = file_get_contents('php://input');
// $signature = $_SERVER['HTTP_X_WEBHOOK_SIGNATURE'] ?? '';
// $secret    = $_ENV['NLNEST_WEBHOOK_SECRET'];
//
// if (!NLnestAPI::verifyWebhookSignature($payload, $signature, $secret)) {
//     http_response_code(401);
//     exit;
// }
//
// $event = json_decode($payload, true);
// if ($event['event'] === 'application.created') {
//     // handle new application...
// }
// http_response_code(200);
```

---

## Error Handling

```php
try {
    $job = $api->getJob(99999);
} catch (RuntimeException $e) {
    // e.g. "NLnest API error [not_found]: Job with ID 99999 does not exist."
    error_log($e->getMessage());
}
```

---

## See Also

- [Python example →](python.md)
- [Node.js example →](nodejs.md)
- [curl examples →](curl.md)
- [Full API documentation →](../docs/authentication.md)
- [NLnest platform →](https://nlnest.com)

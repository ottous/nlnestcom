# NLnest Employer API

[![API Version](https://img.shields.io/badge/API-v1-blue)](https://nlnest.com)
[![License](https://img.shields.io/badge/license-MIT-green)](#license)

[**NLnest**](https://nlnest.com) is a recruitment platform connecting Dutch companies with skilled candidates from across the European Union. With support for **11 languages**, smart job matching, and built-in relocation tools, NLnest makes international hiring straightforward.

This repository contains the official documentation for the **NLnest Employer API v1**.

---

## Platform Overview

[NLnest](https://nlnest.com) gives employers and candidates everything they need for a successful cross-border hire:

- **11 languages supported** - English, Romanian, Dutch, Hungarian, Polish, Spanish, Portuguese, Lithuanian, Bulgarian, Greek, and Ukrainian
- **Smart job matching** - algorithmic scoring based on skills, location, experience, and language preferences
- **Auto-translation** - post a job in English and NLnest automatically translates it to all 11 languages
- **Relocation tools** for candidates:
  - [Salary Calculator](https://nlnest.com/en/salary-calculator) - gross-to-net salary breakdown for the Netherlands
  - [Cost of Living Calculator](https://nlnest.com/en/cost-of-living) - compare living costs across Dutch cities
  - [Relocation Quiz & Checklist](https://nlnest.com/en/relocation-quiz) - personalised relocation roadmap
- **Job alerts** - candidates subscribe to alerts; matched jobs are delivered by email
- **Company profiles** - public employer pages with branding, open positions, and reviews
- **Candidate CRM** - full pipeline management: Kanban board, interviews, scorecards, offers, onboarding
- **Analytics** - views, applications, conversion rates, time-to-hire

---

## API Overview

The NLnest Employer API lets you integrate your ATS, HRIS, or internal tooling directly with the platform. With the API you can:

- **Publish and manage jobs** - create, update, close, and query job listings programmatically
- **Read applications** - pull candidate details, CV files, and screening answers into your own systems
- **Update pipeline status** - move candidates through stages (shortlisted, interview, hired, rejected, etc.)
- **Receive webhook events** - get real-time push notifications when applications arrive or statuses change
- **Query analytics** - retrieve job performance data, conversion rates, and hiring KPIs

All responses are JSON. The API base URL is `https://nlnest.com/api/v1/`.

---

## Quick Start

**1. Get your API key**

Log in to your company account at [nlnest.com](https://nlnest.com), go to **Dashboard → API Keys**, and generate a key. Production keys start with `nlnest_live_`, test keys with `nlnest_test_`.

**2. Make your first request**

```bash
curl https://nlnest.com/api/v1/jobs \
  -H "X-API-Key: nlnest_live_your_key_here"
```

**3. Create your first job**

```bash
curl -X POST https://nlnest.com/api/v1/jobs \
  -H "X-API-Key: nlnest_live_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "title_en": "Warehouse Operative",
    "description_en": "We are looking for a reliable warehouse operative to join our team in Amsterdam.",
    "city_name": "Amsterdam",
    "job_type": "full-time",
    "salary_min": 2550,
    "salary_max": 3200,
    "salary_period": "month",
    "auto_translate": true
  }'
```

That's it. With `auto_translate: true`, NLnest will automatically produce translations in all 11 languages, dramatically increasing your job's visibility across the EU candidate pool.

---

## Table of Contents

| Document | Description |
|----------|-------------|
| [docs/authentication.md](docs/authentication.md) | API keys, headers, base URL |
| [docs/jobs.md](docs/jobs.md) | Create, list, update, and close job listings |
| [docs/applications.md](docs/applications.md) | Read and manage candidate applications |
| [docs/webhooks.md](docs/webhooks.md) | Real-time event notifications |
| [docs/analytics.md](docs/analytics.md) | Job performance and hiring analytics |
| [docs/errors.md](docs/errors.md) | Error codes and troubleshooting |
| [docs/rate-limiting.md](docs/rate-limiting.md) | Rate limits and quota headers |
| [examples/curl.md](examples/curl.md) | curl command examples |
| [examples/php.md](examples/php.md) | PHP integration example |
| [examples/python.md](examples/python.md) | Python integration example |
| [examples/nodejs.md](examples/nodejs.md) | Node.js integration example |

---

## Supported Languages

| Code | Language   |
|------|-----------|
| `en` | English    |
| `ro` | Romanian   |
| `nl` | Dutch      |
| `hu` | Hungarian  |
| `pl` | Polish     |
| `es` | Spanish    |
| `pt` | Portuguese |
| `lt` | Lithuanian |
| `bg` | Bulgarian  |
| `el` | Greek      |
| `uk` | Ukrainian  |

---

## Key Resources

- **Platform**: [https://nlnest.com](https://nlnest.com)
- **Salary Calculator**: [https://nlnest.com/en/salary-calculator](https://nlnest.com/en/salary-calculator)
- **Cost of Living**: [https://nlnest.com/en/cost-of-living](https://nlnest.com/en/cost-of-living)
- **Relocation Quiz**: [https://nlnest.com/en/relocation-quiz](https://nlnest.com/en/relocation-quiz)
- **Employer Dashboard**: [https://nlnest.com](https://nlnest.com) → Log in → Company

---

## License

MIT License. See [LICENSE](LICENSE) for details.

Documentation and examples are provided as-is for integration purposes. The NLnest platform and API are proprietary - see [nlnest.com](https://nlnest.com) for terms of service.

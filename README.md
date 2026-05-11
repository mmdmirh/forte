# Forte

Public-facing umbrella repo for the **Forte** platform — API specs, docs,
and pointers to the apps that consume them.

> 🌐 Website: [forte.help](https://forte.help)
> 📖 Live API reference: [api.forte.help/docs](https://api.forte.help/docs) ·
> [/redoc](https://api.forte.help/redoc) · [/openapi.json](https://api.forte.help/openapi.json)
> 🧭 Hosted Swagger UI (this repo): https://mmdmirh.github.io/forte/

## Quick start — buy credits and call the API

1. **Sign up / sign in** at https://forte.help/developer (uses your
   existing Forte account — same email/password as the iOS app).
2. **Buy a credit pack** on the Billing page. Stripe Checkout opens
   in test or live mode depending on the deployment; Stripe handles
   payment and the webhook credits your balance within a second or
   two. Packs:

   | Pack | Credits | Price |
   |---|---|---|
   | Starter | 1,000 | $10 |
   | Pro | 2,500 | $25 |
   | Scale | 10,000 | $100 |
   | Enterprise | 25,000 | $250 |

3. **Mint an API token** on the Tokens page. The plaintext
   `forte_sk_<32 hex>` value is shown **once** — copy it. We only
   store a SHA-256 hash; we can't recover it.

4. **Call the API.** Each endpoint deducts a fixed credit cost:

```bash
# Parse a job posting URL (5 credits)
curl -X POST https://api.forte.help/api/v1/jobs/by-url \
  -H "Authorization: Bearer forte_sk_..." \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/your-job-posting"}'

# Score a resume against a job (10 credits)
curl -X POST https://api.forte.help/api/v1/match \
  -H "Authorization: Bearer forte_sk_..." \
  -H "Content-Type: application/json" \
  -d '{
        "resume_text": "...",
        "job": {"title": "Backend Engineer", "description": "..."}
      }'
```

5. **Watch your balance** at https://forte.help/developer/usage
   (per-day chart + last 200 calls). You'll get an email at 30
   credits remaining so you know to top up before requests start
   failing with `402 Payment Required`.

## Endpoint pricing

| Endpoint | Credits | Returns |
|---|---|---|
| `POST /api/v1/jobs/by-url` | 5 | Structured job from a posting URL |
| `POST /api/v1/parse-resume` | 5 | Structured profile from a PDF resume |
| `POST /api/v1/match` | 10 | 0–100 fit score + rationale for a (resume, job) pair |
| `POST /api/v1/resume/build` | 10 | LaTeX-rendered PDF from structured data |
| `POST /api/v1/cover-letter` | 15 | Tailored cover letter for a (resume, job) pair |
| `POST /api/v1/jobs/scan` | 20 | Real-time multi-ATS search by titles + locations |

Failed requests (4xx/5xx) **don't** burn credits — they log to your
usage with a `0` charge so abuse / leaked-token bursts are visible
to both us and you.

Rate limits: **60 req/min** and **1,000 req/hour** per token. 429
responses include a `Retry-After` header. If you need higher limits,
email `support@forte.help`.

## What Forte is

Forte is a two-sided hiring platform built around honesty, not optimization:

- **Forte (job seeker app, iOS)** — find roles, run AI-assisted matching
  against your resume, message recruiters who reach out, and apply through
  the in-app Easy Apply flow. App Store: pending. Source:
  [forte-ios-seeker](https://github.com/mmdmirh/forte-ios-seeker) *(private)*.
- **Forte Recruiter (iOS)** — post roles + one-day local gigs, search
  discoverable candidates, run a kanban pipeline, message candidates with
  an optional AI-drafted opener. Pre-launch. Source:
  [forte-ios-recruiter](https://github.com/mmdmirh/forte-ios-recruiter) *(private)*.
- **Backend** — FastAPI on Azure App Service. Source:
  [forte-backend](https://github.com/mmdmirh/forte-backend) *(private)*.

This repo is **public** because the API contract should be readable by
anyone evaluating the platform, building an integration, or auditing what
the apps send. The app source stays private until it ships.

## API at a glance

| Surface | Path prefix | Audience | Auth |
|---|---|---|---|
| Auth | `/auth/*` | both | none → returns bearer |
| Public | `/health`, `/config`, `/plans`, `/legal/*` | both | none |
| Seeker | `/me/*`, `/seeker/v1/*` | job seekers | bearer (seeker role) |
| Recruiter | `/recruiter/v1/*` | recruiters | bearer (recruiter role) |

Newer endpoints live under `/seeker/v1/*` (versioned). Older `/me/*`
endpoints stay registered as aliases for back-compat with shipped App
Store builds — see the comment at the top of each handler in
`backend/app.py` for the alias mapping.

## Browse the spec

Three ways to read the API:

1. **Hosted Swagger UI for this repo** —
   <https://mmdmirh.github.io/forte/> (after GitHub Pages is enabled).
2. **Live Swagger UI on the production server** —
   <https://api.forte.help/docs>.
3. **Raw OpenAPI JSON** — [`openapi.json`](./openapi.json) in this repo
   (snapshot, version-controlled) or
   <https://api.forte.help/openapi.json> (always-fresh).

The snapshot in this repo is updated whenever a backend change touches a
new route. The live spec is authoritative.

## Architecture

```
┌───────────────────────────┐         ┌───────────────────────────┐
│   Forte iOS (seeker app)  │         │ Forte Recruiter iOS app   │
│  com.moenejad.forte       │         │ com.moenejad.forterecruiter│
└──────────────┬────────────┘         └───────────┬───────────────┘
               │  bearer token                    │  bearer token
               │  /me/*  /seeker/v1/*             │  /recruiter/v1/*
               ▼                                  ▼
              ┌──────────────────────────────────────┐
              │  FastAPI · https://api.forte.help    │
              │  Azure App Service (forte-api)       │
              │  SQLite at /home/data/jobs.sqlite    │
              └──────────────────────────────────────┘
                                │
                                ├─ Azure OpenAI (resume scoring, AI draft)
                                ├─ APNs (push notifications)
                                └─ StoreKit / Apple receipts (billing)
```

## Reporting issues

- **API bugs / docs** → file an issue in this repo.
- **App bugs** → use the in-app feedback flow (Settings → Help & Feedback);
  it goes to `support@forte.help`.
- **Security** → email `support@forte.help` directly. Please don't open
  public issues for vulnerabilities.

## License

The OpenAPI spec + docs in this repo are MIT-licensed (see
[LICENSE](./LICENSE)). The app source and backend implementation are
proprietary and licensed separately.

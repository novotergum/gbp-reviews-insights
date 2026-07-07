# GBP Reviews & Insights

A GitHub Actions toolkit that exports **Google Business Profile (GBP) reviews and performance insights** for *all* locations of a GBP account — on a schedule, with no server to run.

Multi-location businesses (retail chains, clinics, franchises, agencies managing local clients) constantly need their Google reviews and performance metrics somewhere else: a spreadsheet, a BI tool, a notification flow, an AI reply assistant. The official APIs exist, but wiring them up — OAuth, pagination, rate limits, per-location fan-out — is the same plumbing every time. This repo is that plumbing, packaged as ready-to-fork scheduled workflows that deliver clean JSON to any webhook you point them at.

Built and used in production by [NOVOTERGUM](https://novotergum.de), a German physiotherapy group, to monitor reviews and local visibility across 90+ locations.

## What's included

| Script | Workflow | What it does |
|---|---|---|
| `gbp-reviews-daily.js` | daily cron | Fetches yesterday's new/updated reviews across all locations, posts them to a webhook; optionally registers each review with a reply-assistant API and attaches a public deep link |
| `gbp-reviews-history.js` | manual | One-off backfill of the full review history (all locations, paginated) |
| `gbp-insights-daily.js` | daily cron | Pulls daily performance metrics (calls, website clicks, direction requests, impressions …) per location |
| `gbp-insights-weekly.js` | weekly cron | Same metrics aggregated for weekly reporting |
| `gbp-insights-monthly.js` | monthly cron | Same metrics aggregated for monthly reporting |
| `gbp-tag-reviews.js` | cron | Ships review batches to a webhook for downstream classification/tagging (e.g. an LLM that labels review topics) |

All scripts share the same core: OAuth2 refresh-token flow, automatic retry with exponential backoff on 429/5xx, per-location fan-out with a configurable concurrency pool, and batched webhook delivery.

## How it works

```
GitHub Actions (cron)
  └─ Node.js script
       ├─ Google Business Profile APIs
       │    ├─ mybusinessbusinessinformation.googleapis.com  (locations)
       │    ├─ mybusiness.googleapis.com/v4                  (reviews)
       │    └─ businessprofileperformance.googleapis.com     (metrics)
       └─ POST → your webhook (Make.com, Zapier, n8n, or any HTTP endpoint)
```

There is no database and no hosted component: state lives in Google's APIs, delivery is fire-and-forget to your automation platform of choice.

## Setup

### 1. Google Cloud project

1. Create a Google Cloud project and request access to the [Business Profile APIs](https://developers.google.com/my-business) (Google approves this per project).
2. Enable **My Business Business Information API**, **My Business API (v4)** and **Business Profile Performance API**.
3. Create an OAuth client (type *Web* or *Desktop*) and obtain a **refresh token** for a Google account that manages your Business Profiles (any standard OAuth playground / helper script works — request scope `https://www.googleapis.com/auth/business.manage`).

### 2. Repository secrets

Fork the repo and set the following Actions secrets:

| Secret | Required | Purpose |
|---|---|---|
| `GBP_CLIENT_ID` / `GBP_CLIENT_SECRET` | ✅ | OAuth client |
| `GBP_REFRESH_TOKEN` | ✅ | Refresh token with `business.manage` scope |
| `GBP_ACCOUNT_ID` | ✅ | Numeric GBP account id (the `accounts/…` owner of your locations) |
| `MAKE_REVIEWS_WEBHOOK_URL` | ✅ (reviews) | Where daily review batches are POSTed |
| `MAKE_INSIGHTS_WEBHOOK_URL_DAILY` / `_WEEKLY` / `_MONTHLY` | ✅ (insights) | Where metric batches are POSTed |
| `MAKE_TAG_REVIEWS_WEBHOOK_URL` | for tagging | Endpoint of your review classifier |
| `PREFILL_API_URL` / `PREFILL_SECRET` / `PUBLIC_APP_URL` | optional | Hook into a reply-assistant app: each review is registered there and the webhook payload carries a ready-to-share reply link |

Despite the `MAKE_*` naming, any HTTP endpoint that accepts JSON works.

### 3. Schedules

Each workflow in `.github/workflows/` has its own cron (UTC) and can also be run manually via *workflow_dispatch*. Adjust the crons to your needs.

> **Note:** GitHub disables scheduled workflows in repos without activity after 60 days — a single commit re-enables them.

## Payloads

Review objects are delivered normalized: location name/store code, star rating, comment, reviewer, create/update timestamps, review state and a Google Maps deep link (`place_id`-based). Insight payloads carry one row per location, day and metric. Timezone for "yesterday"/period boundaries is currently `Europe/Berlin` (see `TZ` constant in the scripts).

## Limitations / roadmap

- Timezone is a constant, not yet an env var
- One GBP account per workflow run (multi-account = fork per account)
- CSV export helpers exist (`papaparse`) but the primary delivery path is webhooks

Issues and PRs welcome.

## License

[MIT](LICENSE)

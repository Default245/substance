# CLAUDE.md — SiftMail v3

## Overview
Full-stack email triage platform with heuristic spam/phish scoring, Gmail + Outlook support, Stripe billing with tiered plans, Chrome Extension, and async BullMQ processing. Monorepo managed via Turborepo + pnpm workspaces.

## Stack
- **Backend**: Fastify + TypeScript, Prisma + PostgreSQL, BullMQ + Redis
- **Frontend**: Next.js 14, CSS variables + `data-theme` theming, utility classes (animations, glass, depth, 3D text), API proxy pattern
- **Worker**: BullMQ workers — ingest, classify, digest, anomaly detection, token refresh
- **Extension**: Chrome Extension (Manifest V3) — popup, background service worker, Gmail content script
- **Billing**: Stripe Checkout + Portal + webhook-driven subscription state machine
- **Security**: AES-256-GCM token encryption at rest, request-id propagation, PII redaction in logs
- **CI/CD**: Railway GitHub integration (API + Worker + Web), per-service Dockerfiles

## Quick Start

### Prerequisites
- Node 20+, pnpm 9+
- PostgreSQL running locally (or use Docker Compose)
- Redis running locally (or use Docker Compose)

### Setup (Manual)
```bash
cd siftmail-v3
pnpm install

# API
cd apps/api
cp .env.example .env   # fill in credentials
pnpm db:push           # create DB tables
pnpm dev               # starts on :4000

# Web (new terminal)
cd apps/web
cp .env.local.example .env.local
pnpm dev               # starts on :3000

# Worker (new terminal)
cd apps/worker
cp ../api/.env.example .env
pnpm dev

# Extension (new terminal)
cd apps/extension
pnpm build             # outputs to dist/
# Load dist/ as unpacked extension in chrome://extensions
```

### Setup (Docker Compose)
```bash
cd siftmail-v3
cp apps/api/.env.example apps/api/.env  # fill in credentials
docker compose -f infra/docker-compose.yml up --build
# API on :4000, Web on :3000, Postgres on :5432, Redis on :6379
```

## Architecture
```
apps/
  api/           Fastify REST API
    src/
      server.ts        Entry point, plugin registration, middleware
      routes/
        auth-google.ts     Google OAuth flow
        auth-microsoft.ts  Microsoft OAuth flow
        gmail.ts           Gmail message list, score, quarantine, undo
        rules.ts           Allow/block/VIP rule management
        mode.ts            Shadow mode, sensitivity, onboarding
        classify.ts        Single-message classification endpoint
        sync.ts            Mailbox sync trigger + status
        actions.ts         Quarantine/release/block/allow with idempotency
        feedback.ts        User label corrections + auto-rule creation
        events.ts          Analytics ingestion + funnel/KPI queries
        billing.ts         Stripe checkout, portal, subscription, invoices
        stats.ts           Dashboard stats (threats, FP rate, time saved)
        stripe.ts          Stripe webhook handler (subscription state machine)
        digest.ts          Audit log query
      lib/
        tier-gate.ts       requireFeature() / requirePlan() middleware
        gmail.ts           Shared Gmail client (getGmailClient + ensureLabel)
  web/           Next.js 14 frontend
    app/
      page.tsx           Landing page (hero, social proof, features, how-it-works, testimonials, CTA)
      layout.tsx         Root layout + theme
      globals.css        Design system: CSS variables, animations, utility classes (glass, depth-card, text-3d, hover-lift, etc.)
      pricing/page.tsx   Pricing page (3 tiers + add-ons + FAQ)
      dashboard/page.tsx Dashboard (stats, messages, quarantine, batch, rules, digest, audit, billing tabs)
      robots.ts          robots.txt (disallow /dashboard, /api/)
      sitemap.ts         sitemap.xml (4 public pages)
      privacy/page.tsx   Privacy policy (branded, with nav + footer)
      terms/page.tsx     Terms of service (branded, with nav + footer)
      api/sift/[...path]/route.ts  API proxy to backend (forwards x-request-id)
      components/        ThemeProvider, ThemeToggle
  worker/        BullMQ async workers
    src/
      worker.ts    Ingest, classify, digest (SMTP), anomaly, token refresh workers
  extension/     Chrome Extension (Manifest V3)
    public/
      manifest.json    Permissions: storage, tabs, scripting, alarms
      popup.html       Extension popup UI
      content.css      Gmail badge injection styles
    src/
      popup.ts         OAuth connect, toggles, scan, quarantine list
      background.ts    Service worker: configurable API URL via chrome.storage.local
      content.ts       Gmail MutationObserver badge injection + inline actions (XSS-safe DOM)
packages/
  utils/         Shared types, tier definitions, queue names, scoring engine, crypto
    src/
      index.ts         Re-exports + QUEUE_NAMES, TIERS, types
      scoring.ts       Heuristic email scoring engine (shared by API + Worker)
      crypto.ts        AES-256-GCM encryptToken / decryptToken (shared by API + Worker)
config/
  pricing.json   Tier structure (Free/Pro/Business) + add-ons
infra/
  docker-compose.yml   Local dev: Postgres, Redis, API, Worker, Web
  Dockerfile.api       Per-service Dockerfile for API (Railway)
  Dockerfile.worker    Per-service Dockerfile for Worker (Railway)
  Dockerfile.web       Per-service Dockerfile for Web (Railway)
```

## Data Model (Prisma — 10 models)
- **User** — email (PK), provider, encrypted tokens, plan, sensitivity, shadow mode, preferences
- **MailConnection** — multi-provider multi-mailbox connections
- **Message** — cached email messages with headers JSON
- **Classification** — per-message scoring results (label, riskScore, action, reasons)
- **Action** — quarantine/release/block/allow with idempotency keys + undo payload
- **Feedback** — user label corrections for scoring calibration
- **Rule** — allow/block/VIP rules with matcher types (exact, contains, domain, regex)
- **AuditLog** — all events with metadata JSON
- **Subscription** — Stripe subscription state machine (active / past_due / grace / canceled)
- **AnalyticsEvent** — funnel/engagement tracking from web, extension, API

## Key Endpoints (API :4000)
All protected routes require `X-API-Key` header. Open routes: /health, /version, /docs, /auth/*, /webhooks/*, /events.

| Method | Path | Description |
|--------|------|-------------|
| GET | /health | Health check |
| GET | /version | API version + uptime |
| GET | /docs | Swagger UI |
| GET | /auth/google/start | Start Google OAuth |
| GET | /auth/google/callback | Google OAuth callback |
| GET | /auth/microsoft/start | Start Microsoft OAuth |
| GET | /auth/microsoft/callback | Microsoft OAuth callback |
| GET | /auth/status?email= | Connection status + settings (extension polling) |
| DELETE | /auth/disconnect | Disconnect all providers |
| GET | /mode?email= | Get shadow mode + sensitivity + preferences |
| POST | /mode | Set shadow mode, sensitivity, auto-quarantine, digest |
| POST | /mode/onboarding | Complete onboarding with protection presets |
| GET | /rules?email= | Get allow/block/VIP rules |
| POST | /rules/allow | Add to allowlist |
| POST | /rules/block | Add to blocklist |
| POST | /rules/vip | Add to VIP list |
| DELETE | /rules/:type/:id | Delete a rule |
| GET | /gmail/messages?email= | List + score inbox |
| POST | /gmail/batch-classify | Bulk score + quarantine |
| POST | /gmail/quarantine | Quarantine single message |
| POST | /gmail/undo | Restore from quarantine |
| POST | /classify | Score a single message (stores classification) |
| POST | /sync/start | Enqueue mailbox sync job |
| GET | /sync/status?email= | Sync status + queue depth |
| POST | /actions/quarantine | Quarantine with idempotency |
| POST | /actions/release | Release quarantined message |
| POST | /actions/block_sender | Block sender |
| POST | /actions/allow_sender | Allow sender |
| POST | /feedback | Submit label correction |
| POST | /events | Ingest analytics events |
| GET | /events/funnel?email= | Activation funnel metrics |
| GET | /events/kpis?email= | KPIs (threats blocked, FP rate, time saved) |
| POST | /billing/checkout | Create Stripe checkout session |
| GET | /billing/portal?email= | Billing portal URL |
| GET | /billing/subscription?email= | Current subscription info |
| GET | /billing/invoices?email= | Invoice list with PDF links |
| GET | /stats?email= | Dashboard stats |
| GET | /digest/audit?email= | Audit log |
| POST | /webhooks/stripe | Stripe webhook |
| POST | /ingest/:provider/:userId | Trigger async ingest |

## Pricing Tiers
| | Free (Shield Lite) | Pro ($15/mo) | Business ($39/mo/user) |
|---|---|---|---|
| Mailboxes | 1 | 3 | Unlimited |
| Daily scans | 100 | Unlimited | Unlimited |
| Auto-quarantine | - | Yes | Yes |
| VIP list | - | Yes | Yes |
| Digest reports | - | Daily/Weekly | Daily/Weekly |
| Anomaly detection | - | - | Yes |
| Priority support | - | - | Yes |

## Shadow Mode
Default ON for all new users. When ON, quarantine actions are logged but NOT executed on Gmail/Outlook. Users must explicitly disable to go live.

## Scoring Algorithm
Heuristic regex scoring (0.0-1.0 float, 0-100 riskScore):
- Phishing subject patterns: +0.35
- Phishing body patterns (password, SSN, credit card...): +0.30
- Spam subject patterns (free, urgent, winner...): +0.30
- Sender patterns (noreply, .ru/.cn/.xyz domains...): +0.25
- Reply-to domain mismatch: +0.20
- Link shorteners in body: +0.15
- Missing List-Unsubscribe header: +0.10
- Very short snippet: +0.05
- Blocklist match: +0.60
- VIP match: score = 0, label = 'vip' (highest priority)
- Allowlist match: score = 0, label = 'safe'

Labels: spam, phish, promo, safe, vip
Actions: allow, quarantine, review, digest
Threshold configurable per user via sensitivity slider (default 0.7).

## Worker Jobs
- **Ingest** (concurrency 3): Pull messages from Gmail/Outlook, enqueue classify jobs
- **Classify** (concurrency 5): Score messages, cache to DB, quarantine if above threshold
- **Digest** (concurrency 2): Generate daily/weekly digests per user, fan-out from scheduler
- **Anomaly** (concurrency 2): Reply-to change detection, domain similarity (Levenshtein), homoglyph checks
- **Token Refresh** (concurrency 1): Proactively refresh tokens expiring within 10 minutes

Scheduled repeatable jobs:
- Token refresh: every 5 minutes
- Digest generation: every 24 hours
- Anomaly scan: every 6 hours (business users only)

## Security
- **Token encryption**: AES-256-GCM at rest via `ENCRYPTION_KEY` env var. Format: `iv:tag:ciphertext`. Backward-compatible with unencrypted legacy tokens.
- **Request-ID**: Generated via `randomUUID()`, propagated in `x-request-id` header for distributed tracing.
- **PII redaction**: Email addresses and tokens redacted from Fastify request logs.
- **API key guard**: All non-public routes require `X-API-Key` header.
- **OAuth state**: HMAC-signed state parameter with httpOnly cookies.
- **Rate limiting**: 200 requests/minute via @fastify/rate-limit.
- **Security headers (API)**: @fastify/helmet.
- **Security headers (Web)**: `next.config.js` adds X-Frame-Options DENY, X-Content-Type-Options nosniff, Referrer-Policy, Permissions-Policy.
- **CORS**: Configurable allowed origins.

## Environment Variables (API)
See `apps/api/.env.example` for full list. Key additions:
- `ENCRYPTION_KEY` — 32-char key for AES-256-GCM token encryption
- `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` — Stripe billing
- `STRIPE_PRICE_PRO`, `STRIPE_PRICE_BUSINESS` — Stripe price IDs
- `APP_URL` — Frontend URL for OAuth callback redirects (default: `http://localhost:3000`)
- `LOG_LEVEL` — Fastify log level (default: info)

## Deployment
All services deploy to Railway via GitHub integration (auto-deploy on push to main).

- **API** → Railway service, Dockerfile: `infra/Dockerfile.api`
- **Web** → Railway service, Dockerfile: `infra/Dockerfile.web`
- **Worker** → Railway service, Dockerfile: `infra/Dockerfile.worker`
- **Extension** → Chrome Web Store (build with `pnpm build` in apps/extension)
- **DB** → Railway PostgreSQL plugin
- **Redis** → Railway Redis plugin

Each Railway service should have its Dockerfile path set in the service settings (e.g., `infra/Dockerfile.api`). Environment variables are configured per-service in the Railway dashboard.

## Frontend Design System
CSS utility classes defined in `apps/web/app/globals.css`:

- **Theming**: CSS variables switch via `data-theme="light"` / `data-theme="dark"` on `<html>`. Colors: `--bg`, `--surface`, `--text`, `--accent`, `--border`, `--muted`, etc.
- **Animations**: `animate-fade`, `animate-slide-up`, `animate-slide-down` with delay classes `.delay-1` through `.delay-6`
- **Text depth**: `.text-3d` (soft glow), `.text-emboss` (inset look), `.text-pop` (stat numbers), `.gradient-text` (clipped gradient), `.gradient-text-3d` (gradient + glow)
- **Cards & surfaces**: `.glass` (frosted surface), `.depth-card` (layered box-shadow), `.raised` (section elevation), `.hover-lift` (translateY on hover)
- **Backgrounds**: `.hero-grid` (CSS grid pattern)

All pages share a consistent nav (frosted glass, sticky, shield logo + "SiftMail" wordmark) and footer (logo + legal links).

## SEO
- **Meta tags**: Open Graph (`og:title`, `og:description`, `og:url`, `og:site_name`, `og:type`) and Twitter Card (`summary_large_image`) in `layout.tsx`
- **robots.txt**: Allows `/`, disallows `/dashboard` and `/api/`, links to sitemap. Generated by `app/robots.ts`.
- **sitemap.xml**: Lists 4 public pages (`/`, `/pricing`, `/privacy`, `/terms`). Generated by `app/sitemap.ts`.

## OAuth Flow
OAuth callbacks (Google + Microsoft) redirect to `/dashboard?connected=gmail&email=...` (or `connected=outlook`). The dashboard:
1. Auto-populates the email input from the query param
2. Loads the user's account (mode, rules, stats)
3. Shows a success toast ("Gmail connected successfully!")
4. Cleans up URL params via `history.replaceState`

The redirect destination is controlled by `APP_URL` env var (default: `http://localhost:3000`).

## Extension
- **API URL**: Configurable via `chrome.storage.local` key `siftmail_api_base` (default: `https://api.siftmail.app`). Both `popup.ts` and `background.ts` use `getApiBase()` to read this.
- **Content script**: Uses safe DOM construction (no `innerHTML`) to prevent XSS from classification data. MutationObserver + immediate initial injection for Gmail badge/inline actions.
- **Host permissions**: `https://mail.google.com/*` and `https://*.siftmail.app/*`.

## Google OAuth Verification Notes
For Play Store / Gmail API production access:
1. Use a verified domain for redirect URIs
2. Publish privacy policy at /privacy
3. Limit scopes to gmail.modify only (no drive, calendar etc.)
4. Shadow Mode demonstrates responsible data handling to reviewers

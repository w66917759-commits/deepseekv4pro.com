# deepseekv4pro.com
# DS-V4-HUUB

DeepSeek information hub + discounted Coding Plan store.

This repository powers a content-led site for DeepSeek news, guides,
benchmarks, localized SEO pages, and a small batch-based store for official API
keys. Content drives traffic; `/pricing` converts users into one-off Coding Plan
purchases.

Read this file before editing pricing, subscriptions, benchmarks, or DeepSeek
V4 content. The project is intentionally not a generic SaaS subscription
marketplace.

---

## Table of Contents

- [Product Model](#product-model)
- [Business Rules](#business-rules)
- [Tech Stack](#tech-stack)
- [Route Map](#route-map)
- [Project Layout](#project-layout)
- [Data Flow](#data-flow)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Supabase Data Model](#supabase-data-model)
- [Common Tasks](#common-tasks)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Things to Avoid](#things-to-avoid)
- [Data Sources](#data-sources)
- [License](#license)

---

## Product Model

The site serves three visitor intents:

1. **Read**: DeepSeek news, guides, use cases, FAQ, and localized articles.
2. **Compare**: model benchmarks, OpenClaw PinchBench rankings, model profiles,
   and an interactive arena.
3. **Buy**: a curated set of discounted Coding Plans for models where we have
   actual API-key inventory.

The commercial path is:

```text
SEO/content pages -> benchmarks/comparison -> /pricing -> checkout -> dashboard key delivery
```

DeepSeek is the headline product. Qwen, MiniMax, GLM, GPT, Claude, Gemini, Grok,
and other models can appear as comparison context, arena participants, or
secondary offers, but hero copy, CTA copy, and SEO positioning should lead with
DeepSeek unless a task explicitly says otherwise.

---

## Business Rules

### One-off product listings, not recurring SaaS subscriptions

- A Coding Plan is a single batch product listed while inventory is available.
- "Monthly" in the UI is the billing unit of the purchased plan window. Do not
  describe the business as recurring SaaS, a marketplace, or an unlimited
  subscription platform.
- Avoid copy that implies perpetual availability, enterprise SLAs, unlimited
  usage, or permanent stock.

### Only list what is in stock

- A model may appear as a purchasable `/pricing` card only when we have secured
  inventory for it.
- Inventory is represented by active `api_keys` rows with available user slots,
  optionally linked to a `subscription_plans.id`.
- If a model is not in stock, it can appear in benchmarks, guides, comparison
  content, or the Coming Soon teaser block. It must not appear as an active
  purchasable plan.

### Benchmarks are independent from purchasable plans

- `src/lib/benchmarks/arena-providers.ts` and related benchmark files define
  comparison/arena support.
- They do not imply that the model can be sold.
- Never add a plan because a model is popular, newly benchmarked, or wired into
  the arena. The gating question is always: "Do we have keys we can resell?"

### Keep plan data consistent

When adding, editing, restocking, or delisting a plan, keep these surfaces in
sync:

- `supabase/migrations/*.sql`: `subscription_plans` rows and key-stock schema.
- `api_keys` inventory: provider, plan linkage, capacity, status, base URL,
  model ID, and API format.
- `src/lib/site-content/pricing.ts`: pricing/support copy and static plan
  metadata.
- `src/components/subscriptions/SubscriptionCards.tsx`: Coming Soon cards only
  for models we intentionally want to tease.
- `src/lib/benchmarks/*`: only if the model also belongs in comparison content.
- Payment product configuration: Creem/PayPal product IDs and webhook handling.

---

## Tech Stack

- **Next.js 16** App Router with Turbopack
- **React 19**
- **TypeScript 5**
- **Tailwind CSS 4**
- **Recharts** for benchmark charts
- **Supabase** for plans, API-key pool, assignments, usage, subscriptions, news,
  and specs seed data
- **Clerk** for auth
- **Creem** and **PayPal** for hosted checkout and payment webhooks

Important implementation detail: `src/lib/supabase.ts` and
`src/lib/supabase-admin.ts` use lazy clients. Importing them without environment
variables is safe; calling them without the required values is not.

---

## Route Map

### Public content

- `/`: DeepSeek-led home page with hero, CTA cards, benchmarks, news, and sales
  framing.
- `/pricing`: store page. Purchasable cards come from `/api/plans` +
  `/api/stock`.
- `/benchmarks`: model comparison, OpenClaw PinchBench leaderboard, and arena.
- `/benchmarks/models/[id]`: model profile/detail pages.
- `/guides`, `/guides/[slug]`: English DeepSeek guides and comparisons.
- `/news/[slug]`: seeded DeepSeek news content.
- `/use-cases`, `/faq`, `/about`, `/contact`, `/documents`: supporting SEO and
  trust pages.
- `/{locale}`, `/{locale}/guides`, `/{locale}/guides/[slug]`,
  `/{locale}/[slug]`: localized SEO pages.

### Authenticated product areas

- `/chat`: DeepSeek chat surface.
- `/dashboard/subscription`: active plan and payment state.
- `/dashboard/api-keys`: delivered API-key assignments.
- `/admin/plans`, `/admin/keys`, `/admin/subscriptions`: internal operations.

### API routes

- `/api/plans`: active `subscription_plans`.
- `/api/stock`: available key slots aggregated by plan or provider.
- `/api/subscriptions/creem/create`: Creem checkout creation.
- `/api/subscriptions/create`: PayPal checkout creation.
- `/api/webhooks/creem`, `/api/webhooks/paypal`: payment webhook handlers.
- `/api/keys/me`, `/api/usage/me`, `/api/subscriptions/me`: user dashboard
  data.
- `/api/arena/stream`: multi-model arena streaming.
- `/api/cron/expire-subscriptions`: scheduled expiration guard.

---

## Project Layout

```text
src/
├── app/
│   ├── page.tsx                         # Home page
│   ├── pricing/page.tsx                  # Store landing page
│   ├── benchmarks/                       # Benchmark and model detail routes
│   ├── guides/                           # English SEO guides
│   ├── news/                             # News route backed by seed data
│   ├── [locale]/                         # Localized landing + article routes
│   ├── chat/                             # Chat surface
│   ├── dashboard/                        # User subscription/key views
│   ├── admin/                            # Internal plan/key/subscription ops
│   └── api/                              # Public, auth, payment, cron APIs
├── components/
│   ├── home/                             # Home shell, navbar, ticker, CTA
│   ├── benchmarks/                       # Arena, cards, leaderboard, scoring
│   ├── subscriptions/                    # Store cards + checkout button
│   ├── documents/                        # Official-docs browsing UI
│   ├── site/                             # Shared shell/footer/nav
│   └── ui/                               # Small reusable UI elements
├── lib/
│   ├── benchmarks/                       # Model profiles, details, arena config
│   ├── subscriptions/                    # Plans, key pool, payments, usage
│   ├── site-content/                     # Pricing, FAQ, contact, use-case copy
│   ├── i18n/                             # Locale metadata + localized articles
│   ├── documents/                        # Official-docs catalog/fetch helpers
│   ├── supabase.ts                       # Lazy public Supabase client
│   └── supabase-admin.ts                 # Lazy service-role Supabase client
├── middleware.ts                         # Clerk matcher, SEO-safe public pages
└── app/globals.css

supabase/migrations/                      # Schema + seed migrations
docs/SEO_INTERNATIONAL.md                 # International SEO operations notes
scripts/fetch_news.py                     # News helper script
```

---

## Data Flow

### Pricing page

```text
/pricing
  -> <SubscriptionCards />
  -> GET /api/plans
      -> subscription_plans where is_active = true
  -> GET /api/stock
      -> api_keys grouped by plan_id
      -> fallback grouping by provider for legacy keys
```

Plan cards are active only if the database returns active plans. Stock badges are
derived from available key capacity:

```text
available_slots = sum(max_users - current_users_count)
```

Keys with `status = active` can receive more assignments. Keys with
`status = full` still count toward total stock but have no available slots.

### Checkout and key delivery

```text
SubscribeButton
  -> /api/subscriptions/creem/create or /api/subscriptions/create
  -> create pending subscriptions row
  -> hosted checkout
  -> payment webhook
  -> activateSubscription()
  -> assign key(s) from api_keys
  -> user sees key(s) in /dashboard/api-keys
```

Single-model plans assign one least-loaded key per provider. Multi-model plans
assign all available keys linked to the plan.

### Auth and SEO

Clerk middleware only runs on API routes and auth pages. Public pages are left
outside the middleware matcher so crawlers can index content without auth
interstitials.

---

## Getting Started

### 1. Install dependencies

```bash
npm install
```

### 2. Create local environment

Create `.env.local` in the repository root. Use the exact variable names shown
below; a few existing names use non-standard casing and the code expects that
casing.

### 3. Run locally

```bash
npm run dev
```

Open <http://localhost:3000>.

### 4. Production-style checks

```bash
npm run build
npm run lint
```

Available npm scripts:

| Command | Purpose |
| --- | --- |
| `npm run dev` | Start the Next.js development server. |
| `npm run build` | Build the app for production. |
| `npm run start` | Start the production server after a build. |
| `npm run lint` | Run ESLint across the project. |

---

## Environment Variables

### Core app and Supabase

| Variable | Required for | Notes |
| --- | --- | --- |
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase APIs, dashboard, admin | Public project URL. |
| `NEXT_PUBLIC_SUPABASE_Anon_KEY` | Public Supabase client | Exact casing is currently used in code. |
| `NEXT_PUBLIC_SUPABASE_service_role_KEY` | Admin/server Supabase calls | Service-role key. Keep server-only in deployment settings. |
| `API_KEY_ENCRYPTION_SECRET` | Storing/revealing API keys | Used by `src/lib/subscriptions/encryption.ts`. |
| `ADMIN_USER_IDS` | Admin access | Comma-separated admin email addresses. |
| `CRON_SECRET` | `/api/cron/expire-subscriptions` | Sent by the scheduled caller. |
| `NEXT_PUBLIC_GOOGLE_SITE_VERIFICATION` | SEO verification | Optional metadata verification token. |

### Clerk

| Variable | Required for | Notes |
| --- | --- | --- |
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Sign-in/sign-up UI | Public Clerk key. |
| `CLERK_SECRET_KEY` | Server auth and admin checks | Required by API routes using `auth()`. |

### Payments

| Variable | Required for | Notes |
| --- | --- | --- |
| `NEXT_PUBLIC_PAYMENT_PROVIDER` | Checkout selection | `creem` or `paypal`; default is `creem`. |
| `CREEM_API_KEY` | Creem checkout | Required when using Creem. |
| `CREEM_WEBHOOK_SECRET` | Creem webhook verification | Required for webhook and redirect signature checks. |
| `CREEM_MODE` | Creem environment | Use `live` for production; anything else uses test API. |
| `CREEM_PRODUCT_ID` | Creem checkout fallback | Optional fallback product ID. |
| `CREEM_PRODUCT_ID_<PLAN_SLUG>` | Per-plan Creem products | Example: `CREEM_PRODUCT_ID_DEEPSEEK_MONTHLY`. |
| `PAYPAL_CLIENT_ID` | PayPal checkout | Required when using PayPal. |
| `PAYPAL_CLIENT_SECRET` | PayPal checkout | Required when using PayPal. |
| `PAYPAL_WEBHOOK_ID` | PayPal webhook verification | Required by PayPal webhook handling. |
| `PAYPAL_MODE` | PayPal environment | `live` for production; otherwise sandbox. |

### Arena and chat models

| Variable | Used by | Notes |
| --- | --- | --- |
| `DEEPSEEK_API_KEY` | `/chat`, arena DeepSeek provider | Native DeepSeek API. |
| `qwen_api_key` | Arena Qwen provider | DashScope OpenAI-compatible endpoint. |
| `glm_api_key` | Arena GLM provider | Zhipu OpenAI-compatible endpoint. |
| `minimax_api_key` | Arena MiniMax provider | Anthropic-compatible endpoint. |
| `GROK_api_key` | Arena Grok provider | xAI OpenAI-compatible endpoint. |
| `claude_api_key` | Arena Claude provider | Proxy-backed via `ARENA_PROXY_BASE_URL`. |
| `gpt_api_key` | Arena GPT provider | Proxy-backed via `ARENA_PROXY_BASE_URL`. |
| `gemini_api_key` | Arena Gemini provider | Proxy-backed via `ARENA_PROXY_BASE_URL`. |
| `ARENA_PROXY_BASE_URL` | Proxy-backed arena models | Defaults to the proxy URL in `arena-providers.ts`. |

---

## Supabase Data Model

Key tables and responsibilities:

| Table | Purpose |
| --- | --- |
| `subscription_plans` | Public plan metadata, price, limits, active status, display fields, plan type, included models. |
| `api_keys` | Encrypted provider keys, model/base URL/API format, capacity, status, optional `plan_id`. |
| `subscriptions` | User purchases and lifecycle state: pending, active, expired, cancelled, suspended. |
| `api_key_assignments` | Which user received which key for which subscription. |
| `usage_logs` | Per-user/provider request and token usage. |
| News/spec tables | DeepSeek V4 content seeded by migrations. |

Important migrations:

- `004_subscription_plans.sql`: base plan schema and seed plans.
- `005_api_keys.sql`: key inventory pool.
- `006_subscriptions.sql`: user subscription lifecycle.
- `007_api_key_assignments.sql`: key assignment tracking.
- `009_plan_display_fields.sql`: price comparison/display fields.
- `010_plan_limit_intervals.sql`: configurable request/token intervals.
- `012_api_keys_plan_id.sql`: links inventory to plans.
- `013_plan_multi_model.sql`: multi-model plan support.
- `014_seed_deepseek_v4_news.sql`: DeepSeek V4 news seed data.
- `015_seed_deepseek_v4_specs.sql`: DeepSeek V4 spec seed data.

---

## Common Tasks

### Add a new Coding Plan when inventory exists

1. Add a new migration under `supabase/migrations/`.
2. Insert or update the `subscription_plans` row with the correct slug, name,
   description, price, provider list, limits, display fields, `is_active`, and
   sort order.
3. Add encrypted API-key inventory through the admin flow or database operation
   so `/api/stock` reports available slots.
4. Link keys to the plan with `api_keys.plan_id` when the stock is plan-specific.
5. Add or update pricing/support copy in `src/lib/site-content/pricing.ts`.
6. Configure `CREEM_PRODUCT_ID_<PLAN_SLUG>` or PayPal product handling for the
   new listing.
7. Update `src/components/subscriptions/SubscriptionCards.tsx` only if the
   Coming Soon teaser should change.
8. Update `src/lib/benchmarks/*` only if the model should also appear in
   comparison content.
9. Run `npm run build` and `npm run lint`.

Do not publish an active plan row without available stock.

### Delist or pause a plan

1. Set `subscription_plans.is_active = false`.
2. Remove or update matching static pricing copy.
3. Disable, revoke, or relink obsolete `api_keys` rows as appropriate.
4. Add a Coming Soon teaser only if we are intentionally restocking that model.
5. Verify `/api/plans` no longer returns the plan.

### Restock an existing plan

1. Add new `api_keys` rows with the correct provider, base URL, model ID,
   `api_format`, `max_users`, and optional `plan_id`.
2. Confirm status starts as `active`.
3. Confirm `/api/stock` reports the expected available slots.
4. Confirm the pricing card copy still matches the actual product being sold.

### Add or change an arena model

1. Update `src/lib/benchmarks/models-config.ts` for model metadata.
2. Update `src/lib/benchmarks/arena-providers.ts` only if the model can be run
   in the live arena.
3. Update `model-profiles.ts`, `model-details.ts`, `showcase-data.ts`, or
   related benchmark files if the model needs profile/detail content.
4. Do not add a purchasable Coding Plan unless inventory exists.

### Update benchmark chart data

- `src/components/ModelBenchmarkChart.tsx` contains the home chart data.
- `src/components/benchmarks/OpenClawView.tsx` contains OpenClaw leaderboard
  display data.
- Keep profile/detail naming aligned when model names change.

### Add DeepSeek V4 news or specs

1. Add a new migration using `014_seed_deepseek_v4_news.sql` or
   `015_seed_deepseek_v4_specs.sql` as the template.
2. Keep copy DeepSeek-led and avoid unsupported model/version claims.
3. Run build/lint after the seed or content change.

### Add a locale

1. Update `LocaleCode` and `LOCALE_CODES` in `src/lib/i18n/locales.ts`.
2. Add localized landing metadata/copy in `src/lib/i18n/locales.ts`.
3. Add localized articles in `src/lib/i18n/articles.ts`.
4. Confirm sitemap and `hreflang` behavior. See `docs/SEO_INTERNATIONAL.md`.

---

## Verification

Run before shipping:

```bash
npm run build
npm run lint
```

Extra checks for pricing/subscription changes:

```bash
curl http://localhost:3000/api/plans
curl http://localhost:3000/api/stock
```

Manual checks:

- `/pricing` lists only active in-stock offerings as purchasable cards.
- Out-of-stock or not-yet-secured models appear only as Coming Soon or content.
- Checkout creates a pending subscription and redirects to the selected payment
  provider.
- Payment webhook activation assigns keys and updates subscription status.
- `/dashboard/api-keys` shows assigned keys for an active user.
- Admin pages still load for emails listed in `ADMIN_USER_IDS`.
- Public SEO pages do not trigger Clerk interstitials.

---

## Troubleshooting

### `Supabase is not configured`

The route called the lazy Supabase client without required env vars. Confirm:

- `NEXT_PUBLIC_SUPABASE_URL`
- `NEXT_PUBLIC_SUPABASE_Anon_KEY`
- `NEXT_PUBLIC_SUPABASE_service_role_KEY` for server/admin operations

### `/pricing` shows no plans

Check:

- `subscription_plans.is_active = true`
- `/api/plans` returns rows
- Supabase service-role env var is available to the server
- The plan was not delisted without updating static pricing copy

### A card shows zero stock

Check:

- Matching `api_keys` rows exist
- `api_keys.status` is `active` or `full`
- `max_users > current_users_count`
- `plan_id` matches the plan, or provider fallback matches the plan provider

### Checkout fails on Creem

Check:

- `NEXT_PUBLIC_PAYMENT_PROVIDER=creem`
- `CREEM_API_KEY`
- `CREEM_WEBHOOK_SECRET`
- `CREEM_PRODUCT_ID_<PLAN_SLUG>` or fallback `CREEM_PRODUCT_ID`
- The plan slug normalizes as expected, e.g. `deepseek-monthly` ->
  `CREEM_PRODUCT_ID_DEEPSEEK_MONTHLY`

### Checkout fails on PayPal

Check:

- `NEXT_PUBLIC_PAYMENT_PROVIDER=paypal`
- `PAYPAL_CLIENT_ID`
- `PAYPAL_CLIENT_SECRET`
- `PAYPAL_WEBHOOK_ID`
- `PAYPAL_MODE`

### Arena model returns missing-key errors

Check the exact env var name in `src/lib/benchmarks/arena-providers.ts`. Several
provider keys intentionally use lowercase names such as `qwen_api_key` and
`gpt_api_key`.

---

## Things to Avoid

- Do not invent pricing tiers such as "Pro", "Team", or "Enterprise".
- Do not promise 24/7 availability, uptime SLAs, unlimited requests, or
  perpetual stock.
- Do not add a plan because a model exists in benchmarks or the arena.
- Do not treat proxy-backed arena models as direct official keys.
- Do not generate URLs, product names, provider claims, or API endpoints that
  are not grounded in this repo.
- Do not update only one of database plan data, pricing copy, stock, payment
  config, and UI teaser state when changing a plan.

---

## Data Sources

- LiveBench: <https://livebench.ai/>
- OpenClaw PinchBench: internal leaderboard reproduced in
  `src/components/benchmarks/OpenClawView.tsx`
- Price references: <https://openrouter.ai/>
- International SEO operations: `docs/SEO_INTERNATIONAL.md`

---

## License

Private / internal.

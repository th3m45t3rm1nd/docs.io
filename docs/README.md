> Construction materials and services marketplace for Delhi NCR. Suppliers list products, customers buy, we facilitate. No warehouse. No inventory risk. Pure marketplace model.

---
## What this repo contains

Node.js REST API powering the Constructmart platform. Handles authentication, product catalog, supplier management, order processing, payments, search, and notifications.

This is a **supplier-to-customer marketplace** — similar to Amazon's third-party seller model. Suppliers onboard, list their products, and fulfill orders. We take a commission per transaction.

---

## Tech stack

| Layer         | Choice                  | Why                                                   |
| ------------- | ----------------------- | ----------------------------------------------------- |
| Language      | Node.js 20 + TypeScript | Team familiarity, rich Indian SDK ecosystem           |
| Framework     | Fastify                 | 2-3x faster than Express, built-in schema validation  |
| Primary DB    | PostgreSQL 15 (AWS RDS) | Relational data, GST joins, ACID compliance           |
| Cache / Queue | Redis (AWS ElastiCache) | Sessions, rate limiting, BullMQ job queues            |
| Search        | OpenSearch (AWS)        | Fuzzy search, Hindi phonetics, function_score ranking |
| Payments      | Razorpay                | UPI + cards + NEFT + EMI, best Indian coverage        |
| SMS           | MSG91                   | Better Indian delivery rates than Twilio              |
| WhatsApp      | Meta Business API       | Order alerts, delivery updates for contractors        |
| Storage       | AWS S3 + CloudFront     | Product images, invoices, supplier documents          |
| Infra         | AWS ap-south-1 (Mumbai) | India data residency, DPDP Act compliance             |
| IaC           | Terraform               | All infrastructure as code, no manual console changes |

Full reasoning behind each choice: [docs/decisions/](https://claude.ai/chat/docs/decisions/)

---

## Quick start (local development)

### Prerequisites

- Node.js 20+
- Docker and Docker Compose
- AWS CLI configured (for S3 access in dev)

### Setup

```bash
# 1. Clone the repo
git clone <repo-url>
cd constructmart

# 2. Set up environment variables
cp .env.example .env
# Fill in the values — see docs/environment.md for explanation of each

# 3. Start local services (PostgreSQL, Redis, OpenSearch)
docker-compose up -d

# 4. Install dependencies
npm install

# 5. Run database migrations
npm run migrate

# 6. Seed reference data (pincodes, categories, GST rates, cities)
npm run seed

# 7. Start the development server
npm run dev
# API is now running at http://localhost:3000

# 8. Verify everything works
curl http://localhost:3000/health
# Expected: { "status": "ok", "db": "ok", "redis": "ok", "search": "ok" }
```

### Useful dev commands

```bash
npm run dev          # Start with hot reload (tsx watch)
npm run build        # Compile TypeScript
npm run migrate      # Run pending migrations
npm run migrate:down # Rollback last migration
npm run seed         # Seed reference data
npm run test         # Run test suite
npm run test:watch   # Run tests in watch mode
npm run lint         # ESLint
npm run typecheck    # TypeScript type check without building
```

---

## Project structure

```
constructmart/
│
├── README.md                   ← you are here
├── CHANGELOG.md                ← what changed in each release
├── .env.example                ← all environment variables (no real values)
├── docker-compose.yml          ← local dev services
├── package.json
├── tsconfig.json
│
├── docs/                       ← all documentation
│   ├── architecture.md         ← system design, deployment overview
│   ├── database.md             ← schema guide, table purposes, key decisions
│   ├── api.md                  ← API conventions, auth flow, error formats
│   ├── environment.md          ← every environment variable explained
│   ├── onboarding.md           ← new engineer checklist (start here)
│   ├── decisions/              ← Architecture Decision Records (ADRs)
│   │   ├── ADR-001-tech-stack.md
│   │   ├── ADR-002-marketplace-model.md
│   │   ├── ADR-003-money-in-paise.md
│   │   ├── ADR-004-outbox-pattern.md
│   │   └── ADR-005-delhi-first-multiregion.md
│   └── runbooks/               ← what to do when things break
│       ├── deployment.md
│       ├── rollback.md
│       ├── database-migration.md
│       └── incidents/
│
├── src/
│   ├── app.ts                  ← Fastify app setup, plugin registration
│   ├── server.ts               ← entry point, starts HTTP server
│   ├── config/
│   │   ├── env.ts              ← validated env vars (zod schema)
│   │   └── constants.ts        ← app-wide constants
│   ├── db/
│   │   ├── client.ts           ← PostgreSQL connection pool
│   │   ├── migrations/         ← numbered SQL migration files
│   │   └── seeds/              ← reference data seed scripts
│   ├── modules/                ← feature modules (one folder per domain)
│   │   ├── auth/               ← OTP login, JWT, session management
│   │   ├── users/              ← user profile, addresses
│   │   ├── suppliers/          ← supplier onboarding, verification, dashboard APIs
│   │   ├── catalog/            ← products, variants, categories, pricing tiers
│   │   ├── search/             ← OpenSearch integration, query building
│   │   ├── cart/               ← cart management, order splitting logic
│   │   ├── orders/             ← order lifecycle, sub-orders, status management
│   │   ├── payments/           ← Razorpay integration, webhook handling, payouts
│   │   └── notifications/      ← WhatsApp, SMS, push notification dispatch
│   ├── workers/                ← background job processors (BullMQ)
│   │   ├── search-sync.ts      ← PostgreSQL → OpenSearch outbox processor
│   │   ├── notification.ts     ← async notification dispatch
│   │   ├── order-confirm.ts    ← post-payment order confirmation
│   │   └── payout-calc.ts      ← supplier payout calculation on delivery
│   └── shared/
│       ├── middleware/         ← auth, rate limiting, request logging
│       ├── errors/             ← error classes, error handler
│       ├── gst/                ← GST calculation service
│       └── utils/              ← shared helpers
│
└── infra/                      ← Terraform infrastructure code
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── README.md               ← how to apply infrastructure changes
```

---

## Environments

|Environment|URL|Branch|Deploy|
|---|---|---|---|
|Local|http://localhost:3000|any|`npm run dev`|
|Staging|https://api-staging.constructmart.in|`main` (auto)|On merge to main|
|Production|https://api.constructmart.in|`main` (manual)|Manual approval in GitHub Actions|

All environments run in **AWS ap-south-1 (Mumbai)**.

---

## Key architectural concepts

**Read these before touching the codebase:**

### 1. One customer order → many supplier sub-orders

When a customer checks out with items from 2 different suppliers, the system creates one parent `ORDER` (customer-facing, single payment) and two `SUB_ORDERS` (one per supplier). Each supplier only sees their own sub-order. Never modify parent order status directly — it is derived from sub-order statuses automatically.

### 2. All money is stored in paise

₹1 = 100 paise. Every monetary column in the database is an integer representing paise. ₹499.00 is stored as `49900`. This prevents floating point rounding bugs in financial calculations. See [ADR-003](ADR-003.md).

### 3. PostgreSQL is always the source of truth for products

OpenSearch is used for search and is eventually consistent with PostgreSQL. If there is ever a discrepancy, PostgreSQL wins. The outbox pattern keeps them in sync. See [ADR-004](ADR-004.md).

### 4. GST rates come from the database, never from code constants

The `gst_rates` table is seeded from the official GST rate schedule. Rates can change (GST Council updates them). Changing a rate requires a database update, not a code deploy.

### 5. Delhi NCR spans three states

"Delhi" for users means Delhi + Noida (UP) + Gurgaon (Haryana) + Faridabad (Haryana) + Ghaziabad (UP). Orders crossing state lines require e-way bills above ₹50,000. The order service handles this automatically using the `pincodes` reference table.

---

## API overview

Base URL: `/api/v1`

Authentication: Bearer JWT token in `Authorization` header. Tokens are obtained via the OTP login flow.

|Domain|Prefix|Description|
|---|---|---|
|Auth|`/auth`|OTP request, OTP verify, token refresh|
|Users|`/users`|Profile, addresses|
|Suppliers|`/suppliers`|Onboarding, verification, dashboard|
|Catalog|`/products`|Product listing, variants, categories|
|Search|`/search`|Full-text search with filters|
|Cart|`/cart`|Add, update, remove items|
|Orders|`/orders`|Checkout, order history, status|
|Payments|`/payments`|Razorpay callbacks, payout history|

Full API conventions and request/response formats: [docs/api.md](https://claude.ai/chat/docs/api.md)

---

## New to this project?

Start with [docs/onboarding.md](https://claude.ai/chat/docs/onboarding.md) — it has a day-by-day checklist and explains the non-obvious parts of the codebase.

---

## Contributing

- All changes via pull request. No direct pushes to `main`.
- Every PR must pass the test suite and TypeScript type check.
- Migration files must be backward-compatible with the previous app version (rolling deploys mean both versions run briefly together).
- Document significant decisions as ADRs in `docs/decisions/`.
- Update `CHANGELOG.md` with every release.

---

## License

Private and confidential. All rights reserved.
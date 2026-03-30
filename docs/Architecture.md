
This document explains how the Constructmart backend is structured, how a request flows through the system, and how all the pieces connect. Read this before reading any code.

---

## What we are building

A **supplier-to-customer construction materials marketplace** for Delhi NCR. Think Amazon third-party sellers, but for cement, steel, tiles, hardware, and construction services.

Three types of humans use the platform:

|Actor|What they do|
|---|---|
|**Customer**|Browses products, places orders, tracks delivery|
|**Supplier**|Lists products, manages stock, fulfills orders, receives payouts|
|**Admin**|Verifies suppliers, manages catalog, handles disputes|

---

## System overview

```
                        ┌─────────────────────────────────────┐
                        │         AWS ap-south-1 Mumbai        │
                        │                                      │
  Customer / Supplier   │   CloudFront CDN                     │
  (browser or app)  ───►│       │                              │
                        │   ALB (Application Load Balancer)    │
                        │       │                              │
                        │   EC2 API Servers (Node.js)          │
                        │    │        │        │               │
                        │  RDS      Redis   OpenSearch         │
                        │  (PG)   (Cache)   (Search)          │
                        │                                      │
                        │   EC2 Background Workers             │
                        │    │        │        │        │      │
                        │  Search  Notif   Order   Payout     │
                        │  Sync    Worker  Confirm  Calc      │
                        │                                      │
                        │   S3 (images, invoices, docs)        │
                        └─────────────────────────────────────┘
                                        │
                        ┌───────────────┼───────────────┐
                        │               │               │
                    Razorpay         MSG91 /        NIC API
                   (payments)       WhatsApp       (e-way bills)
```

---

## Request flow — customer browses and searches

```
User types "sariya 8mm" in search bar
    │
    ▼
CloudFront (CDN edge, Mumbai)
    │  Static assets served from edge
    │  API requests pass through to ALB
    ▼
Application Load Balancer
    │  Routes to a healthy EC2 API server
    ▼
API Server — search module
    │
    ├─► Redis: check for cached result (TTL 60s for popular queries)
    │       └─ Cache hit → return immediately
    │
    └─► OpenSearch: full-text query with function_score ranking
            │  Fuzzy match, synonym expansion, phonetic analysis
            │  Boosted by: sale status, stock availability, rating
            ▼
        Results returned to client
        (OpenSearch never used for checkout pricing —
         always read from PostgreSQL for financial data)
```

---

## Request flow — customer places an order

This is the most complex flow in the system. Read carefully.

```
Customer submits checkout
    │
    ▼
API Server — cart module
    │
    ├─ Read cart items from Redis (session cart)
    ├─ For each item: verify current price + stock from PostgreSQL
    │  (never trust cached/search prices for checkout)
    ├─ Group items by supplier_id
    ├─ Calculate GST per line item (read gst_rates table by HSN code)
    ├─ Determine IGST vs CGST+SGST per supplier
    │  (compare warehouse state_code vs delivery pincode state_code)
    └─ Calculate platform commission per sub-order
    │
    ▼
PostgreSQL transaction (atomic)
    ├─ INSERT INTO orders (parent order, customer-facing)
    ├─ INSERT INTO sub_orders (one per supplier)
    ├─ INSERT INTO order_items (line items per sub-order)
    ├─ RESERVE stock in supplier_products (decrement available qty)
    └─ INSERT INTO search_sync_outbox (stock change → update search)
    │
    ▼
Razorpay — create payment order
    │  Returns payment_order_id to client
    ▼
Customer completes payment (UPI / card / net banking)
    │
    ▼
Razorpay webhook → POST /payments/webhook
    │
    ├─ Verify webhook signature (HMAC-SHA256)
    ├─ Check search_sync_outbox for duplicate (idempotency)
    ├─ UPDATE orders SET status = 'confirmed'
    ├─ UPDATE sub_orders SET status = 'confirmed'
    └─ Enqueue order-confirm job in Redis (BullMQ)
    │
    ▼
order-confirm worker (background)
    ├─ Send WhatsApp notification to customer
    ├─ Send WhatsApp notification to each supplier
    ├─ Generate e-way bill if interstate + value > ₹50,000
    └─ Generate GST invoice PDF → upload to S3
```

---

## Background workers

Workers run on a separate EC2 instance from the API servers. They consume jobs from Redis queues via BullMQ.

|Worker|Queue|Trigger|Responsibility|
|---|---|---|---|
|`search-sync`|`q:search-sync`|Product/stock change in PG|Polls outbox, syncs to OpenSearch|
|`order-confirm`|`q:order-confirm`|Payment webhook captured|Notifications, e-way bill, invoice|
|`notification`|`q:notification`|Various|WhatsApp, SMS, push dispatch|
|`payout-calc`|`q:payout-calc`|Sub-order marked delivered|Calculate and record supplier payout|

**Worker reliability rules:**

- Every job is idempotent — running it twice produces the same result
- Failed jobs retry up to 5 times with exponential backoff
- Dead-lettered jobs (5 failures) trigger a PagerDuty / email alert
- Worker health is monitored — if a worker stops consuming, queue depth alert fires within 5 minutes

---

## Data stores and their roles

### PostgreSQL (AWS RDS Multi-AZ)

The **source of truth** for everything. If any other system disagrees with PostgreSQL, PostgreSQL wins.

- All user, supplier, product, order, payment, and payout data
- GST rates, pincode reference data, city configuration
- `search_sync_outbox` table for OpenSearch consistency
- Read replica for reporting queries and recommendation batch jobs (never run heavy analytics on the primary)

Backup: automated daily snapshots, 7-day retention, point-in-time recovery enabled. Multi-AZ means automatic failover in under 60s if the primary instance fails.

### Redis (AWS ElastiCache)

Used for **ephemeral, fast-access data** only. Never use Redis as a primary store for anything that cannot be reconstructed from PostgreSQL.

|Key pattern|TTL|Purpose|
|---|---|---|
|`session:{token}`|7 days|JWT session data|
|`cart:{user_id}`|24 hours|Active cart items|
|`search:{query_hash}`|60 seconds|Search result cache|
|`rate:{ip}`|1 minute|Rate limit counters|
|`otp:{phone}`|10 minutes|Login OTP|

BullMQ job queues also live in Redis. Queue keys have no TTL — BullMQ manages their lifecycle.

### OpenSearch (AWS managed)

Used **only for search**. Eventually consistent with PostgreSQL via the outbox pattern (see ADR-004).

One index: `products-v1`

Each document represents one supplier listing (one row from `supplier_products` joined with its `product` and `category`). The same physical product listed by 5 suppliers = 5 documents.

Do not use OpenSearch for:

- Checkout price verification (use PostgreSQL)
- Stock reservation (use PostgreSQL)
- Order history (use PostgreSQL)
- Any financial calculation (use PostgreSQL)

### S3 (AWS)

Object storage for binary files. Three buckets:

|Bucket|Contents|Access|
|---|---|---|
|`constructmart-media`|Product images uploaded by suppliers|Public via CloudFront|
|`constructmart-invoices`|GST invoice PDFs, e-way bill PDFs|Private, signed URLs|
|`constructmart-docs`|Supplier verification documents (GSTIN cert, PAN)|Private, admin only|

---

## External integrations

### Razorpay (payments)

Used for: payment order creation, UPI/card/net banking collection, webhook-based payment confirmation, payout to supplier bank accounts.

Integration points:

- `POST /v1/orders` — create payment order at checkout
- `POST /payments/webhook` — receive payment events
- Razorpay X (payout API) — transfer to supplier bank account

Webhook verification: every incoming webhook is verified using HMAC-SHA256 signature with the Razorpay webhook secret. Unverified webhooks are rejected with 400.

### MSG91 (SMS)

Used for: OTP delivery, order status SMS for customers who don't have WhatsApp.

Fallback: if WhatsApp delivery fails, MSG91 SMS is sent automatically by the notification worker.

### Meta WhatsApp Business API

Used for: order confirmation, delivery updates, payout notifications, low stock alerts to suppliers.

WhatsApp has significantly higher open rates than SMS for the contractor and supplier segment. It is the primary notification channel.

Note: Meta Business API approval takes 2-4 weeks. Start the application process before launch.

### NIC E-way Bill API

Used for: generating e-way bills for interstate orders above ₹50,000 in value.

Called by the `order-confirm` worker after payment capture. The e-way bill number is stored on the `sub_orders` table and included in the supplier's shipment documents.

---

## Environments

|Environment|Purpose|Infrastructure|
|---|---|---|
|**Local**|Development|Docker Compose on developer laptop|
|**Staging**|Pre-production testing, QA|AWS ap-south-1, mirrors production|
|**Production**|Live|AWS ap-south-1|

### Local development

`docker-compose up -d` starts:

- PostgreSQL 15 on port 5432
- Redis on port 6379
- OpenSearch on port 9200

Migrations and seeds run automatically on first start. No real API keys needed locally — all integrations have test/mock modes configured via environment variables.

### Staging

Identical infrastructure to production at smaller instance sizes. Connected to Razorpay test mode, MSG91 test sender, and a staging WhatsApp number.

Auto-deploys on every merge to `main` via GitHub Actions.

Staging database is a weekly anonymised copy of production:

- Real product catalog and category structure
- Fake user names, phone numbers, and addresses
- Real order structures but zeroed financial amounts

### Production

Manual deploy approval required in GitHub Actions. Two engineers must approve (currently: engineer + founder).

All production deployments are rolling — zero downtime. The API server instances are updated one at a time while the load balancer keeps traffic flowing.

---

## Security

### Network

- API servers and databases live in private VPC subnets
- No database or Redis instance has a public IP address
- Only the ALB and CloudFront are publicly reachable
- All inter-service communication is within the VPC

### Secrets

- All secrets (database passwords, API keys, JWT secret) live in AWS Secrets Manager
- EC2 instances access Secrets Manager via IAM role — no credentials in environment variables or code
- `.env` files are only used locally for development
- `.env` is in `.gitignore` and must never be committed

### Authentication

- Phone number + OTP login only (no passwords for customers)
- JWTs signed with RS256 (asymmetric) — public key can be shared with future microservices without sharing the secret
- Token expiry: 7 days for customers, 24 hours for suppliers and admins (higher privilege = shorter session)
- All admin routes require a separate admin JWT scope — a supplier JWT cannot access admin endpoints even if the user has admin role in the database

### Rate limiting

- Redis-based, per IP and per user token
- OTP endpoint: 3 requests per phone number per 10 minutes
- Search endpoint: 60 requests per IP per minute
- Checkout endpoint: 10 requests per user per minute
- Admin endpoints: 30 requests per token per minute

---

## Monitoring and alerting

|Tool|Purpose|
|---|---|
|CloudWatch|AWS infrastructure metrics, RDS/Redis/EC2 health|
|Datadog|Application metrics, custom dashboards, APM traces|
|Sentry|Runtime error tracking, error grouping, release tracking|
|PagerDuty|On-call alerting for P0/P1 incidents|

### Alert thresholds (P1 — wake someone up)

- API error rate > 5% for 5 consecutive minutes
- Payment webhook processing lag > 2 minutes
- Search sync outbox queue depth > 500 unprocessed rows
- RDS CPU > 80% for 10 minutes
- Any worker dead-letter queue receives a job

### Runbooks

For each alert there is a runbook in `docs/runbooks/`. Before any incident, read the relevant runbook.

---

## Capacity at launch

Initial infrastructure is deliberately minimal. Scale up only when metrics show actual need.

|Service|Launch size|Upgrade trigger|
|---|---|---|
|API EC2|1× t3.medium|CPU > 70% sustained|
|RDS|db.t3.medium, Multi-AZ|CPU > 60% or query p99 > 200ms|
|ElastiCache|cache.t3.micro|Memory > 70%|
|OpenSearch|1× t3.medium.search|Query p99 > 100ms|
|Worker EC2|1× t3.small|Queue depth growing consistently|

Estimated monthly AWS cost at launch: ₹35,000–40,000.

---

## What this architecture does not include (yet)

These are intentionally deferred and should not be built until the business justifies them:

- **Microservices:** Everything runs as one Fastify monolith. Split into services only when a specific service needs independent scaling or deployment frequency.
- **ML recommendation engine:** Phase 1 recommendations are rule-based SQL queries. Phase 2 (collaborative filtering) starts at ~5,000 orders. See `docs/recommendations.md` when that work begins.
- **Native mobile app:** PWA-first. Native iOS/Android when PWA limitations are actually felt by users.
- **Multi-region:** Delhi NCR only at launch. See ADR-005.
- **Kubernetes:** EC2 with a load balancer is sufficient for years of growth. Kubernetes adds operational overhead that a small team cannot afford.

Every environment variable used by the application is documented here. No variable should exist in `.env.example` without an entry in this file explaining what it is and where to get it.

**Rule:** Never commit real values to the repository. `.env` is in `.gitignore`. In staging and production, all secrets are stored in AWS Secrets Manager and injected at runtime — never in environment variables on the EC2 instance.

---

## How to get values for local development

1. Copy `.env.example` to `.env`
2. The Docker Compose services (PostgreSQL, Redis, OpenSearch) are pre-configured with the values already in `.env.example` — you do not need to change these for local dev
3. Ask the team lead for: Razorpay test keys, MSG91 test key, JWT secret, and S3 credentials

---

## Application

### `NODE_ENV`

**Values:** `development` | `staging` | `production`  
**Required:** Yes  
**Notes:**

- `development` — enables console OTP logging, disables rate limiting, enables detailed error responses
- `staging` — mirrors production behaviour but uses test credentials for all external services
- `production` — strict mode, no debug output

### `PORT`

**Values:** Any valid port number  
**Default:** `3000`  
**Notes:** The port the Fastify HTTP server listens on. The ALB forwards to this port. Do not change in production without updating the ALB target group.

### `API_VERSION`

**Values:** `v1`  
**Default:** `v1`  
**Notes:** Prefixes all API routes as `/api/v1/...` Increment when making breaking API changes.

### `LOG_LEVEL`

**Values:** `trace` | `debug` | `info` | `warn` | `error`  
**Default:** `info` (production), `debug` (development)  
**Notes:** Uses Pino logger. `debug` logs every incoming request and database query. Do not use `debug` in production — it generates significant log volume and may log sensitive request bodies.

---

## Database

### `DATABASE_URL`

**Format:** `postgresql://user:password@host:5432/dbname`  
**Required:** Yes  
**Notes:** Connection string for the primary RDS instance. All write operations and real-time read operations use this. Local default: `postgresql://postgres:postgres@localhost:5432/constructmart`

### `DATABASE_REPLICA_URL`

**Format:** `postgresql://user:password@host:5432/dbname`  
**Required:** No (falls back to `DATABASE_URL` if not set)  
**Notes:** Connection string for the RDS read replica. Used by: batch analytics jobs, recommendation engine, reporting queries. Never use the primary for these. In local development this is not set — reads and writes both go to the same local instance.

### `DATABASE_POOL_MIN`

**Default:** `2`  
**Notes:** Minimum connections in the PostgreSQL connection pool.

### `DATABASE_POOL_MAX`

**Default:** `10`  
**Notes:** Maximum connections in the pool. Each EC2 instance uses up to this many connections. With 3 API server instances, total max connections to RDS = 3 × 10 = 30. RDS `t3.medium` supports up to 170 connections — this headroom is intentional.

---

## Redis

### `REDIS_URL`

**Format:** `redis://host:6379`  
**Required:** Yes  
**Notes:** Connection string for ElastiCache Redis. Local default: `redis://localhost:6379`

### `REDIS_KEY_PREFIX`

**Default:** `cm:`  ￼￼Environment variables
**Notes:** All Redis keys are prefixed with this to avoid collisions if the Redis instance is shared. Change per environment (`cm-staging:`, `cm-prod:`).

### `SESSION_TTL_SECONDS`

**Default:** `604800` (7 days)  
**Notes:** How long a JWT session stays valid in Redis. Customers: 7 days. Suppliers and admins have shorter sessions configured separately in the auth service.

---

## OpenSearch

### `OPENSEARCH_URL`

**Format:** `https://host:9200`  
**Required:** Yes  
**Notes:** AWS OpenSearch endpoint URL. Local default: `http://localhost:9200`

### `OPENSEARCH_USERNAME`

**Required:** In staging and production only  
**Notes:** AWS OpenSearch fine-grained access control username. Not used locally (local OpenSearch has no auth).

### `OPENSEARCH_PASSWORD`

**Required:** In staging and production only  
**Notes:** AWS OpenSearch fine-grained access control password. Stored in AWS Secrets Manager in production.

### `OPENSEARCH_INDEX_PREFIX`

**Default:** `cm`  
**Notes:** Product search index is named `{prefix}-products-v1`. Use different prefixes per environment to avoid index collisions if environments share a cluster.

---

## Authentication

### `JWT_SECRET`

**Required:** Yes  
**Notes:** Secret key for signing JWT tokens (RS256). In production this should be a 256-bit random string. Stored in AWS Secrets Manager. Rotating this secret invalidates all active sessions — coordinate carefully. Generate locally with: `openssl rand -hex 32`

### `JWT_EXPIRY_CUSTOMER`

**Default:** `7d`  
**Notes:** JWT expiry for customer tokens.

### `JWT_EXPIRY_SUPPLIER`

**Default:** `24h`  
**Notes:** JWT expiry for supplier tokens. Shorter than customer because suppliers have write access to catalog and order management.

### `JWT_EXPIRY_ADMIN`

**Default:** `8h`  
**Notes:** JWT expiry for admin tokens. Shortest because admin has the highest privilege level.

---

## Payments — Razorpay

### `RAZORPAY_KEY_ID`

**Format:** `rzp_test_...` (test) or `rzp_live_...` (production)  
**Required:** Yes  
**Notes:** Razorpay API key. Use test keys in local and staging. Never use live keys in non-production environments. Get test keys from Razorpay dashboard → Settings → API Keys.

### `RAZORPAY_KEY_SECRET`

**Required:** Yes  
**Notes:** Razorpay API secret. Stored in AWS Secrets Manager in production. Never log this value.

### `RAZORPAY_WEBHOOK_SECRET`

**Required:** Yes  
**Notes:** Used to verify Razorpay webhook signatures (HMAC-SHA256). Set in Razorpay dashboard → Settings → Webhooks. Every incoming webhook is rejected if the signature does not match. Never disable this check.

### `RAZORPAY_PAYOUT_ACCOUNT_NUMBER`

**Required:** In staging and production  
**Notes:** Your Razorpay X business account number from which supplier payouts are initiated. Not needed for local dev as payouts are mocked.

---

## Notifications — MSG91

### `MSG91_AUTH_KEY`

**Required:** Yes  
**Notes:** MSG91 API authentication key. Used for OTP SMS and transactional SMS. Get from MSG91 dashboard → API. In development, OTPs are logged to console instead of being sent — this key is still required but no SMS credits are consumed unless `NODE_ENV=staging` or `production`.

### `MSG91_SENDER_ID`

**Default:** `CNSTRM`  
**Notes:** 6-character SMS sender ID. Must be registered with MSG91 and TRAI. The default is a placeholder — register your actual sender ID before going live.

### `MSG91_OTP_TEMPLATE_ID`

**Required:** Yes  
**Notes:** MSG91 template ID for OTP SMS. Templates must be pre-approved by MSG91 and follow TRAI DLT guidelines. Get from MSG91 dashboard → SMS → Templates.

### `MSG91_ORDER_TEMPLATE_ID`

**Required:** Yes  
**Notes:** MSG91 template ID for order confirmation SMS.

---

## Notifications — WhatsApp Business API

### `WHATSAPP_ACCESS_TOKEN`

**Required:** In staging and production  
**Notes:** Meta WhatsApp Business API access token. Long-lived token generated from Meta Business Manager. Expires every 60 days — set a calendar reminder to rotate it. Not required locally (WhatsApp messages are logged to console).

### `WHATSAPP_PHONE_NUMBER_ID`

**Required:** In staging and production  
**Notes:** The WhatsApp Business phone number ID from Meta Business Manager. Not the phone number itself — the internal Meta ID for your business phone number.

### `WHATSAPP_BUSINESS_ACCOUNT_ID`

**Required:** In staging and production  
**Notes:** Meta WhatsApp Business Account ID. Used for template management API calls.

---

## Storage — AWS S3

### `AWS_REGION`

**Default:** `ap-south-1`  
**Notes:** AWS region for all AWS services. Always Mumbai for data residency compliance.

### `AWS_ACCESS_KEY_ID`

**Required:** In local development only  
**Notes:** AWS IAM access key for local development. In staging and production, EC2 instances use IAM roles instead of access keys — this variable is not set on those environments.

### `AWS_SECRET_ACCESS_KEY`

**Required:** In local development only  
**Notes:** AWS IAM secret key for local development. Same caveat as `AWS_ACCESS_KEY_ID`.

### `S3_BUCKET_MEDIA`

**Default:** `constructmart-media-{env}`  
**Notes:** S3 bucket for product images uploaded by suppliers. Served publicly via CloudFront. All objects are public-read.

### `S3_BUCKET_INVOICES`

**Default:** `constructmart-invoices-{env}`  
**Notes:** S3 bucket for generated GST invoice PDFs and e-way bill documents. Private — accessed via signed URLs with 1-hour expiry.

### `S3_BUCKET_DOCS`

**Default:** `constructmart-docs-{env}`  
**Notes:** S3 bucket for supplier verification documents (GSTIN certificate, PAN, trade licence). Private — accessible by admin users only via signed URLs.

### `CLOUDFRONT_DOMAIN`

**Format:** `https://xxxx.cloudfront.net`  
**Required:** In staging and production  
**Notes:** CloudFront distribution domain for serving product images. Product image URLs in API responses use this domain, not the S3 bucket URL directly. Not set locally — local dev serves images directly from the local filesystem or a mock S3.

---

## GST and compliance

### `NIC_EWAY_API_URL`

**Default:** `https://einvoice1-uat.nic.in` (staging)  
**Production:** `https://einvoice1.nic.in`  
**Notes:** NIC e-way bill API endpoint. Use the UAT (test) URL in local and staging — the production URL generates real legal documents.

### `NIC_EWAY_USERNAME`

**Required:** In staging and production  
**Notes:** NIC portal username for e-way bill API access. Registered via the GST portal with your GSTIN.

### `NIC_EWAY_PASSWORD`

**Required:** In staging and production  
**Notes:** NIC portal password. Stored in AWS Secrets Manager.

### `PLATFORM_GSTIN`

**Required:** Yes  
**Notes:** The platform company's GST registration number. Appears on all invoices issued by the platform. Format: `29XXXXX1234X1ZX` (15-character alphanumeric).

### `PLATFORM_STATE_CODE`

**Required:** Yes  
**Notes:** GST state code for the platform's registered business address. Used to determine IGST vs CGST+SGST on marketplace commission invoices to suppliers.

---

## Monitoring

### `SENTRY_DSN`

**Required:** In staging and production  
**Notes:** Sentry Data Source Name for error tracking. Get from Sentry → Project Settings → Client Keys. Not set locally — errors are logged to console only.

### `DATADOG_API_KEY`

**Required:** In staging and production  
**Notes:** Datadog API key for metrics and APM traces. Get from Datadog → Organisation Settings → API Keys.

### `DATADOG_SERVICE_NAME`

**Default:** `constructmart-api`  
**Notes:** Service name tag applied to all Datadog traces and metrics. Use `constructmart-workers` for the worker process.

---

## Feature flags

### `FEATURE_WHATSAPP_ENABLED`

**Values:** `true` | `false`  
**Default:** `false`  
**Notes:** Master switch for WhatsApp notifications. Set to `false` until Meta Business API is approved. When `false`, all WhatsApp notifications fall back to SMS.

### `FEATURE_EWAY_BILL_ENABLED`

**Values:** `true` | `false`  
**Default:** `false`  
**Notes:** Master switch for automatic e-way bill generation. Set to `false` until NIC API credentials are obtained and tested. When `false`, e-way bill generation is skipped and ops must generate manually.

### `FEATURE_PAYOUTS_ENABLED`

**Values:** `true` | `false`  
**Default:** `false`  
**Notes:** Master switch for automatic supplier payouts via Razorpay X. Set to `false` until payout timing policy is finalised and Razorpay X KYC is complete. When `false`, payout records are created but money is not transferred automatically — ops initiates payouts manually.

---

## Complete `.env.example`

```bash
# Application
NODE_ENV=development
PORT=3000
API_VERSION=v1
LOG_LEVEL=debug

# Database
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/constructmart
DATABASE_REPLICA_URL=
DATABASE_POOL_MIN=2
DATABASE_POOL_MAX=10

# Redis
REDIS_URL=redis://localhost:6379
REDIS_KEY_PREFIX=cm:
SESSION_TTL_SECONDS=604800

# OpenSearch
OPENSEARCH_URL=http://localhost:9200
OPENSEARCH_USERNAME=
OPENSEARCH_PASSWORD=
OPENSEARCH_INDEX_PREFIX=cm

# Auth
JWT_SECRET=replace-with-random-32-byte-hex-string
JWT_EXPIRY_CUSTOMER=7d
JWT_EXPIRY_SUPPLIER=24h
JWT_EXPIRY_ADMIN=8h

# Razorpay
RAZORPAY_KEY_ID=rzp_test_replace_me
RAZORPAY_KEY_SECRET=replace_me
RAZORPAY_WEBHOOK_SECRET=replace_me
RAZORPAY_PAYOUT_ACCOUNT_NUMBER=

# MSG91
MSG91_AUTH_KEY=replace_me
MSG91_SENDER_ID=CNSTRM
MSG91_OTP_TEMPLATE_ID=replace_me
MSG91_ORDER_TEMPLATE_ID=replace_me

# WhatsApp
WHATSAPP_ACCESS_TOKEN=
WHATSAPP_PHONE_NUMBER_ID=
WHATSAPP_BUSINESS_ACCOUNT_ID=

# AWS S3
AWS_REGION=ap-south-1
AWS_ACCESS_KEY_ID=replace_me
AWS_SECRET_ACCESS_KEY=replace_me
S3_BUCKET_MEDIA=constructmart-media-dev
S3_BUCKET_INVOICES=constructmart-invoices-dev
S3_BUCKET_DOCS=constructmart-docs-dev
CLOUDFRONT_DOMAIN=

# GST / Compliance
NIC_EWAY_API_URL=https://einvoice1-uat.nic.in
NIC_EWAY_USERNAME=
NIC_EWAY_PASSWORD=
PLATFORM_GSTIN=replace_me
PLATFORM_STATE_CODE=07

# Monitoring
SENTRY_DSN=
DATADOG_API_KEY=
DATADOG_SERVICE_NAME=constructmart-api

# Feature flags
FEATURE_WHATSAPP_ENABLED=false
FEATURE_EWAY_BILL_ENABLED=false
FEATURE_PAYOUTS_ENABLED=false
```
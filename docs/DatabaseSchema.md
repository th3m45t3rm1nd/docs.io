This document explains the purpose of every table, the reasoning behind key design decisions, and the rules every engineer must follow when writing queries or migrations.

For the full ERD, see: [link to dbdiagram.io — add this once created]  
For migration files, see: `src/db/migrations/`

---

## Rules every engineer must follow

These are non-negotiable. Read before writing any query or migration.

**1. Never write raw ALTER TABLE on any environment.** Always write a migration file. Every schema change is versioned, reviewed via pull request, and applied consistently across local, staging, and production.

**2. All money is stored in paise.** ₹1 = 100 paise. Every monetary column is an integer suffixed with `_paise`. See ADR-003 for the full reasoning and code examples.

**3. PostgreSQL is always the source of truth.** OpenSearch is for search only and is eventually consistent. Checkout pricing, stock reservation, and all financial calculations always read from PostgreSQL directly.

**4. Never query the primary RDS instance for analytics.** Heavy reporting queries and recommendation batch jobs must use the read replica. Connection strings are separate environment variables (`DATABASE_URL` vs `DATABASE_REPLICA_URL`).

**5. Every migration must be backward-compatible.** Rolling deploys mean the old version of the app and the new version run simultaneously against the same database for ~60 seconds. A migration that renames or drops a column used by the old version will break live traffic. Safe pattern: add column → deploy new code → drop old column in a separate migration.

**6. Always include `city_id` in product and listing queries.** Every product query must be scoped to a city. Use the shared query helper `db.catalog.forCity(cityId)` — never write `SELECT * FROM products` without a city filter.

---

## Core design decisions

### Monetary values in paise

See [ADR-003](./decisions/ADR-003). Short version: all money is integers in paise. ₹499.00 = `49900`. Column names always end in `_paise`.

### City-aware schema from day one

Every product, supplier listing, and service listing carries a `city_id`. At launch the only value is `delhi-ncr`. When a new city launches, one row is inserted into `cities` and suppliers onboard for that city. See ADR-005.

### One customer order → many supplier sub-orders

A customer cart with items from 3 suppliers creates:

- 1 row in `orders` (customer-facing, single payment)
- 3 rows in `sub_orders` (one per supplier)
- N rows in `order_items` (line items, linked to sub_orders)

Never set `orders.status` directly. It is derived from the statuses of its `sub_orders` by the order service.

### Products vs supplier_products

`products` — the shared product definition. Name, description, HSN code, category, attributes. Created by admin or supplier, approved before going live. One row per unique product.

`supplier_products` — the listing. Who sells it, at what price, how much stock, what lead time. One row per supplier per product. The same cement product can have 10 rows here from 10 suppliers.

Think of `products` as the "what" and `supplier_products` as the "who sells it and at what price."

### Self-referencing categories

`categories.parent_id` references `categories.id`. This allows infinite depth: Products → Building Materials → Cement → OPC Cement. `depth_level` is stored denormalized for fast breadcrumb queries. Max practical depth is 5 levels.

### GST rates in a table, never in code

`gst_rates` is seeded from the official GST rate schedule and keyed by HSN code. Application code never hardcodes a GST rate. The GST calculation service always reads from this table so rate changes require a database update, not a code deploy.

---

## Tables — reference domain

These tables are reference data. They are seeded once and updated rarely. They do not change with normal user activity.

### cities

One row per city the platform operates in.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|name|text|"Delhi NCR"|
|slug|text UNIQUE|"delhi-ncr" — used in URLs|
|is_active|boolean|false = city not yet launched|
|launched_at|timestamptz|when the city went live|

### pincodes

Seeded from India Post data. The canonical source for state and district mapping. User addresses are validated against this table — free-text state entry is not allowed.

|Column|Type|Notes|
|---|---|---|
|pincode|text PK|6-digit India Post pincode|
|city_id|uuid FK → cities||
|district|text|e.g. "South Delhi"|
|state|text|e.g. "Delhi", "Uttar Pradesh"|
|state_code|text|GST state code: "07", "09", "06"|
|is_active|boolean|false = not serviceable|

Delhi NCR spans three states:

- Delhi pincodes: state_code = "07"
- UP pincodes (Noida, Ghaziabad): state_code = "09"
- Haryana pincodes (Gurgaon, Faridabad): state_code = "06"

### gst_rates

GST rate schedule keyed by HSN code. Never hardcode rates in application code — always read from this table.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|hsn_code|text|4 or 8 digit HSN|
|description|text|human-readable product description|
|cgst_pct|numeric(5,2)|Central GST % (intra-state)|
|sgst_pct|numeric(5,2)|State GST % (intra-state)|
|igst_pct|numeric(5,2)|Integrated GST % (inter-state)|
|effective_from|date|rate valid from this date|
|effective_to|date|null = currently active|
|is_current|boolean|true = use this row|

Common construction HSN codes:

- 2523: Cement — 28% GST
- 7213: TMT bars / steel — 18% GST
- 2505: Sand — 5% GST
- 6908: Tiles (ceramic) — 18% GST
- 3208: Paint — 18% GST

---

## Tables — user domain

### users

Every human on the platform — customers, suppliers, admins. Phone number is the primary identifier. **OTP-based login only**.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|phone_number|text UNIQUE NOT NULL|primary login identifier|
|email|text UNIQUE|optional, used for invoices|
|full_name|text NOT NULL||
|is_verified|boolean|phone OTP verified|
|is_active|boolean|false = account suspended|
|created_at|timestamptz||
|updated_at|timestamptz||

### user_roles

A user can hold multiple roles. A contractor who also lists products on the platform has both `customer` and `supplier` roles under the same phone number.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|user_id|uuid FK → users||
|role|text|`customer`, `supplier`, `admin`|
|assigned_at|timestamptz||

Unique constraint on (user_id, role).

### addresses

Delivery addresses for customers and site addresses for suppliers. State is always derived from pincode — never entered by the user.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|user_id|uuid FK → users||
|label|text|"Home", "Site A", "Office"|
|line1|text|flat/shop number, building name|
|line2|text|street, area|
|city|text|derived from pincode|
|district|text|derived from pincode|
|state|text|derived from pincode|
|state_code|text|derived from pincode — used for GST|
|pincode|text FK → pincodes||
|latitude|numeric|for delivery mapping|
|longitude|numeric|for delivery mapping|
|is_default|boolean|one default per user|

### otp_requests

Tracks OTP sends to prevent abuse. Rate limit: 3 OTPs per phone per 10 minutes enforced at this table level.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|phone_number|text NOT NULL||
|otp_hash|text|bcrypt hash of the 6-digit OTP|
|expires_at|timestamptz|10 minutes from created_at|
|verified_at|timestamptz|null = not yet used|
|created_at|timestamptz||

---

## Tables — supplier domain

### suppliers

One-to-one extension of `users` for supplier accounts. A user becomes a supplier after completing onboarding and being verified by an admin.

`verified_at` is null until admin approval. A supplier with `verified_at = null` cannot list products.

`commission_pct` is the platform's take rate for this supplier. May vary per supplier based on negotiation. Stored as a numeric percentage (e.g. `8.50` = 8.5%).

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|user_id|uuid FK → users UNIQUE|one supplier profile per user|
|business_name|text NOT NULL|trading name|
|gstin|text UNIQUE|GST registration number|
|pan|text|PAN card number|
|bank_account_number|text|for Razorpay X payouts|
|bank_ifsc|text||
|bank_account_name|text|account holder name|
|commission_pct|numeric(5,2)|platform fee percentage|
|city_id|uuid FK → cities|primary operating city|
|verified_at|timestamptz|null = pending verification|
|is_active|boolean|false = supplier suspended|
|created_at|timestamptz||

### supplier_documents

Verification documents uploaded during onboarding. Stored as S3 URLs. Admin reviews these before setting `suppliers.verified_at`.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|supplier_id|uuid FK → suppliers||
|document_type|text|`gstin_certificate`, `pan_card`, `trade_license`, `cancelled_cheque`|
|s3_url|text|private S3 URL|
|verified|boolean|admin has reviewed this document|
|uploaded_at|timestamptz||

### supplier_service_areas

The pincodes a supplier can deliver to. A supplier in Lajpat Nagar may service all of South Delhi pincodes.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|supplier_id|uuid FK → suppliers||
|pincode|text FK → pincodes||

---

## Tables — catalog domain

### categories

Self-referencing hierarchy. `parent_id` is null for top-level categories.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|parent_id|uuid FK → categories|null = top level|
|name|text NOT NULL|"Building Materials"|
|slug|text UNIQUE|"building-materials"|
|depth_level|integer|0 = top level, 1 = subcategory, etc.|
|icon_url|text|S3 URL for category icon|
|is_active|boolean||
|display_order|integer|for manual category ordering|

### filter_configs

Dynamic filter sidebar configuration per category. Adding a new filter for a category = one INSERT here, no code deploy needed.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|category_id|uuid FK → categories||
|attribute_key|text|"diameter_mm", "grade", "finish"|
|display_label|text|"Diameter", "Grade", "Finish"|
|filter_type|text|`range`, `multi_select`, `boolean`|
|display_order|integer||
|is_active|boolean||

### products

The shared product definition. One row per unique product. Multiple suppliers can list the same product.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|category_id|uuid FK → categories||
|city_id|uuid FK → cities||
|name|text NOT NULL|canonical product name|
|slug|text UNIQUE|for SEO URLs|
|description|text|detailed description|
|hsn_code|text|links to gst_rates|
|unit_of_measure|text|"bag", "kg", "sq_ft", "piece"|
|attributes|jsonb|{ "grade": "OPC 53", "weight_kg": 50 }|
|is_active|boolean|false = delisted from catalog|
|approved_at|timestamptz|null = pending admin approval|
|created_by|uuid FK → users|supplier or admin who created it|
|created_at|timestamptz||

### product_images

Images uploaded by the supplier for a product. `display_order = 0` is the primary image shown in listings.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|product_id|uuid FK → products||
|s3_url|text|public S3 URL via CloudFront|
|display_order|integer|0 = primary image|

### supplier_products

The listing layer. One row per supplier per product. This is what gets indexed in OpenSearch.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|product_id|uuid FK → products||
|supplier_id|uuid FK → suppliers||
|city_id|uuid FK → cities||
|sku|text UNIQUE|supplier's own SKU code|
|price_paise|integer NOT NULL|listed price in paise|
|stock_quantity|integer|total stock available|
|reserved_quantity|integer|reserved for pending orders|
|min_order_quantity|integer|minimum units per order|
|lead_time_days|integer|days to dispatch after order|
|is_active|boolean|supplier can delist without deleting|
|updated_at|timestamptz|used for search sync priority|

Available stock = `stock_quantity - reserved_quantity`. Always compute this in the query — never cache it.

### pricing_tiers

Role-based and quantity-based pricing. Allows different prices for individual buyers vs contractors vs companies, and bulk discounts for large orders.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|supplier_product_id|uuid FK → supplier_products||
|buyer_role|text|`customer`, `contractor`, `company`, or null = all|
|min_quantity|integer|minimum qty for this tier to apply|
|price_paise|integer|price per unit at this tier|
|valid_from|timestamptz||
|valid_until|timestamptz|null = no expiry (sale end date)|

Pricing resolution order:

1. Find all tiers for this supplier_product matching buyer's role and order quantity
2. If multiple match, take the lowest price
3. If no tier matches, fall back to `supplier_products.price_paise`

---

## Tables — order domain

### orders

The parent order. Customer-facing. One per checkout. Never set `status` directly — it is derived from sub-orders.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|order_number|text UNIQUE|human-readable e.g. "CM-2024-00142"|
|buyer_id|uuid FK → users||
|address_id|uuid FK → addresses|delivery address|
|city_id|uuid FK → cities||
|status|text|`pending`, `confirmed`, `partially_delivered`, `delivered`, `cancelled`|
|subtotal_paise|integer|sum of all line items before GST|
|gst_amount_paise|integer|total GST across all sub-orders|
|discount_paise|integer|any coupon or platform discount|
|total_paise|integer|final amount charged to customer|
|buyer_gstin|text|if B2B buyer, their GSTIN for invoice|
|razorpay_order_id|text|Razorpay payment order ID|
|placed_at|timestamptz|when order was created|
|confirmed_at|timestamptz|when payment was captured|

### sub_orders

One per supplier in the parent order. Supplier-facing. Suppliers only see their own sub_orders.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|order_id|uuid FK → orders|parent order|
|supplier_id|uuid FK → suppliers||
|status|text|`confirmed`, `processing`, `dispatched`, `delivered`, `cancelled`|
|subtotal_paise|integer||
|gst_amount_paise|integer||
|total_paise|integer||
|commission_paise|integer|platform fee for this sub-order|
|tcs_paise|integer|TCS deducted (1% of taxable value)|
|payout_amount_paise|integer|total_paise - commission - tcs|
|payout_status|text|`pending`, `processing`, `paid`|
|eway_bill_number|text|for interstate orders > ₹50,000|
|dispatched_at|timestamptz||
|delivered_at|timestamptz||

### order_items

Line items within a sub-order.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|sub_order_id|uuid FK → sub_orders||
|supplier_product_id|uuid FK → supplier_products||
|product_id|uuid FK → products|denormalized for convenience|
|quantity|integer||
|unit_price_paise|integer|price at time of order (snapshot)|
|hsn_code|text|snapshot at time of order|
|gst_rate_pct|numeric(5,2)|snapshot at time of order|
|cgst_paise|integer|0 for interstate orders|
|sgst_paise|integer|0 for interstate orders|
|igst_paise|integer|0 for intra-state orders|
|line_total_paise|integer|(unit_price × quantity) + gst|

Note: `unit_price_paise`, `hsn_code`, and `gst_rate_pct` are **snapshots** taken at checkout time. Even if the supplier later changes their price or the GST rate changes, the order record reflects what the customer was charged.

### shipments

Delivery tracking per sub-order. Supplier updates this as they dispatch and deliver.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|sub_order_id|uuid FK → sub_orders||
|vehicle_number|text|truck/vehicle registration|
|driver_name|text||
|driver_phone|text|customer can track via this|
|tracking_notes|text|free text updates from supplier|
|dispatched_at|timestamptz||
|estimated_delivery|date||
|delivered_at|timestamptz|set by supplier on delivery|

---

## Tables — payment domain

### payments

One payment record per order. Links the Razorpay payment to our order.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|order_id|uuid FK → orders||
|razorpay_payment_id|text UNIQUE|e.g. "pay_ABC123"|
|razorpay_order_id|text|matches orders.razorpay_order_id|
|method|text|`upi`, `card`, `netbanking`, `cod`|
|status|text|`created`, `captured`, `failed`, `refunded`|
|amount_paise|integer||
|gateway_fee_paise|integer|Razorpay's fee|
|captured_at|timestamptz||

### supplier_payouts

Settlement records. One per sub-order, created when the sub-order is marked delivered.

| Column             | Type                 | Notes                                     |     |
| ------------------ | -------------------- | ----------------------------------------- | --- |
| id                 | uuid PK              |                                           |     |
| sub_order_id       | uuid FK → sub_orders |                                           |     |
| supplier_id        | uuid FK → suppliers  |                                           |     |
| gross_amount_paise | integer              | sub_order total                           |     |
| commission_paise   | integer              | platform fee                              |     |
| tcs_paise          | integer              | TCS deducted                              |     |
| net_amount_paise   | integer              | amount transferred to supplier            |     |
| razorpay_payout_id | text                 | Razorpay X payout reference               |     |
| status             | text                 | `pending`, `processing`, `paid`, `failed` |     |
| initiated_at       | timestamptz          |                                           |     |
| settled_at         | timestamptz          | when money reached supplier               |     |

### payment_webhook_log

Every Razorpay webhook is logged here before processing. Used for idempotency — if a webhook arrives twice, the second one is ignored.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|event_type|text|"payment.captured", "payment.failed"|
|razorpay_event_id|text UNIQUE|deduplication key|
|payload|jsonb|full webhook payload|
|processed_at|timestamptz|null = not yet processed|
|error|text|if processing failed|
|received_at|timestamptz||

---

## Tables — search domain

### search_sync_outbox

The outbox for PostgreSQL → OpenSearch sync. See ADR-004.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|entity_type|text|`supplier_product`, `product`|
|entity_id|uuid|the row that changed|
|operation|text|`upsert`, `delete`|
|payload|jsonb|pre-computed OpenSearch document|
|created_at|timestamptz||
|processed_at|timestamptz|null = pending|
|failed_at|timestamptz|null = not failed|
|retry_count|integer|max 5 before dead-lettering|
|error|text|last error message|

### user_events

Behavioral event log. Feeds the recommendation engine and search analytics. Written to on every page view, search, cart add, and purchase.

Never query this table in real-time API paths. Use it only for batch analytics and model training.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|user_id|uuid|null for anonymous sessions|
|session_id|text NOT NULL||
|event_type|text|`view`, `search`, `cart_add`, `wishlist`, `purchase`|
|entity_type|text|`product`, `supplier_product`, `category`|
|entity_id|uuid||
|metadata|jsonb|{ query, dwell_ms, position, source }|
|city_id|uuid FK → cities||
|created_at|timestamptz||

### search_events

Search-specific log. Zero-result searches are reviewed weekly to find missing synonyms or catalog gaps.

|Column|Type|Notes|
|---|---|---|
|id|uuid PK||
|user_id|uuid||
|session_id|text||
|raw_query|text|exactly what user typed|
|parsed_query|jsonb|after query understanding layer|
|filters_applied|jsonb|active filters at search time|
|result_count|integer|0 = synonym or catalog gap|
|clicked_result_id|uuid|null if user bounced|
|clicked_position|integer|ranking position of clicked result|
|city_id|uuid FK → cities||
|created_at|timestamptz||

---

## Key indexes

Beyond primary keys, these indexes are essential for API performance. Add them in the migration that creates the table.

```sql
-- User lookup by phone (login)
CREATE UNIQUE INDEX idx_users_phone ON users(phone_number);

-- Supplier products by city (all catalog queries)
CREATE INDEX idx_supplier_products_city ON supplier_products(city_id)
  WHERE is_active = true;

-- Supplier products by supplier (dashboard queries)
CREATE INDEX idx_supplier_products_supplier ON supplier_products(supplier_id);

-- Orders by buyer (order history)
CREATE INDEX idx_orders_buyer ON orders(buyer_id, placed_at DESC);

-- Sub-orders by supplier (supplier dashboard)
CREATE INDEX idx_sub_orders_supplier ON sub_orders(supplier_id, status);

-- Outbox pending rows (worker polling)
CREATE INDEX idx_outbox_pending ON search_sync_outbox(created_at)
  WHERE processed_at IS NULL AND failed_at IS NULL;

-- Search events zero results (weekly analytics)
CREATE INDEX idx_search_zero_results ON search_events(created_at)
  WHERE result_count = 0;

-- User events by session (recommendation queries)
CREATE INDEX idx_user_events_session ON user_events(session_id, created_at DESC);
```

---

## Migration naming convention

```
src/db/migrations/
  001_initial_users_and_roles.sql
  002_addresses_and_pincodes.sql
  003_cities_reference_data.sql
  004_gst_rates_reference_data.sql
  005_suppliers_and_documents.sql
  006_categories_and_filter_configs.sql
  007_products_and_images.sql
  008_supplier_products_and_pricing.sql
  009_orders_and_sub_orders.sql
  010_order_items_and_shipments.sql
  011_payments_and_payouts.sql
  012_search_outbox_and_events.sql
```

Each file contains:

1. A comment block explaining what this migration does and why
2. The `CREATE TABLE` or `ALTER TABLE` statements
3. The indexes for those tables
4. A `-- rollback` comment block with the `DROP` statements (used by `npm run migrate:down`)
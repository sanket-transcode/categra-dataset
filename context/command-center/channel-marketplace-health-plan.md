# Account Health & Channel / Marketplace Health Plan

Date: 2026-10-09 (rev 4: no snapshot table, failure-only tracking, seller-level fetching; rev 4.1 2026-10-12: metrics payload moved to a seller-level table)
Scope: `api/`, `db-migrations/`, `webapp/`
First consumer: Command Center V2 (**Account Health** pulse tile, **Channel & marketplace health** panel, **data-trust** attention card). Endpoints live in their own resource controllers and are reusable elsewhere.

---

## 1. Goal

Show, per **channel** (Amazon seller account per region / Shopify store) and per **Amazon marketplace**, how the channel actually is:

1. **Account standing**: Amazon account status, Account Health Rating, performance metrics, policy violations; Shopify store active / frozen / locked / inactive / uninstalled.
2. **Connection**: token valid, roles granted.
3. **Data trust**: is a sync currently failing, so the data we show for that scope may be wrong?

**Principles**
- Store the **latest platform values** (headline columns on the existing channel tables, the Amazon metrics payload once per seller × marketplace) and keep **transitions** (not snapshots) in one history table.
- Track **failures only**: a row exists while something is broken and is deleted on the next success. Healthy = no row. That means about 1% of the writes of success-stamping.
- Fetch Amazon data **once per seller** (`seller_partner_id` + marketplace) and fan the result out to every Amazon channel using that seller, across all Categra accounts.
- The displayed status is **derived at read time** with a fixed rule order (§7), so it always explains *why*.

---

## 2. Decisions

| # | Decision |
|---|---|
| D1 | Store Amazon's **exact** `currentAccountStatus` and `previousAccountStatus`. `AT_RISK` no longer maps to `CONNECTED`. |
| D2 | Per-marketplace status, AHR, metrics and policies come from `GET_V2_SELLER_PERFORMANCE_REPORT`. |
| D3 | Marketplace references in new tables use `amazon_marketplace_id` (FK `tbl_amazon_marketplaces`); non-Amazon = `NULL`. |
| D4 | Lean columns. No `*_source`, `*_updated_at`, `*_changed_at`; history only in `tbl_channel_status_events`. |
| D5 | Endpoints in semantic controllers (`channel-health`, `amazon-account-health`), not `command-center`. |
| D6 | One marketplace per report request. |
| D7 | The data-trust attention card is computed automatically from failure rows + existing columns (§6.6). |
| D8 | **No snapshot table.** Only the latest report result is kept: small headline columns (`account_status`, `ahr_status`, `ahr_score`) on `tbl_amazon_channel_marketplaces`, and the metrics/policies payload **once per seller × marketplace** in `tbl_amazon_seller_performance` (§5.2), so ACM reads never carry the jsonb and cross-account sellers store one copy. Changes between reports become `tbl_channel_status_events` rows. Consequence: no metric trend sparkline; the status/metric transition timeline replaces it. |
| D9 | **No success-freshness table.** A new `tbl_channel_sync_failures` mirrors `tbl_product_sync_failures` at channel/marketplace level: insert or update on failure, delete on success. |
| D10 | **No report-request table.** In-flight dedupe and the fallback poll use Redis + the BullMQ job payload. `REPORT_PROCESSING_FINISHED` already carries `sellerId` and `reportId`; `getReportById` returns `marketplaceIds`. |
| D11 | **Seller-level fetching.** Unique work key = `seller_partner_id` + `amazon_marketplace_id` (a marketplace belongs to one region). Results are written with one bulk SQL to every `tbl_amazon_channels` / `tbl_amazon_channel_marketplaces` row of that seller (all accounts, non-deleted channels). Amazon-only state is keyed by `amazon_channel_id` (it lives on Amazon tables). Cross-platform tables (`tbl_channel_status_events`, `tbl_channel_sync_failures`) keep `channel_id`, because Shopify has no `amazon_channel_id` and per-channel rows keep tenant scoping by `account_id`. For Amazon they are filled with one `INSERT … SELECT` over the seller's channels. |
| Q1 | Fetch the report for `DEACTIVATED` channels too. |
| Q2 | Thresholds (constant `CHANNEL_HEALTH_THRESHOLDS`): inventory push failures open ≥ 30 min + breadth rule (§6.6); Amazon order sync `error` or `order_synced_until` > 2 h behind (only with orders in the last 7 d); channel failure rows open ≥ 30 min; Shopify probe ≥ 3 consecutive failures; performance data > 36 h old. |
| Q3 | Show AHR status and score. |
| Q4 | "Near target" warning is phase 2. |
| Q5 | Marketplace `DEACTIVATED` is display only. |
| Q6 | ~~Snapshot retention~~: no longer applicable (D8). |
| Q7 | Rename the unused `QueueNames.AMAZON_ACCOUNT_STATUS_CHANGE` → `AMAZON_SELLER_PERFORMANCE`. |
| Q8 | Manual refresh 1 per channel per hour via `tbl_resource_cooldowns`, plus a seller-level guard (§6.3). |

---

## 3. Current state (verified in code)

| Area | Finding | Location |
|---|---|---|
| Status → connection map | `AT_RISK` → `CONNECTED`. | `core/constants/constants.ts:1672` `AMAZON_ACCOUNT_STATUS_CONNECTION_MAP` |
| Connection overwrite | `NORMAL` → `CONNECTED` unconditionally, so it **overwrites `CONNECTION_LOST`** from a token failure. | `account-status-changed.handler.ts` |
| Previous status / event time | `previousAccountStatus` is never read; `eventTime` is ignored. | `constants.ts:1750`, handler |
| Routing | `AmazonSubscriptionRouteResolver.resolve` returns every `amazon_channel` linked to the provider subscription (filtered by `sellerPartnerId`), so routes already span accounts. A channel whose subscription row is missing or failed is not routed. | `events/routing/amazon-subscription-route.resolver.ts:95-140` |
| Seller identity | `tbl_amazon_channels.seller_partner_id`, `region_id`. | `amazonChannel.entity.ts` |
| Seller performance report | Never requested. `AMAZON_HANDLED_REPORT_TYPES = ['GET_XML_BROWSE_TREE_DATA']`; `ReportProcessingFinishedHandler` has no router by type. | `constants.ts:1697`, `handlers/report/report-processing-finished.handler.ts` |
| Report plumbing | `createReport`, `getReportById`, `getDocumentURL` exist; gzip handling in the category service. | `transport/amazon.service.ts:136-160`, `amazon-categories.service.ts:145` |
| Subscriptions | `ACCOUNT_STATUS_CHANGED` and `REPORT_PROCESSING_FINISHED` are auto-subscribed behind family flags. | `constants.ts:1541`, `:1611` |
| Unused queue | `QueueNames.AMAZON_ACCOUNT_STATUS_CHANGE`. | `global-enums.ts` |
| Token failure | Amazon 401/403/`invalid_grant` and Shopify 401 → `CONNECTION_LOST`, no reason stored. | `amazon-channel-access-token.service.ts:164-190`, `shopify-graphql.service.ts:90-120` |
| Shopify webhooks | No `app/uninstalled`. | `shopify-webhook-record.service.ts:16-42` |
| Amazon order sync | Per marketplace `order_sync_status` (`in-progress / completed / error`) + `order_synced_until`; a 30-min cron flips stuck `in-progress` → `error`. | `amazonChannelMarketplace.entity.ts`, `queue.service.ts:1936-1995` |
| Stock push failures | `tbl_product_sync_failures (product_id, variant_id, channel_id, amazon_marketplace_id, module, raw_data, created_at, updated_at)`. `module=STOCK` row created/updated on 429 / INVALID / Error, **deleted on success**. Shopify stock sync uses the same table (`amazon_marketplace_id = NULL`). | `productSyncFailure.entity.ts`, `product-stock.service.ts:271-1034` |
| FBA inventory | `tbl_amazon_fba_inventories` has no timestamps; fetched per SKU on demand. | `amazon-fba-inventory.service.ts` |
| Cooldowns | `tbl_resource_cooldowns (account_id, module, resource_key, cooldown_until)`; config in `RESOURCE_COOLDOWNS`. | `resourceCooldown.entity.ts`, `constants.ts:1319` |
| Action Center | Channel issues only from `connection_status`. | `action-center.service.ts:3802` |
| Webapp | Mocks: `COMMAND_CENTER_V2_CHANNEL_HEALTH`, attention item `germany-inventory-stale`. | `command-center-v2/_lib/mock-data.ts` |

---

## 4. Platform research

### 4.1 Amazon `ACCOUNT_STATUS_CHANGED`

- Fires on transitions between `NORMAL`, `AT_RISK`, `DEACTIVATED`; payload version `2021-01-01`, SQS.
- `accountStatusChangeNotification.previousAccountStatus` / `.currentAccountStatus` (required).
- Envelope: `eventTime`, `notificationMetadata.{applicationId, subscriptionId, publishTime, notificationId}`.
- **No `marketplaceId`.** Scope = seller × region.

### 4.2 Amazon `GET_V2_SELLER_PERFORMANCE_REPORT`

- Account Health dashboard metrics. JSON, schema `accountHealthReport-2020-11-18`. **Request-only**, sellers only, role **Selling Partner Insights**.
- We send **one marketplace per request**, so one document = one marketplace.

```
{
  accountStatuses: [ { marketplaceId, status: NORMAL|AT_RISK|DEACTIVATED } ],
  performanceMetrics: [ {
    marketplaceId,
    accountHealthRating: { reportingDateRange, accountHealthRatingStatus: GREAT|WARNED /* GitHub schema */,
                           ahrStatus, ahrScore /* docs page */ },
    orderDefectRate: { afn: ODR, mfn: ODR },
      // ODR = { reportingDateRange, orderCount, rate, status, targetValue, targetCondition, fulfillmentType,
      //         orderWithDefects{count,status}, claims{count,status}, chargebacks{count,status}, negativeFeedback{count,status} }
    lateShipmentRate, preFulfillmentCancellationRate, validTrackingRate, onTimeDeliveryRate,
    unitOnTimeDeliveryRate /* US only */, invoiceDefectRate{ invoiceDefect, missingInvoice, lateInvoice },
      // each = { reportingDateRange, rate, status, targetValue, targetCondition, <counts> }
    listingPolicyViolations, productAuthenticityCustomerComplaints, productConditionCustomerComplaints,
    productSafetyCustomerComplaints, receivedIntellectualPropertyComplaints, restrictedProductPolicyViolations,
    suspectedIntellectualPropertyViolations, foodAndProductSafetyIssues, customerProductReviewsPolicyViolations /* docs */,
    otherPolicyViolations, documentRequests,
      // each = { reportingDateRange, defectsCount, status, targetValue, targetCondition }
    policyViolationWarnings: { reportingDateRange, warningsCount } /* docs */,
    warningStates: [] /* deprecated */
  } ]
}
```

- `status`: `GOOD | BAD | NONE`. `targetCondition`: `GREATER_THAN | LESS_THAN | EQUALS`.
- **Schema drift** between the GitHub schema and the docs page: the normaliser accepts both; `ahr_status` is free text. Verify with a real report.

| Report field | Stored as | Use |
|---|---|---|
| `accountStatuses[].status` | `acm.account_status` | Per-marketplace status chip |
| `accountHealthRating` | `acm.ahr_status`, `acm.ahr_score` | Headline |
| Rate metrics, ODR afn/mfn + sub-counts | `tbl_amazon_seller_performance.metrics` jsonb (§5.2) | Metrics table with Amazon's own status + target |
| Policy groups, `policyViolationWarnings`, `documentRequests` | `tbl_amazon_seller_performance.metrics` jsonb | Policy list, Action Center |
| Worst of metric + policy statuses | Not stored; computed from `metrics` at read time (in SQL for lists, §7) | `at_risk` derivation, list badges |
| `marketplaceId`, `fulfillmentType`, sub-count `status`, `warningStates` | Not stored | Row key / redundant with the afn/mfn key / not shown / deprecated |
| Changes vs previous stored value | `tbl_channel_status_events` | What changed, timeline |

### 4.3 Shopify

- No health score. Standing signals: `app/uninstalled` webhook, **402** frozen, **423** locked (verify), "shop is not active", **401** token revoked. THROTTLED / 429 is a data-trust signal, not standing.
- Webhooks fail silently, so a periodic lightweight probe covers reachability.

---

## 5. Data model

Migrations in `db-migrations/` (CLI-generated names); entities in `api/src/entities/`.

### 5.1 Columns on existing tables

| Table | Column | Notes |
|---|---|---|
| `tbl_channels` | `connection_reason` varchar null | `TOKEN_REVOKED`, `INVALID_GRANT`, `APP_UNINSTALLED`, `ACCOUNT_DEACTIVATED` |
| `tbl_amazon_channels` | `account_status` | `NORMAL / AT_RISK / DEACTIVATED`, null = unknown (seller × region) |
| | `previous_account_status` | Exact `previousAccountStatus` |
| | `account_status_event_time` | Out-of-order guard |
| `tbl_amazon_channel_marketplaces` | `account_status` | From report `accountStatuses[]` |
| | `ahr_status` varchar, `ahr_score` int | |
| `tbl_shopify_channels` | `store_status` | `ACTIVE / FROZEN / LOCKED / INACTIVE / UNINSTALLED / UNKNOWN` |

All seller-level columns are written for **every** row of that seller (§6.1), so any account reads its own rows with no cross-account join.

### 5.2 `tbl_amazon_seller_performance` (new): latest report payload, one row per seller × marketplace

**Why not a column on ACM:** ACM is read by order sync, stock sync, publishing and listing flows, mostly via Sequelize `findAll` without an `attributes` list, so a jsonb there would be fetched and de-TOASTed on every one of those reads. The payload is also seller-level data: on ACM it would be copied to every account using the seller. Here it is written once per report, and the hot ACM table only gets three small columns.

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| seller_partner_id | varchar | Stored upper-case |
| amazon_marketplace_id | bigint FK | |
| metrics | jsonb | Normalised shape below |
| report_id | varchar | Last processed report (support / debugging) |
| checked_at | timestamptz | Report processing time; drives the 1 h guard, 36 h staleness and `verifiedAt` |
| created_at / updated_at | timestamptz | |

- Unique `(seller_partner_id, amazon_marketplace_id)`; written with one `INSERT … ON CONFLICT DO UPDATE` per report.
- No `account_id`: tenant access always goes through the account's own ACM row → `tbl_amazon_channels.seller_partner_id`, so an account can only read sellers it has connected.
- Rows whose seller has no remaining non-deleted channel are deleted by the daily cron (§6.3) before requesting.

**`metrics` shape** (normaliser output, schema version `v`; keys present only when Amazon returns them):

```jsonc
{
  "v": 1,
  "metrics": {
    // rate metrics, key = Amazon field name; ODR split into afn / mfn
    "orderDefectRate.afn": { "rate": 0.004, "status": "GOOD", "target": 0.01, "cond": "LESS_THAN",
                             "from": "2026-08-10", "to": "2026-10-09", "orders": 2310,
                             "counts": { "orderWithDefects": 9, "claims": 2, "chargebacks": 1, "negativeFeedback": 6 } },
    "orderDefectRate.mfn": { ... },
    "lateShipmentRate":               { "rate", "status", "target", "cond", "from", "to", "orders", "counts": { "lateShipment" } },
    "preFulfillmentCancellationRate": { ..., "counts": { "cancellation" } },
    "validTrackingRate":              { ..., "counts": { "validTracking", "shipmentCount" } },
    "onTimeDeliveryRate":             { ..., "counts": { "onTimeDelivery", "shipmentCountWithValidTracking" } },
    "unitOnTimeDeliveryRate":         { ..., "counts": { "unitOnTimeDelivery", "totalUnitCount" } },   // US only
    "invoiceDefectRate":              { ..., "counts": { "invoiceDefect", "missingInvoice", "lateInvoice" } }
  },
  "policies": {
    // key = Amazon field name; groups with status NONE and defects 0 are omitted (absent = clean)
    "listingPolicyViolations": { "defects": 2, "status": "BAD", "target": 0, "cond": "EQUALS", "from": "...", "to": "..." },
    "documentRequests":        { ... }
    // also: productAuthenticity / productCondition / productSafety CustomerComplaints, receivedIntellectualPropertyComplaints,
    //       restrictedProductPolicyViolations, suspectedIntellectualPropertyViolations, foodAndProductSafetyIssues,
    //       customerProductReviewsPolicyViolations, otherPolicyViolations
  },
  "warnings": { "count": 1, "from": "...", "to": "..." }   // policyViolationWarnings
}
```

- Exact `counts` field names follow the report; verify against a real document (schema drift, §4.2). Unknown fields are dropped, so a new Amazon field cannot grow the row silently.
- Expected size ≈ 2–4 KB per row (8 rate metrics + up to 12 policy groups, short keys, empty groups omitted).
- Not in the jsonb: `accountStatuses[].status` and `accountHealthRating` (typed ACM columns), `marketplaceId` (row key), `fulfillmentType` (in the afn/mfn key), sub-count `status` (not shown), `warningStates` (deprecated).

### 5.3 `tbl_channel_status_events` (new): transitions only

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| account_id | bigint | Tenant scoping |
| channel_id | bigint FK | Cross-platform (D11) |
| amazon_marketplace_id | bigint null | `NULL` = channel-level or non-Amazon |
| signal | varchar | `AMAZON_ACCOUNT_STATUS`, `AMAZON_AHR_STATUS`, `AMAZON_METRIC_STATUS`, `AMAZON_POLICY_STATUS`, `CONNECTION`, `SHOPIFY_STORE_STATUS` |
| source | varchar | `AMAZON_NOTIFICATION`, `AMAZON_PERFORMANCE_REPORT`, `AMAZON_TOKEN_REFRESH`, `SHOPIFY_WEBHOOK`, `SHOPIFY_API`, `USER` |
| previous_value | varchar null | Exact platform value |
| current_value | varchar | Exact platform value |
| event_time | timestamptz | Platform time / report processing time |
| external_event_id | varchar null | `notificationId`, `X-Shopify-Webhook-Id`, `<reportId>:<signal>:<key>` |
| metadata | jsonb null | e.g. `{ metricKey: 'orderDefectRate.mfn', rate, targetValue }`, `{ policyKey, defectsCount }`, http status |
| created_at | timestamptz | |

- Unique `(channel_id, source, external_event_id)` where `external_event_id` is not null (dedupe per channel; the same Amazon event fans out to several channels).
- Indexes: `(channel_id, signal, event_time desc)`, `(channel_id, amazon_marketplace_id, event_time desc)`.
- Low volume (transitions only); kept indefinitely.

### 5.4 `tbl_channel_sync_failures` (new): channel/marketplace-level twin of `tbl_product_sync_failures`

| Column | Notes |
|---|---|
| id | |
| account_id | |
| channel_id | |
| amazon_marketplace_id | null for channel-level / non-Amazon |
| module | `ORDERS`, `INVENTORY_PULL_FBA`, `PERFORMANCE_REPORT`, `API_PROBE` (Shopify), `WEBHOOK_PROCESSING` (Shopify) |
| error_code | e.g. `MISSING_ROLE`, `RATE_LIMITED`, `HTTP_5XX`, `TIMEOUT`, `THROTTLED`, `DOCUMENT_PARSE_FAILED` |
| raw_data | jsonb (last error detail, like `tbl_product_sync_failures.raw_data`) |
| failure_count | int, incremented per failed attempt |
| created_at | **first failure = "since"** |
| updated_at | last failure |

- Unique index `(channel_id, COALESCE(amazon_marketplace_id, 0), module)`.
- **Write path:**
  - On failure → `INSERT … ON CONFLICT DO UPDATE SET failure_count = failure_count + 1, error_code, raw_data, updated_at`.
  - On success → `DELETE`. This only runs when a Redis marker `chsf:<channelId>:<amazonMarketplaceId|0>:<module>` (set on failure) exists, so the healthy path does **zero** DB writes.
- Seller-level Amazon failures (`PERFORMANCE_REPORT`) are upserted/deleted for all channels of the seller in one statement.
- `ChannelSyncFailureService.recordFailure(...)` / `.clear(...)` in `system/operations/monitoring/` (next to resource-cooldown).
- Amazon **order sync** does not use it: the existing `order_sync_status` / `order_synced_until` already express failure and lag.
- **Inventory push** does not use it: `tbl_product_sync_failures` already holds it.

### 5.5 Index on an existing table

`tbl_product_sync_failures`: add `(channel_id, amazon_marketplace_id, module, created_at)` for the data-trust aggregation (§6.6).

### 5.6 Config

- `RESOURCE_COOLDOWN_TYPES.AMAZON_SELLER_PERFORMANCE_REFRESH` → `{ duration: 1h, requiredKeys: [CHANNEL_ID] }`.
- `CHANNEL_HEALTH_THRESHOLDS` constant (Q2).

---

## 6. Pipelines

### 6.1 Seller fan-out (shared helper)

`AmazonSellerScopeService`:
- `findSellerChannels(sellerPartnerId, regionId)` → all non-deleted `tbl_amazon_channels` (+ `channel_id`, `account_id`) with that seller in that region, across accounts.
- `pickTokenChannel(sellerPartnerId, regionId)` → one channel to call Amazon with: prefer `connection_status = CONNECTED`, freshest token. On token failure (the existing service marks that channel `CONNECTION_LOST`), try the next candidate.
- Bulk writes use raw SQL joined on `seller_partner_id` (+ `region_id` / `amazon_marketplace_id`), with no per-account `BaseService` scoping. Example:

```sql
UPDATE tbl_amazon_channel_marketplaces acm
SET account_status = $3, ahr_status = $4, ahr_score = $5
FROM tbl_amazon_channels ac
JOIN tbl_channels c ON c.id = ac.channel_id AND c.is_deleted = false
WHERE acm.amazon_channel_id = ac.id
  AND UPPER(ac.seller_partner_id) = UPPER($1)
  AND acm.amazon_marketplace_id = $2
RETURNING acm.amazon_channel_id, c.id AS channel_id, c.account_id;
```

`RETURNING` gives the channel list for the event fan-out insert. The metrics payload is not fanned out: it is one upsert into `tbl_amazon_seller_performance` (§5.2).

### 6.2 `ACCOUNT_STATUS_CHANGED` handler

1. Add alias `previousAccountStatus`; read `eventTime` + `notificationId`.
2. Resolve seller + region from the first route (`sellerPartnerId`, `regionId`). Write through `findSellerChannels`, not only the routed channels, so channels with a missing subscription row are covered.
3. One transaction:
   - Bulk `UPDATE tbl_amazon_channels SET account_status, previous_account_status, account_status_event_time` **where** seller + region **and** (`account_status_event_time` is null or `< eventTime`). This is the out-of-order guard.
   - `INSERT INTO tbl_channel_status_events … SELECT` one row per seller channel (`signal=AMAZON_ACCOUNT_STATUS`, exact previous/current, `external_event_id=notificationId`, `ON CONFLICT DO NOTHING`).
   - Connection (delete `AMAZON_ACCOUNT_STATUS_CONNECTION_MAP`), per channel:
     - `DEACTIVATED` → `connection_status=DEACTIVATED`, `connection_reason=ACCOUNT_DEACTIVATED`.
     - `AT_RISK` → connection untouched.
     - `NORMAL` → `CONNECTED` only where the channel is `DEACTIVATED` with reason `ACCOUNT_DEACTIVATED`. It never overrides `CONNECTION_LOST` / `NOT_CONNECTED`.
   - Socket update for affected channels.
4. Enqueue `REQUEST_SELLER` (§6.3) for seller + region with a 15 min delay.
5. Token failures (Amazon, Shopify 401) also write `connection_reason` + a `CONNECTION` event.

### 6.3 Request the performance report (seller-level)

Queue `AMAZON_SELLER_PERFORMANCE` (api-worker, concurrency 1, existing SP-API limiter).

| Job | jobId | Does |
|---|---|---|
| `REQUEST_SELLER` | `perf:<sellerId>:<regionId>` | Distinct enabled marketplaces (`acm.status = true`) across all the seller's channels → one `REQUEST_MARKETPLACE` each |
| `REQUEST_MARKETPLACE` | `perf:<sellerId>:<amazonMarketplaceId>` | See below |
| `POLL_REPORT` | `perf-poll:<reportId>` | Fallback if the notification is missed |

`REQUEST_MARKETPLACE`:
1. **Seller-level guard:** skip if the seller's `tbl_amazon_seller_performance` row for that marketplace has `checked_at` < 1 h ago, or the Redis in-flight key `amazon:report:inflight:GET_V2_SELLER_PERFORMANCE_REPORT:<sellerId>:<marketplaceId>` exists (TTL 2 h). Several accounts sharing a seller (or repeated manual refreshes) therefore cause one Amazon call.
2. `pickTokenChannel` → `createReport({ reportType, marketplaceIds: [<one>] })`.
3. Set the in-flight Redis key `{ reportId, amazonChannelId }` and enqueue `POLL_REPORT` (10 min, backoff to ~2 h) with `{ reportId, sellerId, amazonMarketplaceId, amazonChannelId }` in the payload.
4. 403 / Unauthorized → upsert `PERFORMANCE_REPORT` failure `error_code=MISSING_ROLE` for **all** the seller's channels (marketplace `NULL`, since the role is account-wide). Skip the remaining marketplaces for this run. Other errors → `PERFORMANCE_REPORT` failure for that marketplace.

Triggers:
- **Cron** daily (`30 5 * * *`): one `REQUEST_SELLER` per distinct `(seller_partner_id, region_id)` with at least one non-deleted channel in `CONNECTED` / `DEACTIVATED` (Q1). First deletes `tbl_amazon_seller_performance` rows whose seller has no non-deleted channel left.
- **`ACCOUNT_STATUS_CHANGED`**: §6.2 step 4.
- **Manual refresh**: §8.2.

### 6.4 Process the report

- `REPORT_PROCESSING_FINISHED`: router by `reportType` (`GET_XML_BROWSE_TREE_DATA` → existing category path; `GET_V2_SELLER_PERFORMANCE_REPORT` → `AmazonSellerPerformanceService.processReport`). Add the type to `AMAZON_HANDLED_REPORT_TYPES`.
- `processReport({ reportId, sellerId })`, used by both the notification and `POLL_REPORT`. A Redis lock `amazon:report:processing:<reportId>` makes it idempotent.
  1. `getReportById` (token from the in-flight key or `pickTokenChannel`) → `marketplaceIds[0]`, status. `CANCELLED` = no data → clear the in-flight key; `FATAL` → `PERFORMANCE_REPORT` failure.
  2. Document → download (+ gunzip) → **tolerant normaliser** → the element for that marketplace.
  3. Read the current `tbl_amazon_seller_performance` row (+ `account_status` / `ahr_status` from one seller ACM row) and **diff** it against the new values:
     - `account_status`, `ahr_status` changes
     - each metric `status` change (GOOD↔BAD)
     - each policy group `status` change or `defectsCount` increase
     - new `documentRequests`
  4. One transaction: upsert `tbl_amazon_seller_performance` (`metrics`, `report_id`, `checked_at`); bulk `UPDATE` the three headline columns on ACM rows of the seller + marketplace (§6.1); `INSERT … SELECT` status events for each diff item × each returned channel (`source=AMAZON_PERFORMANCE_REPORT`, `external_event_id=<reportId>:<signal>:<key>`); `clear` the `PERFORMANCE_REPORT` failure (marketplace + `MISSING_ROLE` null-marketplace row) for all seller channels.
  5. Reconcile channel-level `tbl_amazon_channels.account_status` = worst across the seller's ACM rows in that region, through the same `account_status_event_time` guard (event time = report processing time). This heals missed notifications.
  6. Delete the in-flight key.
- The first fill (stored value null) writes no diff events, only the values.

### 6.5 Shopify

1. **`app/uninstalled`** webhook → `CONNECTION_LOST` + `connection_reason=APP_UNINSTALLED`, `store_status=UNINSTALLED`, stop sync flags, `checkAndMarkOrphanedTasks`, `CONNECTION` + `SHOPIFY_STORE_STATUS` events.
2. **Central error classifier** in `shopify-graphql.service.ts`:

   | Detect | Effect | Retry |
   |---|---|---|
   | 401 | `CONNECTION_LOST` / `TOKEN_REVOKED` | no |
   | 402 | `store_status=FROZEN` | no |
   | 423 | `store_status=LOCKED` | no |
   | shop not active | `store_status=INACTIVE` | no |
   | THROTTLED / 429 | none on standing (existing limiter) | yes |

   Each `store_status` change → event + socket. The first success after FROZEN / LOCKED / INACTIVE → `ACTIVE` (+ event). Status changes are rare, so the success path only compares in memory against a cached `store_status`.
3. **Probe cron** (hourly): `shop { id }` per connected store. Failure → `API_PROBE` failure row (`failure_count`); success → `clear`.
4. Webhook processing errors (orders / inventory handlers) → `WEBHOOK_PROCESSING` failure; next successful processing → `clear`.

### 6.6 Data trust (the yellow "stale sync" attention card)

**The card** (mock `germany-inventory-stale`, prototype rank 2): severity `yellow`, `issueType: CONNECTION`, "Data trust · DE only", "Stock data untrusted", "Europe inventory sync stale — Germany only, 2h 18m". Chips: channel, region, `DE stale` (bad), `UK · FR · IT healthy` (good). Description: inventory conclusions for DE are suppressed. CTA "Check connection" → Action Center `CONNECTION` scoped to channel + DE. Shopify variant: "Shoes inventory sync stale 48m".

**Computation**: `ChannelHealthService.getDataTrust(scope)`, from failure rows and existing columns only:

| Domain | Source | Stale when | `since` | Cause |
|---|---|---|---|---|
| **INVENTORY push** (Amazon per marketplace with `sync_stock_with_categra = true`; Shopify per store) | `tbl_product_sync_failures` `module=STOCK` grouped by `(channel_id, amazon_marketplace_id)`: `COUNT(DISTINCT COALESCE(variant_id, product_id))`, `MIN(created_at)`, latest `raw_data` kind | oldest open failure ≥ 30 min **and** (≥ 5 distinct items **or** ≥ 20% of the marketplace's published FBM listings) | `MIN(created_at)` | `RATE_LIMITED` (429), `REJECTED` (INVALID), `TRANSPORT_ERROR` |
| **INVENTORY FBA pull** | `tbl_channel_sync_failures` `INVENTORY_PULL_FBA` | row open ≥ 30 min | `created_at` | row `error_code` |
| **ORDERS** (Amazon) | `acm.order_sync_status`, `acm.order_synced_until`, `tbl_orders` (orders in last 7 d) | `order_sync_status='error'` **or** `order_synced_until` > 2 h behind with orders in the last 7 d | `order_synced_until` | `ORDER_SYNC_ERROR` / `ORDER_SYNC_LAG` |
| **ORDERS / INVENTORY** (Shopify) | `tbl_channel_sync_failures` `WEBHOOK_PROCESSING`, `API_PROBE` | row open ≥ 30 min / probe `failure_count ≥ 3` | `created_at` | `error_code` |

Rules:
- Below the breadth threshold, stock failures stay **product-level** (existing product sync-failure UI), so one bad SKU doesn't flag the marketplace.
- Skipped when the channel is `CONNECTION_LOST` / `NOT_CONNECTED` / `DEACTIVATED` (the critical connection item covers it; `syncAmazonStock` skips pushes then).
- Duration = `now - since`, computed on read. **Self-clearing:** the next successful operation deletes the failure rows, so the issue disappears without a cleanup job.
- **Siblings:** other enabled marketplaces of the same channel, evaluated for the same domain → `healthy` / `stale` chips.
- **Severity:** `yellow`; `red` when older than 24 h or every enabled marketplace of the channel is stale.
- **Known limit:** silence (nothing ran, nothing failed) is not detected. Amazon orders are covered by `order_synced_until` lag; Shopify reachability by the probe; Amazon notification delivery stays on the existing admin receiving-status view.

**Output** (in `/channel-health` and `/channel-health/data-trust`):

```ts
DataTrustIssue = {
  id: `${domain}:${channelId}:${amazonMarketplaceId ?? 0}`,
  domain: 'INVENTORY' | 'ORDERS',
  channelId, amazonMarketplaceId | null, marketplaceCode | null, regionCode | null,
  severity: 'yellow' | 'red',
  cause: string, since: string, durationMs: number,
  affectedCount: number | null,
  siblings: [{ amazonMarketplaceId, marketplaceCode, status: 'healthy' | 'stale' }],
}
```

**Suppression:** each marketplace (and each Shopify channel) carries `trust: { inventory: boolean, orders: boolean }`. `ChannelHealthService.getUntrustedScopes(domain)` is the shared rule: Command Center's Inventory tile / inventory attention items exclude or label untrusted scopes; sales/orders sections do the same for `orders=false`.

---

## 7. Status derivation (read time)

Per marketplace and per channel; **first match wins**. Output `{ status, reasons[] }`, `reason = { code, severity, signal, since, value, link }`.

| # | Status | Amazon | Shopify |
|---|---|---|---|
| 1 | `critical` | channel `connection_status ∈ {CONNECTION_LOST, NOT_CONNECTED, DEACTIVATED}` or marketplace `account_status=DEACTIVATED` | `CONNECTION_LOST`, or `store_status ∈ {UNINSTALLED, FROZEN, LOCKED, INACTIVE}` |
| 2 | `at_risk` | `account_status=AT_RISK` (channel or marketplace), AHR not healthy, any metric or policy group in `tbl_amazon_seller_performance.metrics` with `status=BAD` (list endpoints evaluate this in SQL with `jsonb_path_exists(metrics, '$.metrics.* ? (@.status == "BAD")')` and the same for `$.policies.*`, so only a boolean leaves the DB) (early warning: metrics turn BAD before Amazon sets `AT_RISK`), `documentRequests` BAD | — |
| 3 | `degraded` | any `DataTrustIssue`; `PERFORMANCE_REPORT` failure row (`MISSING_ROLE` → "Permission required", else "Health data unavailable"); `tbl_amazon_seller_performance.checked_at` > 36 h | any `DataTrustIssue`; `API_PROBE` `failure_count ≥ 3` |
| 4 | `attention` | Action Center catalog/listing issues | `SHOPIFY_UNPUBLISHED` / `SHOPIFY_INACTIVE` counts |
| 5 | `healthy` | none of the above, data known | |
| — | `unknown` | no `account_status` and no `tbl_amazon_seller_performance` row → "Not yet checked" (never green) | `store_status=UNKNOWN` and no probe yet |

- Channel status = worst of (channel rules, marketplace statuses).
- `detail` = top reason ("Account at risk since 7 Oct (was NORMAL)", "DE: ODR 1.4% > 1%", "DE inventory sync failing 2h 18m").
- `verifiedAt` = oldest of the time-bearing sources in scope (`tbl_amazon_seller_performance.checked_at`, `order_synced_until`).

---

## 8. API (resource controllers)

### 8.1 `ChannelHealthController`: `@Controller('channel-health')`

`api/src/modules/app/catalog/channel/health/` (module, controller, `ChannelHealthService`). Platform-agnostic, role-scoped.

| Route | Purpose |
|---|---|
| `GET /channel-health?channelIds=` | Status, reasons, marketplaces, trust, data-trust issues |
| `GET /channel-health/data-trust?channelIds=&amazonMarketplaceId=&domain=` | Data-trust issues only |
| `GET /channel-health/:channelId/status-events?amazonMarketplaceId=&signal=&limit=` | Transition timeline (exact previous → current, source, time, metadata) |

```ts
// GET /channel-health
{
  verifiedAt: string | null,
  channels: [{
    channelId, platform: 'AMAZON' | 'SHOPIFY', name, regionCode | null,
    status: 'critical' | 'at_risk' | 'degraded' | 'attention' | 'healthy' | 'unknown',
    detail, reasons,
    connection: { status, reason },
    account: { status, previousStatus, since } | null,          // Amazon; since = account_status_event_time
    store: { status } | null,                                   // Shopify
    failures: [{ module, errorCode, since, failureCount }],     // channel-level failure rows
    trust: { inventory, orders } | null,                        // Shopify
    marketplaces: [{
      amazonMarketplaceId, code, label, enabled,
      status, detail, reasons,
      accountStatus, ahr: { status, score } | null,
      performance: { status, checkedAt },
      failures: [...],
      trust: { inventory, orders }
    }]
  }],
  dataTrust: DataTrustIssue[]
}
```

### 8.2 `AmazonAccountHealthController`: `@Controller('amazon-account-health')`

`api/src/modules/app/platforms/amazon/account-health/` (module, controller, `AmazonSellerPerformanceService`, normaliser, `AmazonSellerScopeService`).

| Route | Purpose |
|---|---|
| `GET /amazon-account-health/:channelId?amazonMarketplaceId=` | From ACM columns + the seller's `tbl_amazon_seller_performance` row (the only endpoint that reads the full jsonb): account status, AHR, metrics (rate, status, target, condition, counts, date range; ODR afn/mfn breakdown), policy groups with `defectsCount > 0` or BAD, warnings, failure state, `checkedAt` |
| `POST /amazon-account-health/:channelId/refresh` | `tbl_resource_cooldowns` check (`AMAZON_SELLER_PERFORMANCE_REFRESH`, key `CHANNEL_ID`) → reject with `retryAfter` while active; else enqueue `REQUEST_SELLER` for the channel's seller + region and upsert the cooldown. The seller-level guard (§6.3) still prevents duplicate Amazon calls across accounts. |

The metric trend endpoint is dropped (D8); the timeline comes from `/channel-health/:channelId/status-events?signal=AMAZON_METRIC_STATUS`.

### 8.3 Webapp URL builders

`channelHealth` and `amazonAccountHealth` in `webapp/src/lib/api.ts`.

---

## 9. Action Center & feeds

- Category `ACCOUNT_HEALTH` (channel/marketplace-scoped): `AMAZON_ACCOUNT_DEACTIVATED`, `AMAZON_ACCOUNT_AT_RISK`, `AMAZON_METRIC_BAD` (per metric key), `AMAZON_POLICY_BAD` (per group), `AMAZON_DOCUMENT_REQUEST`, `AMAZON_HEALTH_PERMISSION_MISSING`, `SHOPIFY_STORE_FROZEN`, `SHOPIFY_STORE_LOCKED`, `SHOPIFY_STORE_INACTIVE`, `SHOPIFY_APP_UNINSTALLED`. Read from current-state columns; auto-clear on recovery.
- Category `CONNECTION` (existing `CHANNEL_CONNECTION_ERROR` / `CHANNEL_NOT_CONNECTED` move under it) + data-trust types `MARKETPLACE_INVENTORY_SYNC_STALE`, `MARKETPLACE_ORDER_SYNC_STALE`, `SHOPIFY_INVENTORY_SYNC_STALE`, `SHOPIFY_ORDER_SYNC_STALE`. Built from `getDataTrust` (same rule, same answer). Drill-down lists the open `tbl_product_sync_failures` / `tbl_channel_sync_failures` rows. Accepts the handoff `{ category: 'CONNECTION', channelId, marketplaceId }`.
- Ranked priorities: account → connection / data trust → stock → listings.
- "What changed": `tbl_channel_status_events` (account, AHR, metric, policy, store, connection transitions).
- Optional notifications on transitions to AT_RISK / DEACTIVATED / FROZEN.

---

## 10. Webapp (Command Center V2 as first consumer)

1. `_lib/types.ts`: add `critical`, `unknown` to `CommandCenterV2ChannelStatus`; reshape `CommandCenterV2ChannelHealth` to the API; tones + translation keys.
2. `channel-health-section.tsx`: `/channel-health` instead of the mock. Marketplace chips are coloured by their own status (tooltip: account status, AHR, top reason). Row expand shows Standing / Connection / Data trust.
3. Account Health tile/card: standing from `/channel-health`, detail from `/amazon-account-health`, timeline from `/status-events`. Refresh button → `POST …/refresh` (disabled with countdown during cooldown).
4. **Attention data-trust card** from `dataTrust`:
   - `severity`; `issueType: 'CONNECTION'`
   - `typeLabel` "Data trust · {code} only" / "Data trust · {store}"
   - `impact` "{Stock|Order} data untrusted"
   - `title` "{Region} {inventory|order} sync stale — {Marketplace} only, {duration}" / "{Store} {domain} sync stale {duration}"
   - `chips` channel, region, `{code} stale` (bad), healthy siblings (good)
   - `description` cause + suppression sentence
   - CTA "Check connection" → Action Center `CONNECTION` with `channelId` + `marketplaceId`
   - Issues of the same channel + domain collapse into one card. Next actions derive from the same issues.
5. `unknown` and "Permission required (Selling Partner Insights)" are shown explicitly.

---

## 11. Phasing

| Phase | Work | Depends |
|---|---|---|
| **P0 correctness** | Handler fix with seller fan-out, out-of-order guard, AT_RISK unmapped, NORMAL never overrides CONNECTION_LOST, delete the map; `connection_reason`; `tbl_channel_status_events`; `tbl_amazon_channels` columns; Shopify `app/uninstalled` + classifier + `store_status` | — |
| **P1 performance report** | ACM columns; `AmazonSellerScopeService`; queue rename + seller jobs + cron + status trigger; Redis in-flight + poll; report router; normaliser; diff → events | P0 |
| **P2 failures + data trust** | `tbl_channel_sync_failures` + `ChannelSyncFailureService` (Redis-marker clear); writers (FBA pull, report, Shopify probe/webhooks); product-failure index; `getDataTrust` + suppression helper | parallel |
| **P3 API + UI** | `channel-health` + `amazon-account-health` modules, derivation, cooldown config, webapp wiring | P0–P2 |
| **P4 actions** | Action Center `ACCOUNT_HEALTH` + `CONNECTION`, priorities, What changed, optional notifications | P3 |
| **Later** | "Near target" warning (Q4) | P1 |

### Verification checklist

- [ ] A real `ACCOUNT_STATUS_CHANGED` payload parses previous/current/eventTime.
- [ ] Same seller connected in two accounts: one notification → both accounts' `tbl_amazon_channels` rows updated and one event row per channel; one daily report per seller + marketplace, written to both accounts' ACM rows.
- [ ] A real `GET_V2_SELLER_PERFORMANCE_REPORT` (single marketplace): AHR shape, compression, exactly one element.
- [ ] 403 without the Selling Partner Insights role → `MISSING_ROLE` failure on all of the seller's channels; it clears after a successful report.
- [ ] Shopify 402 / 423 / shop-not-active error shapes.
- [ ] A NORMAL notification on a `CONNECTION_LOST` channel leaves it `CONNECTION_LOST`.
- [ ] Refresh twice within an hour → second call rejected with `retryAfter`; refresh from two accounts sharing a seller → one Amazon call.
- [ ] Data trust: ≥ 5 STOCK failures in DE older than 30 min → one yellow issue with UK/FR/IT healthy; successful pushes delete the rows → the issue disappears.
- [ ] Below threshold → no marketplace issue.
- [ ] `order_sync_status='error'` on one marketplace → `ORDERS` issue for that marketplace only.
- [ ] Healthy path performs no DB write in `ChannelSyncFailureService.clear` (Redis marker absent).

---

## 12. Implementation status (2026-10-12)

**Built (P0–P3):** migrations (`20261012043031` … `20261012043828`), entities, seller scope + performance pipeline (queue `amazon-seller-performance`, daily cron `30 5`, startup seed when no data is fresh), `ACCOUNT_STATUS_CHANGED` handler rewrite, token-failure reasons, Shopify `app/uninstalled` + store-status classifier + hourly probe + webhook failure rows, FBA pull failure rows, `channel-health` and `amazon-account-health` controllers, and the Command Center V2 wiring (Account Health tile, Channel & marketplace health panel, live connection / account-health / permission / data-trust attention cards, refresh button with cooldown).

**Not built:**
- P4: Action Center `ACCOUNT_HEALTH` / `CONNECTION` categories and priorities, notifications on transitions.
- The "What changed" panel is still the static mock. `/channel-health/:channelId/status-events` exists; an account-wide feed endpoint does not.
- Status rule 4 (`attention`, catalog / listing issues from Action Center) is not emitted yet.
- Inventory / order tiles do not consume `ChannelHealthService.getUntrustedScopes` yet (they are still mock).
- "Near target" warning (Q4).

**Deviations from the text above (decided while building):**
1. **Report normaliser** flattens nested groups generically: `orderDefectRate.afn`, `orderDefectRate.mfn`, `invoiceDefectRate.missingInvoice`, … are separate `metrics` entries, and `counts` takes every non-meta numeric field (or `{count}` object) instead of a hard-coded list, so a renamed Amazon count field does not break the parse. Verify field names against a real document.
2. **Connection derivation (§6.2):** any status other than `DEACTIVATED` (not only `NORMAL`) releases a channel that was deactivated for `ACCOUNT_DEACTIVATED` (or has no reason, for rows deactivated before this change). Otherwise `DEACTIVATED → AT_RISK` would leave the connection stuck.
3. **Channel-level reconcile (§6.4.5):** the channel is `DEACTIVATED` only when every known marketplace is; `AT_RISK` when any marketplace is `AT_RISK` / `DEACTIVATED`; else `NORMAL` (consistent with Q5, marketplace `DEACTIVATED` is display-only).
4. **Poll job id** includes the attempt; seller / marketplace job ids include an hour bucket so a failed job (kept for debugging) cannot block the next request.
5. **Missing role** also sets a 6 h Redis flag so the seller's other marketplace jobs skip instead of repeating the 403.
6. **FBA pull** failure rows are recorded only for transport / auth errors (429, 5xx, network, 401 / 403), not for a 4xx about one SKU.
7. **Breadth rule denominators** are approximate: Amazon = non-deleted, `status = true`, non-FBA `tbl_product_channel_marketplaces` rows of the marketplace; Shopify = non-deleted `tbl_product_channels` rows of the channel.
8. **Stock failure cause** can only distinguish `REJECTED` (Amazon issues) from `TRANSPORT_ERROR`, because 429 and generic errors store the same `rawData`.
9. **Shopify `store_status`** is written only on change; `NULL` means "no known problem". The probe is skipped for `UNINSTALLED` stores.
10. **API envelope:** both controllers return `{ status, data, message }` like the rest of the app; a refresh inside the cooldown returns `status: 'error'` with `data.retryAfterSeconds`.
11. **Webapp:** health is loaded once per page and filtered client-side by channel / marketplace. Attention text is translated by reason `code`; the API's English `message` is never rendered.

**Needs verification before release:**
- Run the migrations on a non-shared database; nothing has been run against a database yet.
- Boot api-main and api-worker once (Nest dependency resolution is only checked by the compiler so far).
- The verification checklist in §11, especially the two-account seller test and a real `GET_V2_SELLER_PERFORMANCE_REPORT`.
- Existing Shopify channels get the `app/uninstalled` webhook on their next webhook reconcile.

---

## Sources

- Amazon notification type values: https://developer-docs.amazon.com/sp-api/docs/notification-type-values
- Performance report types: https://developer-docs.amazon.com/sp-api/docs/report-type-values-performance
- Performance report attributes: https://developer-docs.amazon.com/sp-api/docs/performance-reports-attributes
- Report schema: https://github.com/amzn/selling-partner-api-models/blob/main/schemas/reports/accountHealthReport-2020-11-18.json
- Shopify webhooks: https://shopify.dev/docs/api/webhooks/2026-04.md
- Shopify response codes: https://shopify.dev/docs/api/usage/response-codes
- Shopify GraphQL rate limits: https://shopify.dev/docs/apps/build/apis/graphql-admin/rate-limits

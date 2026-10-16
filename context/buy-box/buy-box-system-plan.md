# Amazon Competition & Buy Box — System Plan

Date: 2026-10-15
Scope: `api/`, `db-migrations/`, `webapp/` (Amazon only; Shopify untouched)
Inputs: `base-requirements.md` (feature level, already decided), the 5 SP-API operations + 2 notifications listed there, and the code as it exists today.
Status: **plan only, nothing implemented.** Items that still need a human decision are in §13 with a recommended default.

---

## 1. Goal

Replace the JSON-blob `tbl_product_current_offers` + read-time `BuyBoxService.buildBuyBoxDetails` with a structured store:

1. **Fetch once per Amazon unit of uniqueness, fan out to every Categra scope** that needs it (a seller id can be connected to many Categra channels in many accounts; the same ASIN is sold by many sellers).
2. **One canonical shape** for offers, regardless of whether the data came from `getListingOffers`, `getCompetitiveSummary`, `ANY_OFFER_CHANGED` or `PRICING_HEALTH`.
3. **Buy Box status is stored, not computed on read**, one row per `product + variant (nullable) + channel + amazon marketplace`, with a stored reason and a transition log.
4. Keep only what is useful: promoted columns for things we filter/sort/count on, small JSON for display-only structures.

Non-goals: Action Center (it is detached, not redesigned, §10.4), "winning chance" estimate, repricing, Shopify, B2B (`CustomerType` is fixed to `Consumer`).

---

## 2. Decisions taken (from the task and the user's answers)

| # | Decision |
|---|---|
| D1 | Status set is exactly `WINNING, AT_RISK, NOT_WINNING, INELIGIBLE, UNAVAILABLE, UNKNOWN`. The old set (`COMPETITIVE, NOT_COMPETITIVE, NO_BUY_BOX`) is deleted everywhere. |
| D2 | **AT_RISK is expected-price first**: winning now **and** `getFeaturedOfferExpectedPriceBatch` says our landed price is above the expected featured price. Other signals are secondary (§6.3). |
| D3 | Status precedence: not published → `UNKNOWN`; published but price missing (in a *complete, successful* data set) → `UNAVAILABLE`; price present → `INELIGIBLE` / `WINNING` / `AT_RISK` / `NOT_WINNING`. |
| D4 | All thresholds live in **one constant object** in `core/constants/constants.ts` (`BUY_BOX_THRESHOLDS`). No threshold anywhere else. |
| D5 | **Seller-level uniqueness**: a seller id can be shared by many channels across many accounts, so polling and notification handling dedupe on the Amazon key and fan out by SQL to every associated Categra scope (same approach as `channel-marketplace-health-plan.md` D11). |
| D6 | Scope = all active published listings. History = latest state + a slim transition log (retention cron). Competitor offers = latest snapshot only. |
| D7 | `tbl_product_current_offers` is dropped, not migrated (nothing to preserve). Every reader is reshaped (§10). |
| D8 | Status is **written** by the evaluator whenever its inputs change; reads are plain joins. |

---

## 3. Source analysis (what each Amazon resource actually gives us)

Verified against the SP-API docs. Rate limits are the documented defaults per seller × app; the `x-amzn-RateLimit-Limit` header can raise them.

| Resource | Key | Batch | Rate (burst) | Gives us | Does **not** give us |
|---|---|---|---|---|---|
| `getListingOffers` (v0, GET) | SKU + marketplace + ItemCondition (`CustomerType=Consumer`) | 1 | 1 rps (2) | `Summary` (`TotalOfferCount`, `NumberOfOffers`, `LowestPrices`, `BuyBoxPrices`, `BuyBoxEligibleOffers`, `CompetitivePriceThreshold`, `ListPrice`, `SuggestedLowerPricePlusShipping`, `SalesRankings`) and up to 20 `Offers` (`MyOffer`, `IsBuyBoxWinner`, `IsFeaturedMerchant`, `IsFulfilledByAmazon`, `PrimeInformation`, `ShippingTime`, `SellerFeedbackRating`, `ListingPrice`, `Shipping`, `Points`). **Only source with `MyOffer`.** | Segments, promotions |
| `getListingOffersBatch` | SKU, up to 20 | 20 | low (≈0.1 rps, burst 1; confirm header) | Same body as above per item | Same |
| `getCompetitiveSummary` (2022-05-01, POST batch) | **ASIN** + marketplace, up to 20 | 20 | 0.033 rps (1) | `featuredBuyingOptions[].segmentedFeaturedOffers[]` (the actual **Featured Offer winner(s)** per customer segment: `sellerId`, `fulfillmentType AFN/MFN`, `listingPrice`, `shippingOptions`, `primeDetails`, `promotions`, `featuredOfferSegments` with `glanceViewWeightPercentage`), `lowestPricedOffers` (Consumer, per condition), `referencePrices` (`CompetitivePriceThreshold`, `CompetitivePrice`, `WasPrice`) | Our own SKU, eligibility |
| `getFeaturedOfferExpectedPriceBatch` | **SKU** + marketplace, up to 40 | 40 | 0.033 rps (1) | `offerIdentifier` (asin, sku, sellerId, fulfillmentType), `resultStatus` (`VALID_FOEP`, `NO_COMPETING_OFFER(S)`, `OFFER_NOT_ELIGIBLE`, `OFFER_NOT_FOUND`, `ASIN_NOT_ELIGIBLE`), `featuredOfferExpectedPrice`, `competingFeaturedOffer`, `currentFeaturedOffer` | Offer list |
| `getMyFeesEstimateForSKU` / `getMyFeesEstimates` | SKU (or ASIN) + price | list (docs state none; **plan for 20**) | 0.5 rps (1) | `FeeDetailList` (`ReferralFee`, `FBAFees` → `FBAPickAndPack`, …), `TotalFeesEstimate` | Competition |
| `ANY_OFFER_CHANGED` (v1.0, SQS) | **ASIN** + marketplace + condition, `SellerId` | — | event | `OfferChangeTrigger` (`OfferChangeType` External / Internal / Featured Offer, `TimeOfOfferChange`), `Summary`, top-20 `Offers` with `IsBuyBoxWinner`, `IsFeaturedMerchant`, `PrimeInformation`. Only for items where the seller has an active offer. | **No SKU, no `MyOffer` flag** (match by `SellerId`), no FOEP, no segments |
| `PRICING_HEALTH` (v1.0, SQS) | **ASIN** + marketplace, `sellerId` | — | event | `issueType` (`BuyBoxDisqualification`), `merchantOffer` (condition, fulfillmentType, listing/shipping/landed), `summary` (counts, `buyBoxPrices`), `referencePrice` (`averageSellingPrice`, `competitivePriceThreshold`, `retailOfferPrice`, `msrpPrice`) | No SKU, **no recovery event** |

### Observations that shape the design

1. **Two natural keys, not one.** Market facts (who holds the Featured Offer, lowest offers, reference prices) are per **marketplace + ASIN**. Seller facts (my price, eligibility, FOEP, fees, pricing-health) are per **seller + marketplace + SKU**. Categra data is per `product/variant + channel + marketplace` (+ account). Three levels → three storage levels (§5).
2. **Notifications are ASIN-keyed with no SKU.** A seller can list one ASIN under several SKUs (e.g. FBA + FBM), so notification → seller rows needs `(seller, marketplace, asin[, fulfillmentType])`. `asin` must be stored on the seller-offer row.
3. **Throughput is dominated by the 0.033 rps APIs** (one call per 30 s): 1,000 ASINs ⇒ 50 CS calls ≈ 25 min, 1,000 SKUs ⇒ 25 FOEP calls ≈ 12.5 min *per seller × marketplace*. Dedupe by ASIN/SKU before calling, and notifications must be able to skip polling.
4. **Same field, different spellings across sources** (`IsFulfilledByAmazon` bool vs `fulfillmentType` AFN/MFN; `PrimeInformation.IsPrime/IsNationalPrime` vs `primeDetails.eligibility`; `ListingPrice.Amount` vs `listingPrice.amount`; `NO_COMPETING_OFFER` in text vs `NO_COMPETING_OFFERS` in the sample). One mapper per source into one canonical shape (§4); handle both FOEP spellings.
5. **Landed price** = listing + shipping − points (points only exist in JP). One helper everywhere.
6. `ANY_OFFER_CHANGED` subscriptions are currently `29/29 ERROR` and `PRICING_HEALTH` has never delivered (`amazon-events-analysis.md`). Phase 0 verifies subscribe works (payload version, `eventFilter`) before anything depends on it. Polling is the safety net either way.

---

## 4. Canonical shapes (code + JSON)

One TypeScript module `core/types/buy-box.type.ts`, one mapper file per source. All money is `{ amount: number, currency: string }` with currency kept **per value** (marketplaces can differ from account base currency).

**`OfferSnapshot`** (used for competitors, featured offers, my offer):
`sellerId`, `condition`, `subCondition`, `fulfillmentType: 'AFN'|'MFN'`, `listingPrice`, `shippingPrice`, `landedPrice`, `primeEligibility: 'NATIONAL'|'REGIONAL'|'NONE'|null`, `isFeaturedOffer: boolean|null`, `isFeaturedMerchant: boolean|null`, `shippingTime { minHours, maxHours }|null`, `sellerFeedback { ratingPct, count }|null`, `promotions[]|null`, `segments[{ membership, weightPct }]|null`.

Mappers: `fromListingOffer(offer)`, `fromCompetitiveSummaryOffer(offer, isFeatured, segments)`, `fromAnyOfferChanged(offer)`. Missing information is `null`, never `0` (the old code wrote `0`/`''` for unknowns, which is how "unknown" turned into "price 0").

**`MarketSummary`**: counts by `(condition, fulfillmentType)` for total and buy-box-eligible offers, lowest landed per fulfillment type, featured landed price, `competitivePriceThreshold`, `listPrice`, `referencePrices` (`wasPrice`, `competitivePrice`, `averageSellingPrice`, `retailOfferPrice`, `msrpPrice` when present), `salesRanks[]`.

**Status reasons** (`BUY_BOX_STATUS_REASONS`, constant, one place):

| Status | Reasons |
|---|---|
| `UNKNOWN` | `NOT_PUBLISHED`, `FETCH_PENDING`, `FETCH_FAILED`, `DATA_STALE`, `DATA_INCOMPLETE` |
| `UNAVAILABLE` | `NO_PRICE`, `NOT_BUYABLE` (only if §13 OI-4 accepted) |
| `INELIGIBLE` | `PRICE_TOO_HIGH`, `OFFER_NOT_ELIGIBLE`, `ASIN_NOT_ELIGIBLE`, `NOT_FEATURED_MERCHANT` |
| `AT_RISK` | `ABOVE_EXPECTED_PRICE`, `COMPETITOR_UNDERCUT` |
| `NOT_WINNING` | `ANOTHER_OFFER_FEATURED`, `NO_FEATURED_OFFER` (only if §13 OI-3 accepted) |
| `WINNING` | `FEATURED_OFFER_HELD` |

---

## 5. Database design

Names follow the repo convention (`tbl_`, snake_case, audit columns, `createdAt/updatedAt`). Enumerations are `VARCHAR` validated in code via constants (enum types were already purged in earlier cleanups). New high-volume tables use `BIGINT` ids. Everything is one migration set in `db-migrations/` + entity files in `api/.../entities/` exported from `entities/index.ts`.

### 5.1 `tbl_amazon_market_offers` — market facts, shared by everyone

Unique key: `(amazon_marketplace_id, asin, item_condition)`. Written by `getCompetitiveSummary`, `getListingOffers`, `ANY_OFFER_CHANGED`. **Account-agnostic** (public marketplace data).

| Column | Type | Notes |
|---|---|---|
| `id` | BIGINT PK | |
| `amazon_marketplace_id` | INT FK `tbl_amazon_marketplaces` | |
| `asin` | VARCHAR(20) | |
| `item_condition` | VARCHAR(20) | `New`… (Categra mapped condition) |
| `currency_code` | VARCHAR(3) | |
| `total_offer_count` | INT | |
| `buy_box_eligible_offer_count` | INT | |
| `lowest_landed_price` / `lowest_landed_price_afn` / `lowest_landed_price_mfn` | NUMERIC(14,2) | promoted for gap/sort |
| `featured_seller_id` | VARCHAR(20) | holder of the Featured Offer in the **default/primary segment** |
| `featured_fulfillment_type` | VARCHAR(3) | |
| `featured_landed_price` | NUMERIC(14,2) | |
| `competitive_price_threshold` | NUMERIC(14,2) | |
| `featured_offers` | JSONB | `OfferSnapshot[]` per segment (usually 1–2) |
| `top_offers` | JSONB | up to 20 `OfferSnapshot[]` (competitor list) |
| `summary` | JSONB | rest of `MarketSummary` (counts by condition/fulfilment, reference prices, sales ranks) |
| `observed_at` | TIMESTAMPTZ | Amazon's time when known (`TimeOfOfferChange`), else request time. **Out-of-order guard**: a write with `observed_at` older than stored is ignored. |
| `last_source` | VARCHAR(30) | `COMPETITIVE_SUMMARY` / `LISTING_OFFERS` / `ANY_OFFER_CHANGED` |
| `fetched_at` | TIMESTAMPTZ | last successful write |

Indexes: unique key above; `(fetched_at)` for sweeps.

### 5.2 `tbl_amazon_seller_offers` — seller-specific facts, shared across channels/accounts

Unique key: `(seller_partner_id, amazon_marketplace_id, sku)`.

| Column | Type | Notes |
|---|---|---|
| `id` | BIGINT PK | |
| `seller_partner_id` | VARCHAR(20) | = `tbl_amazon_channels.seller_partner_id` |
| `amazon_marketplace_id` | INT FK | |
| `sku` | VARCHAR(80) | |
| `asin` | VARCHAR(20) NULL | |
| `item_condition` | VARCHAR(20) | |
| `fulfillment_type` | VARCHAR(3) NULL | `AFN`/`MFN` as Amazon reports it |
| `currency_code` | VARCHAR(3) | |
| `has_offer` | BOOLEAN | `MyOffer` found in a *complete* response (distinguishes "no offer" from "call failed") |
| `listing_price` / `shipping_price` / `landed_price` | NUMERIC(14,2) NULL | NULL = unknown/missing, never 0 |
| `is_featured_offer` | BOOLEAN NULL | I hold the Featured Offer (any segment, §13 OI-2) |
| `is_featured_merchant` | BOOLEAN NULL | |
| `is_buy_box_eligible` | BOOLEAN NULL | from `BuyBoxEligibleOffers` for my condition + fulfilment |
| `prime_eligibility` | VARCHAR(10) NULL | |
| `shipping_min_hours` / `shipping_max_hours` | INT NULL | |
| `foep_result_status` | VARCHAR(30) NULL | `VALID_FOEP`… (both `NO_COMPETING_OFFER`/`S` normalised) |
| `foep_expected_price` | NUMERIC(14,2) NULL | `featuredOfferExpectedPrice.listingPrice` (a listing price, compared with our listing price) |
| `foep_details` | JSONB NULL | `competingFeaturedOffer`, `currentFeaturedOffer` as `OfferSnapshot` |
| `pricing_health_issue` | VARCHAR(40) NULL | `BuyBoxDisqualification` or NULL |
| `pricing_health_at` | TIMESTAMPTZ NULL | |
| `pricing_health_details` | JSONB NULL | `referencePrice` block |
| `fees_price_basis` | NUMERIC(14,2) NULL | price the estimate was made at |
| `referral_fee` / `fba_pick_and_pack_fee` / `other_fees` / `total_fees` | NUMERIC(14,4) NULL | promoted (Product Overview + orders margin read them) |
| `fees_details` | JSONB NULL | full `FeeDetailList` for the rare consumer |
| `offers_fetched_at` / `foep_fetched_at` / `fees_fetched_at` | TIMESTAMPTZ NULL | per-source freshness |
| `last_error` | JSONB NULL | `{ source, code, at }` of the last failed source call |

Indexes: unique key; `(seller_partner_id, amazon_marketplace_id, asin)` for notification fan-out; `(offers_fetched_at)` for sweeps.

### 5.3 `tbl_product_buy_box_statuses` — the stored status (the answer to the task)

Unique key: `(product_id, COALESCE(variant_id, 0), channel_id, amazon_marketplace_id)` (unique expression index, same trick as other nullable-variant keys in the schema).

| Column | Type | Notes |
|---|---|---|
| `id` | BIGINT PK | |
| `account_id` | INT | tenant scope |
| `product_id` | INT FK | |
| `variant_id` | INT NULL FK | NULL = standalone product. Parents of variants get **no** row. |
| `channel_id` | INT FK | |
| `amazon_marketplace_id` | INT FK | NOT NULL (Amazon only) |
| `seller_partner_id`, `sku`, `asin` | VARCHAR | denormalised lookup keys to `tbl_amazon_seller_offers` / `tbl_amazon_market_offers`; set when the row is created and refreshed on publish/BS/SKU change; a reconcile cron fixes drift |
| `item_condition` | VARCHAR(20) | |
| `status` | VARCHAR(20) | §1 D1 |
| `status_reason` | VARCHAR(40) | §4 |
| `status_since` | TIMESTAMPTZ | only changes when `status` changes |
| `landed_gap_vs_lowest` | NUMERIC(14,2) NULL | our landed − lowest competing landed (promoted: sort/filter) |
| `data_as_of` | TIMESTAMPTZ | oldest relevant input timestamp (drives `DATA_STALE`) |
| `evaluated_at` | TIMESTAMPTZ | |

Indexes: the unique key; `(channel_id, amazon_marketplace_id, status)` (list filters + Command Center counts); `(seller_partner_id, amazon_marketplace_id, asin)` and `(seller_partner_id, amazon_marketplace_id, sku)` (fan-out); `(account_id, status)`.

Displayed numbers (own landed, lowest, offer count, eligibility, fulfilment, prime, competitors, fees) are **read by join** to 5.1/5.2, not copied again, so the status row stays small and there is one copy of each fact.

### 5.4 `tbl_product_buy_box_status_events` — transition log

`id, account_id, product_id, variant_id, channel_id, amazon_marketplace_id, from_status, to_status, from_reason, to_reason, trigger` (`POLL`/`ANY_OFFER_CHANGED`/`PRICING_HEALTH`/`PUBLISH`/`MANUAL`/`RECONCILE`), `occurred_at`. Insert only when `(status, reason)` changes. Keys are copied (no FK to the status row) so the log survives status-row deletion. Index `(channel_id, amazon_marketplace_id, occurred_at)` and `(product_id, variant_id, occurred_at)`. Retention cron, 90 days (constant; mirrors the amazon-events inbox retention service pattern).

### 5.5 Dropped

`tbl_product_current_offers` (+ entity, service, controller, module, `CurrentOffersData`/`AmazonProductOffersPayload` types that only served it). Migration `down()` re-creates the empty structure only.

---

## 6. Status evaluation

A single pure function `evaluateBuyBoxStatus(input): { status, reason, landedGapVsLowest }` in `buy-box-status.evaluator.ts`, unit-tested table-style, no I/O. Inputs come from one SQL join (status row ⟕ seller offer ⟕ market offer ⟕ publication facts).

### 6.1 Order of rules (first match wins)

1. **Not published** → `UNKNOWN / NOT_PUBLISHED`. Published = PCM/VM `status = true`, sync status ≠ `NOT_PUBLISHED`, and an `asin` exists. (This is also why rows exist for unpublished listings: to show `UNKNOWN`, not to fetch.)
2. **No fetch yet** → `UNKNOWN / FETCH_PENDING`. **Last fetch of a required source failed with no usable previous data** → `UNKNOWN / FETCH_FAILED`. **`data_as_of` older than `BUY_BOX_THRESHOLDS.STALE_AFTER_MS`** → `UNKNOWN / DATA_STALE`. **Response complete flag false** (a required source returned an error/partial item) → `UNKNOWN / DATA_INCOMPLETE`.
3. **Price missing in a complete response** (`has_offer = false` or `landed_price IS NULL`) → `UNAVAILABLE / NO_PRICE`. Missing price in an *incomplete/failed* response is rule 2, never UNAVAILABLE (satisfies "don't infer unavailability from a missing price alone").
4. **Ineligible** → `INELIGIBLE` when any of: FOEP `OFFER_NOT_ELIGIBLE` / `ASIN_NOT_ELIGIBLE`; an active `PRICING_HEALTH` `BuyBoxDisqualification` newer than the last successful eligibility confirmation; `is_buy_box_eligible = false`; `is_featured_merchant = false`. Reason precedence: `ASIN_NOT_ELIGIBLE` > `PRICE_TOO_HIGH` (pricing-health disqualification, or our landed price above `competitive_price_threshold`) > `OFFER_NOT_ELIGIBLE` > `NOT_FEATURED_MERCHANT`.
5. **Holds the Featured Offer** (`is_featured_offer`, cross-checked with `featured_seller_id` and fulfilment) → `AT_RISK` if §6.3, else `WINNING`.
6. Otherwise → `NOT_WINNING / ANOTHER_OFFER_FEATURED`.

### 6.2 Gap and tolerance

`landed_gap_vs_lowest = my landed − lowest competing landed` (competitors = `top_offers` excluding my seller id, same condition). NULL when there is no competitor. `PRICE_EPSILON` (constant) avoids flapping on rounding.

### 6.3 AT_RISK (only reachable from rule 5)

1. **Primary (D2)**: FOEP `resultStatus = VALID_FOEP` and `my listing price > foep_expected_price + PRICE_EPSILON` → `ABOVE_EXPECTED_PRICE`.
2. **Secondary**: a competitor offer in the same-or-better prime tier is cheaper than me by more than `AT_RISK_UNDERCUT_PCT` → `COMPETITOR_UNDERCUT`.
3. FOEP `NO_COMPETING_OFFER(S)` ⇒ no risk from FOEP. FOEP missing/old ⇒ rule 2 decides only for staleness; otherwise fall to secondary.

### 6.4 `BUY_BOX_THRESHOLDS` (single constant)

`STALE_AFTER_MS`, `POLL_INTERVAL_MS`, `FOEP_MAX_AGE_MS`, `FEES_MAX_AGE_MS`, `FEES_PRICE_CHANGE_PCT`, `AT_RISK_UNDERCUT_PCT`, `PRICE_EPSILON`, `EVENT_RETENTION_DAYS`, `NOTIFICATION_DEBOUNCE_MS`, `MANUAL_REFRESH_MIN_AGE_MS`, batch sizes (`COMPETITIVE_SUMMARY_BATCH=20`, `FOEP_BATCH=40`, `LISTING_OFFERS_BATCH=20`, `FEES_BATCH=20`). Defaults in §13 OI-1.

---

## 7. Data flow

### 7.1 Work key and fan-out

- Work unit = **`(seller_partner_id, amazon_marketplace_id)`**. API token = any healthy connected channel of that seller (try next on auth failure); region from that channel. Rate limiter key stays `waitForRateLimit(sellerPartnerId, api)`; add the new `AmazonApis` entries and `AMAZON_API_DELAYS`.
- Results are written to 5.1/5.2 **once**, then a single SQL re-evaluation selects all `tbl_product_buy_box_statuses` rows matching `(seller_partner_id, amazon_marketplace_id, sku|asin)` (every account/channel) and upserts status + event rows in bulk (`unnest` arrays). No per-channel API call, no per-account loop of Amazon calls.

### 7.2 Queue

New queue `GlobalEnums.QueueNames.BUY_BOX_REFRESH` + one `@Processor` class (existing one-processor-per-queue pattern), concurrency from `queueConfigs`. Job payload `{ sellerPartnerId, amazonMarketplaceId, skus?: string[], asins?: string[], sources?: ('OFFERS'|'COMPETITIVE_SUMMARY'|'FOEP'|'FEES')[], trigger }`. **Job id = `bbx:{seller}:{marketplace}`**, with requested SKUs/ASINs coalesced into a Redis set; a running job picks up what accumulated (so 50 accounts publishing at once or 500 notifications produce few jobs). Pipeline inside a job:

1. Resolve unique SKUs/ASINs (all accounts' status rows for this key, or the requested subset).
2. `getCompetitiveSummary` per unique **ASIN** (20/call, `includedData: featuredBuyingOptions, referencePrices, lowestPricedOffers`, `lowestPricedOffersInputs: Consumer` per distinct condition) → 5.1.
3. `getListingOffersBatch` per unique **SKU** (20/call; single `getListingOffers` when ≤ 2 SKUs) → `MyOffer`, eligibility, summary → 5.2 (+ 5.1 as a second source, newest `observed_at` wins).
4. `getFeaturedOfferExpectedPriceBatch` per unique SKU (40/call), skipped when `foep_fetched_at` is younger than `FOEP_MAX_AGE_MS` and no price/featured change was seen.
5. `getMyFeesEstimates` per SKU **only when** price changed more than `FEES_PRICE_CHANGE_PCT` vs `fees_price_basis` or the estimate is older than `FEES_MAX_AGE_MS` (fees move rarely; this removes ~all fee calls from routine polling). Price basis = my Amazon landed price, else Categra's configured offer price (the existing `getPriceFromDb` logic, kept for fees only, never for status).
6. Per-source success/failure is recorded on 5.2 (`*_fetched_at`, `last_error`); a failed source never overwrites good data with empties.
7. Re-evaluate and fan out (7.1).

### 7.3 Scheduled poll (safety net)

`@Cron` in api-main (worker has no cron) enumerates distinct `(seller, marketplace)` from active status rows and enqueues jobs **staggered** (deterministic jitter by hash) so the 0.033 rps APIs never see all sellers at once. A per-key Redis lock + job id prevent duplicates across restarts. A second cheap cron expires `DATA_STALE` (re-evaluates rows whose `data_as_of` crossed the threshold) and runs retention and key-drift reconcile.

### 7.4 Notifications

Register `ANY_OFFER_CHANGED` and `PRICING_HEALTH` in `AMAZON_NOTIFICATION_DEFINITIONS` (one entry each, per CLAUDE.md), destination SQS, payload version `1.0`, `marketplaceFilter: true`, new lane `PRICING` (or reuse LISTINGS if lane budgets allow), `autoSubscribe` behind `familyFlag`s. `ANY_OFFER_CHANGED` uses `eventFilter` + `aggregationSettings` (5-minute aggregation) to cut volume. One handler (`handlers/pricing/buy-box.handler.ts`):

- **ANY_OFFER_CHANGED**: parse via `fromAnyOfferChanged`; dedupe on `(marketplace, asin, condition, TimeOfOfferChange)`; if payload time ≤ stored `observed_at` → ignore; upsert 5.1 directly (no API call); mark my seller row(s) by `SellerId` + fulfilment (`is_featured_offer`, `is_featured_merchant`, landed price, prime); evaluate + fan out; then enqueue a **debounced** FOEP refresh for the affected SKUs (`NOTIFICATION_DEBOUNCE_MS`) because AOC carries no FOEP.
- **PRICING_HEALTH**: set `pricing_health_issue/at/details` on seller rows matched by `(seller, marketplace, asin, fulfillmentType)`; evaluate; enqueue FOEP refresh. Because Amazon sends **no recovery event**, the flag is cleared by the next successful FOEP/offer fetch that shows eligibility (rule order in 6.1.4 uses "newer than last eligibility confirmation").
- Routing: the existing `AmazonSubscriptionRouteResolver` already returns every channel across accounts for the provider subscription. The handler uses routes only to derive the distinct `(seller, marketplace)` set and to return `UNMATCHED` when none is connected; it does **not** loop per channel doing work. No new routing scope is needed.
- Orders of magnitude: AOC is the noisiest type in the registry; the handler is O(1) SQL per delivery and never calls Amazon.

### 7.5 Triggers that create/remove/refresh status rows

`BuyBoxStatusLifecycleService` (`ensureRows`, `markNotPublished`, `remove`, `refreshKeys`), called from: Amazon backward-sync BSQ4 (replaces `refreshAllSellableEntities`), forward-sync publish success, publish/unpublish of PCM/VM, SKU change, product/variant/channel deletion, product merge (`migrateProductCurrentOffers` becomes "repoint variant rows, delete standalone rows"). Rows start as `UNKNOWN / FETCH_PENDING` (or `NOT_PUBLISHED`) and a refresh job is enqueued for the key.

### 7.6 Manual refresh

Keep the user-facing refresh and the 1-hour `AMAZON_PRODUCT_OFFERS` resource cooldown. It now enqueues a targeted job for the SKU/ASIN; if the shared rows are younger than `MANUAL_REFRESH_MIN_AGE_MS` (another account just refreshed the same seller SKU) it skips the API calls and only re-evaluates.

---

## 8. Concurrency & correctness

- **Idempotent upserts** on the natural keys; `observed_at` guard on 5.1; per-source timestamps on 5.2 so a late poll cannot undo a newer notification.
- **Single evaluator** used by poll, notification and manual paths, so a status can never depend on which path wrote last.
- **No whole-document writes**: columns plus small JSON per source (lesson from the BS lifecycle race on whole-doc `JSON.SET`).
- Cross-account safety: shared tables hold no account data; only 5.3/5.4 carry `account_id`, and every read of them is channel-scoped through the existing `getUserWiseChannels` checks.
- Seller ↔ channel changes (reconnect with another seller id): reconcile cron rewrites `seller_partner_id` on status rows from `tbl_amazon_channels`.
- Condition: derived from `conditionType` via `CONDITION_TYPE_MAPPING` exactly as today; `CustomerType=Consumer` fixed on all three calls that accept it.

---

## 9. Rate/cost model

Per `(seller, marketplace)` with N active SKUs (≈ N ASINs): CS `N/20` calls at 30 s each, FOEP `N/40` at 30 s, listing offers batch `N/20`, fees ≈ 0 routinely. N = 1,000 ⇒ ≈ 25 + 12.5 min of limiter time for CS + FOEP, so default poll interval must be ≥ that; notifications carry the real-time load. Dedupe means a seller connected in 5 accounts costs the same as 1.

---

## 10. Code reshaping inventory (what changes where)

### 10.1 `db-migrations/`
1. Create 5.1–5.4; drop `tbl_product_current_offers` (one migration, generated with `migrate:create`).

### 10.2 `api/` — new
`entities/{amazonMarketOffer,amazonSellerOffer,productBuyBoxStatus,productBuyBoxStatusEvent}.entity.ts` (+ `index.ts` exports); `core/types/buy-box.type.ts`; `core/constants`: `BUY_BOX_STATUSES` (new set), `BUY_BOX_STATUS_REASONS`, `BUY_BOX_THRESHOLDS`, `AmazonApis` + delays, `QueueNames.BUY_BOX_REFRESH` + `queueConfigs`, two `AMAZON_NOTIFICATION_DEFINITIONS` entries; transport methods in `amazon.service.ts` (`getListingOffersBatch`, `getMyFeesEstimates`, `getCompetitiveSummary`, `getFeaturedOfferExpectedPriceBatch`); a `modules/app/catalog/products/listings/buy-box/` folder (module, `buy-box-refresh.service`, `buy-box-status.evaluator`, `buy-box-status-lifecycle.service`, `buy-box.mappers`, processor in `sync/queue/processors/`, cron service); notification handler under `platforms/amazon/events/handlers/pricing/`. JSDoc on all functions per lint rule.

### 10.3 `api/` — rewritten / removed
- `product-current-offers.{service,controller,module}.ts`, `ProductCurrentOffers` entity, `CurrentOffersData`/`AmazonProductOffersPayload`/old `BuyBoxDetails` types: removed; the refresh endpoint moves to the new buy-box module (same URL if the webapp already calls it).
- `overview/buy-box/buy-box.service.ts`: `buildBuyBoxDetails` + suggestion generation removed; replaced by a read mapper that builds the response from status + joined shared rows. `product-overview.service.ts`: the CTE at `:1804`, `findAll`/`findOne` at `:3279/:3580`, `buildFeesDetails` (`:312`, now reads fee columns), and the three `buildBuyBoxDetails` call sites (`:2150`, `:3406`, `:3762`).
- `order.service.ts:1605` margin query: join status → seller offer fee columns instead of parsing `raw_data` JSON (orders margin depends on referral fee, so fee columns are a hard requirement of the schema).
- Deletion/cleanup: `product-deletion.service.ts` (5 sites) and `productDeletionTables.txt` (4 lists), `channel.service.ts:1474`, `shopify-product.service.ts:3290` (the Amazon-offers delete there is dead for Shopify; remove), `product-structure-management.service.ts:1348` (merge).
- `amazon-product-bsq4.service.ts:306`: swap the post-commit step for `ensureRows + enqueue`.
- Module wiring: `app.module.ts`, `catalog.module.ts`, `product.module.ts`, `product-overview.module.ts`, `sync-orchestration.module.ts`.
- Subscriptions: provisioner/reconciler pick up the new definitions automatically; the old note "stop ANY_OFFER/PRICING_HEALTH" in the events refactor is explicitly reversed by this plan.

### 10.4 Action Center (out of scope, but must compile and not lie)
`action-center.service.ts` injects `BuyBoxService` (`:1616`) and reads `tbl_product_current_offers` (`:5151`) plus 90+ buy-box references. This plan does **not** redesign it: the minimal change is to remove those inputs so the module compiles and buy-box issue families produce nothing until Action Center is reshaped on top of the new status table. `actionEngineSettings` buy-box keys stay untouched. (Decision needed: §13 OI-9.)

### 10.5 `webapp/`
- `lib/constants.ts`: `BUY_BOX_STATUSES` + `BUY_BOX_STATUSE_COLORS` to the new set (WINNING success, AT_RISK alert, NOT_WINNING alert, INELIGIBLE destructive, UNAVAILABLE destructive/secondary, UNKNOWN secondary; final colours with design); delete unused `BUY_BOX_ALGORITHM` if dead.
- Types: `types/product.types.ts`, `types/productOverview.type.ts` (`BuyBoxDetails`: `status`, `statusReason`, `statusSince`, `dataAsOf`, `offerCount`, `yourLanded`, `lowestLanded`, `gapVsLowest`, `isEligible`, `fulfillment`, `isPrime`, `competitors[]`, fees; `suggestions` removed).
- Components: `marketPricingCard.tsx` (remove suggestions block, add reason + "as of"), `performanceByChannel.tsx`, `VariantPerformance.tsx`, `variant-columns.ts`, `variant-cell-renderer.tsx`, `simulatePriceDialog.tsx`, `details/util-components.tsx`.
- Translations (5 languages): status labels + reason labels in `translationServices/*/common.ts`.
- URL builders: any renamed refresh endpoint goes through `lib/api.ts` (no hardcoded URLs).
- Action Center webapp files that import buy-box types are only type-fixed, not redesigned.

---

## 11. Command Center

Attention cards read counts straight from 5.3 with the family mapping from the requirements: **Buy Box lost** (warning) = `NOT_WINNING`, **at risk** (warning) = `AT_RISK`, **ineligible** (critical) = `INELIGIBLE`. One grouped query `SELECT status, COUNT(*) … WHERE channel_id = ANY(:allowedChannels) [AND amazon_marketplace_id …] GROUP BY status` served by the `(channel_id, amazon_marketplace_id, status)` index. Count unit = status rows (listing = product/variant × channel × marketplace); whether to also expose "distinct products" is OI-8. `UNAVAILABLE`/`UNKNOWN` are not cards (the "Missing price" / data-trust cards already cover their cause).

---

## 12. Phases

| Phase | Content | Exit check |
|---|---|---|
| P0 | Verify live subscribe for `ANY_OFFER_CHANGED` / `PRICING_HEALTH` (why 29/29 ERROR), capture 3 real payloads each, confirm `getListingOffersBatch` and `getMyFeesEstimates` limits from headers | payload fixtures saved |
| P1 | Migration, entities, types, constants, mappers (unit-tested against doc examples + captured payloads) | mapper specs green |
| P2 | Transport methods, refresh service + queue + processor, evaluator (+ table-driven specs for every rule in §6), lifecycle service, BSQ4 swap | one seller×marketplace refreshes end to end |
| P3 | Cron poll/stale/retention/reconcile, notification definitions + handler | AOC and PH change a status without any API call |
| P4 | Product Overview read mapper, orders margin query, deletion/merge/channel cleanup, drop old module/table | no reference to the old table remains (grep) |
| P5 | Webapp types/components/translations; Command Center counts; Action Center detach | lint/format/build in all three repos |

---

## 13. Open items — RESOLVED 2026-10-15 (before implementation)

Answers: OI-1 defaults accepted; OI-2 any segment; OI-3 `NOT_WINNING / NO_FEATURED_OFFER`; OI-4 `UNAVAILABLE / NOT_BUYABLE`; OI-7 no parent rows; OI-8 rows; OI-9 strip Action Center inputs; **OI-6 changed: suggestions are NOT removed — replaced by new guidance generated from the stored `status + status_reason` ("what phase is this listing in, and what exactly to do"), exact unambiguous wording, translated in all 5 languages, no free-form heuristics.** OI-5/OI-10 defaults stand.

Original table (kept for reference; the OI-6 row is superseded):

| # | Question | Default in this plan |
|---|---|---|
| OI-1 | Numbers: poll interval, stale window, FOEP age, fee refresh, undercut %, epsilon | `POLL_INTERVAL` 6 h, `STALE_AFTER` 24 h, `FOEP_MAX_AGE` 1 h, `FEES_MAX_AGE` 7 d, `FEES_PRICE_CHANGE_PCT` 5, `AT_RISK_UNDERCUT_PCT` 1, `PRICE_EPSILON` 0.01, debounce 5 min, manual min age 10 min (all in `BUY_BOX_THRESHOLDS`) |
| OI-2 | Several Featured Offer segments (Prime / non-Prime / default): when are we "winning"? | Winning if we hold it in **any** segment; segment weights are stored in `featured_offers` for later |
| OI-3 | Eligible, price present, but **no** featured offer exists (suppressed Buy Box) | `NOT_WINNING / NO_FEATURED_OFFER` (the old `NO_BUY_BOX` status is gone) |
| OI-4 | Listing exists with a price but Amazon listing status lacks `BUYABLE` | `UNAVAILABLE / NOT_BUYABLE` using the already-stored `external_status`; confirm or drop |
| OI-5 | `NOT_WINNING` is now the only "behind" state (no `NOT_COMPETITIVE`): is the price gap (stored) enough to describe "far off"? | yes, gap is shown, no extra status |
| OI-6 | The old rule-based "suggestions" text on the product overview is removed (not in the new requirements) | removed |
| OI-7 | Should variants-parent rows exist? | no row for parents with variants |
| OI-8 | Command Center counts: listings (rows) or distinct products? | rows |
| OI-9 | Action Center detach: strip its buy-box inputs now, or leave the build broken until its redesign? | strip inputs now (minimal) |
| OI-10 | "Winning chance" estimate from the requirements' clarifications | not built (no source data in the listed APIs) |

---

## 14. Implementation status (2026-10-15/16)

Built: migration `20261015115539-buy-box-status-tables-and-drop-product-current-offers.js`; 4 entities; types/constants (`BUY_BOX_THRESHOLDS` in `core/constants/buy-box.constants.ts`); mappers, evaluator, guidance builder (+ specs, 33 api tests green); store/queue/refresh/lifecycle/read/manual-refresh/schedule services + `BuyBoxStatusModule`; `buy-box-refresh` queue + processor; crons (hourly poll/stale, daily reconcile/purge); `ANY_OFFER_CHANGED` + `PRICING_HEALTH` definitions and handler; old module/table/service removed; overview, orders margin, deletion, merge, BSQ4, listing-notification hooks reshaped; Action Center buy-box inputs stripped; Command Center families BUY_BOX_LOST / AT_RISK / INELIGIBLE; webapp types, card, dialog, translations (5 languages).

Deviations from the plan above: `foep_expected_price` is a listing price (compared with our listing price); status table gained `is_published` / `is_buyable`; seller table gained `eligibility_confirmed_at`; refresh endpoint is `POST buy-box/refresh` (old `product-current-offers/*` removed); competitors response is a flat list (`landed/listing/shipping`); `PRICING_HEALTH` payload mapper is defensive until P0 fixtures exist.

Not done / needs a human: run the migration; P0 live capture of notification payloads and header rate limits; Action Center redesign; webapp Action Center files still reference the string `NO_BUY_BOX` (dead).

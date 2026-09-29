# Command Center v2 — Analysis (API focus, Amazon first)

> Scope: `api/` repo only. Input: `categra_command_center_full_prototype_v3.html`, `initial-findings.md`, the current Command Center (v1) backend + webapp page, and live SP-API docs (checked 2026-09-29).
> Status legend used throughout:
> - ✅ **Present**: the data exists in Categra today and can be read directly.
> - 🟡 **Partial**: some data or plumbing exists, but it is incomplete, runs only on demand, has no history, or has a bug.
> - 🔴 **External (SP-API)**: needs a new Amazon SP-API integration (report, API call or notification) we don't use yet.
> - ⛔ **Outside the project**: needs a separate product or API outside SP-API (Amazon Ads API) or new data the user must enter (such as product cost).

---

## 0. TL;DR

1. **v1 is mostly a view over Action Center**, not an intelligence layer. v1's `now`, `next`, `atRiskProducts` and `stock` sections all come from one call, `ActionCenterService.getSalesOperationsSnapshot()`. Only `trend` (orders), `catalogStatus` (readiness), `capacity` (plan limits) and `amazonEvents` (event pipeline) come from anywhere else. v2 needs **new data sources**, not just a new UI.
2. **We already have a lot of Amazon plumbing:** notification ingest (SQS + EventBridge), auto-subscription, raw event storage, a multi-stage event pipeline, listing status and issues handling, orders, FBA inventory table, offers and fees snapshots, and warehouse stock. v2 can build on it.
3. **Two existing parsers look broken against real Amazon payloads** (found in code, still to be confirmed against stored raw messages). Both must be fixed before v2 ranks anything on them:
   - `ACCOUNT_STATUS_CHANGED`: Amazon sends `payload.accountStatusChangeNotification.currentAccountStatus` ∈ `NORMAL | AT_RISK | DEACTIVATED`. Neither handler reads that field (details in §4.1).
   - `ANY_OFFER_CHANGED`: the buy-box normalizer uses free-text regexes such as "lost the buy box". It never finds the seller's own offer via `SellerId`/`IsBuyBoxWinner`, and it extracts no prices (§4.4).
4. **Periodic Amazon data pulls do not exist.** `cron.service.ts` has no Amazon inventory, offers, health, traffic or finance jobs. FBA inventory and offers refresh only during backward sync, order processing or a manual button. v2 needs a scheduled ingestion layer: cron in api-main → BullMQ jobs in api-worker → snapshot tables.
5. **Some prototype cards can't be backed today without new SP-API roles:**
   - Account health metrics need *Selling Partner Insights*.
   - Conversion and sessions need *Brand Analytics*.
   - Real fees and profit need *Finance and Accounting*.
   - Stranded and aged FBA inventory need *Amazon Fulfillment*.
   - Advertising needs the separate **Amazon Ads API**, with its own onboarding.

---

## 1. Open questions (need your decision before implementation)

These change the design, so they're collected here instead of guessed. My suggested default is in *italics*.

| # | Question | Why it matters | Suggested default |
|---|---|---|---|
| Q1 | Which **SP-API roles** does our registered app have today (Selling Partner Insights, Brand Analytics, Amazon Fulfillment, Finance and Accounting, Pricing)? The code has no record of them. | Decides which pulse tiles can be real in phase 1 versus shown as "Permission required". | *Check the Developer Central app profile; build the per-channel capability check (§6.9) either way.* |
| Q2 | **Scope selector granularity.** The prototype uses "Amazon NA · Seller A", which in Categra = one `Channel` (seller + region). Is a marketplace-level filter still needed? v1 already accepts `marketplaceId`. | Affects DTO, cache keys and every aggregate. | *Channel + optional marketplace, same as v1.* |
| Q3 | **Product cost (COGS) source for profit.** Only `tbl_warehouse_stocks.purchase_price` exists: per stock entry, per warehouse, and absent for FBA-only sellers. | Profit and gross margin are meaningless without a trusted cost. | *Add an explicit per-variant cost field (user-maintained) and fall back to the latest `purchase_price`.* |
| Q4 | **"Shared stock" definition.** The prototype example ("Shared pool 12 — Amazon FBA holds 14 → oversell by 2") mixes FBA units, which physically sit in Amazon warehouses, with a Categra pool. In our model, FBA stock is never part of a Categra warehouse pool. | Decides what a "cross-channel conflict" really is. | *Define a conflict as: a Categra warehouse pool mapped to 2+ channels (Amazon FBM + Shopify) where remote quantities ≠ what Categra pushed, or the sum of reservations > on hand.* |
| Q5 | **Easy Ship.** Should it be a separate inventory bucket? Easy Ship = merchant-held stock where Amazon collects the parcel. For inventory it behaves like FBM; it differs only in the shipping flow. Easy Ship API support is limited to certain marketplaces. | Adds a third bucket or not. | *Treat it as FBM for stock; tag Easy Ship orders separately in order signals.* |
| Q6 | **Signal freshness and "data trust".** Should stale data *suppress* downstream conclusions, as the prototype's "Categra suppresses Germany inventory conclusions" says? | Needs per-marketplace freshness tracking, which doesn't exist. | *Yes, but only once scheduled syncs exist; until then, show "last synced" without suppression.* |
| Q7 | **Notifications vs Command Center.** Initial findings say "show in command center **or as notification**". Should Amazon events also push in-app/email alerts? A flagged pipeline exists: `AMAZON_EVENTS_NOTIFICATIONS_*`. | Scope of the work. | *CC + Action Center first; reuse the existing notifications pipeline for critical families only.* |
| Q8 | **Auto-repricing.** Initial findings 4.1.5.2 mention "update price automatically". | This is an action engine, not a dashboard feature, and it carries risk: the in-code event catalog warns that `ANY_OFFER_CHANGED` "as a direct repricing trigger can create rapid offer churn". | *Out of v2 CC scope; CC shows the suggested price (FOEP) and deep-links to pricing.* |
| Q9 | **History retention** for offer, traffic and health snapshots (volatility and trends need history). | Table growth. | *Daily rollups 13 months; raw offer snapshots 30 days.* |
| Q10 | Should the v1 endpoints (`/command-center/summary`, etc.) be **removed or kept** while the new webapp is built? | Migration plan. | *Add `/command-center/v2/*` endpoints and delete v1 once the webapp switches.* |

---

## 2. What the current Command Center (v1) does — explained

**Backend:** `api/apps/api-main/src/modules/app/system/operations/orchestration/command-center/`
- **Controller:** `command-center.controller.ts`. All endpoints are `POST` and need the `ACTION_CENTER_VIEW` subscription capability.
- **Body:** `{ channelId?, marketplaceId?, period?: 7d|30d|90d|6m|12m }`.
- **Cache:** every section is cached in Redis for **90 s**. The key includes account, user, channel access, channel, marketplace, period and currency (`command-center.service.ts` → `buildCacheKey`).

| Endpoint | v1 section (webapp) | Where the data comes from | What it means |
|---|---|---|---|
| `summary` | all below (except amazonEvents) | fan-out of the 4 below | One-shot payload for the page |
| `operational-summary` | **Now**, **Next**, **At-risk products**, **Stock** | `ActionCenterService.getSalesOperationsSnapshot()` (`action-center.service.ts:2924`) → rows normalized by `operational-rows.ts` | **Now** (`now.builder.ts`): counts of open urgent / in-progress / blocked Action Center work items, plus the top issue signals grouped by type. **Next** (`next.builder.ts`): canned recommendations per issue type ("Restock unavailable products", …). **At-risk** (`at-risk.builder.ts`): top products by a fixed issue priority. **Stock** (`stock.builder.ts`): count of OUT_OF_STOCK / LOW_STOCK rows. |
| `catalog-status` | **Catalog status** | `catalog-status.service.ts` (readiness rows + publish rows per channel/marketplace) | Readiness buckets (ready / almost / blocked / needs attention) and publishing buckets (published / not published / inactive / out of sync) |
| `capacity-summary` | **Plan capacity** | `PackageSubscriptionManagerService.getPackageResourcesSnapshot` | Channels / products / storage used vs plan limit |
| `trend-summary` | **Trend chart** | `OrderService.getCommandCenterTrendData` (orders table, converted to base currency) | Revenue and orders for the current vs previous period, plus a daily series |
| `amazon-events` | **Amazon events** | `tbl_amazon_event_projection_candidate` (active projection generation), feature-flagged | Up to 5 cards: critical account restriction, degraded account, price-policy risk, and Buy Box lost/excluded on **protected** listings (`product_channel.is_listing_protected`) |

**Key takeaway:** v1 "Now / Next / At-risk / Stock" are *Action Center rows re-counted*. v2's ranked priorities can keep using that snapshot for product-level issues. But **account-level, data-trust, business and fulfillment-breakdown signals have no source today.**

### 2.1 The Action Center model v1 depends on
- **Issue types** (legacy, `action-center.service.ts` ~L1560): `CHANNEL_NOT_CONNECTED`, `CHANNEL_CONNECTION_ERROR`, `CHANNEL_SETUP_INCOMPLETE`, `SYNC_FAILED`, `OUT_OF_SYNC`, `AMAZON_ERROR`, `AMAZON_WARNING`, `CHANNEL_ERROR/WARNING`, `MISSING_PRICE`, `OUT_OF_STOCK`, `LOW_STOCK`, `SHOPIFY_UNPUBLISHED/INACTIVE`, `LISTING_STATUS_PENDING`, `AMAZON_NOT_SEARCHABLE`, `AMAZON_MISSING_OFFER`, `PRICE_POLICY_BLOCKED`, `BUY_BOX_LOST/EXCLUDED`, plus readiness (`LOW_READINESS`, `MISSING_MEDIA/CONTENT/ATTRIBUTES`).
- **Compound families** (`actionCenter.compound.ts`) group listing issues (e.g. `LISTING_REJECTED`, `COMPLIANCE_RESTRICTION`, `INVALID_REQUIRED_ATTRIBUTES`).
- **Priority groups** (configurable per account, `action-engine-settings`), in order:
  - Pinned first: `CRITICAL_SYSTEM_BLOCKERS`.
  - Sortable after it: `LISTING_BLOCKERS`, `OUT_OF_STOCK`, `PRICING_ISSUES`, `LISTING_WARNINGS`, `LOW_STOCK`, `READINESS_CONTENT_QUALITY`, `INFORMATIONAL_ADVISORY`.
- **Health:** `BLOCKED` / `AT_RISK` / `NEEDS_ATTENTION` / `HEALTHY`. **Workflow:** `OPEN`, `IN_PROGRESS`, `PENDING_VERIFICATION`, `WAITING_BLOCKED`, `SNOOZED`, `CLOSED`, `CLEARED`, with owners and assignment.
- **Handoff contract** already used by CC → Action Center: `CommandCenterActionCenterFilters` (`mode: 'SALES_OPERATIONS', issueType, channelId, marketplaceId, canonicalGroupKey, severity, workflowStatus…`).
  - This matches the prototype's "handoff payload" idea.
  - The prototype's categories `ACCOUNT_HEALTH` and `CONNECTION(DE)` do **not** exist as Action Center categories yet.

### 2.2 The Amazon events pipeline ("shadow" pipeline) — what those tables are
The chain is synchronous per notification. It is triggered from the raw ingest (`amazon-events-raw.service.ts` → `amazon-events-scope.service.ts:1976-2086`).

```
SQS/EventBridge POST /amazon-notifications/{sqs|event-bridge}
  → tbl_amazon_event_raw_(delivery|message)         raw payload, dedupe, processing attempts
  → scope resolution (tbl_amazon_event_scope_*)       map seller/marketplace/ASIN/SKU → account/channel/product
  → normalizers: account | price-policy | buy-box     → tbl_amazon_event_normalized_observation
  → occurrence lifecycle (tbl_amazon_event_occurrence[_revision])  OPEN / REFRESHED / COMPLETED / SUPERSEDED
  → suppression edges (stronger signal hides weaker)
  → projection candidates (target: ACTION_CENTER / COMMAND_CENTER / BOTH / ADMIN_ONLY)
  → Action Center materialization (only PRICE_POLICY_BLOCKED, BUY_BOX_LOST, BUY_BOX_EXCLUDED)
  → user notifications (in-app/email; flag-gated)
```
- **"Shadow"/"generation":** every rule set has a version (e.g. `projection-shadow-v3`). Results are written into a *generation* so rules can be re-evaluated without touching live data. CC reads only the `ACTIVE` projection generation.
- **Heavily flag-gated.** Flags include `AMAZON_EVENTS_LIVE_ENABLED`, `AMAZON_EVENTS_COMMAND_CENTER_*`, `AMAZON_EVENTS_ACTION_CENTER_*`, `AMAZON_EVENTS_AUTO_SUBSCRIBE_*` and `AMAZON_EVENTS_NOTIFICATIONS_*`, plus account and channel allowlists.
- **Auto-subscribed notification types** (`amazonEventsAutoSubscription.constants.ts`): `ACCOUNT_STATUS_CHANGED`, `ORDER_CHANGE`, `PRICING_HEALTH`, `ANY_OFFER_CHANGED`, `FULFILLMENT_ORDER_STATUS`. Flags also exist for `LISTINGS_ITEM_STATUS_CHANGE`, `LISTINGS_ITEM_ISSUES_CHANGE` and `ITEM_PRODUCT_TYPE_CHANGE`.
- **Legacy handlers** (`amazon-notifications.service.ts`) run separately and directly update products and channels:
  - listing status and issues
  - product type change (+ platform notification)
  - feed and report finished
  - account status
  - order change

---

## 3. Prototype walkthrough — section by section, with purpose explained

| # | Prototype section | Purpose (plain words) | Verdict |
|---|---|---|---|
| A | **Top bar:** scope select + "Verified 2m ago" | Filter everything to one channel (seller + region) or store. "Verified" = how fresh the underlying data is. | **Keep.** Scope = `channelId` (+ optional `marketplaceId`). "Verified" must be the *oldest* freshness among the sources used, not the request time (v1 `meta.generatedAt` is request time). |
| B | **Intelligence pulse** (6 tiles) | One-glance state per domain: Account Health, Inventory, Listings, Pricing, Sales & Profit, Advertising. | **Keep, modify.** Add a **Catalog/Readiness** tile (from v1 catalog-status). Show Advertising only once connected (not a "Future" placeholder). Each tile needs 3 states: *value* / *permission required* / *not connected*. |
| C | **Business pulse** (Sales, Gross margin, Conversion, Orders + "Why") | Business outcomes with a short cause explanation. | **Modify.** Sales and Orders are ✅. Conversion needs the traffic report (🔴). Gross margin needs cost + fees (🟡/🔴). The "Why" line needs an attribution engine; start rule-based (§6.10) and hide it when there's no confident cause. |
| D | **Ranked priorities** (top 2 expanded + 3 compact) | One ordered list across *all* domains, most damaging first. | **Keep** — this is the core of v2. Needs a new cross-domain ranker (§6.1). Existing Action Center priority groups cover product-level items only. |
| E | **Recommended sequence** | Fix order that respects dependencies (account → connection/data trust → stock → listings). | **Modify.** It mostly duplicates D. Suggest merging it into D as a "do this first" badge, or deriving it from D's ranks with fixed dependency rules. v1 `next.builder.ts` already has canned recommendation texts. |
| F | **Fulfillment & availability** (FBA / FBM / shared stock / Shopify) | "Out of stock" has different causes and owners per fulfillment type. | **Keep.** FBA and FBM counts are possible today (🟡). Sub-reasons (stranded, inbound delayed, aged) need FBA reports (🔴). Redefine "shared stock" per Q4. |
| G1 | **Channel & marketplace health** | Per seller account / region / store status. | **Keep.** Connection status ✅. Account health ⚠ (bug §4.1). Per-marketplace staleness 🔴. Move the secondary "Permissions" item here as a badge. |
| G2 | **What changed?** | Timeline of recent external changes. | **Keep.** Sources exist but are spread out: event occurrences, Action Center lifecycle events, product type change notifications, Shopify webhooks. Needs a unified feed query (§6.8). |
| G3 | **Cross-channel conflict** | Oversell risk across channels sharing stock. | **Modify** per Q4. Nothing detects it today (`data-reconciliation.service.ts` is an 11-line stub). |
| H | **Secondary intelligence** (Returns, Fees & margin, A+ content, Advertising, Permissions) | Lower-priority signals. | **Keep, trim.** Returns 🔴, fees 🟡, A+ 🔴 (brand registry). Merge Advertising into B. Move Permissions to G1. |
| I | **Footer + Action Center drawer** | Everything actionable routes to Action Center with scope preserved. | **Keep.** The contract already exists (`CommandCenterActionCenterFilters`). Add new Action Center categories if D includes account or connection items. |

**Removed from v1 with no prototype equivalent — suggestions:**
- **Plan capacity:** only show as a banner when usage is ≥ 80% (it's billing info, not operations).
- **Trend chart:** fold into Business pulse (sparkline).
- **At-risk products list:** covered by D + Action Center.
- **Catalog status:** becomes tile B "Catalog".

**Suggested additions (not in prototype):**
- **Orders needing action:** unshipped FBM orders past the latest ship date, buyer cancellation requests. Data ✅ (§4.2).
- **Days of cover / stock-out forecast:** needed for findings #1 "near threshold in 3 days" (§4.2).
- **Notification pipeline health** (admin-side, or a small "data trust" signal): whether SQS deliveries are arriving. `AMAZON_EVENT_ADMIN_RECEIVING_STATUSES` already exists.

---

## 4. Domain analysis — what exists, what's partial, what's external

### 4.1 Account Health

| Item | Status | Where / How |
|---|---|---|
| Subscription to `ACCOUNT_STATUS_CHANGED` | ✅ | Auto-subscription (SQS, grantless destination). Payload version `2021-01-01`. |
| Status value `NORMAL / AT_RISK / DEACTIVATED` | ⚠ **Bug** | Details below this table. |
| Account health metrics: ODR, late shipment, cancellation, valid tracking, on-time delivery, policy violations (IP, authenticity, safety, listing policy), AHR warning states | 🔴 | `GET_V2_SELLER_PERFORMANCE_REPORT` (structured, AFN/MFN split). `GET_V1_SELLER_PERFORMANCE_REPORT` is XML. Both are **request-only (not schedulable)**, sellers only, and need role **Selling Partner Insights**. |
| Periodic pull + "major drop" detection | 🔴 | New cron → createReport → the `REPORT_PROCESSING_FINISHED` handler (exists, but only routes `GET_XML_BROWSE_TREE_DATA`, `amazon-notifications.service.ts:2315`) → parse → snapshot table → diff against previous. |

**The status bug in detail:**
- **Legacy handler** (`amazon-notifications.service.ts:3804`, `resolveAccountStatusFromPayload`):
  - It reads `accountStatus/AccountStatus/status`, but Amazon sends `currentAccountStatus`.
  - It then falls back to a regex over the whole JSON. That regex matches `NORMAL` from `previousAccountStatus`, so **AT_RISK is mapped to CONNECTED**.
- **Events normalizer** (`amazon-events-account.service.ts` L74-165) matches phrases like "suspended" and "under review". None of these appear in the enum values, so `ACCOUNT_DEGRADED` is very likely never produced.
- **Verify:** run `SELECT payload FROM tbl_amazon_event_raw_message WHERE raw_notification_type='ACCOUNT_STATUS_CHANGED'`.

**Scope note:** `ACCOUNT_STATUS_CHANGED` has no marketplace in its payload. It maps to the seller account per region, i.e. one Categra `Channel` (`tbl_amazon_channel` has `region_id`). The prototype's "North America" is therefore channel-level, which matches our model.

**Suggested approach:**
1. Fix both parsers to read `currentAccountStatus` / `previousAccountStatus` structurally.
2. Map `AT_RISK` → `ACCOUNT_DEGRADED` and `DEACTIVATED` → `ACCOUNT_CRITICAL_RESTRICTION`.
3. Keep `channel.connection_status` for DEACTIVATED only.
4. Add `tbl_amazon_account_health_snapshot` with these columns:

   | Column | Content |
   |---|---|
   | ids | channel_id, amazon_channel_id, marketplace_id |
   | captured_at | snapshot time |
   | account_status | from the notification or report |
   | metrics | ahr_status, odr, lsr, cancel_rate, vtr, otdr |
   | policy_violation_counts_json | counts per violation category |
   | report_id | source report |

5. Trigger the report daily **and** immediately on every `ACCOUNT_STATUS_CHANGED`.
6. Flag a "major drop" when a metric crosses Amazon's target (ODR > 1%, LSR > 4%, cancel > 2.5%, VTR < 95%) or the AHR state worsens.

### 4.2 Inventory (FBA / FBM / Easy Ship / Shared / Shopify)

**How stock is resolved today** (Action Center SQL, `action-center.service.ts` ~L14270-14316):
- **FBA** (`variant_marketplace.is_fba` / `product_channel_marketplace.is_fba`, or a warehouse flagged as an Amazon fulfillment center): stock = `tbl_amazon_fba_inventories` fulfillable qty. **No low-stock threshold for FBA** (threshold forced to 0).
- **FBM with mapped Categra warehouses** (`tbl_channel_product_warehouses`): stock = sum of warehouse stock; threshold = `tbl_product_variant_stocks.low_stock_threshold`.
- **Otherwise:** `amazon_stock` column (last value pushed/pulled).

| Item | Status | Where / How |
|---|---|---|
| FBA/FBM flag per product/variant/marketplace | ✅ | `is_fba` on `tbl_product_channel_marketplaces` / `tbl_variant_marketplaces` |
| FBA quantities (fulfillable, inbound working/shipped/receiving, reserved, researching, unfulfillable, total) | 🟡 | `tbl_amazon_fba_inventories`, filled by `AmazonFbaInventoryService.fetchFbaInventory` (FBA Inventory API `getInventorySummaries`, **one SKU per call**). Only called from backward sync BSQ2, Amazon order processing and the manual stock UI. **No schedule, no history.** |
| FBA near-real-time updates | 🔴 | `FBA_INVENTORY_AVAILABILITY_CHANGES` (SQS; payload has per-marketplace Fulfillable, Unfulfillable, Inbound/Reserved breakdowns, FutureSupplyBuyable). Not subscribed. The catalog marks it "high noise". |
| FBA stranded | 🔴 | `GET_STRANDED_INVENTORY_UI_DATA` (request-only; Pricing or Amazon Fulfillment role) |
| FBA aged / excess, days of supply, recommended action | 🔴 | `GET_FBA_INVENTORY_PLANNING_DATA` (Amazon Fulfillment) |
| FBA inbound delayed | 🔴 | Fulfillment Inbound API (shipment status) — new integration |
| FBA restock recommendation | 🔴 | `GET_RESTOCK_INVENTORY_RECOMMENDATIONS_REPORT` |
| FBM stock (Categra warehouses) | ✅ | `tbl_warehouse_stocks`, `tbl_warehouse_products`, `tbl_channel_product_warehouses`; per-marketplace `sync_stock_with_categra` flag |
| FBM quantity on Amazon vs Categra (mismatch) | 🔴 | `LISTINGS_ITEM_MFN_QUANTITY_CHANGE` notification (not subscribed), or a periodic `GET_MERCHANT_LISTINGS_ALL_DATA` compare |
| FBM late handling | 🟡 | `orderAttention.ts:155` `isAmazonLateShipWindow` (MFN orders, `latest_ship_date`). Only evaluated during order sync, not scheduled; not aggregated for CC. |
| FBM cancellation risk | 🟡 | `tbl_order_items.buyer_requested_cancel`, `tbl_orders.order_status`; needs an aggregate query |
| Easy Ship | 🟡 | No explicit handling. Orders store `fulfillment_channel` and `shipment_service_level_category`; the raw order JSON (`raw_order_data`) carries Easy Ship fields. See Q5. |
| "Near threshold in 3 days" (findings #1.1) | 🟡 | Needs **sales velocity**. Units/day per variant/marketplace can be computed from `tbl_order_items` (✅ data). Days of cover = available / avg daily units (7d or 30d). New computation + daily cron. |
| Shared-pool oversell | 🟡/🔴 | Categra warehouses can map to many channels, but there is no conflict detection (`data-reconciliation.service.ts` is a stub). See Q4. |
| Shopify inventory | ✅ | `inventory_levels/update` webhook (`shopify-webhook.service.ts`); Shopify stock flags in Action Center |
| Channel `last_inventory_sync` | 🟡 | Written only on backward sync (import), so it is **not** a freshness signal today (`amazon-product-bs.service.ts:535`) |

**Suggested approach:**
1. Replace the per-SKU FBA fetch with a **scheduled bulk pull**: `getInventorySummaries` with `details=true`, paginated per marketplace, or `GET_FBA_MYI_UNSUPPRESSED_INVENTORY_DATA`. Write `tbl_amazon_fba_inventories` plus a daily history table.
2. Optionally subscribe to `FBA_INVENTORY_AVAILABILITY_CHANGES` for faster updates. It is observability only; it must never overwrite Categra FBM stock.
3. Add a daily **velocity/days-of-cover** job over `tbl_order_items`, and emit `STOCK_OUT_FORECAST` when cover ≤ N days (N configurable, default 3).
4. Weekly FBA health reports: stranded, planning, restock.

### 4.3 Listings (suppression, buyability, structure)

| Item | Status | Where / How |
|---|---|---|
| `LISTINGS_ITEM_STATUS_CHANGE` (BUYABLE / DISCOVERABLE) | ✅ | Legacy handler `processListingItemStatusChange`; stored in `external_status` |
| `LISTINGS_ITEM_ISSUES_CHANGE` (issues, severities, enforcement actions such as search/listing suppression) | ✅ | `processListingItemIssueChange`; stored in `listing_errors`, `external_errors`, `amazon_enforcement_actions`, `tbl_channel_product_errors` |
| Action Center issue types | ✅ | `AMAZON_ERROR`, `AMAZON_WARNING`, `AMAZON_NOT_SEARCHABLE`, `AMAZON_MISSING_OFFER`, `LISTING_STATUS_PENDING`, compound `LISTING_REJECTED`, `COMPLIANCE_RESTRICTION`, `INACTIVE_LISTING` |
| Product type changed → remap | ✅ | `ITEM_PRODUCT_TYPE_CHANGE` → `processProductTypeChangeNotification` + platform notification (`amazon-notifications.service.ts:3552/3835`). This is the prototype's "Product Type changed — 4 listings need remap". |
| Product type *definition* (schema) changed | 🔴 | `PRODUCT_TYPE_DEFINITIONS_CHANGE` not subscribed |
| Readiness / publishing | ✅ | v1 `catalog-status` |

**Approach:** mostly aggregation. The Listings tile = count of open Action Center rows in `LISTING_BLOCKERS` + `LISTING_WARNINGS`, split into suppression / not buyable / structure (PTD) / other.

### 4.4 Pricing & Competitiveness (Buy Box / Featured Offer)

| Item | Status | Where / How |
|---|---|---|
| `ANY_OFFER_CHANGED` ingest | ✅ | Auto-subscribed per marketplace (`MarketplaceIds` filter), SQS, raw stored |
| Buy Box lost/excluded detection | ⚠ **Heuristic** | Details below this table. |
| Lowest price / Buy Box price / offer count / competitive threshold | 🟡 | `tbl_product_current_offers.raw_data` holds the `getListingOffers` response (`Summary.LowestPrices`, `BuyBoxPrices`, `NumberOfOffers`, `Offers[]`) + `feesEstimate`. Refreshed only manually (`refreshProductOffers`/`refreshAllSellableEntities`) and at the end of backward sync (BSQ4). One row per entity, **no history**. |
| `PRICING_HEALTH` (offer not eligible for Featured Offer due to uncompetitive price; includes Buy Box price, competitive price threshold, 60-day average selling price, retail offer price) | 🟡 | Subscribed and flows through the price-policy normalizer (same text-matching style; verify it against real payloads). Materialized into Action Center as `PRICE_POLICY_BLOCKED`. |
| Featured Offer Expected Price (FOEP) = "target price" | 🔴 | Product Pricing v2022-05-01 `getFeaturedOfferExpectedPriceBatch` (≤40 SKUs/call, **0.033 rps ≈ 1 call per 30 s**). Result statuses: `VALID_FOEP`, `NO_COMPETING_OFFERS`, `OFFER_NOT_ELIGIBLE`, `OFFER_NOT_FOUND`, `ASIN_NOT_ELIGIBLE`. |
| Competitive summary / reference prices | 🔴 | `getCompetitiveSummary` (≤20 ASINs/call, 0.033 rps): featured buying options, lowest priced offers, reference prices (`CompetitivePriceThreshold`, `WasPrice`, `CompetitivePrice`) |
| Volatility (frequent Buy Box flips), competitive pressure (offer count / price spread change) | 🔴 | Needs a **history table** of structured offer snapshots (from `ANY_OFFER_CHANGED` + scheduled refresh) |
| Price opportunity (margin increase while keeping the Buy Box) | 🔴 | FOEP − own price > 0 while winning → headroom. Needs cost for a margin (Q3). |
| Shipping / Prime / fulfillment change detection | 🔴 | Offer snapshot history (`Offers[].ShippingTime`, `PrimeInformation`, `IsFulfilledByAmazon`) diff |

**Why the Buy Box detection is only heuristic:**
- The normalizer (`amazon-events-buy-box.service.ts` L99-175, 1035-1070) regex-matches phrases like "lost the buy box" and a few boolean paths.
- Real `ANY_OFFER_CHANGED` payloads are **structured**: `Summary.BuyBoxPrices`, `LowestPrices`, `NumberOfOffers`, `NumberOfBuyBoxEligibleOffers`, `ListPrice`, `SalesRankings`, and `Offers[]` with `SellerId`, `IsBuyBoxWinner`, `IsFeaturedMerchant`, `ListingPrice`, `Shipping`, `ShippingTime`, `PrimeInformation`, `IsFulfilledByAmazon`.
- The code never compares `Offers[i].SellerId` to our `amazon_channel.seller_partner_id`, so it cannot know whether *we* lost the Buy Box.
- No prices are extracted.

**Suggested approach:**
1. Add a **structured offer parser**:
   - own offer = the `Offers[]` entry whose `SellerId` = our seller id
   - `ownIsWinner` = `IsBuyBoxWinner`
   - read BuyBox landed price, lowest landed price (FBA/MFN), offer counts, `CompetitivePriceThreshold`
2. Persist to `tbl_amazon_offer_snapshot` (append-only, 30-day retention) and upsert the latest into `tbl_product_current_offers` or a typed "latest" table.
3. Derive the Buy Box families from facts:
   - **LOST:** previous own winner → now not
   - **EXCLUDED:** own offer is not among the eligible offers
   - **UNAVAILABLE:** no Buy Box price at all
4. Scheduled FOEP batch for **listing-protected and top-revenue SKUs only**, because of the rate limits.
5. Volatility = number of Buy Box winner changes per ASIN over 24h/7d.

### 4.5 Sales, Traffic & Conversion

| Item | Status | Where / How |
|---|---|---|
| Orders count, revenue, period comparison, daily series, currency conversion | ✅ | `OrderService.getCommandCenterTrendData` (`order.service.ts:1489`); orders arrive via `ORDER_CHANGE` notifications + `GET_FLAT_FILE_ALL_ORDERS_DATA_BY_ORDER_DATE_GENERAL` report + Shopify webhooks. Exchange rates via cron every 2 days. |
| Per-product sales trend / declining products | 🟡 | Computable from `tbl_order_items` (product/variant linked). New aggregate needed. |
| Sessions, page views, Buy Box %, unit session % (conversion), ordered product sales/units by ASIN/SKU | 🔴 | `GET_SALES_AND_TRAFFIC_REPORT` (JSON; role **Brand Analytics**, brand registry **not** required; request or schedule; granularity DAY/WEEK/MONTH × PARENT/CHILD/SKU; ≤2y lookback; 7-30-day ranges recommended). Alternative: Data Kiosk (GraphQL). |
| Brand Analytics (search terms, search query performance, market basket, repeat purchase) | 🔴 | Require **Brand Registry** — optional, per Q1 |
| Hourly glance views | 🔴 | `DETAIL_PAGE_TRAFFIC_EVENT` notification (hourly; up to 24h delayed) |

**Approach:** a daily cron that requests the Sales & Traffic report per channel/marketplace for the last 2-3 days (late data restates), SKU granularity, upserted into `tbl_amazon_sales_traffic_daily`. The Business pulse "Conversion" tile = sum(units) / sum(sessions).

### 4.6 Profit

| Item | Status | Where / How |
|---|---|---|
| Revenue | ✅ | Orders |
| Product cost | 🟡 | Only `tbl_warehouse_stocks.purchase_price` (per stock entry). No canonical COGS (Q3). |
| Amazon fee **estimate** per SKU (referral, FBA pick/pack, …) | 🟡 | `tbl_product_current_offers.raw_data.feesEstimate` (`getMyFeesEstimateForSKU`), on-demand only |
| **Actual** fees, refunds, reimbursements, adjustments | 🔴 | Finances API `2024-06-19 listTransactions` (date-range, "Transaction View" equivalent) or `GET_V2_SETTLEMENT_REPORT_DATA_FLAT_FILE_V2`. Role **Finance and Accounting**. |
| Storage fees | 🔴 | `GET_FBA_STORAGE_FEE_CHARGES_DATA`; estimated fees `GET_FBA_ESTIMATED_FBA_FEES_TXT_DATA` (max 1/day) |
| Fee change detection ("FBA fee +$0.42/unit") | 🔴 | Diff of the estimated-fees report over time; `FEE_PROMOTION` notification for fee promos |
| Ad spend | ⛔ | Amazon Ads API (§4.7) |

**Approach:**
- **Phase A:** estimated contribution margin = price − estimated fees − cost (Q3). Label it "estimated".
- **Phase B:** actuals from the Finances API into `tbl_amazon_financial_event` (daily rollup per SKU).

### 4.7 Advertising
- ⛔ **Amazon Ads API is separate from SP-API.** It needs:
  - its own LwA security profile
  - Ads API access approval (the `advertising::campaign_management` scope; approval can take ~72h)
  - a separate seller OAuth consent
  - a profile per marketplace
- Data comes from async reporting (v3). Amazon Marketing Stream is available for near-real-time data.
- Nothing exists in Categra. **Recommendation:** leave it out of v2; show the tile only after an Ads connection flow exists.

### 4.8 Returns, Feedback, Content, Permissions (secondary)

| Item | Status | Source |
|---|---|---|
| FBA returns + reasons | 🔴 | `GET_FBA_FULFILLMENT_CUSTOMER_RETURNS_DATA` (daily) |
| FBM returns | 🔴 | `GET_FLAT_FILE_RETURNS_DATA_BY_RETURN_DATE` |
| Seller feedback | 🔴 | `GET_SELLER_FEEDBACK_DATA` |
| Refunds (order level) | 🟡 | `tbl_orders.total_refunded_amount` (mainly Shopify) |
| A+ / enhanced content | 🔴 | A+ Content API (brand-registered only) |
| Missing images / content (Categra side) | ✅ | Readiness: `MISSING_MEDIA`, `MISSING_CONTENT` |
| Permissions / roles | 🔴 | No API lists granted roles. Detect per channel by calling each needed operation and recording 403 `Unauthorized` → `tbl_channel_capability` (§6.9). |

### 4.9 Channel & data trust

| Item | Status | Source |
|---|---|---|
| Connection status | ✅ | `tbl_channels.connection_status` (CONNECTED / NOT_CONNECTED / CONNECTION_LOST / DEACTIVATED) + Action Center channel rows (`CHANNEL_NOT_CONNECTED`, …) |
| Marketplace enabled | ✅ | `tbl_amazon_channel_marketplaces.status` |
| Per-marketplace order sync state | ✅ | `order_synced_until`, `order_sync_status` |
| Per-marketplace inventory/catalog freshness | 🔴 | Not tracked. Channel-level `last_*_sync` is written only on import. |
| Notification delivery health | 🟡 | Admin receiving status (`AMAZON_EVENT_ADMIN_RECEIVING_STATUSES`, 30-day lookback) |
| Shopify store: unpublished / inactive | ✅ | Action Center `SHOPIFY_UNPUBLISHED`, `SHOPIFY_INACTIVE` |

---

## 5. Master availability matrix (prototype element → source)

| Prototype element | Status | Primary source (existing or new) |
|---|---|---|
| Scope list (channels / stores) | ✅ | `ChannelService.getRoleBasedChannelsForSetup` + `ActionCenterService.getChannelAccessScope` |
| Account Health tile + priority #1 | 🟡→✅ after bug fix | `ACCOUNT_STATUS_CHANGED` (fix parser) |
| Account metrics (ODR, …) | 🔴 | `GET_V2_SELLER_PERFORMANCE_REPORT` |
| Inventory tile (FBA/FBM counts) | 🟡 | Action Center snapshot (OUT_OF_STOCK/LOW_STOCK by `is_fba`) |
| FBA stranded / inbound delayed / aged | 🔴 | FBA reports / Inbound API |
| FBA unfulfillable | 🟡 | `tbl_amazon_fba_inventories.unfulfillable_quantity` (needs a schedule) |
| FBM late handling / cancel risk | 🟡 | `tbl_orders` + `orderAttention` |
| FBM qty mismatch | 🔴 | `LISTINGS_ITEM_MFN_QUANTITY_CHANGE` / merchant listings report |
| Shared stock conflicts | 🔴 (define per Q4) | New reconciliation job |
| Listings tile (suppression, PTD) | ✅ | Action Center + listing notifications |
| Pricing tile (Featured Offer, price gaps) | 🟡 | Events pipeline (after parser fix) + `tbl_product_current_offers` |
| Target price (FOEP) | 🔴 | `getFeaturedOfferExpectedPriceBatch` |
| Sales, Orders (+Δ vs 7d) | ✅ | Orders |
| Conversion | 🔴 | Sales & Traffic report |
| Gross margin | 🟡/🔴 | Cost (Q3) + fees estimate/actuals |
| "Why" explanations | 🔴 (new logic) | Rule-based attribution over the signals above |
| Data trust / stale marketplace | 🔴 | New freshness tracking |
| What changed? | 🟡 | Event occurrences, lifecycle events, notifications (unify) |
| Returns & feedback | 🔴 | Returns / feedback reports |
| Fees & margin change | 🔴 | Estimated fees report diff |
| A+ content | 🔴 | A+ Content API |
| Advertising | ⛔ | Amazon Ads API |
| Permissions | 🔴 | Capability probe |
| Action Center handoff | ✅ | `CommandCenterActionCenterFilters` |

---

## 6. Proposed approach (backend architecture)

### 6.1 Layering
```
[Ingest]  cron (api-main, ENABLE_CRON) ─► BullMQ jobs (api-worker) ─► SP-API (reports/APIs)
          SQS/EventBridge notifications ─► existing raw → scope → normalize pipeline
[Store]   typed snapshot tables (per channel/marketplace/sku, with captured_at) + "latest" views
[Derive]  signal builders (pure functions, like v1 *.builder.ts) → CommandCenterSignal[]
[Rank]    cross-domain ranker (severity × blast radius × dependency tier × freshness)
[Serve]   POST /command-center/v2/{overview|pulse|priorities|fulfillment|channels|changes|secondary}
          Redis cache 60–90 s (same pattern as v1), meta.freshness per section
```
- **Unified signal shape:**

  ```
  CommandCenterSignal {
    id, domain, family, severity, tier, scope{channelId, marketplaceId, productIds[]},
    blastRadius{accounts, marketplaces, products, revenueAtRisk?}, freshness{sourceAt, stale},
    title, explanation, handoff: CommandCenterActionCenterFilters | {route, params}
  }
  ```
- **Dependency tiers for the "Recommended sequence":**
  - `T0` account health
  - `T1` connection / data trust
  - `T2` sellability (out of stock, suppression, Buy Box excluded)
  - `T3` competitiveness (price gaps)
  - `T4` growth / optimization

  Ranking = tier first, then severity, then blast radius (products × revenue share from orders).
- Reuse the Action Center priority-group settings for the product-level ordering *within* a tier, so user preferences carry over.

### 6.2 Report lifecycle (shared for all report-based sources)
1. A `tbl_amazon_report_request` row records `channel, marketplace(s), report_type, options, report_id, status, requested_at, document_id, processed_at, error`.
2. `createReport` → wait for `REPORT_PROCESSING_FINISHED`. The handler exists; extend `processReportProcessingFinished` with a **router by reportType**. Poll `getReport` as a fallback.
3. `getReportDocument` → download (gzip handling) → a parser per report type → upsert snapshot rows.
4. Throttling: the Reports API has tight per-operation limits (createReport is very low-rate). Stagger per account/region, and reuse `AMAZON_API_DELAYS` / `GlobalEnums.AmazonApis` buckets.

### 6.3 Suggested schedules (tunable)

| Job | Frequency | Output |
|---|---|---|
| Seller performance V2 | daily + on `ACCOUNT_STATUS_CHANGED` | account health snapshot |
| Sales & Traffic (SKU, DAY) | daily (last 3 days) | sales_traffic_daily |
| FBA inventory summaries | every 2–4h | fba_inventories + daily history |
| FBA planning / stranded / restock | daily/weekly | fba_health |
| Offer refresh (getListingOffers / competitive summary) | protected + top SKUs every 1–4h | offer_snapshot |
| FOEP batch | protected + top SKUs every 4–6h | offer_snapshot.foep |
| Velocity / days of cover | daily | derived signal |
| Estimated fees report | daily (1/day limit) | fee snapshot + diff |
| Finances listTransactions | daily | financial_event |
| Shared-stock reconciliation | hourly | conflict signal |

### 6.4 New tables (indicative — migration + entity per project rules)
- `tbl_amazon_report_request`
- `tbl_amazon_account_health_snapshot`
- `tbl_amazon_sales_traffic_daily`
- `tbl_amazon_fba_inventory_history` (+ reuse `tbl_amazon_fba_inventories` as "latest")
- `tbl_amazon_fba_health` (stranded / aged / restock)
- `tbl_amazon_offer_snapshot` (structured `ANY_OFFER_CHANGED` + API refresh, incl. FOEP)
- `tbl_amazon_financial_event` (phase B)
- `tbl_channel_capability` (role/permission probe results)
- `tbl_channel_marketplace_freshness` (last successful pull per source × marketplace)

### 6.5 Fixes to existing code (pre-requisites)
1. Parse `ACCOUNT_STATUS_CHANGED` structurally in both handlers (§4.1).
2. Parse `ANY_OFFER_CHANGED` structurally, identifying our offer by `SellerId` (§4.4). Then re-evaluate the buy-box families (a new normalization version, which the generation model supports).
3. Verify the `PRICING_HEALTH` normalizer against real stored payloads.
4. Extend the `REPORT_PROCESSING_FINISHED` router.
5. Start writing per-marketplace freshness whenever a pull succeeds.

### 6.6 Notifications to add to auto-subscription (candidates)
`FBA_INVENTORY_AVAILABILITY_CHANGES`, `LISTINGS_ITEM_MFN_QUANTITY_CHANGE`, `PRODUCT_TYPE_DEFINITIONS_CHANGE`, optionally `DETAIL_PAGE_TRAFFIC_EVENT` / `FEE_PROMOTION`.
- All of these are high-noise, so aggregate them (latest-state upsert). Don't create per-event work items.

### 6.7 Multi-account / multi-region mapping
- Prototype "Amazon NA · Seller A" = `tbl_channels` row (type AMAZON) + `tbl_amazon_channels` (seller_partner_id, region).
- Prototype "US · CA · MX · BR" = `tbl_amazon_channel_marketplaces`.
- A Shopify store = `tbl_channels` (type SHOPIFY) + `tbl_shopify_channels`. It has no marketplace dimension, matching the prototype's "no marketplace flags".

### 6.8 "What changed?" feed
Union query, newest first, limited per scope, over:
- `tbl_amazon_event_occurrence_revision` (opened / completed)
- `tbl_action_center_lifecycle_events`
- product type change platform notifications (`tbl_notifications`)
- Shopify webhook records
- report-derived diffs (health drop, fee change)

### 6.9 Capability probe (permissions)
- On connect and daily, call one cheap operation per role, e.g. `createReport` for a small report type per role, or `getCompetitiveSummary` with 1 ASIN.
- Store `granted | denied | unknown` per channel and capability.
- Pulse tiles read it to show "Permission required" instead of zeros.

### 6.10 "Why" attribution (rule-based start)
Only emit a "why" when a rule matches with data present. Examples:
- **Sales ↓ and sessions ≈ flat and unit session % ↓** → conversion problem. Then check whether, in the same window, the top SKUs lost the Buy Box, went out of stock or got suppressed → name the cause.
- **Sales ↓ and sessions ↓** → traffic problem (ads/search; ads data is out of scope).
- **Margin ↓** → fee estimate ↑ or price ↓ on top SKUs.

---

## 7. Glossary (concepts in the prototype and findings)

| Term | Meaning |
|---|---|
| **Buy Box / Featured Offer** | The offer shown in "Add to cart" on the product page. Most sales go to it. "Lost" = another seller holds it; "Excluded / not eligible" = our offer can't compete for it (price, account or fulfillment reasons). |
| **FOEP** (Featured Offer Expected Price) | Amazon's computed price at or below which our offer can expect to become the Featured Offer. Used as a "target price". |
| **Competitive Price Threshold / Reference price** | Price benchmark (including prices from other retailers) above which Amazon may make our offer ineligible (PRICING_HEALTH). |
| **PRICING_HEALTH** | Notification that our offer lost Featured Offer eligibility due to uncompetitive pricing. |
| **ANY_OFFER_CHANGED** | Notification when any of the top ~20 offers on an ASIN changes (price, Buy Box winner, count). High volume. |
| **AFN / MFN** | Amazon Fulfillment Network (FBA) / Merchant Fulfillment Network (FBM). |
| **FBA** | Amazon stores, picks and ships. Stock sits in Amazon fulfillment centers. |
| **FBM** | Seller stores and ships. Stock is in the seller's (Categra) warehouses. |
| **Easy Ship** | FBM variant where Amazon picks up the seller's package and delivers it (selected marketplaces). |
| **Fulfillable / Reserved / Inbound / Researching / Unfulfillable** | FBA quantity buckets: sellable; held for orders or transfers; on the way to the fulfillment center; being investigated; damaged/defective. |
| **Stranded inventory** | FBA units in a fulfillment center with no active offer (listing issue), so they can't sell while still incurring storage fees. |
| **Aged / excess** | FBA units stored too long or beyond demand; incur long-term storage fees. |
| **Days of cover** | Available units ÷ average daily units sold. |
| **Suppression** | Listing hidden from search or detail page because of an issue (enforcement action). |
| **BUYABLE / DISCOVERABLE** | Listing status flags: can be purchased / appears in search. |
| **PTD** | Product Type Definition: Amazon's JSON schema of required attributes per product type. A change can require remapping. |
| **Account Health Rating (AHR)** | Amazon's aggregate account score; "At Risk" can precede deactivation. |
| **ODR** | Order Defect Rate (A-to-z claims, chargebacks, negative feedback). Target < 1%. |
| **LSR / Cancellation / VTR / OTDR** | Late shipment rate (< 4%), pre-fulfillment cancel rate (< 2.5%), valid tracking rate (> 95%), on-time delivery rate — FBM metrics. |
| **Sessions / Page views / Unit session % / Buy Box %** | Visits / page loads / units ÷ sessions (conversion) / share of page views where we held the Buy Box. |
| **SP-API roles** | Permission groups granted to the app and the seller (Pricing, Amazon Fulfillment, Selling Partner Insights, Brand Analytics, Finance and Accounting, …). Missing role → 403. |
| **Grantless operation** | SP-API call made with app credentials, not a seller token (e.g. creating notification destinations). |
| **SQS vs EventBridge** | The two notification delivery targets. Account/offer/pricing/order types need SQS; listing types use EventBridge. |
| **Listing protection** | Categra flag `product_channel.is_listing_protected`. Marks listings the user cares most about; boosts their events in projection. |
| **Shadow generation** | A versioned re-computable run of event rules; CC/Action Center read only the ACTIVE generation. |
| **Blast radius** | How much is affected: account → region → marketplace → products → revenue share. |

---

## 8. Suggested delivery phases

| Phase | Content | Depends on |
|---|---|---|
| **P0 — correctness** | Account status parser fix; structured `ANY_OFFER_CHANGED` parser; `PRICING_HEALTH` verification; report router; freshness writes | none |
| **P1 — v2 on existing data** | New v2 endpoints; unified signal model + ranker; pulse tiles for Account status / Inventory (FBA/FBM split from Action Center) / Listings / Pricing (events) / Sales & Orders / Catalog; Channel health; What changed; Action Center handoff for new categories | P0, Q2, Q10 |
| **P2 — scheduled Amazon pulls** | FBA inventory bulk schedule + history; velocity/days of cover; offer refresh + FOEP for protected/top SKUs; seller performance report | Q1 roles |
| **P3 — business intelligence** | Sales & Traffic (conversion); estimated margin; fee change diff; FBA health reports; returns/feedback; capability probe | Q1, Q3 |
| **P4 — finance & ads** | Finances API actuals; Ads API connection | new onboarding |

---

## 9. Key file references
- v1 CC backend: `api/apps/api-main/src/modules/app/system/operations/orchestration/command-center/` (`command-center.service.ts`, `*.builder.ts`, `commandCenter.types.ts`)
- Action Center: `…/system/operations/action-center/action-center.service.ts` (`getSalesOperationsSnapshot` L2924; stock resolution ~L14270-14316; issue sets ~L1560)
- Priority groups: `…/system/operations/configuration/action-engine-settings/actionEngineSettings.types.ts`, `…/action-center/actionCenter.priority-ordering.ts`
- Notifications (legacy): `…/platforms/amazon/amazon-notifications/amazon-notifications.service.ts` (dispatch L457/L530, account status L3685/L3804, report finished L2315, PT change L3552)
- Events pipeline: `…/platforms/amazon/amazon-events/{raw,scope,account,buy-box,price-policy,suppression,projection}/`, `…/amazon-events-action-center-materialization/`, `…/amazon-events-notifications/`
- Events constants and catalog: `api/apps/api-main/src/core/constants/amazon-events/*` (catalog = in-code description of every notification type)
- Auto-subscription: `…/platforms/amazon/notifications/amazon-events-auto-subscription/`, `amazonEventsAutoSubscription.constants.ts`
- FBA inventory: `…/sync/productSyncing/amazon/amazonFbaInventory/amazon-fba-inventory.service.ts`, entity `amazonFbaInventory.entity.ts`
- Offers & fees: `…/catalog/products/listings/offers/product-current-offers.service.ts`, entity `productCurrentOffers.entity.ts`, types `core/types/product.type.ts` (`CurrentOffersData`)
- Orders: `order.entity.ts`, `orderItem.entity.ts`, `…/commerce/orders/order/utils/orderAttention.ts`, `order.service.ts:1489`
- SP-API transport: `…/platforms/amazon/transport/amazon.service.ts`
- Cron: `…/system/cron/cron.service.ts` (no Amazon data jobs today)
- Webapp v1 page: `webapp/src/app/(index)/(menu-layout)/command-center/main.tsx`

## 10. External sources (checked 2026-09-29)
- [Notification type values (payloads for ACCOUNT_STATUS_CHANGED, ANY_OFFER_CHANGED, FBA_INVENTORY_AVAILABILITY_CHANGES, …)](https://developer-docs.amazon/sp-api/docs/notification-type-values)
- [Seller performance reports (V1/V2, Selling Partner Insights role)](https://developer-docs.amazon/sp-api/docs/report-type-values-performance)
- [Analytics reports — GET_SALES_AND_TRAFFIC_REPORT (Brand Analytics role)](https://developer-docs.amazon/sp-api/docs/report-type-values-analytics)
- [FBA report types (inventory, planning, stranded, restock, fees, returns, ledger)](https://developer-docs.amazon/sp-api/docs/report-type-values-fba)
- [Product Pricing v2022-05-01 model (getCompetitiveSummary, getFeaturedOfferExpectedPriceBatch)](https://github.com/amzn/selling-partner-api-models/blob/main/models/product-pricing-api-model/productPricing_2022-05-01.json)
- [FOEP availability update](https://developer-docs.amazon.com/sp-api/changelog/update-product-pricing-api-v2022-05-01-now-supports-the-getfeaturedofferexpectedpricebatch-operation-in-all-marketplaces-except-japan)
- [Pricing FAQ / PRICING_HEALTH](https://developer-docs.amazon/sp-api/docs/pricing-faq)
- [Settlement reports](https://developer-docs.amazon.com/sp-api/docs/report-type-values-settlement) · [Finances "Transaction View" discussion](https://github.com/amzn/selling-partner-api-models/discussions/4015)
- [Easy Ship API v2022-03-23](https://developer-docs.amazon/sp-api/reference/easy-ship-v2022-03-23)
- [Amazon Ads API onboarding](https://advertising.amazon.com/API/docs/en-us/guides/onboarding/overview)

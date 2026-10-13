# Command Center V2 — Top-Row Pulse Cards (middle 4) — Data Plan & Estimate

Date: 2026-10-12
Scope: `api/` + `webapp/` (+ `db-migrations/` only if a new table is needed — none planned here)
Cards in scope (**the middle 4 of the 6 top-row pulse cards**):
**Inventory · Listings · Pricing · Sales & Profit**
Out of scope (already handled elsewhere): Account Health (live from `channel-health`), Advertising (placeholder until the Ads API connection exists), all Business Pulse cards, Ranked priorities, Fulfillment panel, Channel health panel, What changed.

Goal: replace the static mock payload of these 4 cards (`webapp/.../command-center-v2/_lib/mock-data.ts` → `COMMAND_CENTER_V2_PULSE_CARDS`) with live, published-catalog-scoped counts derived from existing data, **without adding new tables** wherever possible.

---

## 1. Decisions locked with the requester

| # | Decision |
|---|---|
| D1 | "Middle 4 cards" = **Inventory, Listings, Pricing, Sales & Profit** (the middle 4 of the top-row pulse cards). |
| D2 | **Inventory at risk** = out-of-stock **+** low-stock **+** stock sync issues (published only). |
| D3 | **Pricing** counts **missing price only** (offer/effective published price empty or ≤ 0). No price-policy / Buy Box / sync-gap pricing troubles in v1 of the card. |
| D4 | **Sales & Profit status** = Growing / Stable / Declining from revenue + orders trend vs the previous comparable period, **downgraded to At Risk** when gross margin is negative or unknown (cost coverage too low). |
| D5 | **Conversion** stays `unavailable`; the Amazon Sales & Traffic integration remains a **later phase** (documented in §9, not built here). |
| D6 | Scope/unit: Inventory & Pricing over **Amazon + Shopify**; Listings **Amazon-only**. Counting unit = one row per **variant** when the product has variants, else one row for the **product**. Parents with variants are never counted. |
| D7 | **Pricing must respect the product Pricing tab inheritance chain** (channel/marketplace node wins only when non-inherited, else defer to master; no marketplace→channel inheritance). See §2.2 and §5. |
| D8 | Amazon listing issues use a **de-coupled, code-based, extensible bucket taxonomy** (new module, versioned) — not the Action Center presentation labels. Errors → **blockers**; if there are none, fall back to a second stage counting **warnings** with a different badge. See §4 and §2.3. |
| D9 | **Every threshold/constant and every classification rule lives in the shared constants/utils layer** (`src/core`), never inline in the builder. See §2.3. |

---

## 2. Base conditions (shared by every card)

### 2.1 Only the published catalog
"Published" = the product/variant is published **inside a channel** (non-Amazon) or **marketplace** (Amazon), `sync_status ∈ {SUCCESS, OUT_OF_SYNC}` (`CONSTANTS.CHANNEL_PUBLISHED_STATUSES`).

Already encoded by the reusable CTE `buildChannelListingKeysCteSql()` in
`api/apps/api-main/src/modules/app/catalog/products/listings/channel-listing-keys.sql.ts`.
It yields `listing_keys` = **one row per sellable channel listing key**:

- Amazon → `(product, channel, marketplace)` and `(variant, channel, marketplace)`
- non-Amazon (Shopify) → `(product, channel)` and `(variant, channel)`
- `sellableOnly: true` drops product keys of products that have at least one non-deleted variant ⇒ **the exact unit rule of D6** (variant holds stock/price; parent suppressed).
- live-row filters already applied: channel not deleted + connection status ∈ {CONNECTED, CONNECTION_LOST, DEACTIVATED}, product/variant not deleted, `product_channels` / `product_variant_channels` active, marketplace enabled, etc.
- exposes `product_id, variant_id, channel_id, channel_type, language_code, sync_status`.

**Published filter** = `listing_keys.sync_status IN ('SUCCESS','OUT_OF_SYNC')`. (For Amazon variant keys this is `tbl_variant_marketplaces.sync_status`; for Amazon product keys `tbl_product_channel_marketplaces.sync_status`; for Shopify `tbl_product_channels` / `tbl_product_variant_channels.sync_status`.)

**Scope** = the requesting user's allowed channels (`ChannelService.getUserWiseChannels` → `resolveListingSummaryChannelIds`) plus an optional `channelId` and (Amazon only) `marketplaceId`. Reuse `channelListingKeysReplacements(accountId, channelIds, marketplaceId)`.

### 2.2 Published values, not local values
- **Price**: read the **published** price using the **same inheritance chain as the product Pricing tab** (D7):
  1. **Row lookup is exact-scope only** — Amazon: `channel_id = :channelId AND amazon_marketplace_id = :marketplaceId AND currency = marketplace currency`; Shopify: `channel_id = :channelId AND amazon_marketplace_id IS NULL AND currency = shop default currency`. Master fallback is `channel_id IS NULL AND amazon_marketplace_id IS NULL` (same currency). **There is no marketplace→channel inheritance** (a channel row never defers to a channel-level row, only to master).
  2. **Per price type** (`listPrice`, `offerPrice`, `salePrice`, `businessPrice`): if the scope row's node exists **and** `pricing_details[type].isInherited === false` → use the scope node; otherwise (missing or inherited) → use the **master** node. `offerPrice` additionally falls back to the legacy `tbl_product_variant_prices.price` column.
  3. **Baseline wins for the published value** — if a `tbl_price_sync_baselines` row exists for `(product, variant, channel, marketplace, price_type)`, the published node is `previous_price_data` (its `price`, plus `isInherited`/`startAt`/`endAt`). Apply the same rule: a baseline node with `isInherited = true` resolves through the current **master** node; a non-inherited baseline node is taken as-is. When no baseline row exists, nothing was pushed since the last publish, so the step-2 chain value **is** the published value.
- **Stock**: **no baseline system** — read directly from the primary tables (FBA table / warehouse stock / `amazon_stock` / `external_stock`), exactly like the product **Stock tab** and the Action Center stock CTEs.
- Other attributes/media/misc: out of scope for these 4 cards.

### 2.3 Constants, thresholds & classification modules (single source of truth)
Per D9, no magic numbers or label maps inline in builders. All shared values live in `src/core`:

- **`api/apps/api-main/src/core/constants/constants.ts`** — add a `COMMAND_CENTER_V2` block next to the existing `CHANNEL_HEALTH_THRESHOLDS`, e.g.:
  - `COMMAND_CENTER_V2.inventory.atRiskShareRedPercent` (badge red threshold, default 25)
  - `COMMAND_CENTER_V2.salesProfit.flatRevenuePercent` (default 3), `growingThresholdPercent` (default 3), `atRiskCostCoverageFloorPercent` (default 30)
  - `COMMAND_CENTER_V2.listings.maxIssuesScanned` (row cap), `detailTopBuckets` (default 3)
- **`api/apps/api-main/src/core/constants/command-center-v2/amazon-listing-issue-buckets.ts`** (new, versioned) — the de-coupled taxonomy (§4): bucket **codes**, labels, ordering, and the ordered keyword/error-code → bucket rule list. Adding a bucket later = adding one entry + rules + a spec case; nothing else changes.
- **`api/apps/api-main/src/modules/app/system/operations/orchestration/command-center/utils/commandCenterV2Pulse.ts`** (new) — pure functions (`classifyAmazonListingIssue`, `resolveSalesProfitStatus`, `computeInventoryAtRiskState`, tone mappers). Mirrors the existing `order/utils/businessPulse.ts` pattern.
- Specs colocated (`*.spec.ts`) enumerate every classification rule and threshold boundary.

### 2.4 Fulfillment / stock resolution (source of truth)
Mirror `readiness/engine/readiness-stock.resolver.ts` and the Action Center CTEs
(`action-center.service.ts` `buildAmazonSalesQuery` ~L14341-14526, `buildShopifySalesQuery` ~L13864-13980):

| Case | Published stock |
|---|---|
| Amazon **FBA** (`is_fba` true **or** a mapped warehouse with `tbl_warehouse.amazon_fulfillment_center = true`) | `tbl_amazon_fba_inventories.fulfillable_quantity` |
| Amazon **FBM** with mapped non-FC warehouses (`tbl_channel_product_warehouses`) | `SUM(tbl_warehouse_products.total_quantity)` |
| Amazon FBM with **no** mapped warehouse | `variable_marketplace.amazon_stock` / `product_channel_marketplace.amazon_stock` |
| Shopify with mapped warehouses | `SUM(tbl_warehouse_products.total_quantity)` |
| Shopify with no mapped warehouse | `product_variant_channels.external_stock` / `product_channels.external_stock` |

Low-stock threshold: `tbl_product_variant_channels.low_stock_threshold` / `tbl_product_channels.low_stock_threshold`. Effective threshold is **0** for FBA and for no-warehouse listings (so they never count as "low", same as Action Center).

**Shared stock** = a Categra warehouse (`tbl_warehouse`) that is mapped via `tbl_channel_product_warehouses` to **2 or more channels** for the same product (a pool serving Amazon FBM and/or Shopify). The unit's stock bucket is `SHARED` when its resolved stock comes from such a multi-channel pool; otherwise `FBA` / `FBM`.

---

## 3. Card 1 — Inventory

Prototype: badge `74 at risk`, state `FBA · FBM · Shared`, detail `22 FBA · 31 FBM · 21 shared`.

**Data**
- One published `listing_keys` row per unit (D6), Amazon + Shopify.
- Resolve published stock per §2.4; classify bucket ∈ `{FBA, FBM, SHARED}`.
- A unit is **at risk** when **any** holds (counted once per unit):
  1. `resolved_stock <= 0` → **out of stock**
  2. `resolved_stock > 0 AND effective_threshold > 0 AND resolved_stock <= effective_threshold` → **low stock**
  3. **stock sync issue** → one of:
     - `SYNC_GAP`: warehouse-managed FBM where the remote value differs from the Categra value (`amazon_stock != categraStock`, `external_stock != categraStock`)
     - an open `tbl_product_sync_failures` row with `module = STOCK` for that unit/channel/marketplace (already used by the channel-health data-trust card)
     - (optional, if the requester wants it) the channel/marketplace is in `ChannelHealthService.getUntrustedScopes('INVENTORY')` — mark "untrusted" instead of counting, consistent with the health plan §6.6
- Counts returned: `fbaAtRisk`, `fbmAtRisk`, `sharedAtRisk`, `atRiskTotal = sum`, plus breakdown counters `outOfStock`, `lowStock`, `syncIssues` for the tooltip.

**SQL sketch** (extends the existing CTE; lateral joins resolve stock per key)
```sql
WITH ${buildChannelListingKeysCteSql({ restrictChannels, restrictMarketplace, sellableOnly: true })},
published AS (
  SELECT lk.* FROM listing_keys lk WHERE lk.sync_status IN ('SUCCESS','OUT_OF_SYNC')
),
-- per-key warehouse mapping (non-FC), FBA detection, shared-pool detection, remote stock
resolved AS (
  SELECT p.*,
    -- ... lateral: is_fba, fba_fulfillable, warehouse_stock, mapped_non_fc_count,
    --     has_fc_mapping, amazon_stock, external_stock, threshold, distinct_channel_count_of_pool
    CASE WHEN is_fba THEN CASE WHEN resolved_stock<=0 THEN 'OOS' ... END END AS state
  FROM published p
)
SELECT
  COUNT(*) FILTER (WHERE state IN ('OOS','LOW','SYNC') AND bucket='FBA')   AS fba_at_risk,
  COUNT(*) FILTER (WHERE state IN ('OOS','LOW','SYNC') AND bucket='FBM')   AS fbm_at_risk,
  COUNT(*) FILTER (WHERE state IN ('OOS','LOW','SYNC') AND bucket='SHARED') AS shared_at_risk,
  COUNT(*) FILTER (WHERE reason='OOS')  AS out_of_stock,
  COUNT(*) FILTER (WHERE reason='LOW')  AS low_stock,
  COUNT(*) FILTER (WHERE reason='SYNC') AS sync_issues
FROM resolved;
```

**Card output**
- badge: `{atRiskTotal} at risk` (tone amber; red if a configurable share of published units is at risk, e.g. ≥ 25%)
- state: `FBA · FBM · Shared`
- detail: `{fba} FBA · {fbm} FBM · {shared} shared`

**Caveat**: `is_fba` and `amazon_stock` live on `tbl_variant_marketplaces` (variants) and `tbl_product_channel_marketplaces` (standalone). Use `fulfillable_quantity` (not `total_quantity`) to match the Stock tab / Action Center. FBA stock is pulled only on demand today (no schedule) — the card reflects the last pulled value; that is acceptable for v1 and is flagged by the data-trust signal.

---

## 4. Card 2 — Listings (Amazon only)

Prototype: badge `11 blockers`, state `Needs attention`, detail `Suppression · PTD · structure`.

**Data**
- Source: **`tbl_channel_product_errors` where `type = 'AMAZON'`** (`GlobalEnums.syncErrorCodes.AMAZON`). Per the brief, every AMAZON row is a listing issue. Rows carry `severity` (`error` | `warning`), `error_code` (Amazon issue code), `error_keyword` (`issue.categories`), `error`, `action_type`.
- Join to the **published Amazon** `listing_keys` (product×marketplace and variant×marketplace) via `(product_id, variant_id, channel_id, amazon_marketplace_id)` so only published listings count and the D6 unit rule holds.
- **Shopify excluded** (its listing health is a different model; the code does write `type='SHOPIFY'` rows on the forward-sync path, but we deliberately filter them out).

**Two-stage counting (D8, errors first, warnings only as fallback)**
1. **Stage 1 — blockers:** distinct published listing units (per D6) with **at least one AMAZON `severity <> 'warning'`** row (i.e. `error`, plus the defensive `COALESCE(severity,'error')` case). If `blockers > 0` → badge `{n} blockers`.
2. **Stage 2 — warnings:** only when `blockers = 0`, count distinct units with **AMAZON `severity = 'warning'`** rows → badge `{n} warnings` (different label/tone).
3. Both 0 → badge `Healthy` / tone green.
Counts are **distinct listing units**, not raw error rows (a listing commonly produces many).

**De-coupled, code-based bucket taxonomy (D8, extensible)**
Create a new module `core/constants/command-center-v2/amazon-listing-issue-buckets.ts`, **independent of the Action Center presentation layer** so buckets can be added/renamed without touching Action Center and vice-versa. Shape:

```ts
export const AMAZON_LISTING_ISSUE_BUCKET = {
  SUPPRESSION_VISIBILITY: 'SUPPRESSION_VISIBILITY',
  NOT_BUYABLE:           'NOT_BUYABLE',
  STRUCTURE_PRODUCT_TYPE:'STRUCTURE_PRODUCT_TYPE',
  COMPLIANCE:            'COMPLIANCE',
  ATTRIBUTES_CONTENT:    'ATTRIBUTES_CONTENT',
  MEDIA:                 'MEDIA',
  OTHER_VALIDATION:      'OTHER_VALIDATION',
} as const;

export const AMAZON_LISTING_ISSUE_BUCKETS = [
  { code: 'SUPPRESSION_VISIBILITY', labelKey: 'Cc_V2_Listing_Bucket_Suppression', order: 1 },
  { code: 'NOT_BUYABLE',            labelKey: 'Cc_V2_Listing_Bucket_NotBuyable',    order: 2 },
  { code: 'STRUCTURE_PRODUCT_TYPE', labelKey: 'Cc_V2_Listing_Bucket_Structure',     order: 3 },
  { code: 'COMPLIANCE',             labelKey: 'Cc_V2_Listing_Bucket_Compliance',    order: 4 },
  { code: 'ATTRIBUTES_CONTENT',     labelKey: 'Cc_V2_Listing_Bucket_Attributes',    order: 5 },
  { code: 'MEDIA',                  labelKey: 'Cc_V2_Listing_Bucket_Media',         order: 6 },
  { code: 'OTHER_VALIDATION',       labelKey: 'Cc_V2_Listing_Bucket_Other',         order: 7 },
] as const;

// ordered; first match wins. Each rule inspects structured fields only.
export const AMAZON_LISTING_ISSUE_BUCKET_RULES = [
  { bucket: 'COMPLIANCE',              match: { keywords: ['compliance','restricted','trademark','hazmat','brand approval'] } },
  { bucket: 'SUPPRESSION_VISIBILITY',  match: { keywords: ['suppress','not discoverable','searchable','inactive','removed'] } },
  { bucket: 'NOT_BUYABLE',             match: { keywords: ['not buyable','not sellable','out of stock','buy box'] } },
  { bucket: 'STRUCTURE_PRODUCT_TYPE',  match: { codes: ['...'], keywords: ['variation','product type','category','structure','template','condition'] } },
  { bucket: 'ATTRIBUTES_CONTENT',      match: { keywords: ['attribute','required','missing','invalid','content','description','title','bullet'] } },
  { bucket: 'MEDIA',                   match: { keywords: ['image','media','photo','main image'] } },
  { bucket: 'OTHER_VALIDATION',        match: { always: true } },
] as const;
```

- The classifier `classifyAmazonListingIssue({ severity, errorCode, errorKeyword, errorText })` (in `commandCenterV2Pulse.ts`) returns a bucket **code** from this table. It uses `error_code` + `error_keyword` first, then `error` text; the `OTHER_VALIDATION` rule is the guaranteed fallback (`always: true`) so **every issue gets a code** (no "unknown").
- Buckets are keyed by stable **codes** (not display strings); labels are i18n keys. Future bucketing = one new entry + rules + spec case, no builder change.
- A bucketed count map is returned for the tooltip; the card shows the top `detailTopBuckets` (constant, default 3) labels in `detail`.

**Severity caveat** (evidence-backed): current writes always persist AMAZON rows as `severity='error'` (`processAmazonIssues` filters to error; every FS inserter hard-codes `.error`). The physical column is a plain `VARCHAR DEFAULT 'error'`, so legacy rows may hold `warning`, and the product list derives a display-level `warning` from a `LOW_STOCK` text heuristic. The two-stage logic therefore reads the stored `severity` **and** reuses the same display heuristic for the warning stage, so both real warning rows and the derived ones are counted consistently. (Optionally extend the classifier with a `readTimeWarning` helper that mirrors `product.service.ts:getSyncErrorDisplaySeveritySql`.)

**Card output**
- badge: Stage 1 → `{n} blockers`; else Stage 2 → `{n} warnings`; else `Healthy`.
- tone: red (blockers on a `is_listing_protected` listing) / amber (blockers) / blue (warnings) / green (none).
- state: top bucket label, or `Needs attention` / `Healthy`.
- detail: top `detailTopBuckets` bucket labels joined by ` · `.

---

## 5. Card 3 — Pricing (missing price only)

Prototype: badge `5 alerts`, state `Review`, detail `2 Featured Offer · 3 price gaps` → **replace** with `{n} missing prices` since only missing price is counted (D3).

**Data**
- Scope: published `listing_keys` (Amazon + Shopify), D6 unit rule.
- Resolve the **published effective price** per unit/channel/marketplace following the **Pricing tab inheritance chain** (D7 / §2.2), summarized:
  1. **Baseline first**: if `tbl_price_sync_baselines` has a row for `(product_id, variant_id, channel_id, amazon_marketplace_id, price_type)`, the published node is `previous_price_data`. If its `isInherited = true`, resolve it through the **current master** node; otherwise take its `price`.
  2. **Else use the current price rows**: exact scope row (Amazon `channel_id + amazon_marketplace_id`; Shopify `channel_id + marketplace NULL`, marketplace currency / shop default currency). Its per-type node is used only when `pricing_details[type].isInherited === false`. Otherwise (missing or inherited) fall back to the **master** row (`channel_id IS NULL AND amazon_marketplace_id IS NULL`). `offerPrice` additionally falls back to the legacy `tbl_product_variant_prices.price` column.
  3. **No marketplace→channel inheritance** — a channel row never defers to a channel-level row, only to master.
- **Effective price** = **active `salePrice.price`** (when `startAt <= now <= endAt`) **else `offerPrice.price`**. This matches the publish payload (`resolvePrices`/`buildAmazonPayload`: active sale wins over offer) and the Action Center `price_resolution` CTEs.
- **Missing** = effective published price is `NULL` or `<= 0`.
- Count = number of published units with a missing price. Split Amazon vs Shopify for the tooltip.

`price_type` values: `GlobalEnums.PRICE_TYPES` = `{listPrice, offerPrice, salePrice, businessPrice}` (`global-enums.ts:2538`).

**Correctness trap to avoid:** do **not** collapse to "channel+marketplace → master-marketplace → master". The pricing tab has **no master-marketplace tier**; the only fallback is master (`channel_id IS NULL AND amazon_marketplace_id IS NULL`). (The Action Center Amazon CTE does rank a master-marketplace tier — that is a divergence we deliberately do not copy here.)

**SQL sketch (exact-scope inheritance + baseline; no master-marketplace tier)**
```sql
WITH ${buildChannelListingKeysCteSql({ restrictChannels, restrictMarketplace, sellableOnly: true })},
published AS (SELECT * FROM listing_keys WHERE sync_status IN ('SUCCESS','OUT_OF_SYNC')),
nodes AS (
  SELECT p.*,
    scope.pricing_details  AS scope_pd,   scope.price  AS scope_legacy_price,
    master.pricing_details AS master_pd,  master.price AS master_legacy_price,
    -- per-type baseline fragments at exact scope (offerPrice + salePrice)
    (SELECT bl.previous_price_data FROM tbl_price_sync_baselines bl
      WHERE bl.product_id = p.product_id AND bl.variant_id IS NOT DISTINCT FROM p.variant_id
        AND bl.channel_id = p.channel_id
        AND bl.amazon_marketplace_id IS NOT DISTINCT FROM p.amazon_marketplace_id
        AND bl.price_type = 'offerPrice') AS bl_offer,
    (SELECT bl.previous_price_data FROM tbl_price_sync_baselines bl
      WHERE ... AND bl.price_type = 'salePrice') AS bl_sale
  FROM published p
  LEFT JOIN tbl_product_variant_prices scope
    ON scope.product_id = p.product_id AND scope.variant_id IS NOT DISTINCT FROM p.variant_id
   AND scope.channel_id = p.channel_id
   AND scope.amazon_marketplace_id IS NOT DISTINCT FROM p.amazon_marketplace_id   -- Shopify: both NULL
  LEFT JOIN tbl_product_variant_prices master
    ON master.product_id = p.product_id AND master.variant_id IS NOT DISTINCT FROM p.variant_id
   AND master.channel_id IS NULL AND master.amazon_marketplace_id IS NULL
),
-- resolve_node(node, master, baseline, legacyMasterPrice):
--   if baseline exists   -> baseline.isInherited ? resolve(master node) : baseline.price
--   else if node exists and node.isInherited = false -> node.price
--   else -> master node price (offerPrice also falls back to legacy master price column)
resolved AS (
  SELECT *,
    resolve_node(scope_pd->'salePrice', master_pd->'salePrice', bl_sale, NULL)  AS sale_price,
    resolve_node(scope_pd->'offerPrice', master_pd->'offerPrice', bl_offer, master_legacy_price) AS offer_price
  FROM nodes
)
SELECT
  COUNT(*) FILTER (
    WHERE COALESCE(NULLIF(sale_price,0), NULLIF(offer_price,0)) IS NULL     -- active sale>0 else offer>0
  ) AS missing_price_total
FROM resolved;
```

**Caveat**: `tbl_price_sync_baselines.previous_price_data` for `salePrice` stores a `starAt` key (typo) instead of `startAt`; the missing-price check only needs `price`, so it is unaffected.

**Card output**
- badge: `{n} missing prices` (tone blue when n>0, green when 0)
- state: `Review` / `Healthy`
- detail: `Offer price not published` (+ `{a} Amazon · {s} Shopify` if scoped to all)

---

## 6. Card 4 — Sales & Profit

Prototype: badge `Stable`, state `No major decline`, detail `Current: orders + revenue`.

**Data** — already fully available from the live business-pulse pipeline; this card is a compact summary of it:
- Sales, orders: `OrderService.getBusinessPulse` / `getCommandCenterTrendData`
  (`order.service.ts:1753` and `:1513`), revenue = `SUM(quantity_shipped × item_price_amount)` over `tbl_order_items` + `tbl_orders`, base-currency converted, current vs previous comparable window (`sellerTrendMetrics.ts` / `businessPulse.ts`).
- Margin: `businessPulse.ts` `computeGrossMarginPercent` (revenue − COGS(WAC from `tbl_stocks.purchase_price`) − discounts − refunds − estimated Amazon fees), plus `costCoveragePercent` / `feeEstimateCoveragePercent`.

**Status derivation (D4)** — add a small builder (constant thresholds):
| Status | Rule |
|---|---|
| **At Risk** | gross margin `< 0` **or** margin unknown (`costCoveragePercent` below a floor, e.g. `< 30%`, or `status = 'unavailable'`) |
| **Growing** | revenue Δ ≥ `+3%` (and not At Risk) |
| **Declining** | revenue Δ ≤ `-3%` (and not At Risk) |
| **Stable** | otherwise (`|Δ| < 3%`) |

Order trend is used as the `detail` line and as a tie-breaker (if revenue and orders disagree strongly, show the weaker one as the headline driver).

**Card output**
- badge: `Growing` / `Stable` / `Declining` / `At Risk` (tone green / blue / amber / red)
- state: short sentence (`No major decline` / `Revenue is falling` / `Margin below zero`)
- detail: `orders {↑/→/↓} {x}% · revenue {↑/→/↓} {y}%`

No new tables. Reuses the existing `order/business-pulse` (or the trend method if a lighter payload is preferred).

---

## 7. API design

Add one v2 endpoint (keep it out of the v1 `command-center/*` surface per the v1→v2 migration plan):

```
POST /command-center/v2/pulse
body: { channelId?, marketplaceId?, range? }   // range default LAST_30_DAYS, used by Sales & Profit
→ { meta: { generatedAt, scope, currency },
    cards: {
      inventory: { tone, badge, state, detail, counts: { fbaAtRisk, fbmAtRisk, sharedAtRisk, atRiskTotal, outOfStock, lowStock, syncIssues } },
      listings:  { tone, badge, state, detail, counts: { stage: 'blockers'|'warnings'|'healthy', blockers, warnings, byBucket: { [bucketCode]: count } } },
      pricing:   { tone, badge, state, detail, counts: { missingPriceTotal, amazon, shopify } },
      salesProfit:{ tone, badge, state, detail, counts: { status, revenueDeltaPercent, ordersDeltaPercent, marginPercent } }
    } }
```

- Controller: new `CommandCenterV2Controller` + `CommandCenterV2PulseService` inside
  `api/.../system/operations/orchestration/command-center/`, guarded by `JwtAuthGuard` + `@RequiresSubscriptionCapabilities('ACTION_CENTER_VIEW')` (same as v1).
- Reuse `buildChannelListingKeysCteSql` + `channelListingKeysReplacements`; add a companion SQL module for the stock/price resolution (so Listings/Pricing/Inventory share one `listing_keys` backbone).
- Redis cache 60–90 s keyed by account, user, channel scope, channel, marketplace, range, currency (same pattern as v1 `CommandCenterService.buildCacheKey`).
- Access scope via `ChannelService.getUserWiseChannels` (view-only, no master requirement for these cards).
- The Sales & Profit builder calls the existing order service; no duplication.

`webapp/src/lib/api.ts`: add `GET_COMMAND_CENTER_V2_PULSE: 'command-center/v2/pulse'`.

---

## 8. Webapp changes

- `_components/intelligence-pulse-section.tsx`: source `inventory`, `listings`, `pricing`, `sales-profit` from the new endpoint instead of `COMMAND_CENTER_V2_PULSE_CARDS`; keep Account Health live and Advertising as the placeholder. Show the existing loading/unavailable fallback per card (as Account Health does today).
- `_lib/types.ts`: extend `CommandCenterV2PulseCard` usage with the `counts` payload (tooltips) — keep the visual contract (`badge/state/detail/tone`) unchanged.
- Map tones: red/amber/blue/green/gray already exist (`CommandCenterV2Tone`).
- Tooltips: reuse the card `detail` + counts; no new page.
- Action Center handoff per card (optional v1 parity): Inventory → `INVENTORY`, Listings → `LISTINGS`, Pricing → pricing capability, Sales & Profit → sales scope. V2 currently navigates bare to `/actionCenter`; wiring the `CommandCenterActionCenterFilters` contract is a small follow-up (not required for the card data).

---

## 9. Data-quality caveats & deferred work

1. **Conversion** (not in these 4 cards, but adjacent): still `unavailable`; requires `GET_SALES_AND_TRAFFIC_REPORT` (Brand Analytics role), a report router entry, and a new daily table. **Deferred** (D5).
2. **FBA stock freshness**: `tbl_amazon_fba_inventories` is only refreshed on demand/backward sync — the Inventory card shows the last pulled value; the channel-health data-trust signal covers staleness. A scheduled bulk FBA pull is a separate phase.
3. **`tbl_product_sync_failures` index**: the stock-sync-issue part of Inventory wants `(channel_id, amazon_marketplace_id, module, created_at)` (already proposed in the channel-health plan §5.5).
4. **COGS coverage**: margin is `unavailable`/partial for FBA-only sellers (no `tbl_stocks.purchase_price`) → Sales & Profit will read **At Risk** for those accounts until cost is entered or a canonical cost field is added. This is the intended D4 behavior but call out in the card `why`.
5. **Shopify errors**: `tbl_channel_product_errors` does contain `type='SHOPIFY'` rows in some forward-sync paths — the Listings card filters to `type='AMAZON'` only, as agreed.
6. **Amazon `severity`**: current writers persist AMAZON rows as `severity='error'` only (`processAmazonIssues` filters to error; all FS inserters hard-code `.error`), but the column is a plain `VARCHAR DEFAULT 'error'` so legacy `warning` rows can exist, and the product list derives a display `warning` from a `LOW_STOCK` text heuristic. The two-stage Listings logic reads the stored severity **and** applies the same display heuristic so both sources funnel into the "warnings" stage.
7. **Amazon refunds**: `tbl_orders.total_refunded_amount` is Shopify-only; Amazon margin is estimate-based (fees estimate, no Finances API).

---

## 10. Effort estimate

Estimates are **ideal dev-days** for one backend + one frontend engineer, including unit tests and a manual verification pass against a read-only DB (GET permission granted). Shared plumbing is amortised across the 4 cards.

| # | Work item | Days |
|---|---|---|
| 1 | v2 endpoint + `CommandCenterV2PulseService` skeleton, DTO, access scope, Redis cache, `lib/api.ts` builder | 1.5 |
| 2 | Shared SQL backbone: `published listing_keys` + stock resolution lateral (FBA/FBM/warehouse/Shopify) + shared-pool detection | 2.0 |
| 3 | **Inventory** builder: buckets, out/low/sync-issue classification, dedupe, counts | 2.0 |
| 4 | **Listings** builder: two-stage blockers/warnings + de-coupled bucket taxonomy module + classifier + spec | 2.5 |
| 5 | **Pricing** builder: Pricing-tab inheritance resolver (exact-scope node → master, baseline interaction) + missing-price count | 2.5 |
| 6 | **Sales & Profit** builder: status rules over existing order/business-pulse data | 1.0 |
| 7 | Webapp: wire 4 cards live, tones, loading/unavailable, tooltips | 1.5 |
| 8 | Tests (spec) + verification queries (published-only, variant/parent rule, inheritance, baseline fallback, blocker/warning stages, FBA/FBM/shared) | 2.5 |
| 9 | Buffer / integration polish + review | 1.5 |
| | **Total** | **~17 dev-days (~3.5 weeks calendar)** |

**Risk drivers**: the warehouse/shared-pool resolution and the Pricing-tab inheritance resolver (baseline × inherited-node interaction) are the two areas most likely to overrun. The de-coupled bucket taxonomy is bounded (new module + specs) and removes coupling to Action Center. No schema migration is required, which removes the largest schedule risk.

---

## 11. Suggested phasing

| Phase | Content |
|---|---|
| **P1** | Backend endpoint + Inventory + Pricing + Sales & Profit; wire those 3 cards (Listings still mock). |
| **P2** | Listings card (AMAZON error aggregation + classification) and wire it. |
| **P3** | Verification hardening: FBA schedule, `tbl_product_sync_failures` index, data-trust suppression on Inventory. |
| **Later** | Conversion (Sales & Traffic report), pricing troubles beyond missing price, Action Center handoff filters from v2. |

---

## 12. Verification checklist

- [ ] Only `sync_status ∈ {SUCCESS, OUT_OF_SYNC}` units are counted; `NOT_PUBLISHED`/`ERROR` excluded.
- [ ] A product with variants contributes **only** its variants; a product without variants contributes itself.
- [ ] Amazon FBA uses `fulfillable_quantity`; no-warehouse listings never count as "low".
- [ ] A `tbl_price_sync_baselines` row with `previous_price_data.price` empty ⇒ counted as missing price even when the local price is set (matches the brief's V1/P1/C1/M1 example).
- [ ] With no baseline row, the resolved value follows the Pricing tab: exact-scope node when `isInherited=false`, else **master**; a channel row is never resolved from a channel-level row; `offerPrice` falls back to the legacy `price` column. **No master-marketplace tier.**
- [ ] A baseline node with `isInherited=true` resolves through the **current master**; with `isInherited=false` its stored `price` is used.
- [ ] Active `salePrice` (start/end window) wins over `offerPrice`; an inactive sale does not.
- [ ] Listings counts distinct **listing units**, not raw error rows; only `type='AMAZON'`.
- [ ] Listings Stage 1 counts only non-warning rows; when Stage 1 = 0, Stage 2 shows the warning count with the **warnings** label; both 0 → `Healthy`.
- [ ] Every AMAZON issue maps to exactly one bucket code (no "unknown"); adding a rule/bucket needs only the taxonomy module + spec change.
- [ ] Sales & Profit reads At Risk when margin `< 0` or cost coverage is too low; otherwise Growing/Stable/Declining from revenue Δ (±3%).
- [ ] Scope by channel and (Amazon) marketplace works, and honours per-user channel access.
- [ ] No threshold/constant/classification rule is inline in a builder — all come from the `src/core` constants/utils layer.
- [ ] All four cards render with the existing pulse-card skeleton while loading.

---

## 13. Key file references

- Pulse card mock/types (webapp): `webapp/src/app/(index)/(menu-layout)/command-center-v2/_lib/mock-data.ts`, `_lib/types.ts`, `_components/intelligence-pulse-section.tsx`, `_components/business-pulse-section.tsx`.
- Published-catalog CTE: `api/.../catalog/products/listings/channel-listing-keys.sql.ts`.
- Existing v2 summaries (reuse patterns): `.../listings/product-channel-sync-summary.service.ts`, `.../readiness/views/readiness-summary.service.ts`.
- Stock resolution: `.../readiness/engine/readiness-stock.resolver.ts`; Action Center CTEs `action-center.service.ts` (~L14235-14526 Amazon, ~L13755-13980 Shopify); Stock tab `.../stock/product-stock.service.ts` (`getVariantStockByAmazonChannel` L3374, `getAmazonChannelData` L2190, `getShopifyStockData` L2548).
- Pricing inheritance (authoritative): `.../catalog/products/pricing/product-pricing.service.ts` (`getPriceData` L4894, `normalizePrice` L5332, `getPriceDataByAmazon` L5363, `getPriceDataByShopify` L5959, `resolvePrices` L4594, `buildAmazonPayload` L4612); webapp Pricing tab `webapp/.../products/[product]/_components/productPricing/`.
- Baselines: `.../sync/baselines/price-sync-baseline/price-sync-baseline.service.ts`; `sync-baseline.service.ts` (`queryPricing` L1828, `reconcileSyncStatus` L300); entities `priceSyncBaseline.entity.ts` + the other 5 `*_sync_baselines`.
- Listing errors: `entities/channelPorductError.entity.ts`; writer `.../amazon/backwardSync/amazon-product-bs.service.ts` `processAmazonIssues` (L5598, error-only filter L5650) + `getSyncErrorDisplaySeveritySql` (`product.service.ts` L12961); **new** taxonomy `core/constants/command-center-v2/amazon-listing-issue-buckets.ts` + classifier `.../command-center/utils/commandCenterV2Pulse.ts`.
- Constants: `core/constants/constants.ts` (new `COMMAND_CENTER_V2` block; pattern: `CHANNEL_HEALTH_THRESHOLDS`).
- Orders / margin: `.../commerce/orders/order/order.service.ts` (`getBusinessPulse` L1753, `getCommandCenterTrendData` L1513); `order/utils/businessPulse.ts`, `order/utils/sellerTrendMetrics.ts`; entity `warehouseStocks.entity.ts` (`tbl_stocks.purchase_price`); `productCurrentOffers.entity.ts` (fees estimate).
- v1 Command Center (reference for cache/scope/controller): `.../orchestration/command-center/` (`command-center.service.ts`, `commandCenter.types.ts`, builders).
- Action Center handoff contract: `.../orchestration/command-center/commandCenter.types.ts` `CommandCenterActionCenterFilters`.

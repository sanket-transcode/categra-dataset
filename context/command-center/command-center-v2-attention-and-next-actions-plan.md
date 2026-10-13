# Command Center V2 — "What needs attention now" & "Next best actions" — Plan & Estimate

Date: 2026-10-12
Scope: `api/` + `webapp/` (+ `db-migrations/` only if a new column is required — none planned)
Sections in scope (the two remaining live sections of Command Center V2):
**What needs attention now** (max 5 cards) · **Next best actions** (max 4 actions)

Goal: replace the static mock payloads (`webapp/.../command-center-v2/_lib/mock-data.ts` →
`COMMAND_CENTER_V2_ATTENTION_ITEMS`, `COMMAND_CENTER_V2_NEXT_ACTIONS`) and the leftover
client-only health composition with **API-sourced, de-coupled, reusable** attention items and
actions, without adding new tables.

Companion docs (decisions/context that this plan builds on):
`command-center-v2-pulse-cards-plan.md` (pulse cards D1–D9), `channel-marketplace-health-plan.md`,
`command-center-v2-analysis.md`, `command-center-v2-ui-prompt.md`, `initial-findings.md`.

---

## 1. Objective & constraints

- **Objective:** one ordered, severity-driven list of what to fix now, + a short dependency-ordered
  action sequence. Reuse the investigation the rest of Command Center V2 already performs.
- **Gathering is API-side; limiting is frontend-side.** The API returns **all** attention items and
  actions; the webapp slices to 5 / 4 (constants in the webapp), so the display cap can change
  without an API change.
- **De-coupled, unambiguous, reusable.** Severities, ordering and action definitions live in a shared
  constants/utils layer — never inline in a builder. New signals are added by extending a table, not
  by editing branches scattered across services.
- **Redirects:** every attention card deep-links into the **Action Center** with the correct filters.
  Unsupported patterns still redirect with the closest available filters.
- **Ambiguity handling:** the feature has several reasonable interpretations (severity of some issues,
  listing grouping, main-image classification). They are enumerated in §14 and must be confirmed
  before implementation.

---

## 2. Locked decisions (carried from the requester + prior plan)

| # | Decision |
|---|---|
| D1 | Sections = **What needs attention now** (≤5 cards, sorted severity → issue type) and **Next best actions** (≤4, non-interactive). |
| D2 | Severity colours: **red = cannot sell**, **yellow = can sell but cannot trust/safely operate on data**, **black = can sell but needs fixing/completing** (§3). |
| D3 | API returns the full list; the webapp caps (5 / 4) via webapp constants. |
| D4 | Counting unit = one per **variant** when the product has variants, else one per **product**; parents with variants are suppressed (reuse `buildChannelListingKeysCteSql({ sellableOnly: true })` — same as the pulse cards D6). |
| D5 | "Published" = `sync_status ∈ {SUCCESS, OUT_OF_SYNC}` (`CONSTANTS.CHANNEL_PUBLISHED_STATUSES`). |
| D6 | Every card carries a machine-readable **redirect filter payload** (issueType / channelId / marketplaceId / health / workflowStatus) understood by `/actionCenter` today (§7.4 of the webapp research). |
| D7 | Attention items are **grouped** per (channel [, marketplace]) × issue family (one card per group with a count), not per unit. Grouping reuses the Action Center `operationalGrouping.canonicalGroupKey` semantics where available. |
| D8 | All thresholds, severity mappings, issue-family and action definitions live in `src/core/constants/command-center-v2/`; the pure logic lives in `.../command-center/utils/commandCenterV2Attention.ts`, mirroring `commandCenterV2Pulse.ts`. |
| D9 | Reuse live sources: `ChannelHealthService.getHealth` (connection/account/marketplace/data-trust), the pulse SQL backbone (price/stock/listing issues), and the Action Center snapshot for issue-type/redirect semantics. No new tables. |
| D10 | "Later on" cases (Conversion rate, Buy box lost, ad intelligence, deeper pricing troubles, notification-driven live updates) are documented in §9, not built. |
| D11 | **Low stock** = **yellow** (sellable, imminent risk). |
| D12 | **Missing main image** = **black**; missing vs unsupported-type shown as **one card with two chips** (no red, no second card). |
| D13 | **Amazon listing blockers** = **one aggregate card** with per-bucket chips (not one card per bucket). |
| D14 | **Product-type-changed** redirects to the **Action Center with the closest category/product-type filter** (no dedicated issue type exists). |
| D15 | **Catalog completeness** (missing content, missing attributes, low readiness) **is included** as black cards. |
| D16 | Amazon signals group **per channel** with **marketplace chips** (not per channel×marketplace). |
| D17 | **Data-trust composition moves API-side** (into the `/v2/attention` endpoint); the channel-health section is unchanged. |
| D18 | **Sales decline / negative margin does not create an attention card** — it stays in the Sales & Profit pulse card only. |

---

## 3. Severity & ordering model (the crux)

Severity answers exactly one question: **"can the seller sell / trust / finish?"**

| Colour | Meaning | Sellability test |
|---|---|---|
| **red** | Because of this, I cannot sell. | The unit (or the channel) is not sellable / not visible / not reachable right now. |
| **yellow** | I can still sell, but I cannot trust or safely operate on the data. | Sales are possible, but the underlying data is stale/out-of-sync/untrusted, or a risk is imminent. |
| **black** | I can sell, but something needs to be fixed or completed. | Sales work; there is unfinished/corrective work (structure, media, content, remap). |

**Confirmed issue → severity map:**

| Issue family | Severity | Rationale |
|---|---|---|
| Connection lost / not connected / token revoked / app uninstalled | **red** | Categra cannot operate the channel at all. |
| Account deactivated / account at risk | **red** | Selling privileges affected. |
| Amazon marketplace deactivated / at risk | **red** | Marketplace scope cannot sell. |
| Data trust stale (inventory or orders sync stale) | **yellow** | Can sell, but data cannot be trusted/safely operated on. |
| Missing price (published) | **red** | No price ⇒ cannot sell. |
| Out of stock (published) | **red** | No stock ⇒ cannot sell. |
| Low stock (published) | **yellow** | Still sellable; imminent risk (D11). |
| Stock sync issue / sync failed / out-of-sync | **yellow** | Can sell, data untrusted. |
| Amazon listing blockers (suppression / not buyable / search-suppressed / compliance / structure) | **red** | Listing blocked on Amazon side. |
| Amazon listing warnings (no blockers) | **black** | Sellable; correction needed. |
| Amazon not searchable (published, no `DISCOVERABLE`) | **red** | Not discoverable to buyers. |
| Shopify not active (`external_status ≠ ACTIVE`, published) | **red** | Not visible/purchasable to customers. |
| Missing main image (published) | **black** | Completeness blocker; fix required (D12). |
| Unsupported main-image type for the channel (published) | **black** | Sync/correction needed; listing may lag (D12). |
| Missing content / attributes (catalog readiness) | **black** | Incomplete catalog. |
| Amazon product type changed by Amazon | **black** | Sellable; remap/accept needed. |
| Amazon product type recheck in progress | **black** | Informational pending state. |

**Ordering** (deterministic, applied API-side; the webapp only slices):
1. severity rank: red `0` → yellow `1` → black `2`;
2. issue-family rank (stable domain order; reuse Action Center `ISSUE_PRIORITY` ranks where they map,
   else a fixed family order table);
3. affected count desc, then `priorityScore` desc (Action Center score where available),
   then label asc (stable tiebreak).

---

## 4. Attention item catalog (source → severity → count → redirect)

Each item is produced as a **group**: `{ id, severity, issueFamily, typeLabel, impact, title,
description, chips[], affectedCount, redirect, priorityScore }`.

| # | Case (requester) | Source (existing = live) | Unit | Redirect (`/actionCenter?…`) |
|---|---|---|---|---|
| 1 | Account / channel-level health (groupable: connection lost, not connected, deactivated, at risk) | **existing** `ChannelHealthService.getHealth` → channel `reasons` / `connection` / `account` | per channel | `issueType=CHANNEL_NOT_CONNECTED` \| `CHANNEL_CONNECTION_ERROR` `&channelId` (alt: `/channelManagement?channelId=`) |
| 2 | Amazon channel **marketplace-level** health (deactivated / at risk / AHR / metric / policy) | **existing** `ChannelHealthService.getHealth` → `marketplaces[].reasons` | per channel (+marketplace chips) | `channelId`, `marketplaceId`; `issueType` from reason code family |
| 3 | Data trust — inventory/orders sync stale | **existing** `channel-health.dataTrust` (`buildHealthAttentionItems`) | per (channel, domain) | `issueType=CHANNEL_NOT_CONNECTED` + channelId (data-trust) |
| 4 | Missing price (published) | **reuse** pulse `buildPricingKeysQuery` (group by channel/marketplace) | sellable entity | `issueType=MISSING_PRICE` |
| 5 | Missing stock (out of stock, published) | **reuse** pulse `buildInventoryKeysQuery` (state `OOS`) | sellable entity | `issueType=OUT_OF_STOCK` |
| 6 | Low stock (published) | **reuse** pulse `buildInventoryKeysQuery` (state `LOW`) | sellable entity | `issueType=LOW_STOCK` |
| 7 | Stock sync issues (state `SYNC` + `tbl_product_sync_failures` module `STOCK`) | **reuse** pulse inventory `SYNC` | sellable entity | `issueType=SYNC_FAILED` \| `OUT_OF_SYNC` |
| 8 | Amazon listing **blockers** — **one aggregate card** with per-bucket chips | **reuse** pulse `buildListingIssuesQuery` (blocker stage) + `amazon-listing-issue-buckets` taxonomy | listing unit | `issueType=AMAZON_ERROR` (+ optional bucket/`issueGroupKey`) |
| 9 | Amazon listing **warnings** (when no blockers) | **reuse** pulse listings (warning stage) | listing unit | `issueType=AMAZON_WARNING` |
| 10 | Amazon **not searchable** (published, `external_status` lacks `DISCOVERABLE`) | **new** query over published Amazon keys | sellable entity | `issueType=AMAZON_NOT_SEARCHABLE` |
| 11 | Shopify **not active** (published, `tbl_product_channels.external_status ≠ ACTIVE`) | **new** query (variant inherits parent status) | sellable entity | `issueType=SHOPIFY_UNPUBLISHED` \| `SHOPIFY_INACTIVE` |
| 12 | **Missing main image** (published; product and each variant need one) — **one black card with two chips** | **new** query (`tbl_product_media_linkers.is_main_image`, channel→master) | sellable entity | `issueType=MISSING_MEDIA` |
| 13 | **Unsupported main-image type** for the channel — same card as #12, second chip | **new** query (`ALLOWED_FILE_EXTENSIONS` per channel type) | sellable entity | `issueType=MISSING_MEDIA` (+ media chip) |
| 14 | Amazon **product type changed by Amazon** (`tbl_product_channels.suggested_product_attribute_group_id IS NOT NULL`) | **new** query | product (channel scope) | Action Center, closest category/product-type filter |
| 15 | Missing content / attributes / low readiness | **existing** Action Center `MISSING_CONTENT` / `MISSING_ATTRIBUTES` / `LOW_READINESS` | product | `issueType=MISSING_CONTENT` \| `MISSING_ATTRIBUTES` |

Notes:
- Cases 12 & 13 produce the requester's desired split: *"5 have missing main image, 3 have unsupported
  main image"* → surfaced as **one black card with two chips** (D12).
- Business decline / margin (Sales & Profit) stays in the pulse card only; **not** repeated here (D18).
- "Other worth-noticing" signals (`SOURCE_DATA`, low-readiness) are included behind the same catalog
  and can be toggled by config, not by code.

---

## 5. Next best actions model (≤4, non-interactive)

Derived from the **same** collected attention items — the investigation is shared; only the
presentation differs. Actions are dependency-ordered (do not fix downstream before access/trust).

**Proposed action catalog** (`src/core/constants/command-center-v2/command-center-v2-attention.constants.ts`),
one entry per family, each with `id`, `impact`, `i18nKey` + param builder, and an ordering rank:

| Rank | Action | Triggered by | Impact |
|---|---|---|---|
| 1 | Restore access / data trust (`Reconnect {channel}` / `Restore trusted {domain} data`) | connection, account, marketplace health, data trust | high |
| 2 | Fix sellability (`Add prices`, `Restock`) | missing price, out of stock | high |
| 3 | Fix visibility & listings (`Fix Amazon listings`, `Reactivate Shopify listings`) | listing blockers, not searchable, not active | high |
| 4 | Complete the catalog (`Add main images`, `Fix image formats`, `Remap product type`, `Replenish low stock`) | main image, unsupported type, PTD changed, low stock, content/attributes | medium |

- API returns all triggered actions; the webapp caps at 4 (`COMMAND_CENTER_V2_NEXT_ACTIONS_LIMIT`).
- No button/route on an action card (per the brief); the section is informational.
- Titles/descriptions are **i18n keys** with count params (fixes the current hard-coded English in
  `mock-data.ts`).

---

## 6. API design

### 6.1 Endpoint

```
POST /command-center/v2/attention
body: { channelId?: number; marketplaceId?: number }
→ {
    meta: { generatedAt, scope: { channelId, marketplaceId } },
    items: CommandCenterV2AttentionItem[],     // ALL items, already ordered
    actions: CommandCenterV2NextAction[]        // ALL triggered actions, already ordered
  }
```

- Added to the existing `CommandCenterV2Controller` (`command-center/v2`), `JwtAuthGuard` +
  `@RequiresSubscriptionCapabilities('ACTION_CENTER_VIEW')` (same as `/pulse`), envelope via
  `GlobalResponses.formatResponse`.
- Redis cache 60–90 s keyed like the pulse service (account, user, scope, channel, marketplace).

### 6.2 Service & builder files (new, inside `.../orchestration/command-center/`)

- `command-center-v2-attention.service.ts` — scope resolution (reuse `resolveListingSummaryChannelIds`
  + `ChannelService.getUserWiseChannels` exactly as the pulse service), caching, orchestration.
- `command-center-v2-attention.sql.ts` — the new grouped signal queries (cases 10–14) + grouped
  variants of the pulse price/stock/listing queries (cases 4–9) sharing the
  `buildChannelListingKeysCteSql` backbone.
- `attention.builder.ts` — composes channel-health items (injected `ChannelHealthService`) + signal
  items + grouping/dedupe + redirect payloads, then orders via the utils.
- `next-actions.builder.ts` — maps ordered items → the action catalog.
- `utils/commandCenterV2Attention.ts` — pure functions: `resolveAttentionSeverity`,
  `resolveIssueFamilyRank`, `orderAttentionItems`, `resolveNextActions` (+ spec).
- `dto/command-center-v2-attention.dto.ts`, types appended to `command-center-v2.types.ts`.
- `command-center.module.ts` — add the service to providers/exports and import `ChannelHealthModule`.

### 6.3 Reuse map

| Concern | Reused artifact |
|---|---|
| Published sellable keys | `buildChannelListingKeysCteSql({ sellableOnly: true })` + `channelListingKeysReplacements` |
| Stock resolution / buckets | pulse inventory SQL + `classifyInventoryState` / `resolveInventoryBucket` |
| Pricing inheritance | pulse pricing SQL + `resolvePublishedEffectivePrice` |
| Listing blockers/warnings + buckets | pulse listings SQL + `classifyAmazonListingIssue` + taxonomy |
| Account/channel/marketplace/data-trust | `ChannelHealthService.getHealth` |
| Redirect semantics | Action Center URL contract (`issueType` / `issueGroupKey` / `channelId` / `marketplaceId` / `health` / `workflowStatus`) |

`webapp/src/lib/api.ts`: add `GET_COMMAND_CENTER_V2_ATTENTION: 'command-center/v2/attention'`.

---

## 7. Constants & classification modules (single source of truth)

- `src/core/constants/command-center-v2/command-center-v2-attention.constants.ts` (new):
  - `COMMAND_CENTER_V2_ATTENTION_THRESHOLDS` (e.g. low-stock band if needed, media min).
  - `COMMAND_CENTER_V2_ATTENTION_SEVERITY` — issue-family → severity table (§3).
  - `COMMAND_CENTER_V2_ATTENTION_FAMILY_ORDER` — stable family ranking for the issue-type tiebreak.
  - `COMMAND_CENTER_V2_ATTENTION_REDIRECT` — family → `{ issueType|issueGroupKey, health?, workflowStatus? }`.
  - `COMMAND_CENTER_V2_NEXT_ACTIONS` — the action catalog (§5).
- Reuse the existing `amazon-listing-issue-buckets.ts` taxonomy unchanged.
- Pure logic in `utils/commandCenterV2Attention.ts`; specs colocated enumerate every mapping,
  rank boundary and redirect payload.

---

## 8. Webapp changes

- `_components/attention-section.tsx`: source `items` from the new endpoint (via a
  `_lib/use-section-request.ts` call) instead of `mock-data` + client-only health composition; keep
  the existing sort/skeleton/expanded-vs-compact layout; cap to 5 via
  `COMMAND_CENTER_V2_ATTENTION_LIMIT`.
- `_components/next-actions-section.tsx`: source from the endpoint, cap 4, render API title/description
  keys; **enrich UI** to match the rest of Command Center V2 (impact badge, richer card styling,
  spacing, section-entry animation) rather than the current bare numbered strip.
- `main.tsx`: stop composing `attentionItems` from `channelHealth` + mock; pass the live items; make
  `openActionCenter(item)` serialize `item.redirect` into the `/actionCenter` query string (the v2 page
  currently pushes a bare `/actionCenter`; the old Command Center already shows the serializer pattern).
- `_lib/types.ts`: add `CommandCenterV2AttentionItem` (API shape) + `CommandCenterV2NextAction`.
- `_lib/attention-sort.ts`: keep as the webapp-side ordering/limit guard (severity → issue type → cap).
- i18n: add `Command_Center_V2_Attention_*` / `Command_Center_V2_Next_*` item keys + `Command_Center_V2_Cta_*`
  in `translationServices/english/command-center-v2.ts` only (`fallbackLng` covers the rest).

---

## 9. Data-quality caveats, gaps & deferred ("later on") items

1. **Shopify unpublished/inactive flags are dead** in the Action Center today
   (`shopifyUnpublished`/`shopifyInactive` are never set true; the Shopify read model filters
   `external_status = 'ACTIVE'` and hardcodes `listingStatus = 'PUBLISHED'`). Case 11 is therefore a
   **new** query, not a flag read.
2. **"Unsupported main-image type" has no existing signal.** It is computed by comparing the unit's
   main-image asset `file_type` against `ALLOWED_FILE_EXTENSIONS[IMAGE][channelType]`
   (`/CHANNEL_SYNCABLE_MEDIA_TYPES`). No readiness rule or Action Center flag exists today.
3. **Action Center "missing media" is count-based**, not a main-image check
   (`ReadinessMediaResolver` counts rows; `getPrimaryMediaMinimum` reads only the master rule). Case 12
   deliberately applies the stricter per-sellable-entity main-image rule from the brief.
4. **Product type change has no aggregate counter** — only the per-product banner
   (`getProductTypeChangeAlert`, `suggested_product_attribute_group_id IS NOT NULL`). Case 14 adds the
   first aggregation; the signal is channel-scoped (not marketplace-scoped).
5. **Marketplace-level product-type change** is not modeled (the suggested group lives on
   `tbl_product_channels`), so case 14 counts at channel scope.
6. **`ScopeItem.routing` is unpopulated** for product rows (only channel rows). Redirect payloads are
   built from the item's issue type + scope, not from `routing`.
7. **Data-trust items** are currently composed client-side in `_lib/channel-health-view.ts`; this plan
   moves composition API-side for consistency (§8). The channel-health section itself stays unchanged.
8. **Deferred / later on:** Conversion rate (`GET_SALES_AND_TRAFFIC_REPORT`), Buy box lost/price
   opportunities (offers API), ad intelligence, pricing troubles beyond missing price, notification-
   driven real-time attention refresh, scheduled FBA stock freshness.

---

## 10. Effort estimate

Ideal dev-days for one backend + one frontend engineer, including unit tests and a read-only DB
verification pass. Shared plumbing is amortised with the pulse cards.

| # | Work item | Days |
|---|---|---|
| 1 | `/command-center/v2/attention` endpoint + service skeleton, DTO, scope, cache, `lib/api.ts`, types | 1.0 |
| 2 | New grouped signal SQL: not-searchable, Shopify not-active, main-image (missing/unsupported), product-type-changed | 3.0 |
| 3 | Grouped variants of pulse price/stock/listing queries + reuse map | 1.5 |
| 4 | Severity/order/family/redirect constants + pure utils + spec | 1.5 |
| 5 | Attention composer (health items + signal items + grouping/dedupe + redirects) | 2.0 |
| 6 | Next-actions builder + catalog + spec | 1.0 |
| 7 | Webapp: live attention + next-actions, cap, mapping, skeletons, CTA deep-link, UI enrichment | 2.0 |
| 8 | i18n keys (en) + polish | 0.5 |
| 9 | Tests + verification queries (published-only, unit rule, each signal, severity/order, redirects) | 2.0 |
| 10 | Buffer / integration polish / review | 1.5 |
| | **Total** | **~16 dev-days (~3.3 weeks calendar)** |

**Risk drivers:** the main-image missing/unsupported resolution (channel→master, per sellable entity,
extension allow-lists) and the grouped signal SQL (four new signals + grouping by channel/marketplace)
are the areas most likely to overrun. No schema migration is required.

---

## 11. Suggested phasing

| Phase | Content |
|---|---|
| **P1** | Endpoint + reuse-based items (price, stock, listings, account/channel/marketplace/data-trust) + next actions; wire both sections. |
| **P2** | New signals: Amazon not-searchable, Shopify not-active, main-image missing/unsupported, product-type-changed. |
| **P3** | Hardening: Action Center redirect payloads per family, config toggles for optional families, verification pass. |
| **Later** | Conversion, Buy box, ad intelligence, deeper pricing, notification-driven refresh. |

---

## 12. Verification checklist

- [ ] Only `sync_status ∈ {SUCCESS, OUT_OF_SYNC}` units are counted; a variant parent is never counted.
- [ ] Every item carries a valid `/actionCenter` redirect (channelId/marketplaceId in scope preserved).
- [ ] Severity map matches §3 exactly; ordering is severity → family → count → score → label (stable).
- [ ] API returns all items; the webapp caps attention at 5 and actions at 4 via webapp constants.
- [ ] Connection/account/marketplace/data-trust items match the live channel-health section.
- [ ] Amazon not-searchable counts published units lacking `DISCOVERABLE`; Shopify not-active counts
      published units whose product channel `external_status ≠ ACTIVE` (variant inherits parent).
- [ ] Missing main image is per sellable entity (product or variant), channel main → master main;
      unsupported-type is split out via the channel extension allow-list; both render as one black card
      with two chips.
- [ ] Catalog completeness (missing content / attributes / low readiness) renders as black cards and is
      still capped by the frontend limit along with every other item.
- [ ] Product-type-changed counts published Amazon products with `suggested_product_attribute_group_id`.
- [ ] Next actions are dependency-ordered, ≤4, non-interactive, and use i18n keys with counts.
- [ ] No threshold/severity/family/action definition is inline in a builder — all from `src/core`.
- [ ] Both sections render the existing skeleton while loading and the `SectionLoadState` on error/restricted.

---

## 13. Key file references

- **Target section components:** `webapp/src/app/(index)/(menu-layout)/command-center-v2/_components/attention-section.tsx`,
  `.../_components/next-actions-section.tsx`, `.../main.tsx`, `.../_lib/attention-sort.ts`,
  `.../_lib/mock-data.ts`, `.../_lib/channel-health-view.ts`, `.../_lib/use-section-request.ts`,
  `.../_lib/styles.ts` (`COMMAND_CENTER_V2_SEVERITY_CLASS`), `.../_lib/types.ts`.
- **Pulse backbone to reuse:** `api/.../command-center/command-center-v2-pulse.service.ts`,
  `command-center-v2-pulse.sql.ts`, `command-center-v2.types.ts`,
  `.../utils/commandCenterV2Pulse.ts`, `core/constants/command-center-v2/amazon-listing-issue-buckets.ts`.
- **Health source:** `api/.../catalog/channel/health/channel-health.service.ts` (`getHealth`),
  `channel-health.types.ts`; webapp `.../_lib/channel-health-view.ts`.
- **Action Center (issue types, snapshot, redirect semantics):**
  `api/.../operations/action-center/action-center.service.ts` (`ScopeItem`, `getSalesOperationsSnapshot`,
  `ISSUE_PRIORITY`), webapp `actionCenter/main.tsx` URL parsing, `actionCenter/_lib/redesign.ts`
  (`ActionCenterCompactFilters`), `webapp/src/lib/action-center-remediation-navigation.ts`.
- **Published keys CTE:** `api/.../catalog/products/listings/channel-listing-keys.sql.ts`.
- **Media:** `entities/productMediaLinker.entity.ts` (`is_main_image`), `entities/digitalAssets.entity.ts`
  (`file_type`), `core/constants/constants.ts` (`ALLOWED_FILE_EXTENSIONS`, `CHANNEL_SYNCABLE_MEDIA_TYPES`,
  `CHANNEL_SYNCABLE_FILE_EXTENSIONS`), `.../media/digital-assets/productMedia/product-media.service.ts`.
- **Product type change:** `entities/productChannel.entity.ts` (`suggested_product_attribute_group_id`,
  `amazon_product_type_id`, `is_product_type_rechecking`), `product.service.ts` (`getProductTypeChangeAlert`),
  webapp `products/[product]/_components/action-center/productTypeChangeBanner.tsx`.
- **External status:** `entities/productChannelMarketplace.entity.ts`, `variantMarketplace.entity.ts`
  (Amazon `external_status` JSONB), `entities/productChannel.entity.ts` (Shopify `external_status` scalar).
- **Constants:** `core/constants/constants.ts` (`CHANNEL_PUBLISHED_STATUSES`), `core/constants/global-enums.ts`.

---

## 14. Resolved decisions

All feature-level ambiguities were confirmed before implementation:

| Question | Resolution |
|---|---|
| Severity of "low stock" | **Yellow** (sellable, imminent risk) — D11. |
| Severity / layout of "missing main image" | **Black**, **one card with two chips** (missing vs unsupported-type) — D12. |
| Amazon listing blockers grouping | **One aggregate card** with per-bucket chips — D13. |
| Product-type-changed redirect | **Action Center, closest category/product-type filter** — D14. |
| Include catalog content/attributes as cards | **Yes**, as black cards — D15. |
| Amazon grouping scope | **Per channel + marketplace chips** — D16. |
| Data-trust composition | **API-side** in `/v2/attention` — D17. |
| Sales decline / negative margin | **Pulse card only** (no attention card) — D18. |

Remaining implementation-time detail (not feature-level): the exact Action Center `issueGroupKey`/
`issueType` used for the product-type-changed redirect, and the per-family optional-card toggles —
both resolved during P1/P3 using the existing Action Center filter metadata.

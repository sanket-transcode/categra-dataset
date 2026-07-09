# Delta / Selective Sync — Low-Level Implementation Plan

> Scope of this doc: turn the [high-level-plan](./high-level-plan.md) into concrete, file-level work.
> **Backend** = one new route on the already-built `syncReview` module (delete stale baselines + flip stalled `OUT_OF_SYNC` → `SUCCESS`).
> **Frontend** = everything else: build the change-review dialog from [delta-sync-review.html](./delta-sync-review.html) and wire the `Publish Changes` / `Publish to External Channel` flow into the **Variants tab**, **Products list**, and **Channels tab**.
>
> Companion context: [present-sync-changes.md](./present-sync-changes.md) (selection rules + payload format), [sync-change-summary.md](./sync-change-summary.md) (exact `get-change-summary` response shape).
>
> **All open questions from the first review round are resolved** — decisions are baked into the sections below and summarized in [§ Resolved decisions](#resolved-decisions).

---

## 0. What already exists (do not rebuild)

| Piece | Location | Returns |
|---|---|---|
| `POST /sync-review/get-change-counts` | [syncReview.controller.ts](../../../api/apps/api-main/src/modules/app/syncReview/syncReview.controller.ts) | `ProductChangeCounts[]` — per-product group counts, **only products with `total > 0`**. Empty array ⇒ no real baseline deltas. |
| `POST /sync-review/get-change-summary` | same | `MarketplaceChange[]` — full grouped diff for one product × channel (× all/one marketplace). |
| `POST /sync/product-syncing/publish-products-to-channel` | [productSyncing.controller.ts](../../../api/apps/api-main/src/modules/app/syncing/productSyncing/productSyncing.controller.ts) | Body `{ channelId, itemToSync, marketplaceIds?, ignoreWarning? }` **+ a new selective key** (see [F3 Submit](#f3-submit)). The dialog's **Submit** calls this. |
| `POST /sync/product-syncing/sync-product-to-channel` | same | Body `{ productId, channelId, channelMarketplaceIds?, variantIds }`. The existing "publish a Not-Published product" path — unchanged. |
| `productService.clearSyncBaselines(...)` | [product.service.ts](../../../api/apps/api-main/src/modules/app/products/product.service.ts) (~26268) | Deletes baseline rows for a product/variant/marketplace scope. Reused by the new route (see [B1.3](#b13-service)). |

Sync-status enum ([globalEnums.ts](../../../api/apps/api-main/src/core/config/globalEnums.ts) `GlobalEnums.syncStatus`): `NOT_PUBLISHED | SUCCESS | ERROR | OUT_OF_SYNC`.

Status is stored per level/channel-type. **For Amazon, sync status lives ONLY on the marketplace-scoped tables** — `tbl_product_channels` / `tbl_product_variant_channels` `sync_status` are irrelevant for Amazon and must never be read or written by this feature for an Amazon channel:

| Channel type | Level | Table (sync_status) |
|---|---|---|
| Non-Amazon | parent | `tbl_product_channels` |
| Non-Amazon | variant | `tbl_product_variant_channels` |
| Amazon | parent | `tbl_product_channel_marketplaces` |
| Amazon | variant | `tbl_variant_marketplaces` |

"Enabled/in-channel" = `is_active`/`status` boolean on those rows (+ `is_deleted = false`). The active-membership join paths are already spelled out in `SyncReviewService.getChangeCounts` (`active_products` / `active_variants` CTEs) — reuse those exact joins.

---

# BACKEND

## B1. New route — `POST /sync-review/mark-in-sync`

**Purpose:** the "stalled out-of-sync" repair. When `get-change-counts` returns `[]` for a selection (no baseline deltas surface) but the entity's `sync_status` is still `OUT_OF_SYNC`, this route (1) **deletes any leftover/stale baseline rows** for the scope and (2) flips those status rows `OUT_OF_SYNC` → `SUCCESS`. It **only** touches rows currently `OUT_OF_SYNC` — never `NOT_PUBLISHED` / `ERROR` / `SUCCESS` — and never pushes anything to Amazon/Shopify. **No capability gate** (it performs no external sync).

### B1.1 DTO — `dto/markInSync.dto.ts`

Mirror `GetChangeCountsDTO` (same `products` map convention).

```ts
import { IsInt, IsObject, IsOptional } from 'class-validator';

export class MarkInSyncDTO {
	@IsObject()
	products: Record<string, number[]>; // productId -> variantIds

	@IsInt()
	channelId: number;

	@IsOptional()
	@IsInt()
	amazonMarketplaceId?: number | null; // global acm.amazon_marketplace_id (NOT the channel-marketplace join id)
}
```

**`products` map semantics (identical to `get-change-counts`):**
- key = productId, value = variantId[]
- **empty array** ⇒ *all enabled variants **+** the parent itself*
- **non-empty array** ⇒ *only those variants; **parent excluded** (never touch parent status/baselines).*

### B1.2 Controller method — add to `SyncReviewController`

Follow the file's existing shape exactly (guard, `formatResponse`, try/catch → error response):

```ts
@UseGuards(JwtAuthGuard)
@Post('mark-in-sync')
async markInSync(
	@Req() req: AuthenticatedRequest,
	@Res() res: Response,
	@Body() body: MarkInSyncDTO,
): Promise<object> {
	try {
		return res.json(
			this._globalResponses.formatResponse(
				req,
				GlobalEnums.responseStatuses.success,
				await this._service.markInSync(req, body),
				null,
			),
		);
	} catch (error) {
		return res.json(this._globalResponses.formatResponse(req, GlobalEnums.responseStatuses.error, null, null));
	}
}
```

### B1.3 Service method — `SyncReviewService.markInSync(req, body)` {#b13-service}

Resolve channel + account scope the same way `getChangeCounts` does:

```ts
const channelId = Number(body.channelId);
const channel = await this._channelService.findOne(req, { id: channelId, accountId: req.user.accountId }, undefined, ['id', 'channelType']) as Channel;
if (!channel) return { updated: 0, deletedBaselines: 0 };
const isAmazon = isAmazonChannel(channel.channelType);
const amazonMarketplaceId = Number(body.amazonMarketplaceId) || null;
const entries = Object.entries(body.products).map(([pid, vids]) => [Number(pid), (vids || []).map(Number).filter(Boolean)]);
if (!entries.length) return { updated: 0, deletedBaselines: 0 };
```

Two units of work — run **inside one transaction** (atomicity: we must not flip status to SUCCESS if the baseline delete failed, and vice-versa). `_channelService.rawQuery` / `productService.clearSyncBaselines` both accept a `transaction`; open one via the Sequelize instance the service already reaches through `_channelService`.

#### Step 1 — Delete stale baselines (B4)

Reuse `productService.clearSyncBaselines` — it already implements full-product-vs-specific-variant resolution and marketplace scoping:

```ts
await this._productService.clearSyncBaselines({
	req,
	channelId,
	productIds: entries.map(([pid]) => pid),
	variantSelectionByProduct: Object.fromEntries(entries), // Record<productId, variantId[]> — [] = full product
	specificMarketplaceIds: <see note>,
	transaction,
});
```

> **Marketplace-id caveat:** `clearSyncBaselines.specificMarketplaceIds` expects the **external string `marketplaceId`** (e.g. `ATVPDKIKX0DER`) which it resolves to `AmazonMarketplace.id` internally — whereas our DTO carries the **numeric `amazonMarketplaceId` (= `AmazonMarketplace.id`)** directly. So `markInSync` must map numeric → external string before calling (one lookup on `tbl_amazon_marketplaces`), **or** we add an overload/param to `clearSyncBaselines` that accepts the numeric id directly. Prefer the small lookup in `markInSync` to avoid touching the shared method. When `amazonMarketplaceId` is null (all markets) omit `specificMarketplaceIds` entirely.
>
> Media baselines are channel-level (no marketplace) — `clearSyncBaselines` already handles that internally; nothing extra to do.

#### Step 2 — Flip status `OUT_OF_SYNC` → `SUCCESS`

Build the same `input(product_id, variant_ids)` `VALUES` table used in `getChangeCounts` (reuse that exact `valuesSql` builder — `ARRAY[...]::bigint[]` / `ARRAY[]::bigint[]`). Then run **one UPDATE per level**, each guarded by `sync_status = OUT_OF_SYNC` and active membership. **Row-selection rules:** parent rows only when the product's `variant_ids` array is empty (`array_length(i.variant_ids,1) IS NULL`); variant rows when empty (all variants) **or** `variant_id = ANY(i.variant_ids)`.

**Non-Amazon (Shopify)** — updates `tbl_product_channels` (parent) + `tbl_product_variant_channels` (variant):

```sql
-- Parent
WITH input(product_id, variant_ids) AS (VALUES /* valuesSql */)
UPDATE tbl_product_channels pc
SET sync_status = :SUCCESS
FROM input i
WHERE pc.product_id = i.product_id
  AND array_length(i.variant_ids, 1) IS NULL
  AND pc.channel_id = :channelId AND pc.is_active = true AND pc.is_deleted = false
  AND pc.sync_status = :OUT_OF_SYNC;

-- Variant (join tbl_product_variants for product scoping)
WITH input(product_id, variant_ids) AS (VALUES /* valuesSql */)
UPDATE tbl_product_variant_channels pvc
SET sync_status = :SUCCESS
FROM tbl_product_variants pv
JOIN input i ON i.product_id = pv.product_id
WHERE pvc.product_variant_id = pv.id
  AND pvc.channel_id = :channelId AND pvc.is_active = true AND pvc.is_deleted = false
  AND pvc.sync_status = :OUT_OF_SYNC
  AND (array_length(i.variant_ids, 1) IS NULL OR pv.id = ANY(i.variant_ids));
```

**Amazon** — updates `tbl_product_channel_marketplaces` (parent) + `tbl_variant_marketplaces` (variant), **only** these two tables. `amazonMarketplaceId` optional → all marketplaces of the channel when null:

```sql
-- Parent
WITH input(product_id, variant_ids) AS (VALUES /* valuesSql */)
UPDATE tbl_product_channel_marketplaces pcm
SET sync_status = :SUCCESS
FROM tbl_product_channels pc
JOIN input i ON i.product_id = pc.product_id
JOIN tbl_amazon_channel_marketplaces acm ON acm.id = pcm.amazon_channel_marketplace_id AND acm.status = true
WHERE pcm.product_channel_id = pc.id
  AND array_length(i.variant_ids, 1) IS NULL
  AND pc.channel_id = :channelId AND pc.is_active = true AND pc.is_deleted = false
  AND pcm.status = true AND pcm.is_deleted = false
  AND pcm.sync_status = :OUT_OF_SYNC
  {AND acm.amazon_marketplace_id = :amazonMarketplaceId};

-- Variant
WITH input(product_id, variant_ids) AS (VALUES /* valuesSql */)
UPDATE tbl_variant_marketplaces vm
SET sync_status = :SUCCESS
FROM tbl_product_channel_marketplaces pcm
JOIN tbl_product_channels pc ON pc.id = pcm.product_channel_id
JOIN input i ON i.product_id = pc.product_id
JOIN tbl_amazon_channel_marketplaces acm ON acm.id = pcm.amazon_channel_marketplace_id AND acm.status = true
WHERE vm.product_channel_marketplace_id = pcm.id
  AND pc.channel_id = :channelId AND pc.is_active = true AND pc.is_deleted = false
  AND pcm.status = true AND pcm.is_deleted = false
  AND vm.status = true AND vm.is_deleted = false
  AND vm.sync_status = :OUT_OF_SYNC
  AND (array_length(i.variant_ids, 1) IS NULL OR vm.product_variant_id = ANY(i.variant_ids))
  {AND acm.amazon_marketplace_id = :amazonMarketplaceId};
```

**Return shape:** `{ updated: number, deletedBaselines?: number }` (sum of affected status-rows). The FE uses success only to flip local state; the exact number is informational.

### B1.4 Wiring / housekeeping
- Import the DTO in the controller.
- `ChannelService` (for `findOne` + `rawQuery`) is already injected into `SyncReviewService`. **Add `ProductService`** (for `clearSyncBaselines`) to `SyncReviewModule` imports + constructor (`forwardRef` to match the module's existing pattern).
- No migration (pure UPDATE + baseline deletes on existing tables).
- **Multi-tenant safety:** every statement is scoped by `channel_id` + the `accountId` check on `channelService.findOne`. Keep the channel-id filter everywhere.
- JSDoc on new methods (repo lint requires it). Tabs, single quotes, 120 cols.

### B1.5 Correctness guarantees (from clarifications)
- **The `sync_status = OUT_OF_SYNC` predicate is mandatory on every UPDATE** — this is the single guard that makes the route safe to call "wisely" from the FE. It can never corrupt `NOT_PUBLISHED` / `ERROR` / `SUCCESS` rows, and it makes the all-markets partial case a non-issue (clean markets flip, dirty markets are handled by the dialog). No extra all-markets logic needed.
- **Parent is never touched when the variant array is non-empty** (neither status nor baselines) — enforced by the `array_length(...) IS NULL` guard on parent statements and by `variantSelectionByProduct` in `clearSyncBaselines`.
- Empty `products` map ⇒ early return (no table-wide writes).

---

# FRONTEND

Four deliverables: (F1) API URL builders + types, (F2) an orchestration hook implementing "counts → decide → flip-or-dialog", (F3) the shared **Sync Change Review Dialog** (adapted from the prototype), (F4) button wiring at the entry points.

## F1. API layer — `webapp/src/lib/api.ts`

Add (nothing hardcoded in components, per root CLAUDE.md URL contract):

```ts
GET_CHANGE_COUNTS: 'sync-review/get-change-counts',
GET_CHANGE_SUMMARY: 'sync-review/get-change-summary',
MARK_IN_SYNC: 'sync-review/mark-in-sync',          // NEW
// reused: PUBLISH_PRODUCTS_TO_CHANNEL (getProductApiUrl), SYNC_PRODUCT_TO_CHANNEL (getGeneralApiUrl)
```

All calls go through the `genericService` singleton (`.post`).

**Types** — add `types/sync-review.ts` mirroring [sync-change-summary.md](./sync-change-summary.md) (`MarketplaceChange`, `ChangedEntityChange`, `EntityGroups`, and each `*Result`) plus request bodies (`GetChangeCountsBody`, `MarkInSyncBody`) and the selective publish payload `SyncSelectionPayload` from [present-sync-changes.md §3](./present-sync-changes.md).

**Marketplace ids:** everywhere in this feature use the **global `amazonMarketplaceId`** (`acm.amazon_marketplace_id`) for `get-change-counts` / `get-change-summary` / `mark-in-sync`. The channel-marketplace join id (`amazonChannelMarketplaceId`) is only needed where the *existing* publish/sync calls already use it — and at those call sites it's already in scope.

## F2. Orchestration hook — `usePublishChanges`

Single hook so the branch logic lives in one place. Roughly:

```ts
publishChanges({
  channelId,
  channelType,                 // 'AMAZON' | 'SHOPIFY'
  channelName, channelLogo,    // for the dialog header (F3)
  amazonMarketplaceId,         // optional; only when a marketplace filter is applied
  products,                    // Record<productId, variantId[]>  (same map convention as BE)
  hasNotPublished,             // channels-tab routing (see below)
  onStatusFlipped,             // (products) => void — caller flips local OUT_OF_SYNC -> SUCCESS
})
```

Flow:

1. `POST get-change-counts` with `{ products, channelId, amazonMarketplaceId }`.
2. **If array empty** (no real deltas):
   - **Channels-tab NOT_PUBLISHED case** — if `hasNotPublished` (≥1 enabled entity is `NOT_PUBLISHED`), call the existing `sync-product-to-channel` (real publish, unchanged) instead. **Not** `mark-in-sync`.
   - **Otherwise** → `POST mark-in-sync` with the same `{ products, channelId, amazonMarketplaceId }`. On success: **info toast + `onStatusFlipped(products)`** (flip local status `OUT_OF_SYNC` → `SUCCESS`; leave `NOT_PUBLISHED` untouched). **No task manager opened** — `mark-in-sync` is instant and does no external work.
3. **Else** (deltas exist) → open the shared **Change Review Dialog** (F3), seeded with the returned product(s), first product selected, and `channelType` / `channelName` / `amazonMarketplaceId` context. The dialog fetches `get-change-summary` per product.

`mark-in-sync` is therefore invoked **only** when the entities are `OUT_OF_SYNC` but no actual change surfaces, and either "Publish Changes" was chosen or every entity is already published.

## F3. Sync Change Review Dialog (adapted from `delta-sync-review.html`)

**Shared, global, single-source-of-truth component** — mounted once (like [`TaskManager`](../../../webapp/src/components/task-manager/task-manager.tsx) in [`(menu-layout)/layout.tsx`](../../../webapp/src/app/(index)/(menu-layout)/layout.tsx)) and opened via a Zustand store (mirroring `useGlobalStore.openTaskManager` / `setOpenTaskManager`). All three entry points push their context into the store; the dialog owns all selection/cascade state.

> **Deviate from the prototype where functionality requires it — don't copy it 1:1.** Concrete deltas from the prototype below.

### Layout
- **Top bar:** title + subtitle. **Remove the prototype's `Revert selected` / `Revert whole product` buttons entirely** (out of scope). Keep only the primary **`Sync selected`** = Submit.
- **Channel is fixed & single** (always known before opening) — **remove the prototype's channel switcher**. Instead show the **current channel name + logo** top-left (same channel chip style used elsewhere in the app).
- **Marketplace switcher stays for Amazon only** — chips to switch the *view*, but selections for **every** marketplace are retained in state (see Selection state).
- **Metric cards:** `Changes in Current View`, `Variants in This View`, `Selected for Sync`. **Remove `Products with Changes`.** When only one product is in scope (Variants/Channels tabs) it's meaningless.
- **Left product-list panel:** shown only for **multi-product** invocations (Products-list bulk). **Hidden entirely for single-product** invocations (Variants tab, Channels tab) — go straight to the detail view.

### Group model
Order `attributes → pricing → media → structure → variantAttributes → miscellaneous`. Amazon attribute hierarchies `purchasable_offer` / `list_price` fold into **Pricing** — the backend already returns them under `pricing.attachedAttributes`; FE just renders.

### Block / level rules (present-sync-changes §1F)
- Product-level block: Attributes, Pricing, Media, Misc (no Structure / Variant Attributes).
- Variant-level block: all six groups; Amazon variant blocks grouped by **wrapper** (`current.wrapper` in `VariantStructureResult`).
- **Media** shows a "Channel-wide" badge and its selection is **shared across marketplaces**; it does **not** re-render on marketplace switch.

### Selection units + rules

| Group | Unit granularity | Rule |
|---|---|---|
| Attributes (Amazon direct / Shopify) | one checkbox per `attributeId` | independent |
| Attributes (Amazon hierarchy) | one checkbox per `hierarchyId` root; children read-only under it | select whole root |
| Pricing | single checkbox | BE expands to `purchasable_offer`/`list_price` |
| Media | single checkbox, shared across marketplaces | channel-wide |
| Variant Structure | per-variant checkbox | cascade (below) |
| Variant Attributes | single checkbox per variant (changed attrs read-only) | pulls structure (below) |
| Misc | one checkbox per `baselineType` | independent |

### Cascade / constraint engine (mirror prototype `forcedStructureKeys` / `siblingStructureKeys` / `toggleUnit`; **backend is source of truth**)
- **Amazon structure:** selecting variant V's structure auto-selects + locks all sibling variants in V's **wrapper** (same marketplace). Deselect cascades off.
- **Shopify structure:** selecting any variant's structure selects **all** variants' structure.
- **Variant attributes → structure:** selecting V's variant-attributes, if V's structure changed, auto-selects V's structure (Amazon → wrapper siblings; Shopify → all variants' variant-attributes + structure).
- Auto-pulled units render an **"Auto"** badge and are **locked** while the dependency holds.

### Selection state (Amazon multi-marketplace)
Track selection keyed by **(productId, amazonMarketplaceId, unit)**. Switching the marketplace chip only changes the *view*; each marketplace keeps its own selection. Media is keyed by (productId, channel) only (shared). `get-change-summary` returns `MarketplaceChange[]` — one entry per marketplace when opened at all-markets scope, or a single entry when a marketplace filter was applied.

### Change-type rendering (present-sync-changes §1G)
Added/removed/updated chips for attributes; NEW/DEL + reorder for media; axis add/remove for structure. Empty scope → "No changes for {scope}".

### Submit {#f3-submit}
Build the selective payload as a **new array key** on the `publish-products-to-channel` body — an array of per-product-per-marketplace objects following the `SyncSelectionPayload` schema in [present-sync-changes.md §3](./present-sync-changes.md) (`productId`, `channelId`, `channelType`, `amazonMarketplaceId?`, `product`, `variants[]`). Emit **one payload entry per (product, marketplace)** that has any selection; **all** retained marketplace selections go into the array (not just the currently-viewed one). Media deduped channel-wide.
- On `res.data.hasVariationThemeChanged` (Amazon), show the variation-theme warning and re-submit with `ignoreWarning: true` — same pattern as [amazon-variants-core.tsx](../../../webapp/src/app/(index)/(menu-layout)/products/[product]/_components/variants/_components/nonMasterChannel/amazonChannel/amazon-variants-core.tsx) (~line 717).
- On success: `toastSuccess`, close dialog, **open the task manager** (this path does real external sync), and flip local `syncStatus` for affected entities.

> The `publish-products-to-channel` endpoint accepting this new selective array key is the agreed contract (payload shape per present-sync-changes §3). Backend acceptance of that key is outside this plan's single BE deliverable (`mark-in-sync`) and tracked separately.

## F4. Entry-point wiring

Button model:
- **`Publish to External Channel`** — always shown; existing behavior, unchanged.
- **`Publish Changes`** — added in **Variants list** and **Products list** only; shown when the relevant entity is `OUT_OF_SYNC` (bulk: ≥1 selected entity `OUT_OF_SYNC`). On click → `usePublishChanges`.
- **Channels tab** — **no new button**; the existing single **`Publish`** button just works dynamically (see F4.3).

**Status data availability (F6):** each entry point already has `syncStatus` for parent + variants at its channel/marketplace scope. **Products list is the exception:** for **non-Amazon** the parent carries an **array of statuses**; for **Amazon** it's a **singleton** per (parent/variant, marketplace). Drive both button-visibility and the optimistic local flip off that existing data — no new status fields required from list APIs.

### F4.1 Variants tab
Files: [amazon-variants-core.tsx](../../../webapp/src/app/(index)/(menu-layout)/products/[product]/_components/variants/_components/nonMasterChannel/amazonChannel/amazon-variants-core.tsx), the Shopify variant equivalent, bulk bar [selected-row-action.tsx](../../../webapp/src/app/(index)/(menu-layout)/products/[product]/_components/variants/_components/selected-row-action.tsx).
- Individual variant: `Publish Changes` if that variant `OUT_OF_SYNC`; `Publish to External Channel` always.
- Bulk: `Publish Changes` if ≥1 selected variant `OUT_OF_SYNC`.
- Builds `products = { [productId]: selectedVariantIds }` (parent never included) + `channelId` + `amazonMarketplaceId` (Amazon). Single-product dialog (left panel hidden).

### F4.2 Products list
Files: [List.tsx](../../../webapp/src/app/(index)/(menu-layout)/products/_components/list/List.tsx), [selected-row-action.tsx](../../../webapp/src/app/(index)/(menu-layout)/products/_components/list/_components/selected-row-action.tsx), [table-actions.tsx](../../../webapp/src/app/(index)/(menu-layout)/products/_components/list/_lib/header/table-actions.tsx).
- Works at **non-Amazon channel**, **Amazon selected-marketplace**, and **Amazon all-markets** levels.
- **All-markets Amazon:** `Publish Changes` when the entity is `OUT_OF_SYNC` for ≥1 marketplace (bulk: ≥1 selected entity in ≥1 marketplace). Call counts/`mark-in-sync` **without** `amazonMarketplaceId`.
- **Selected-marketplace / non-Amazon:** pass `amazonMarketplaceId` (Amazon) or omit (non-Amazon). Non-Amazon parent status is an array → `OUT_OF_SYNC` if any element is.
- Build `products` from selection: parent selected → `{ [pid]: [] }` (all variants + parent); specific variants → `{ [pid]: [vids] }` (parent excluded). Multiple products → multi-key map, dialog shows the left product panel, first selected by default.

### F4.3 Channels tab
Files: [channel-detail.tsx](../../../webapp/src/app/(index)/(menu-layout)/products/[product]/_components/channels/channel-details/channel-detail.tsx), [renderMarketplaceRows.tsx](../../../webapp/src/app/(index)/(menu-layout)/products/[product]/_components/channels/channel-details/nonMasterChannels/amazonChannel/productOverView/renderMarketplaceRows.tsx).
- Single dynamic **`Publish`** button (no label change, no extra button).
- **Disabled** when: standalone product is `SUCCESS`; or product-with-variants where *all enabled variants + parent* are `SUCCESS` for the channel/marketplace.
- On click: `get-change-counts` with `{ [productId]: [] }` + channelId (+ `amazonMarketplaceId` for a market-level button).
  - **Empty counts:** if ≥1 enabled variant/parent is `NOT_PUBLISHED` → existing `sync-product-to-channel` (unchanged); else → `mark-in-sync` with `{ [productId]: [] }` + info toast + local flip (no task manager).
  - **Non-empty counts:** dialog (single product, left panel hidden) → Submit → `publish-products-to-channel`.
- This is the entry point that sets `hasNotPublished` on `usePublishChanges`.

## F5. Cross-cutting FE conventions (webapp CLAUDE.md)
- All strings via `useTranslate()` / `translate(...)`, `Underscore_CamelCase` keys (add `Publish_Changes` etc.). No hardcoded English.
- Feedback via `toastSuccess/Info/Error` from `@/components/ui/sonner` — never `alert`.
- Gate buttons with `RenderIfPermitted` using the same permissions the current publish buttons use.
- Client-side only (`genericService`), no server actions.
- Prettier: 2-space, single quotes, ES5 trailing commas (**webapp uses spaces, not tabs** — unlike api).

---

## Build / verify order

1. **BE** `mark-in-sync` (DTO → module wiring for `ProductService` → controller → service: baseline delete + status UPDATEs in one transaction) → smoke test Amazon (all + single marketplace) and Shopify, parent-only and variant-only maps; confirm only `OUT_OF_SYNC` rows flip and stale baselines are deleted.
2. **FE F1/F2** API + `usePublishChanges` with a stub dialog → validate empty-counts → `mark-in-sync` (+ NOT_PUBLISHED → `sync-product-to-channel`) branch and local flip.
3. **FE F3** dialog: read-only render from `get-change-summary` → selection + cascade → multi-marketplace selection state → Submit.
4. **FE F4** wire the three entry points; verify button visibility per level.

---

## Resolved decisions {#resolved-decisions}

| # | Decision |
|---|---|
| B1 | Route `POST /sync-review/mark-in-sync`, **no** capability gate. |
| B2 | Amazon sync status lives **only** on `tbl_product_channel_marketplaces` / `tbl_variant_marketplaces`; never touch `tbl_product_channels` / `tbl_product_variant_channels` for Amazon. |
| B3 | When the variant array is non-empty, **never** touch parent status or parent baselines. |
| B4 | `mark-in-sync` also **deletes stale baselines** for the scope via `productService.clearSyncBaselines` (mind the string-vs-numeric marketplace-id caveat). |
| B5 | Safety comes solely from the `sync_status = OUT_OF_SYNC` guard on every UPDATE; no special all-markets handling. |
| B6 | `amazonMarketplaceId` = global `acm.amazon_marketplace_id`. |
| B7 | Wrap baseline-delete + status-flip in **one transaction** (method supports it; they must be atomic). |
| F1 | Selective publish is a **new array key** on the publish body, schema per present-sync-changes §3. |
| F2 | Use `amazonMarketplaceId` at all counts/summary/mark-in-sync call sites; `amazonChannelMarketplaceId` only where existing publish already uses it (already in scope there). |
| F3 | **Remove** `Revert selected` / `Revert whole product`. |
| F4 | Dialog is a **shared global** component (mounted like `TaskManager`), single source of truth. |
| F5 | Single-product invocations **hide the left product list** and **remove the `Products with Changes`** metric. |
| F6 | `syncStatus` already present for parent + variants at channel/marketplace scope; products-list non-Amazon parent = **array of statuses**, Amazon = singleton. |
| F7 | **No channel switcher** (single known channel; show name + logo top-left). Amazon marketplaces still switchable; **retain per-marketplace selections**; on Save all marketplace selections go into the payload. |
| F8 | `mark-in-sync` fires only when entities are `OUT_OF_SYNC` with no real changes **and** ("Publish Changes" chosen **or** everything already published) → info toast + local flip, **no task manager**. |
| F9 | Standalone (no-variant) product encoded as `{ [pid]: [] }` (parent only). |
| F10 | Add `Publish Changes` in **Variants list** + **Products list**; Channels tab keeps only its dynamic `Publish` button. |

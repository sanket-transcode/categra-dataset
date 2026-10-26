# `getChannelsSellableCount` — Explained

**File:** `api/apps/api-main/src/modules/app/catalog/products/product.service.ts`
**Lines:** `9579–9984`

## Purpose

For every active channel (or one specific channel, if `channelId` is passed), this method computes a
single row of **sync/publish health metrics** for that account — the numbers that back the "out of
sync" / channel health dashboards (`getOutOfSyncProducts`, channel overview cards, etc.). It answers,
per channel: *how many sellable items exist, how many are in sync, out of sync, blocked by an
internal error, blocked by an external (channel-side) error, never synced at all, and how many have
open issues.*

It is a single big raw SQL query (via `this.rawQuery`) built from a chain of CTEs (`WITH ... AS (...)`),
executed once per call, rather than N+1 ORM queries per channel.

## Signature

```ts
async getChannelsSellableCount({
	req,
	channelId,
	transaction,
}: GetChannelsSellableCount): Promise<ChannelSellableCounts[]>
```

- `req` — used only for `req.user.accountId` (everything is scoped to the caller's account).
- `channelId` *(optional)* — if provided, every CTE adds `AND c.id = :channelId` and the result
  collapses to a single-row array; if omitted, the query returns one row **per channel** on the account.
- `transaction` — passed through to `rawQuery` for use inside a larger DB transaction.

Returns `ChannelSellableCounts[]`, each row shaped as:

```ts
{
	channelId, sellableCount, approvedCount, inSyncCount, outOfSyncCount,
	importFailed, importedWithIssues, notSyncedCount, issueCount
}
```

## Why it's complicated: standalone products vs. variant products

A product in Categra can be **standalone** (no variants) or have **variants** (e.g. size/color). Sync
state is tracked at different granularity for each:

- Standalone product → synced via `tbl_product_channels` (+ for Amazon, `tbl_product_channel_marketplaces`).
- Variant → synced via `tbl_product_variant_channels` (+ for Amazon, `tbl_variant_marketplaces`).

Almost every CTE in this query therefore comes in a **standalone/variant pair**, and the two are later
`UNION ALL`'d together and summed.

## Why it's complicated: Amazon vs. non-Amazon (Shopify)

Amazon and non-Amazon channels track "is this item actually live/approved" differently:

- **Amazon**: a product/variant can be pushed to *multiple marketplaces* (via
  `tbl_product_channel_marketplaces` / `tbl_variant_marketplaces`), each with its own `sync_status`.
  So "in sync" for Amazon means **all** of an item's marketplaces report `SUCCESS`
  (`bool_and(COALESCE(pcm.sync_status = 'SUCCESS', FALSE))`, so a NULL status is not in sync), and "out of sync" means **any** marketplace reports
  `OUT_OF_SYNC` (`bool_or(...)`). Channel-error detection also happens per marketplace, matched via
  `tbl_channel_product_errors` filtered on `type != CATEGRA` (errors Amazon itself returned, as
  opposed to Categra's own validation errors).
- **Non-Amazon (Shopify)**: there's one `sync_status` directly on `tbl_product_channels` /
  `tbl_product_variant_channels`, so counting is a plain `GROUP BY ... sync_status`.

This is why the query has parallel CTE pairs like `standalone_amazon_per_product` /
`standalone_non_amazon_success` and `variant_amazon_per_variant` / `variant_non_amazon_success`.

## CTE-by-CTE walkthrough

1. **`standalone_products`** — count of sellable standalone products per channel (product is linked
   to the channel via an active `tbl_product_channels` row, and has no non-deleted variants).
2. **`variant_products`** — count of sellable variants per channel (product has variants, each variant
   individually linked to the channel via `tbl_product_variant_channels`).
   → `sellableCount` in the final SELECT = `standalone_products + variant_products`. This is the
   denominator: "how many items *should* be tracked for this channel."

3. **`standalone_amazon_per_product`** — one row per (channel, standalone product) on Amazon channels.
   Aggregates across all of that product's marketplaces to derive `in_sync`, `out_of_sync`, and
   `has_external_error` flags. Excludes **syncing** products: any active PCM with `is_loading` or
   `sync_in_progress` set (`HAVING NOT bool_or(...)`, i.e. any marketplace syncing → whole product syncing).
4. **`standalone_non_amazon_success`** — per (channel, sync_status) count of standalone products on
   non-Amazon channels, restricted to `sync_status IN CHANNEL_PUBLISHED_STATUSES` (`SUCCESS` or
   `OUT_OF_SYNC` — see `CONSTANTS.CHANNEL_PUBLISHED_STATUSES`, `constants.ts:176-180`; `ERROR` is
   explicitly commented out as "not being used currently") and PC not syncing (`is_loading` and
   `sync_in_progress` both false).
5. **`variant_amazon_per_variant`** — the variant-level mirror of #3, joined through
   `tbl_variant_marketplaces`. A variant is syncing if any of its PCM **or** VM rows has either flag set.
6. **`variant_non_amazon_success`** — the variant-level mirror of #4; syncing if PC **or** VC has either flag set.

**Syncing rule:** Amazon reads only PCM/VM flags; other channels read only PC/VC flags. Syncing entities
stay in `sellableCount`, drop out of every Sync bucket (In Sync / Out of Sync) and every Listing count
(Approved / Imported with Issues) and `issueCount`, so they land in `notSyncedCount`. Joins require
`is_deleted = FALSE` on P/V/PC/VC/PCM/VM and `is_active`/`status = TRUE` on PC/VC/PCM/VM/ACM.

7. **`issue_count_agg`** — counts open entries in `tbl_channel_product_errors` where
   `type != CATEGRA` (i.e., **external** errors surfaced from the channel itself, not Categra's own
   validation) that belong to a product/variant *currently known to be published* (matched via
   `EXISTS` against the `standalone_amazon_per_product` / `variant_amazon_per_variant` CTEs). This
   feeds `issueCount` — "problems on already-live listings," distinct from `importedWithIssues`.

8. **`import_failed_agg`** — counts **Categra-side** errors (`type = CATEGRA`) that are channel-level
   (`product_id IS NULL`) from imports (`action_type = IMPORT`). Counted distinctly by `error_code` (Shopify external id; task-level rows with `error_code = CATEGRA` excluded) for
   Shopify, by `sku` for other channel types (reflecting how each channel type reports these errors).
   Feeds `importFailed`.

9. **`imported_with_issues_agg`** — the more complex one. Counts items that are synced (in_sync or
   out_of_sync) and **do** have some
   (`type != CATEGRA`) channel error recorded against them (Amazon: `has_external_error`, per marketplace) — i.e., "it's live, but the
   channel rejected/flagged it." Built as a `UNION ALL` of all four combinations (standalone/variant ×
   Amazon/non-Amazon). Feeds `importedWithIssues`.

10. **`in_sync_agg`** / **`out_of_sync_agg`** — union together the `in_sync`/`out_of_sync` counts (or
    matching `sync_status`) across all four standalone/variant × Amazon/non-Amazon combinations, then
    sum per channel. Feeds `inSyncCount` / `outOfSyncCount`.

## Final SELECT — how the output columns are derived

```
channelId          = c.id
sellableCount       = standalone_products.sellable_count + variant_products.sellable_count
approvedCount       = inSyncCount + outOfSyncCount - importedWithIssues
inSyncCount         = in_sync_agg.count
outOfSyncCount      = out_of_sync_agg.count
importFailed = import_failed_agg.count
importedWithIssues = imported_with_issues_agg.count
notSyncedCount      = c.uncreated_sellable_count + sellableCount - inSyncCount - outOfSyncCount - importFailed
issueCount          = issue_count_agg.count
```

Notes:
- `approvedCount` = items that made it through sync (in or out of sync) **minus** those flagged with an
  external error — i.e., "successfully live on the channel, channel-side, right now."
- `notSyncedCount` starts from `c.uncreated_sellable_count` (a column on `tbl_channels` tracking items
  that were never even attempted) and adds the sellable items that exist but haven't landed in either
  sync bucket nor been marked with an internal error — the "still pending" pool, which includes every
  entity whose BS loader (`is_loading`) or FS flag (`sync_in_progress`) is currently set.
- Counts are pushed over the channel socket by BS per batch and by FS at publish start, per Amazon FS
  job complete/fail, per Shopify FS batch/finish/fail, and on orphaned-task cleanup
  (`ChannelService.refreshChannelSellableCounts`).
- All joins from the main `SELECT` are `LEFT JOIN`s against `tbl_channels`, so **every channel on the
  account appears in the result even if it has zero activity** (all counts `COALESCE`'d to 0).

## How it's used

- `getChannelsSellableCount` (no `channelId`) is called from:
  - `getOutOfSyncProducts` (`product.service.ts:9434`) — merges these counts with pending-webhook /
    pending-product counts per channel for the "out of sync" summary screen.
  - `channel.service.ts:1098` — channel list/overview endpoint.
- Called with a specific `channelId` (single-row result) from:
  - `amazon-product-bs.service.ts` (Amazon backward-sync) and `shopify-products-bs.service.ts`
    (Shopify backward-sync) — likely to report updated sellable/sync counts after a sync run for that
    one channel.

The raw result is shaped for the frontend via `populateChannelSellableCounts()`
(`api/apps/api-main/src/common/lib/utils.ts:982-995`), which null-coalesces every field to `0` and
passes the columns through unchanged (`importFailed`, `importedWithIssues`).

## Related sibling: `getMasterSellableCount`

Just above this method (`product.service.ts:9510-9577`) is `getMasterSellableCount`, a much simpler
version of the same "standalone + variant" counting pattern — it only returns a single sellable-item
count (optionally scoped to a channel and/or a specific list of `productIds`), with no sync-status
breakdown. Useful as a simpler reference for the same standalone/variant split logic.

# Amazon Backward Sync: SKU-only matching plan

Status: implemented on 2026-10-03 (uncommitted). Decisions taken on 2026-10-03.
Paths are relative to `api/apps/api-main/src/`. Line numbers are as of 2026-10-03.

---

## 1. Final rules

| # | Rule |
|---|---|
| **R-SKU** | A wrapper with a SKU matches on **(sku, marketplace)**. Within one product, channel and marketplace there is **at most one wrapper per SKU**. ASIN is never part of a match; it is only copied over from the latest Amazon data. |
| **R-NOSKU** | No-SKU wrappers (invisible parent `(null, asin)` and temp parent `(null, null)`) in one product and marketplace are **grouped by variation theme**. Each group is merged into its **highest-score** wrapper. Several no-SKU wrappers can therefore coexist in a marketplace, as long as their themes differ. |
| **R-MIX** | A wrapper with a SKU and a no-SKU wrapper can coexist in the same marketplace. |
| **R-KIND** | Entries in the Redis parent list match on **(sku, kind)**, where kind is standalone or variation parent. If a SKU is standalone in DE and a parent in UK, it stays as two entries. At product level they share one group, so **the variation parent wins** and the standalone listing is not attached (product SKUs are unique per account; decided 2026-10-03). |
| **R-VAR** | Variants match on **SKU only**, everywhere. |
| **R-BASE** | When comparing variant-structure baselines, `wrapper.asin` is **left out**. The stored snapshot keeps the ASIN, and it is used only when a restore has to create a wrapper. |
| **Out of scope** | No-SKU wrappers keep `sku = NULL` in the DB. Filling them with the product SKU is a separate plan, because it changes forward sync. |

Tie-break when choosing the surviving wrapper (R-NOSKU merge, and legacy duplicates under R-SKU):
**higher score → has an ASIN → more variants → already exists in the DB (`wrapperId`) → lower id.**

---

## 2. Changes by area

### 2.1 Matching helper (`amazon-product-bs.service.ts:1511`, `:1530`)
Replace `findWrapperFromList` / `findWrapperIndexFromList` with one helper:

```ts
// sku present        → match w.sku === sku (and marketplace / kind when given)
// sku null, asin set → match a no-SKU entry with the same asin   (Redis list-building only, see 2.2)
// sku null, theme    → match a no-SKU entry with the same theme   (DB matching / merging, see 2.3)
// nothing to go on   → no match
```

### 2.2 Q1, building the Redis lists (`amazon-product-bsq1.service.ts`)
| Where | Change |
|---|---|
| `addOrUpdateRedisProduct` (L3213) | Parent list: match on (sku, kind). Child list: match on sku. Update `isStandaloneProduct` only for entries of the same kind. Gather ASINs into `asinByMarketplace` (D1). |
| `removeFromRedisProductList` (L3291) | Drop `&& p.asin === asin`. |
| Invisible parents while the lists are built | Still keyed by parent ASIN, so unrelated invisible families stay apart until they are grouped. Temp parents still never match. |
| `ParentProduct` / `ChildProduct` (`core/types/sync/amazonSync.interface.ts`) | Add `asinByMarketplace: Record<string, string \| null>`. Keep `asin` as the base or first ASIN for existing readers. |
| Raw-data key `getProductKey` (`amazon-product-bs.service.ts:1131`) | Key by SKU when there is one (`AM:sku-{ch}-{task}-{sku}`), otherwise by ASIN. All 48 call sites go through this function, and the value is already split by marketplace. **Deploy note:** restart BS tasks that are running during the deploy. |

Side effect fixed: today a variant with a different ASIN per marketplace creates two child entries, and Q2 `processVariants` (`bsq2:1474`) builds two `variantData` rows for one SKU, which can create a duplicate variant. SKU-only child matching removes this.

### 2.3 Q1, `createCPList`
New order of steps for each group of related parents:

1. **Product lookup** (`wrapperConditions`, L1757): match on `(sku)` only, across **all channels** (used only to find the existing product). The product's wrappers are then loaded explicitly for the **current channel** only (D2). No-SKU wrappers are **not** used for the product lookup; variant SKUs resolve those families.
2. **Build `combinedList`** (L2028–2120):
   - Wrappers from Redis are upserted onto DB wrappers by `(sku, marketplace)`.
   - No-SKU wrappers are upserted by `(null sku, marketplace, theme)`. That is today's theme condition, now used for invisible parents too, where today they match on ASIN.
3. **Collapse duplicate SKUs:** same SKU and marketplace → one wrapper, using the tie-break. After 2.2 this only happens with old DB duplicates that the SQL clean-up hasn't removed yet.
4. **Score** (existing logic, L2144–2190). The theme dedupe at L2121 and the most-repeated-theme check at L2164 become SKU-only.
5. **Collapse no-SKU wrappers (R-NOSKU):** group no-SKU wrappers by `(marketplace, variationTheme)`, with a null theme as its own group. Merge each group into its top wrapper: combine variants (deduplicated by SKU), keep the survivor's ASIN, and record merged-away DB wrapper ids in a new CP field, `mergedWrapperIds: Array<{ from: number; to: number \| 'self' }>`.
6. **Score again**, then pick the winner and primary (L2192+). Merging changes variant counts, so the scores from step 4 are stale.
7. **Partial imports** (`productVariantsMap` set): merge a duplicate DB SKU wrapper only when the **themes are equal**. Otherwise leave it for the next full import, so variants outside the request don't change structure.

### 2.4 Q2 (`amazon-product-bsq2.service.ts`)
| Where | Change |
|---|---|
| `processWrappers` (L1336) | No matching change (it works off the CP `wrapperId`). |
| **New:** `absorbMergedWrappers` (after `processVariants`) | For each `mergedWrapperIds` entry, in the Q2 transaction: move `tbl_variant_marketplaces.wrapper_id`, including variants not in this import; move `tbl_wrapper_variant_attributes` rows whose attribute the survivor doesn't have and delete the rest; set `tbl_variant_structure_baselines.previous_meta_data.wrapper.id` to the survivor; then delete the merged wrapper. |
| `cleanupEmptyProductWrappers` (L1642) | Unchanged. The two event-scope tables are stale and will be dropped, so they are deliberately not handled. |

Q3 to Q6 use `wrapperId` and raw-data keys only, so they are covered by 2.2 (`getProductKey`) and need no other change.

### 2.5 Other places that match wrappers
| Where | Change |
|---|---|
| `sync/productSyncing/productStructureManagement/product-structure-management.service.ts:1959–1980` | Use the same helper: SKU wrappers on `(sku, marketplace)`, no-SKU wrappers on `(marketplace, theme)`. Otherwise merging two products re-creates duplicates. |
| `platforms/amazon/channel-config/product-type/amazon-channel-product-type.service.ts:1853`, `:2050` | **Check:** it creates a wrapper with the product SKU. Make sure it first reuses an existing wrapper with that SKU in the marketplace (R-SKU). |
| `catalog/products/listings/productSearchByAsin/amazon-product-cbaq2.service.ts` | Deletes and recreates wrappers. No matching change. |

### 2.6 Variant-structure baseline (`sync/baselines/variant-structure-sync-baseline/variant-structure-baseline.service.ts`)
| Where | Change |
|---|---|
| L473 `isMetaDataChanged`, L481 `isCurrentMetaDataSameAsBaseline` | Compare `toStructureMatchKey(metaData)`, which is the metadata with `wrapper.asin` removed. |
| Stored snapshot / upsert (L204–264) | Unchanged; the ASIN stays in `previous_meta_data`. |
| Restore (L726–760) | Find the wrapper by `(product, channel, marketplace, sku)` for a SKU baseline, or `(…, sku null, theme)` for a no-SKU baseline. If it is found with a different theme, reset it to the baseline theme (D3). Create a wrapper, with the **baseline ASIN**, only when none is found. |
| Producers (`product.service.ts:22173/22292`, `product-variants-flow.service.ts:6573`, `amazon-channel-product-type.service.ts:2128`) | Unchanged; they keep capturing the ASIN. |

---

## 3. Case matrix

### Within Redis (one family, one marketplace)
| # | Input | Result |
|---|---|---|
| R1 | `(P1, A)` + `(P1, B)` | One wrapper P1, with the ASIN for each marketplace from `asinByMarketplace` |
| R2 | `P1` + `P2` | Two wrappers |
| R3 | `P1` + `(null, B0X)` | Coexist (R-MIX) |
| R4 | `(null, B0X, theme T)` + `(null, B0Y, theme T)` | Merged into the higher score; B0X is kept if ahead on the tie-break |
| R5 | `(null, B0X, T)` + `(null, null, T)` | Merged; the invisible parent (has an ASIN) beats a temp parent with the same score |
| R6 | `(null, B0X, T1)` + `(null, B0Y, T2)` | Two no-SKU wrappers (different themes) |
| R7 | Standalone P1, DE `A` / UK `B` | One standalone entry with per-marketplace ASINs |
| R8 | P1 standalone in DE and a parent in UK | Two Redis entries (R-KIND); one product group, variation parent wins |
| R9 | Standalone P1 is a variant of another parent | Unchanged (enrich Task 2 already matches by SKU) |

### Redis ↔ DB (same marketplace)
| # | Case | Before | After |
|---|---|---|---|
| M1 | `P1(A)` ↔ DB `P1(A)` | match | match |
| M2 | `P1(A)` ↔ DB `P1(B)` | new wrapper; the old one is cleaned up | **match**, DB ASIN updated |
| M3 | `(null, B0X, T)` ↔ DB `(null, B0X, T)` | match | match |
| M4 | `(null, B0Y, T)` ↔ DB `(null, B0X, T)` | new wrapper | **match**, ASIN updated |
| M5 | `(null, B0Y, T2)` ↔ DB `(null, B0X, T1)` | new wrapper | new wrapper (different theme group) |
| M6 | `(null, null, T)` ↔ DB `(null, B0X, T)` | match | match |
| M7 | `P1` ↔ DB no-SKU wrapper | none | none (coexist) |
| M8 | DB has two `P1` in one marketplace (old data) | — | Collapsed in step 3; DB rows cleaned by `absorbMergedWrappers` or the SQL clean-up |
| M9 | Two DB no-SKU wrappers with the same theme | — | Collapsed in step 5 |

### Baseline
| # | Case | Result |
|---|---|---|
| B1 | BS changes a wrapper's ASIN (M2/M4), then a structure trigger fires | No false out-of-sync |
| B2 | A baseline points at a merged-away wrapper id | `absorbMergedWrappers` or the SQL clean-up rewrites `wrapper.id`, so the comparison still works |
| B3 | Restore `P1/T`, but DB has `P1/T2` | Reuse P1 and reset it to T (D3) |
| B4 | Restore `(null, T)`, but no DB no-SKU wrapper with theme T | Create one with the baseline ASIN |

---

## 4. One-time SQL clean-up (`AMAZON_WRAPPER_DEDUPE.sql` at the monorepo root; no migration, no unique index)

A plain SQL script with a read-only preview, the merge in one transaction, and a read-only report:

1. **SKU duplicates:** group by `(product_id, channel_id, amazon_marketplace_id, sku)` where `sku IS NOT NULL`, having more than one row.
2. **No-SKU duplicates:** group by `(product_id, channel_id, amazon_marketplace_id, COALESCE(amazon_current_variation_theme, ''))` where `sku IS NULL`, having more than one row.
3. **Survivor:** `score DESC, (asin IS NOT NULL) DESC, active VM count DESC, id ASC`.
4. Repoint, as in `absorbMergedWrappers`: variant marketplaces; wrapper variant attributes (move rows the survivor lacks, delete the rest); `jsonb_set` on the structure baselines' `previous_meta_data.wrapper.id`. The survivor takes a loser's ASIN when it has none. The two stale event-scope tables are not handled.
5. Delete the losing wrappers.
6. **Report only:** the same SKU used by more than one product in the same channel and marketplace. Products can't be merged in SQL; the next BS structural merge (`resolveStructuralIssues`) handles them.
7. Irreversible once committed; run the preview first.

Without an index, the code (2.2–2.5) is the only thing that stops new duplicates.

---

## 5. Defaults applied (not asked)
- **D1:** per-marketplace ASIN map, and raw-data keys by SKU (2.2).
- **D2:** the product lookup searches wrappers on every channel (only to find `commonProductId`); the wrappers used for matching and merging are loaded for the current channel only.
- **D3:** restore resets the theme to the baseline's.
- **D4:** only `wrapper.asin` is left out of baseline comparison. `score` is still compared, and BS rewrites it on every sync, so some baselines may never clear on their own. That is an existing issue and not part of this plan.

---

## 6. Implementation order
1. Matching helper + Q1 Redis lists (2.1, 2.2), including the `getProductKey` change.
2. `createCPList` steps 1–7 (2.3).
3. Q2 `absorbMergedWrappers` (2.4).
4. Structure merge + product-type reuse (2.5).
5. Baseline match key + restore (2.6).
6. SQL clean-up (4). Run it after steps 1–5 are deployed, so new duplicates aren't created while it runs.

## 7. Tests (colocated `*.spec.ts`)
- Helper: R1–R8 inputs give the expected entries (kind separation, temp parents never match).
- `createCPList`: M1–M9 give the expected combined list; R-NOSKU groups by theme and keeps the top wrapper by tie-break; the partial-import guard (step 7).
- Child list: a variant with a different ASIN per marketplace gives one child entry and one `variantData`.
- `absorbMergedWrappers`: VMs, WVA rows and baseline `wrapper.id` all move to the survivor, then the merged wrapper is deleted.
- Baseline: an ASIN-only change leaves the variant in sync; restore cases B3 and B4.
- SQL clean-up: fixture with SKU duplicates and no-SKU duplicates (same theme vs different themes) gives one survivor per group, with all references repointed.

Shorthand used below:
- Parent entry = { sku, asin, marketplaces[], variationTheme, variants[], isStandaloneProduct, productType, productName }
- Invisible parent = a parent with no SKU (sku: null) but a known ASIN
- Temp parent = a parent with sku: null, asin: null

---

Phase 0: The upsert rule everything goes through

Every write to the parent or child list goes through addOrUpdateRedisProduct (amazon-product-bsq1.service.ts:3236), which uses findWrapperFromList (amazon-product-bs.service.ts:1511) to find an existing entry:

- It matches on SKU + ASIN together, where a null value only matches null. It ignores marketplace, so one entry gathers marketplaces[] across marketplaces.
- If both SKU and ASIN are null, it never matches. Every temp parent is a new entry.
- On a match, it merges variants (deduplicated by SKU), merges marketplaces, and overwrites variationTheme, productType and productName when the new value is present.

So (P1, B0X) and (P1, B0Y) become two separate parent entries. Phase 2 merges them back together.

---

Phase 1: Sorting each listing (getAmazonProductsRecursively, ~L990–1152)

For each Listings API item in each scoped marketplace, the code reads the listing relationships, the catalog relationships and the variation theme. The theme comes from relationships[0].variationTheme.theme, falling back to the variation_theme attribute. Each item then falls into one of these cases:

┌─────────────┬───────────────────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│    Case     │                           Condition                           │                                                      Result                                                       │
├─────────────┼───────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Standalone  │ no variation theme                                            │ parent entry (sku, asin), isStandaloneProduct: true, no variants                                                  │
├─────────────┼───────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Real parent │ has a theme, plus childSkus (listing) or childAsins (catalog) │ parent entry (sku, asin) with variants: []. Children add themselves on their own turn.                            │
├─────────────┼───────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Child       │ has a theme, no children                                      │ resolves its parent as described below, adds itself as a variant of that parent, and also goes into childProducts │
└─────────────┴───────────────────────────────────────────────────────────────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

How a child resolves its parent (L1048–1136)

1. Collect candidates: parentSku from listingRelationships[0].parentSkus[0] and parentASIN from catalogRelationships[0].parentAsins[0].
2. Same-SKU guard: if parentSku === sku, the parent SKU is ignored and the case is logged to sameParentChildSKUs. Some forward-synced listings report themselves as their own parent.
3. Check the parent ASIN when a valid parent SKU exists. The ASIN the child's catalog reports is sometimes wrong, so:
   - If a parent with that SKU is already in parentProducts for this marketplace, its ASIN is used.
   - Otherwise it calls getProductBySku(parentSku) and uses the parent's own summaries[0].asin. If that call fails, it falls back to the ASIN the child reported (parentASIN).
4. No parent SKU but a parent ASIN: this is an invisible parent. The ASIN is added to invisibleWrappers (one entry per ASIN, with the marketplaces it appears in).
5. No parent SKU and no parent ASIN: this is an edge case (edgeCaseProducts). The parent becomes a temp parent (null, null).
6. Upsert the parent as (validParentSku | null, parentASIN | null) with variants: [{ sku, asin }], then upsert the child into childProducts.

Fetching data for invisible parents (processInvisibleWrappersRecursively, L1218)

After the listing pages for a marketplace are done, the invisible-parent ASINs are fetched from the Catalog API in batches of 20. Each one gets its raw data stored under the product key (null, asin). It is then upserted again into parentProducts as (null, asin), which merges into the entry the children created and adds productType and productName.

Retry imports (specificImport.crawlRelationships)

For a retry of specific SKUs, the first pass collects every related parentSkus and childSkus from the listing relationships. A second pass then fetches those SKUs (crawlRelationships: false), so the whole family gets resolved and not just the SKUs that were asked for.

---

Phase 1b: Fixing the structure

enrichParentChildStructure (L1395)

This makes two corrections where marketplaces disagree with each other:

- Task 1, temp parent that is really a parent: a temp parent (null, null) has exactly one variant, and that variant's SKU is a parent SKU somewhere else (usually in another marketplace).
  - That SKU is promoted to a parent entry (variant.sku, variant.asin) with isEnriched: true.
  - The temp parent is removed (by index), and the SKU is removed from childProducts for those marketplaces.
- Task 2, standalone that is a variant elsewhere: a standalone (sku, asin) appears as a variant of another parent.
  - A new temp parent (null, null) is created with that SKU as its only variant and the other parent's variationTheme.
  - The SKU is moved into childProducts, and the standalone parent entry is removed.

populateMissingParents (L1495)

Some non-standalone parents have a SKU but no raw data in Redis, because only their children mentioned them and their own listing never came back in the pages. These are collected per marketplace, recorded under MISSING_PRODUCTS in the sync-tracking key, and fetched by SKU.

The fetch uses preventRelationshipsEvaluation: true, so it only stores raw data. It does not change the parent or child lists. skuDataMap provides the known ASIN as a fallback.

removeUnlistedSellableProducts (L1592)

- Drops variants that are missing either a valid SKU or an ASIN (isListedAmazonSellableProduct).
- Drops non-standalone parents that have no variants left. The parent itself does not need a SKU or ASIN; it only needs sellable children.
- Drops standalones that are missing a SKU or ASIN.

sortParentsByMostVariants then puts the biggest families first, so they claim their related parents in Phase 2 before the smaller fragments do. The lists are saved to Redis.

---

Phase 2: Grouping parents into one product (createCPList, L1622)

For each parent that has not been processed yet (!isProcessed):

2a. Finding the related parents (L1697–1748)

This only runs for non-standalone parents. A parent counts as "identical" if it matches in one of three rounds:

1. It has the same parent SKU, or shares at least one variant SKU, with the current parent.
2. It has the same SKU as a parent already found in round 1.
3. It shares a variant SKU with any parent found so far.

This is how the split (P1, B0X) / (P1, B0Y) entries, invisible parents and temp parents rejoin their real family. Each related parent is then marked isProcessed, so it isn't handled twice.

2b. groupParents (amazon-product-bs.service.ts:4511)

The group is bucketed by productType|variationTheme|marketplaces:

- Parents without a productName are merged into the named listing in their bucket. This applies to children-only references and to missing parents whose data was never fetched.
- If there is no named listing, temp parents (null, null) in the same bucket are combined into one.
- Any parent left with more than one marketplace is split into one parent per marketplace (wrapperObjects).

2c. Matching the group to a Categra product (L1784–1993)

Three parallel queries look for product IDs:
- Wrapper rows matching any (sku, asin) pair in the group
- variant-level SKU attributes matching any variant SKU
- product-level SKU attributes matching any parent SKU

The outcome depends on how many product IDs come back:
- 0: a new product.
- 1: that product becomes commonProductId.
- More than 1: a structural conflict.
  - If a backward sync is already running on those products and a full import is in progress, the group is skipped and the products are deferred (deferredProductIds). The task is then queued again.
  - Otherwise resolveStructuralIssues merges them into targetProductId. Variant SKUs from products that fail to merge become conflitingSKUs and are left out of every parent.

2d. Choosing the primary and winner parent (L2036–2357)

combinedList is built from two sources:
- The product's existing DB wrappers, marked isOldWrapper and isActiveWrapper. They keep only the variants that are not being re-imported now.
- The Redis parents.
  - A Redis parent with a SKU or ASIN is matched to a DB wrapper by (sku, asin, marketplace). A temp parent is matched by marketplace and variation theme.
  - On a match it updates that wrapper; otherwise it is added as a new one.

Each parent is then scored:

┌─────────────────────────────────────────────────────────┬───────────────────────────┐
│                        Criterion                        │          Points           │
├─────────────────────────────────────────────────────────┼───────────────────────────┤
│ Has a SKU (required for the points below)               │ 5 per variant             │
├─────────────────────────────────────────────────────────┼───────────────────────────┤
│ Uses the most common variation theme                    │ +10                       │
├─────────────────────────────────────────────────────────┼───────────────────────────┤
│ Is in an active marketplace                             │ +7                        │
├─────────────────────────────────────────────────────────┼───────────────────────────┤
│ Its marketplace language is the account's base language │ +7                        │
├─────────────────────────────────────────────────────────┼───────────────────────────┤
│ No SKU, but has an ASIN                                 │ 5 (in place of the above) │
└─────────────────────────────────────────────────────────┴───────────────────────────┘

From the scores:
- The winner is the highest score. Its raw summaries and attributes become the source data, and its marketplace is sourceMarketplace.
- The primary is the existing DB wrapper if there is one, otherwise the winner. It supplies the CP item's sku and asin.
- Per-marketplace primary: after sorting by score, the first wrapper in each marketplace gets isPrimaryWrapper.
- DB wrappers that are not the winner and have metadata are added to cleanupWrappers.

The CP item is created only when:
- it belongs to a marketplace in scope,
- it was not deferred,
- it has active wrappers, and
- it is standalone or has active variants.

---

Things I noticed while tracing

1. The conflict check never runs (L2296). requestedVariantSKUs.length is read on a Record, which has no length, so the value is always undefined. The PARTIAL_VARIANT_STRUCTURE_IMPORT_CONFLICT check therefore never fires.
2. The "primary is the active DB wrapper" rule takes the first DB wrapper (L2228–2229). Every DB wrapper is pushed with isActiveWrapper: true, so findIndex returns whichever DB wrapper comes first, not the one that is actually active.
3. Invisible-parent fetching can drop batches (L1269–1274, L1366). If any batch returns no items, the return exits all remaining batches. A nextToken restarts the whole loop from batch 0.
4. Grouping is not fully transitive (L1718–1745). Rounds 2 and 3 run once each, in index order. A chain that is linked only through a parent found late in the same round can be missed.
5. groupParents changes the original parent objects. It pushes variants into validListing.variants, and that is the same object as the parentProducts entry that is saved to Redis at L2441. Variants can therefore be duplicated in the saved list.
6. Wrong value in a log (L1089). receivedParentASIN: asin logs the child's ASIN, not the parent ASIN the child reported.
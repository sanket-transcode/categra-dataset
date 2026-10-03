# Amazon Backward Sync: Parent Resolution Fixes (open items)

File: `api/apps/api-main/src/modules/app/sync/productSyncing/amazon/backwardSync/amazon-product-bsq1.service.ts`, inside `createCPList`.
Line numbers are as of 2026-10-02. Status: #1, #2, #4 implemented on 2026-10-03 (uncommitted); the wrapper re-fetch in #1 was intentionally kept.

Already fixed separately: the wrong `receivedParentASIN` log value, batches dropped by `processInvisibleWrappers`, and `groupParents` mutating the Redis parent list.

Suggested order: **#4, then #2, then #1**. Each one can change the CP list for existing products, so compare the CP list (`cpKey`) before and after on a real channel.

---

## #1: The partial-import conflict check never runs

**Where:** L2281–2327

**Problem**
- `requestedVariantSKUs` is a `Record<string, number>`, so `requestedVariantSKUs.length` is always `undefined`. The whole `PARTIAL_VARIANT_STRUCTURE_IMPORT_CONFLICT` block is skipped.
- Inside the block, `attriutes` is misspelled twice (L2292, L2300). Sequelize ignores it and loads every column.
- The block fetches wrappers for `commonProductId` again, although they were already loaded into `commonProductWrappers` at L1982–2002.
- If it did run, `throw new UnrecoverableError(...)` inside the per-parent loop would fail the **whole** fetch job, so one conflicting product would block every other product in the import.

**Suggested solution**
1. Change the condition to `commonProductId && Object.keys(requestedVariantSKUs).length`. Build `const requestedIds = new Set(Object.values(requestedVariantSKUs))` once.
2. Remove the second `findAll`:
   - Add `status` to the `VariantMarketplace` attributes of the first wrapper query (L1997: `['id', 'variantId', 'asin', 'status']`).
   - Filter `commonProductWrappers` in memory by `channelId === channel.id` and `vm.status`.
3. Only report a conflict when **both** themes are present and differ (`aw.variationTheme && dbW.amazonCurrentVariationTheme && aw.variationTheme !== dbW.amazonCurrentVariationTheme`), **and** the DB wrapper has an active variant that is not in `requestedIds`. A temp parent with a null theme must not count as a conflict.
4. **Don't throw from inside the loop.** Record the conflict as a failure on that one product, using the per-product failure path Q1 already has (for example the `FAILED_KEYS_KEY` tracking). Then mark its identical parents `isProcessed` and `continue`.

```ts
const requestedIds = new Set(Object.values(requestedVariantSKUs));

if (commonProductId && requestedIds.size) {
	const hasConflict = activeWrappers.some((aw) => {
		const dbWrapper =
			aw.wrapperId &&
			commonProductWrappers.find((w) => Number(w.id) === Number(aw.wrapperId) && w.channelId === channel.id);

		if (!dbWrapper || !aw.variationTheme || !dbWrapper.amazonCurrentVariationTheme) return false;
		if (aw.variationTheme === dbWrapper.amazonCurrentVariationTheme) return false;

		return dbWrapper.variantMarketplaces.some((vm) => vm.status && !requestedIds.has(Number(vm.variantId)));
	});

	if (hasConflict) {
		// record PARTIAL_VARIANT_STRUCTURE_IMPORT_CONFLICT for commonProductId, mark identical parents processed, continue
	}
}
```

**Risk:** this turns on a check that has never actually run. Partial imports that pass today will start reporting conflicts. Tell QA before releasing.

---

## #2: The primary wrapper is just the first DB wrapper

**Where:** L2070 (`isActiveWrapper: true` on every DB wrapper), L2214 (`findIndex(item => item.isActiveWrapper)`)

**Problem**
- The `Wrapper` entity has no "active" column, and every DB wrapper is pushed into `combinedList` with `isActiveWrapper: true`.
- So when a product already exists, the primary wrapper (which becomes the CP item's `sku`/`asin`) is whichever DB wrapper Postgres returns first.
- The parent SKU of an existing product can therefore change between syncs, depending on row order.

**Suggested solution**

The real identity of an existing product is its **product-level SKU attribute**. It is already loaded: `productSKUs` comes from `getProductSKUs` (L2001), and the entry with `variantId` null is the product-level SKU.

1. Pick the primary in this order:
   1. The DB wrapper whose `sku` equals the product-level SKU. If several marketplaces have it, prefer the base-language marketplace, then an active marketplace.
   2. Otherwise, the DB wrapper with the highest score from this run.
   3. Otherwise, the winner (`highestScoreWrapperIdx`). This is the current behaviour for new products.
2. Push DB wrappers with `isActiveWrapper: false`. Set `isActiveWrapper = true` only on the chosen index, so the flag that goes into `activeWrappers` means what it says.
3. Remove `isActiveWrapperPresent`.

```ts
const productLevelSku = productSKUs.find((p) => !p.variantId)?.sku || null;

const pickPrimary = (): number => {
	const dbIdxs = combinedList.map((w, idx) => (w.isOldWrapper || w.wrapperId ? idx : -1)).filter((idx) => idx !== -1);
	if (!dbIdxs.length) return highestScoreWrapperIdx;

	const skuMatches = productLevelSku ? dbIdxs.filter((idx) => combinedList[idx].sku === productLevelSku) : [];
	const pool = skuMatches.length ? skuMatches : dbIdxs;

	return pool.reduce((best, idx) => (rank(combinedList[idx]) > rank(combinedList[best]) ? idx : best), pool[0]);
	// rank = base-language marketplace > active marketplace > score
};
```

Note: `isOldWrapper` is set to `false` for DB wrappers that get re-matched by a Redis wrapper (L2108). Use `wrapperId` to tell whether a wrapper exists in the DB.

**Risk:** the CP `sku` can change for existing products whose current first wrapper isn't their product-level SKU. Check `replaceTempSKUWithWinnerSKU` and other places that use `cp.sku` (`amazon-product-bs.service.ts`).

---

## #4: Grouping related parents isn't fully transitive

**Where:** L1681–1733 (the three "identical parents" rounds), L2401 (`isProcessed`)

**Problem**
- Rounds 2 and 3 each run **once**, in index order. A family linked only through a parent that is found later in the same round (A↔B↔C↔D) can be split into separate CP items. That produces duplicate products, or a structural issue on the next sync.
- Each round rescans the whole parent list with nested `.some`, once per parent. The cost grows roughly with the square of the parent count times the variant count, which is slow for large catalogues.

**Suggested solution:** compute every group once, before the loop, with union-find (disjoint sets).

1. Build two indexes in one pass:
   - `parentSku → idx[]` (including standalones, which matches the current SKU rule)
   - `variantSku → idx[]` (standalones have no variants, so they still never match here)
2. `union` every index list in both maps.
3. Group the indexes by `find(idx)`. Order the groups by their **lowest index**, so the "most variants first" sort (`sortParentsByMostVariants`) still decides who leads.
4. Replace `isProcessed` and the three `reduce`/`forEach` rounds with `for (const group of groups)`. The loop body uses `group[0]` as `parentProduct` and `group.map((i) => parentProducts[i])` as the identical parents.

```ts
const parentIdx = parentProducts.map((_, i) => i);
const find = (i: number): number => (parentIdx[i] === i ? i : (parentIdx[i] = find(parentIdx[i])));
const union = (a: number, b: number) => { const ra = find(a), rb = find(b); if (ra !== rb) parentIdx[Math.max(ra, rb)] = Math.min(ra, rb); };

const bySku = new Map<string, number[]>();
const byVariantSku = new Map<string, number[]>();

parentProducts.forEach((p, i) => {
	if (p.sku) bySku.set(p.sku, [...(bySku.get(p.sku) || []), i]);
	if (!p.isStandaloneProduct) p.variants.forEach((v) => v.sku && byVariantSku.set(v.sku, [...(byVariantSku.get(v.sku) || []), i]));
});

for (const idxs of [...bySku.values(), ...byVariantSku.values()]) idxs.slice(1).forEach((i) => union(idxs[0], i));

const groups = new Map<number, number[]>();
parentProducts.forEach((_, i) => { const r = find(i); groups.set(r, [...(groups.get(r) || []), i]); });
// iterate groups in ascending root order (root = lowest index, because union keeps the min)
```

(Union by minimum index keeps the root as the lowest index, which keeps the "biggest family first" order.)

**Risk:** with full transitivity, one wrong relationship from Amazon now joins two whole families, where today it only pulls in one extra level. That fits the intent (a variant SKU is unique per seller, and step 2c merges products anyway), but make it visible: `consoleTrack` any group with more than one distinct parent SKU **and** more than one `productType`.

---

## Tests to add (colocated `*.spec.ts`)

- **#1:** a conflicting theme together with a variant that isn't requested reports a failure for that product only. The other products in the batch still produce CP items.
- **#2:** the result is the same when the DB wrappers come back in shuffled order. An existing product keeps its product-level SKU as the CP `sku`.
- **#4:** a chain A↔B (shared SKU), B↔C (shared variant), C↔D (shared variant) where D sits at a lower index than C ends up in one group. Unrelated families stay separate.

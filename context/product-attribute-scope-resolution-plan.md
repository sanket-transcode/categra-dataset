# Product attribute scope resolution — consistent structure plan

Status: **implemented 2026-09-18** (API + webapp). Decisions on the former open points are recorded in §8.

Covers `getProductAttributesWithRawQuery` / `getEnrichedProductAttributes` (primary attributes, master group attributes, Amazon group attributes) and every other place that re-implements the same value ladder.

---

## 1. Vocabulary

| Term | Meaning |
|---|---|
| **Scope** | `product id + variant id (or null) + channel id (or null) + language code` |
| **Level** | `Parent` (variant id null) or `Variant` (variant id set) |
| **Channel** | `Master` (channel id null) or `Channel` (channel id set, e.g. Amazon) |
| **Language** | `Base` (language == account base language) or `NonBase` |
| **Entry scope** | The scope the caller arrives from (the URL / tab the user is looking at) |
| **Tier** | One of the up-to-6 scopes that contribute a value to a row: `self`, `fallbackValues`, `parentValues`, `parentFallbackValues`, `baseValues`, `parentBaseValues` |
| **Root scope** | `Parent + Master + Base` — the only scope that has nothing above it |

Tier → scope mapping (this never changes, regardless of entry scope):

| Tier key | Level | Channel | Language |
|---|---|---|---|
| *(row itself)* | entry | entry | entry |
| `fallbackValues` | entry | Master | entry |
| `parentValues` | Parent | entry | entry |
| `parentFallbackValues` | Parent | Master | entry |
| `baseValues` | entry | Master | Base |
| `parentBaseValues` | Parent | Master | Base |

Note there is deliberately **no** `Channel + Base language` tier. A channel scope in a non-base language falls to Master (same language) before it falls to the base language. This matches every existing ladder in the codebase (`pickProductAttributeValue`, `pickAttributeValue`, readiness resolver, product-list SQL).

---

## 2. Quick conditions (the whole rule set in 5 lines)

1. **`fallback*` keys exist ⇔ entry channel is non-master.** (Master has nothing to fall back to at the same language.)
2. **`base*` keys exist ⇔ entry language ≠ base language.** (Base language has nothing to fall back to at the same channel.)
3. **`parent*` keys exist ⇔ entry level is Variant AND `includeParentValues` is true.** (Primary attributes never inherit from the parent product — see `isInheritFromParentAllowed`.)
4. **Key presence depends only on the entry scope, never on whether a DB row exists.** Missing rows produce the key with every value field `null` and `isInherited: null`, `id: null`.
5. **A tier is never omitted because another tier already answered.** The response is a flat snapshot of all tiers; *choosing* between them is the consumer's job (see §6).

Derived: `parentFallbackValues` ⇔ (1) ∧ (3). `parentBaseValues` ⇔ (2) ∧ (3).

---

## 3. Entry-scope matrix (all 8 combinations)

`includeParentValues = true` (group attributes, master + Amazon):

| # | Entry scope | Row keys always present | Keys that must NOT be present |
|---|---|---|---|
| 1 | Parent + Master + Base | *(own values only)* | fallbackValues, parentValues, parentFallbackValues, baseValues, parentBaseValues |
| 2 | Parent + Master + NonBase | baseValues | fallbackValues, parentValues, parentFallbackValues, parentBaseValues |
| 3 | Parent + Channel + Base | fallbackValues | parentValues, parentFallbackValues, baseValues, parentBaseValues |
| 4 | Parent + Channel + NonBase | fallbackValues, baseValues | parentValues, parentFallbackValues, parentBaseValues |
| 5 | Variant + Master + Base | parentValues | fallbackValues, parentFallbackValues, baseValues, parentBaseValues |
| 6 | Variant + Master + NonBase | baseValues, parentValues, parentBaseValues | fallbackValues, parentFallbackValues |
| 7 | Variant + Channel + Base | fallbackValues, parentValues, parentFallbackValues | baseValues, parentBaseValues |
| 8 | Variant + Channel + NonBase | fallbackValues, parentValues, parentFallbackValues, baseValues, parentBaseValues | — |

`includeParentValues = false` (primary attributes / product structure): same table with every `parent*` key removed, i.e. rows 5–8 collapse to the key sets of rows 1–4.

Row 8 is the reported bug: today, when there is no `V + Channel + NonBase` row **and** no `V + Master + NonBase` row, the JS merge synthesises the row from the base query and emits only `baseValues`/`parentBaseValues`, dropping `fallbackValues`/`parentFallbackValues` (see `utils.ts:700-712`, the `? :` branch). Under rule 4 all five keys are present, with the two fallback ones null-filled.

Resolution ladder order (per row, top → bottom), used by every consumer in §6:

```
self → fallbackValues → parentValues → parentFallbackValues → baseValues → parentBaseValues
```

Tiers not present for the entry scope are simply skipped; the ladder is the same list for all 8 entries.

---

## 4. Why the current structure collapses

1. **Anchored on the entry scope.** The main `SELECT … FROM tbl_product_attributes pa WHERE <entry scope>` only yields attributes that have a row at the entry scope. Everything else is invisible to the SQL, so the tier `LEFT JOIN`s never get a chance to fill it.
2. **JS merge re-synthesises missing rows with partial keys.** `getEnrichedProductAttributes` runs the raw query 2–3 times at *different* entry scopes (channel, master, base) and stitches the results. A synthesised row gets `fallbackValues`+`parentFallbackValues` **or** `baseValues`+`parentBaseValues`, never both, and its `parentValues` is copied from the *source* scope's parent, not the entry scope's parent.
3. **Merge-of-a-merge.** Amazon group attributes and product structure run `getEnrichedProductAttributes` twice (`firstMerge` → `combinedAttributes`); the second pass sees already-synthesised rows and treats them as "current", so their key sets are frozen at whatever the first pass produced.
4. **`buildProductAttributeJsonObject` hides values by `is_inherited`.** Each tier is nulled when its row is `is_inherited = TRUE`, with a hand-tuned `keepInheritedValue` boolean per tier per scope. This bakes the *choice* of value into the fetch, so the response cannot be re-evaluated on the FE when the user flips the inherit switch, and it is the reason the merge branches differ per scope.
5. **Language replacement default.** `languageCode: languageCode || baseLanguageCode` is applied in the replacements but `languageCode !== baseLanguageCode` is evaluated on the raw argument, so an `undefined` language builds `base_attributes` CTEs for the base language itself.

---

## 5. Proposed structure (API)

### 5.1 One query, universe-first, tiers as LEFT JOINs

Replace the anchored query + JS merge with a single query shaped as:

```
WITH
  scope_rows AS (
    -- every row for the group ids that belongs to ANY tier of the entry scope
    SELECT pa.* FROM tbl_product_attributes pa
    WHERE pa.product_attribute_group_id IN (:productAttributeGroupIds)
      AND pa.account_id = :accountId
      AND (
        -- self
        (variant_id <entry> AND channel_id <entry> AND language_code = :languageCode)
        -- fallback            (only when channelId)
        OR (variant_id <entry> AND channel_id IS NULL AND language_code = :languageCode)
        -- parent              (only when variantId && includeParentValues)
        OR (variant_id IS NULL AND channel_id <entry> AND language_code = :languageCode)
        -- parentFallback      (only when channelId && variantId && includeParentValues)
        OR (variant_id IS NULL AND channel_id IS NULL AND language_code = :languageCode)
        -- base                (only when languageCode <> base)
        OR (variant_id <entry> AND channel_id IS NULL AND language_code = :baseLanguageCode)
        -- parentBase          (only when languageCode <> base && variantId && includeParentValues)
        OR (variant_id IS NULL AND channel_id IS NULL AND language_code = :baseLanguageCode)
      )
  ),
  universe AS (
    -- the row universe: one entry per (group, attribute) present at ANY tier  (decision: union of all tiers)
    SELECT DISTINCT product_attribute_group_id, attribute_id FROM scope_rows
  ),
  attr            AS ( ... unchanged: labels, translations, filtered config ... ),
  attr_with_config AS ( ... unchanged ... ),
  t_self   AS (SELECT * FROM scope_rows WHERE <self scope>),
  t_fb     AS (SELECT * FROM scope_rows WHERE <fallback scope>),         -- only emitted when the tier exists
  t_par    AS (SELECT * FROM scope_rows WHERE <parent scope>),
  t_pfb    AS (SELECT * FROM scope_rows WHERE <parentFallback scope>),
  t_base   AS (SELECT * FROM scope_rows WHERE <base scope>),
  t_pbase  AS (SELECT * FROM scope_rows WHERE <parentBase scope>)
SELECT
  u.attribute_id, u.product_attribute_group_id,
  -- row identity is ALWAYS the entry scope, even when t_self is missing  (constraint: "current scope is compulsory")
  :variantId  AS "variantId", :channelId AS "channelId", :languageCode AS "languageCode",
  s.id, s.value_group_id, s.sequence, s.is_inherited, s.<value columns>,  -- null when no self row
  (s.id IS NULL)          AS "isSynthesized",
  <tier json>(fb)   AS "fallbackValues",          -- emitted only when the tier exists for the entry scope
  <tier json>(par)  AS "parentValues",
  <tier json>(pfb)  AS "parentFallbackValues",
  <tier json>(base) AS "baseValues",
  <tier json>(pbase) AS "parentBaseValues",
  a.display_label, <attribute json>, <sequenceData json>
FROM universe u
JOIN attr_with_config a ON a.id = u.attribute_id
LEFT JOIN t_self  s     ON s.attribute_id = u.attribute_id AND s.product_attribute_group_id = u.product_attribute_group_id
LEFT JOIN t_fb    fb    ON ...
LEFT JOIN t_par   par   ON ...
LEFT JOIN t_pfb   pfb   ON ...
LEFT JOIN t_base  base  ON ...
LEFT JOIN t_pbase pbase ON ...
WHERE <bypassAttributeNames / bypassAttributePatterns on a.name>
ORDER BY s.sequence NULLS LAST, s.value_group_id NULLS FIRST, s.id ASC;
```

Key points:

- **Which tier CTEs / columns are emitted is decided by the entry scope only** (§2 rules), exactly the same booleans that decide the `scope_rows` OR-branches. A single helper `resolveTierPlan({ variantId, channelId, languageCode, baseLanguageCode, includeParentValues })` returns `{ fallback: bool, parent: bool, parentFallback: bool, base: bool, parentBase: bool }` and is the **only** place these conditions live — the SQL generator, the TS type, the FE type and the tests all derive from it.
- **`universe` is the union of all tiers.** An attribute present only at, e.g., the parent-channel tier still gets a row (decision confirmed).
- **Rows with no `self` record are stamped with the entry scope** and `isSynthesized: true`, `isInherited: null`, `id: null`. Saving from the FE creates the row at the entry scope (which today already happens through `isProductAttributeRecordExist`).
- **Tier JSON = value fields + `id` + `isInherited`**, with **no** `is_inherited`-based masking. `buildProductAttributeJsonObject` and its `keepInheritedValue` argument are deleted. The tier object shape becomes:

  ```ts
  interface ProductAttributeTierValues extends ProductAttributeValues {
    id: number | null;          // null ⇒ no row at that tier
    isInherited: boolean | null;
  }
  ```

  A missing tier row still produces the object (`JSON_BUILD_OBJECT` over a NULL-row LEFT JOIN yields all-null fields), satisfying rule 4 with zero JS post-processing.
- **Multi-value attributes (`value_group_id`).** Today the row grain is one DB row per (attribute, value group). Keep that: `t_self` is joined on `(attribute_id, group)` and tier CTEs are joined on `attribute_id` only, so each self value-group row carries the same tier objects. For synthesised rows (no self row) exactly one row per attribute is produced. For tiers that themselves hold multiple value groups, the tier object exposes the **first by `value_group_id NULLS FIRST, id`** (matches today's implicit behaviour where the LEFT JOIN would already multiply rows). *Open point, see §8.*
- `languageCode` is normalised once (`languageCode ??= baseLanguageCode`) **before** any `!==` comparison.
- `account_id` is added to `scope_rows` (currently only group ids constrain the query).

### 5.2 Delete the JS merge

`getEnrichedProductAttributes` is removed. All four call sites become a single `getProductAttributesWithRawQuery` call at the entry scope:

| Call site | Today | After |
|---|---|---|
| `getMasterProductAttributes` (per group) | 2 queries (current, base) + 1 merge | 1 query, `includeParentValues: true` |
| `getAmazonProductAttributes` | 3 queries + 2 merges | 1 query, `includeParentValues: true` |
| `getAmazonProductAttributesForAmazon` | 2 queries + 1 merge | 1 query |
| `getProductStructure` (primary attributes) | 3 queries + 2 merges (+ primary media lookup) | 1 query, `includeParentValues: false`; primary-media override logic unchanged |

`fillProductAttributeValues` / `getEmptyProductAttributeValues` stay (used elsewhere) but are no longer needed here.

### 5.3 Types

`api/src/core/types/attribute.type.ts`

```ts
export interface ProductAttributeTierValues extends Partial<ProductAttributeValues> {
  id: number | null;
  isInherited: boolean | null;
}
export interface ProductAttributeValuesChain {
  fallbackValues?: ProductAttributeTierValues;
  parentValues?: ProductAttributeTierValues;
  parentFallbackValues?: ProductAttributeTierValues;
  baseValues?: ProductAttributeTierValues;
  parentBaseValues?: ProductAttributeTierValues;
}
export interface EnrichedProductAttribute extends ProductAttribute, ProductAttributeValuesChain {
  isSynthesized: boolean;
}
```

Optional (recommended for the FE): a discriminated union keyed by entry scope so TypeScript enforces §3 — `ScopeKeys<Entry>` computed from `resolveTierPlan`.

---

## 6. Proposed value-resolution rule (shared by FE + API + readiness + SQL)

### 6.1 The rule

```
effective(row, isChecked, isBaseValue, valueKey):
  if (!isChecked || isBaseValue)           return row[valueKey]          // switch off, or root scope
  ladder = [fallbackValues, parentValues, parentFallbackValues, baseValues, parentBaseValues]
             .filter(tier => tier present on row)                        // §3 key set
  for tier in ladder except last:
    if (tier.isInherited === false)         return tier[valueKey]        // explicit override wins, EVEN IF the value is null/empty
  return last(ladder)[valueKey]                                          // terminal tier = root scope, used unconditionally
```

- `isChecked` = the UI switch (= `row.isInherited !== false` on load, see `useAttributeSwitches`), unchanged.
- `isBaseValue` = `!variantId && channel is master && language is base`, unchanged.
- **Removed:** the `isNotEmpty(x) ? x : next` missing-value bypass. A tier with `isInherited === false` and a null value is an explicit "blank override" and stops the walk.
- **Removed:** treating `isInherited` null/undefined as "not inherited" anywhere on the ladder; only `=== false` counts.
- **Terminal tier** (`parentBaseValues` when parent tiers are present, otherwise `baseValues`; if neither is present — entry rows 1, 3, 5, 7 — then the last present tier, or `null` when the ladder is empty) is used regardless of its `isInherited` flag: it is the root scope and has no inheritance concept. Confirmed decision.

### 6.2 Where the ladder is re-implemented today, and what changes

| Location | Today | Change |
|---|---|---|
| `webapp/src/lib/utils.ts` `pickProductAttributeValue` | 6-deep `isNotEmpty` ternary | Replace with §6.1. `getProductAttributeEvaluatedValue` and `useProductAttributeSwitchDetector` keep their signatures. |
| `webapp/src/lib/utils.ts` `getProductAttributeOwnValue` | own value only | unchanged |
| `webapp/src/types/attribute.types.ts` `ProductAttributeInfo` | tier keys typed as `ProductAttributeValues` | Use `ProductAttributeTierValues` (adds `id`, `isInherited`) |
| `webapp … details-footer.tsx` lines ~517-651, ~708-729 | nulls own values when `isInherited` on save | unchanged in behaviour — it operates on the row's own `isInherited`, not on tiers |
| `api … common/lib/utils.ts` `pickAttributeValue` | same 6-deep `isNotEmpty` ternary on a flat row list (`isChecked = valueRecord?.isInherited !== false`) | Build tiers from the row list with the §1 mapping, then apply §6.1. Callers: `attribute-sync-baseline.service.ts` (×10), `amazon-channel-product-type-schema.service.ts`. |
| `api … readiness/engine/readiness-attribute-value.resolver.ts` `buildLadder` / `resolve` | ladder order already identical; steps use `requireOwnValue` + `is_inherited !== true` **and** skip empty values | `requireOwnValue` steps must test `is_inherited === false` (explicit); an explicit-override row with an empty value is returned as the resolved (empty) value, not skipped. Terminal steps unchanged (`requireOwnValue: false`). |
| `api … productList/productListQueryBuilder.ts` (~L185-215 name/SKU, ~L495-515 search) | `ROW_NUMBER` ranking with `is_inherited = FALSE` for channel/master-language, plus `value_varchar IS NULL` pushed last | Drop the `value_varchar IS NULL` ordering term so an explicit blank override ranks first; keep `is_inherited = FALSE` for non-terminal ranks, no flag test on the base-language rank. Search CTE rank 2 currently uses `value_varchar IS NOT NULL` instead of `is_inherited = FALSE`, and rank 3 tests `is_inherited = FALSE` on base language — both align to the same rule. |
| `api … overview/product-overview.service.ts` (~L275-305, ~L1495-1540) | `COALESCE(channel explicit, master-language explicit, base)` | `COALESCE` skips NULLs, which is the missing-value bypass. Replace with a `CASE` that returns the first rank whose row *exists with is_inherited = FALSE*, else the base row. |
| `api … pickAttributeValue` consumers that compute `skipEmptyValues` with `isMasterBaseLanguage` (`amazon-product-cbaq2`, `amazon-product-bsq2`) | write-side | Out of the ladder; listed for awareness only. |
| `getVariantAttributeValue` / FE `pickVariantAttributeValue` / `edit-variant.tsx` / `variant-cell-renderer.tsx` / `VariantPerformance.tsx` | variant-attribute values (no inherit flag) | **Excluded** per instruction. |

A single exported constant `PRODUCT_ATTRIBUTE_TIER_ORDER = ['fallbackValues','parentValues','parentFallbackValues','baseValues','parentBaseValues']` in both repos (API `core/constants`, FE `lib/constants`) is the source of truth for the order; every implementation above iterates it rather than hard-coding the chain.

---

## 7. Verification plan (before/after, no data changes)

1. **Scope matrix test (API, unit on the SQL generator).** For each of the 8 entry scopes × `includeParentValues` ∈ {true,false}, assert the emitted column list equals §3 exactly (present and absent keys).
2. **Empty-tier test (API, integration).** Seed one attribute with a row only at `Parent + Master + Base`; query from entry #8; assert one row with all five tier keys, `isSynthesized: true`, `fallbackValues.id === null`, `parentBaseValues.id !== null`.
3. **Bug reproduction.** Seed `V + Channel + NonBase` present and `V + Master + NonBase` absent; entry #8; assert `fallbackValues` and `parentFallbackValues` keys are present (null-filled) — this is the case that fails today.
4. **Ladder equivalence.** One fixture set of rows, resolved through FE `pickProductAttributeValue`, API `pickAttributeValue`, readiness `resolve`, and the product-list SQL; assert all four return the same effective value for each of the 8 entries. Include the "explicit blank override in the middle" case.
5. **Call-site parity.** Snapshot responses of the 4 endpoints for a product with variants, Amazon channel, and a second language before/after; differences must be explained by §2 (extra null-filled keys) or §6 (blank-override semantics) only.

---

## 8. Decisions taken at implementation (formerly open points)

1. **Multi-value tiers → first row only.** A non-self tier exposes one row per (group, attribute), picked by `value_group_id NULLS FIRST, id` (`DISTINCT ON` in the tier CTEs). The self tier keeps every value-group row.
2. **No `isSynthesized` flag.** A row with no record at the entry scope has `isInherited: null` and all own values null.
3. **Row `id` for synthesized rows = `COALESCE(self.id, <first present tier>.id …)`** (same for `sequence` / `sequenceData`). The FE keys row identity on `id` in ~40 places (change detection, store updates, React keys, attachments), so `null` would collide; this keeps parity with the old merge, which carried the source-scope row id.
4. **Product-list / overview SQL ladders aligned** to "explicit blank override wins" (rows matching no ladder step are filtered out; `COALESCE` on values replaced by a `CASE` on row existence — `buildPrimaryAttributeValueLadderSql`).
5. **Root scope for `pickAttributeValue` / readiness:** `Master + base language`, and for a variant only when it cannot inherit from its parent (primary attributes) — matches FE `isPrimaryAttributeBaseValue` / `isGroupAttributeBaseValue`.

### Files changed

API: `core/constants/constants.ts` (`PRODUCT_ATTRIBUTE_TIER_ORDER`), `core/types/attribute.type.ts`, `common/lib/utils.ts` (`resolveProductAttributeTierPlan`, `getProductAttributeTierKeys`, `toProductAttributeTierValues`, `pickProductAttributeTierValue`, rewritten `pickAttributeValue`, `buildPrimaryAttributeValueLadderSql`; removed `getEnrichedProductAttributes`, `buildProductAttributeJsonObject`), `catalog/products/product.service.ts` (single-query `getProductAttributesWithRawQuery`, 4 call sites), `readiness/engine/readiness-attribute-value.resolver.ts`, `productList/productListQueryBuilder.ts` (`buildPrimaryAttributeLadder`), `overview/product-overview.service.ts`, new spec `common/lib/product-attribute-tiers.spec.ts` (26 tests).

Webapp: `lib/constants.ts` (`PRODUCT_ATTRIBUTE_TIER_ORDER`), `types/attribute.types.ts` (`ProductAttributeTierValues`), `lib/utils.ts` (`pickProductAttributeValue`).

# Sync Change Summary Feature

---

## API

### Endpoint

| Item | Value |
|------|-------|
| Route | `POST /sync-review/get-change-summary` |
| Controller | `SyncReviewController.getChangeSummary(req, res, body)` |
| Service | `SyncReviewService.getChangeSummary(req, body): Promise<MarketplaceChange[]>` |
| Guard | `JwtAuthGuard` |
| DTO | `GetChangeReviewDTO` — `dto/getChangeSummary.dto.ts` |
| Interface file | `interface/changeReview.interface.ts` |

### DTO

```typescript
export class GetChangeReviewDTO {
  @IsInt()                               productId: number;
  @IsInt()                               channelId: number;
  @IsOptional() @IsInt()                 amazonMarketplaceId?: number | null;
  @IsOptional() @IsArray() @IsInt({ each: true }) specificVariantIds?: number[];
}
```

---

### Language Codes — Three Distinct Values

| Variable | Source | Role |
|----------|--------|------|
| `interfaceLanguage` | `user.activeLanguage` (`tbl_users`) | First-priority in all label translation subqueries |
| `languageCode` | `acm.language` (Amazon) or `shopifyDefaultLanguage.language` (Shopify) | Current-value attribute inheritance joins; second-priority label fallback |
| `baseLanguageCode` | `accountConfigurationService.getAccountBaseLanguage()` → `data.baseLanguage` | Final fallback for current-value joins (l3/lv5); SKU lookup in variant structure |

No `currency` is resolved or returned.

---

### Pre-flight (`getChangeSummary`)

**Step 1 — Parallel service calls**

```typescript
const [baseLanguage, user, channel] = await Promise.all([
  this._accountConfigurationService.getAccountBaseLanguage(req),
  this._userService.findOneById(req, req.user.id, undefined, ['activeLanguage']) as User,
  this._channelService.findOne(req, { id: channelId, accountId: req.user.accountId }, undefined, ['id', 'channelType']) as Channel,
]);
const baseLanguageCode: string = baseLanguage.data.baseLanguage;
const interfaceLanguage: string = user.activeLanguage;
```

**Step 2 — Variant IDs** (service call, not raw SQL)

```typescript
if (body.specificVariantIds?.length) {
  variantIds = body.specificVariantIds;
} else {
  const variantChannels = await this._variantsChannelsService.findAll(
    req,
    { channelId, isActive: true },
    [{ model: ProductVariant, required: true, where: { productId } }],
    ['id', 'productVariantId'],
  );
  variantIds = variantChannels.map((vc) => Number(vc.productVariantId));
}
```

**Step 3a — Amazon: marketplace list** (raw SQL, fetches `languageCode` here — no separate per-marketplace language query)

```sql
SELECT DISTINCT
  acm.amazon_marketplace_id::int AS "amazonMarketplaceId",
  acm.language                   AS "languageCode"
FROM tbl_product_channel_marketplaces pcm
JOIN tbl_product_channels pc
  ON pc.id = pcm.product_channel_id
  AND pc.product_id = :productId AND pc.channel_id = :channelId AND pc.is_active = true
JOIN tbl_amazon_channel_marketplaces acm
  ON acm.id = pcm.amazon_channel_marketplace_id AND acm.status = true
  [AND acm.amazon_marketplace_id = :amazonMarketplaceId  -- only when body.amazonMarketplaceId present]
WHERE pcm.status = true
```

**Step 3b — Shopify: default language**

```typescript
const shopifyDefaultLanguage = await this._shopifyLanguageMappingService.findOne(
  req, { channelId, isDefault: true }, undefined, ['language'],
);
// languageCode = shopifyDefaultLanguage.language
```

---

### Marketplace Aggregation & Zero-Change Filtering

**Amazon** — runs `buildMarketplaceChange` per marketplace in parallel, filters zero-change results:

```typescript
const marketplaceChanges = await Promise.all(
  amazonMarketplaces.map(async ({ amazonMarketplaceId, languageCode }) => {
    const changes = await this.buildMarketplaceChange(...);
    const changeCount = changes.reduce((s, c) => s + c.changeCount, 0);
    const selectableCount = changes.reduce((s, c) => s + c.selectableCount, 0);
    return { amazonMarketplaceId, changes, changeCount, selectableCount };
  })
);
return marketplaceChanges.filter((mp) => mp.changeCount > 0);
```

**Shopify** — single call, returns `[]` if no changes:

```typescript
const changeCount = changes.reduce((s, c) => s + c.changeCount, 0);
const selectableCount = changes.reduce((s, c) => s + c.selectableCount, 0);
return changeCount > 0 ? [{ amazonMarketplaceId: null, changes, changeCount, selectableCount }] : [];
```

---

### `buildMarketplaceChange`

```typescript
private async buildMarketplaceChange(
  productId: number, channelId: number, amazonMarketplaceId: number | null,
  variantIds: number[], interfaceLanguage: string, languageCode: string, baseLanguageCode: string,
): Promise<ChangedEntityChange[]>
```

Runs 6 parallel raw SQL queries, then resolves variant-attribute labels in JS, then calls `assemble()`:

```typescript
const [attrRows, priceRows, mediaRows, structureRows, vattrRows, miscRows] = await Promise.all([
  this.queryAttributes(params),
  this.queryPricing(params),
  this.queryMedia(params),
  this.queryVariantStructure(params),
  this.queryVariantAttributes(params),
  this.queryMisc(params),
]);
await this.resolveVariantAttrLabels(vattrRows, interfaceLanguage, languageCode);
return this.assemble(attrRows, priceRows, mediaRows, structureRows, vattrRows, miscRows);
```

---

### SQL Queries — Common Conventions

- Every query starts `FROM` its baseline table; current values are `LEFT JOIN`ed onto baseline rows only.
- `variantIds` is interpolated inline: `(table.variant_id IN ('id1','id2',...) OR table.variant_id IS NULL)` — not via a bound parameter.
- `amazonMarketplaceId` filter is conditionally appended: `${amazonMarketplaceId ? 'AND table.amazon_marketplace_id = :amazonMarketplaceId' : ''}`.

---

### `queryAttributes` — `tbl_attribute_sync_baselines`

**Params**: `productId`, `channelId`, `amazonMarketplaceId`, `variantIds`, `interfaceLanguage`, `languageCode`, `baseLanguageCode`

**Label fallback** (2-level — no `name` fallback in SQL):
```sql
COALESCE(
  (SELECT translation FROM tbl_attribute_translations WHERE attribute_id = a.id AND language_code = :interfaceLanguage AND key = 'label' LIMIT 1),
  (SELECT translation FROM tbl_attribute_translations WHERE attribute_id = a.id AND language_code = :languageCode   AND key = 'label' LIMIT 1)
) AS "attrLabel"
```

**Hierarchy label** (3-level):
```sql
COALESCE(h.translations->>:interfaceLanguage, h.translations->>:languageCode, h.title) AS "hLabel"
```

**`typeKey`** — computed inline in SQL to select the right value column; NOT returned as a column in `RawAttrRow`:
```sql
CASE
  WHEN a.is_primary_attribute = true AND a.name = 'product_description' THEN 'value_text'
  WHEN a.is_primary_attribute = true                                     THEN 'value_varchar'
  WHEN a.attribute_type IN ('TEXT','TEXTAREA','AMAZON_MULTISELECT')      THEN 'value_array'
  WHEN a.attribute_type = 'DROPDOWN'                                     THEN 'value_varchar'
  WHEN a.attribute_type = 'CHECKBOX'                                     THEN 'value_bool'
  WHEN a.attribute_type = 'NUMBER'                                       THEN 'value_decimal'
  WHEN a.attribute_type = 'MULTI_TEXT'                                   THEN 'value_text'
  WHEN a.attribute_type = 'DATE_AND_TIME' THEN
    CASE (a.attribute_configuration->>'type')
      WHEN 'both' THEN 'value_date_and_time'
      WHEN 'date' THEN 'value_date'
      WHEN 'time' THEN 'value_time'
      ELSE 'value_varchar'
    END
  ELSE 'value_varchar'
END
```

**Current-value joins** — 6 aliases, always present:

| Alias | `variant_id` | `channel_id` | `language_code` |
|-------|-------------|--------------|-----------------|
| l1 | IS NULL | `:channelId` | `:languageCode` |
| l2 | IS NULL | IS NULL | `:languageCode` |
| l3 | IS NULL | IS NULL | `:baseLanguageCode` |
| lv1 | `= asb.variant_id` | `:channelId` | `:languageCode` |
| lv2 | `= asb.variant_id` | IS NULL | `:languageCode` |
| lv5 | `= asb.variant_id` | IS NULL | `:baseLanguageCode` |

- **Product-level** (`pag.is_product_structure = true`): chain l1 → l2 → l3 (skip if `is_inherited`)
- **Variant-level** (`pag.is_product_structure = false`): chain lv1 → lv2 → l1 → l2 → lv5 → l3

`currentValue` is built as a nested `CASE (typeKey) WHEN 'value_varchar' THEN COALESCE(...) ... END`, wrapping each column with `to_jsonb(...)`.

**Columns returned** → `RawAttrRow`: `baselineId`, `variantId`, `attributeId`, `hierarchyId`, `changeType`, `value`, `attrId`, `attrName`, `isPrimaryAttribute`, `attributeType`, `attrLabel`, `hId`, `hTitle`, `hKey`, `hIsPrimary`, `hLabel`, `currentValue`

---

### `queryPricing` — `tbl_price_sync_baselines`

**Params**: `productId`, `channelId`, `amazonMarketplaceId`, `variantIds`

Current value — 2-level from `tbl_product_variant_prices`:
- `pvp_c`: `channel_id = :channelId`, `amazon_marketplace_id IS NOT DISTINCT FROM psb.amazon_marketplace_id`, variant match
- `pvp_m`: `channel_id IS NULL`, same marketplace/variant match

```sql
COALESCE(
  CASE WHEN pvp_c.is_inherited = false THEN pvp_c.pricing_details->psb.price_type END,
  pvp_m.pricing_details->psb.price_type
) AS "currentPriceData"
```

**Columns returned** → `RawPriceRow`: `baselineId`, `variantId`, `priceType`, `priceData`, `currentPriceData`

---

### `queryMedia` — `tbl_product_media_sync_baselines`

**Params**: `productId`, `channelId`, `variantIds`  
**No `amazonMarketplaceId` — channel-level, baseline runs once and its result is included in every marketplace entry.**

Baseline media: `JSONB_ARRAY_ELEMENTS(pmsb.previous_sync_value)` lateral → JOIN `tbl_digital_assets` on `digitalAssetsId` → aggregated, ordered `isMainImage DESC, sequence ASC NULLS LAST, id ASC`.

Current media: subquery on `tbl_product_media_linkers` → JOIN `tbl_digital_assets`, matched by `product_id + channel_id + variant_id`.

**Columns returned** → `RawMediaRow`: `baselineId`, `variantId`, `baselineMedia`, `currentMedia`

---

### `queryVariantStructure` — `tbl_variant_structure_baselines`

**Params**: `productId`, `channelId`, `amazonMarketplaceId`, `variantIds`, `interfaceLanguage`, `languageCode`, `baseLanguageCode`  
Variant-level only — no `OR variant_id IS NULL`.

- **Baseline attributes**: lateral `JSONB_ARRAY_ELEMENTS_TEXT(vsb.previous_variant_attribute_ids)` → JOIN `tbl_product_variant_attributes` → JOIN `tbl_attributes`; label: interfaceLanguage → languageCode.
- **Current attributes**: subquery on `tbl_product_variant_attributes` WHERE `product_id = :productId AND is_active = true`, ordered by `pva.sequence ASC NULLS LAST`.
- **Current wrapper** (Amazon only; `NULL` for Shopify via `CASE WHEN :amazonMarketplaceId IS NULL THEN NULL ELSE (...) END`): `tbl_wrappers` → JOIN `tbl_variant_marketplaces` on `vm.product_variant_id = vsb.variant_id AND w.channel_id AND w.amazon_marketplace_id`; `associatedVariants` resolved from `tbl_variant_marketplaces vm2`; SKU from `tbl_product_attributes` (primary structure group, `a.name = 'sku'`, `channel_id IS NULL`, `language_code = :baseLanguageCode`).

**Columns returned** → `RawStructureRow`: `baselineId`, `variantId`, `metaData`, `baselineAttributes`, `currentAttributes`, `currentWrapper`

---

### `queryVariantAttributes` — `tbl_variant_attribute_sync_baselines`

**Params**: `productId`, `channelId`, `amazonMarketplaceId`, `variantIds`, `interfaceLanguage`, `languageCode`  
Variant-level only.

Current value — 2-level from `tbl_variants_attributes_values`:
1. `channel_id = :channelId AND language_code = :languageCode AND is_inherited = false`
2. `channel_id IS NULL AND language_code = :languageCode`

Attribute label: interfaceLanguage → languageCode.

**Columns returned** → `RawVattrRow`: `baselineId`, `variantId`, `pvaId`, `value`, `attrId`, `attrName`, `attrLabel`, `currentValue`

---

### `queryMisc` — `tbl_misc_sync_baselines`

**Params**: `productId`, `channelId`, `amazonMarketplaceId`, `variantIds`, `interfaceLanguage`, `languageCode`

**`baselineEnriched`** — `CASE msb.baseline_type WHEN ... THEN JSONB_BUILD_OBJECT(singleKey, ...) ELSE '{}'::jsonb END`:

| `baseline_type` | `baselineEnriched` content |
|----------------|---------------------------|
| `TAGS` | `{ tags: previous_sync_value->>'tags' }` |
| `SHOPIFY_CATEGORY` | `{ shopifyCategory: { id, name, fullName } }` via LEFT JOIN `tbl_shopify_categories sc_base` |
| `SHOPIFY_THEME_TEMPLATE` | `{ themeTemplate: previous_sync_value->>'themeTemplate' }` |
| `SHOPIFY_EXTERNAL_PUBLICATIONS` | `{ externalPublications: previous_sync_value->'externalPublications' }` |
| `SHOPIFY_COLLECTIONS` | `{ shopifyCollections: [{id, label}] }` via subquery + JOIN `tbl_shopify_collections` |
| `AMAZON_CONDITION_TYPE` | `{ conditionType: { value, label } }` via LEFT JOIN `tbl_amazon_condition_types act_base` (lang fallback: interfaceLanguage → languageCode) |
| `AMAZON_SHIPPING_TEMPLATE` | `{ shippingTemplate: { value: external_id, label: name } }` via LEFT JOIN `tbl_amazon_channel_shipping_templates ast_base` |
| `AMAZON_CATEGORY` (recommended_browse_nodes) | `{ amazonCategory: { categoryType: 'recommended_browse_nodes', amazonCategory: { id, name, hierarchy } } }` via LEFT JOIN `tbl_amazon_categories ac_base` |
| `AMAZON_CATEGORY` (item_type_keyword) | `{ amazonCategory: { categoryType: 'item_type_keyword', itemTypeKeyword: '...' } }` from raw stored value |
| fallback | `'{}'::jsonb` |

**Current value columns** (one conditional subquery per column, gated by `CASE WHEN msb.baseline_type = '...' THEN (...) END`):

| Column | Source |
|--------|--------|
| `cur_tags` | `tbl_product_channels.tags` |
| `cur_shopifyCategory` | `tbl_product_channels` + LEFT JOIN `tbl_shopify_categories` |
| `cur_themeTemplate` | `tbl_product_channels.theme_template` |
| `cur_externalPublications` | `tbl_product_channels.external_publications` (`TO_JSONB`) |
| `cur_shopifyCollections` | `tbl_product_channel_collections` (is_active=true) + JOIN `tbl_shopify_collections` |
| `cur_conditionType` | product: `tbl_product_channel_marketplaces`; variant: `tbl_variant_marketplaces`; label via `condTypeJoin` helper (interfaceLanguage → languageCode) |
| `cur_shippingTemplate` | same two paths; JOIN `tbl_amazon_channel_shipping_templates`; `{ value: external_id, label: name }` |
| `cur_amazonCategoryType` | `CASE WHEN itKeywordExistsSql THEN 'item_type_keyword' ELSE 'recommended_browse_nodes' END` |
| `cur_itemTypeKeyword` | when ITK: linked attribute lookup → `tbl_product_attributes` (`channel_id = :channelId`, `language_code = :languageCode`, `variant_id IS NULL`) |
| `cur_amazonCategoryObj` | when not ITK: `tbl_amazon_categories` via `category_id` from `tbl_product_channel_marketplaces` (product) or `tbl_variant_marketplaces` (variant) |

**Reusable SQL fragments**:

`amazonChannelMarketplaceIdSql`:
```sql
(SELECT id FROM tbl_amazon_channel_marketplaces
 WHERE amazon_marketplace_id = :amazonMarketplaceId
   AND amazon_channel_id = (SELECT amazon_channel_id FROM tbl_amazon_channels WHERE channel_id = :channelId)
 LIMIT 1)
```

`itKeywordExistsSql`:
```sql
EXISTS (
  SELECT 1 FROM tbl_amazon_attribute_hierarchies h2
  JOIN tbl_amazon_channel_product_types acpt ON acpt.id = h2.channel_product_type_id
  JOIN tbl_product_channels pc2 ON pc2.amazon_product_type_id = acpt.amazon_product_type_id
    AND pc2.product_id = :productId AND pc2.channel_id = :channelId
  WHERE h2.key = 'item_type_keyword'
    AND h2.channel_marketplace_id = <amazonChannelMarketplaceIdSql>
)
```

**Columns returned** → `RawMiscRow`: `baselineId`, `variantId`, `baselineType`, `value`, `baselineEnriched`, `cur_tags`, `cur_shopifyCategory`, `cur_themeTemplate`, `cur_externalPublications`, `cur_shopifyCollections`, `cur_conditionType`, `cur_shippingTemplate`, `cur_amazonCategoryType`, `cur_itemTypeKeyword`, `cur_amazonCategoryObj`  
Ordered by `msb.id`.

---

### `resolveVariantAttrLabels` (post-query, in `buildMarketplaceChange`)

1. Collect all distinct `attrId`s and all distinct value keys from both `currentValue` and `value` arrays across all rows.
2. Single query: `tbl_attribute_translations` WHERE `attribute_id IN (...)` AND `language_code IN (:interfaceLanguage, :languageCode)` AND `key IN (...)`.
3. Build `Map<attrId, Map<key, { ul: string|null, il: string|null }>>` — `ul` = interfaceLanguage match, `il` = languageCode match.
4. Per row: `row._resolvedLabels = currentValue.map(v => entry.ul || entry.il || rawKey)`.

Labels are resolved only for `currentValue` keys (baseline value keys are fetched from the translations query but not stored separately).

---

### `assemble` — Entity Map & Count Computation

**Entity map**: `Map<variantId: number|null, EntityGroups>`

**Row routing**:

| Condition | Destination |
|-----------|-------------|
| `attrRow` — `hKey IN ('purchasable_offer', 'list_price')` | `entity.pricing.attachedAttributes` |
| `attrRow` — `!hierarchyId \|\| hIsPrimary` | `entity.attributes.direct` |
| `attrRow` — non-primary hierarchy | `entity.attributes.hierarchies[hId].children` |
| `priceRow` | `entity.pricing.root` |
| `mediaRow` | `entity.media` (single record, not an array) |
| `structureRow` | `entity.variantStructure` (single record) |
| `vattrRow` | `entity.variantAttributes.records` |
| `miscRow` | `entity.misc.records` |

Groups are initialized with `changeCount: 0, selectableCount: 0` when first created.

**Count rules** (applied at the map → array step):

| Group | `changeCount` | `selectableCount` |
|-------|--------------|-------------------|
| `attributes` | `direct.length + Σ hierarchies[*].children.length` | `direct.length + hierarchies.length` (each group = 1) |
| `pricing` | `root.length + attachedAttributes.length` | `cc > 0 ? 1 : 0` (whole pricing = 1) |
| `media` | `1` (one baseline row = one change) | `1` |
| `variantStructure` | `1` | `1` |
| `variantAttributes` | `records.length` | `records.length > 0 ? 1 : 0` (all records together = 1) |
| `misc` | `records.length` | `records.length` (linear) |

Entity `changeCount`/`selectableCount` = sum over all present groups.

**Sort**: `null` (product-level) first, then ascending `variantId`.

**Return shape**: `{ variantId, groups, changeCount, selectableCount }[]`

---

### `buildMiscCurrent` — Current Field Dispatch

Pure function; dispatches on `row.baselineType` and returns only the relevant `MiscEnrichedFields` subset:

| `baselineType` | Returns |
|---------------|---------|
| `TAGS` | `{ tags: row.cur_tags }` |
| `SHOPIFY_CATEGORY` | `{ shopifyCategory: row.cur_shopifyCategory \|\| null }` |
| `SHOPIFY_THEME_TEMPLATE` | `{ themeTemplate: row.cur_themeTemplate }` |
| `SHOPIFY_EXTERNAL_PUBLICATIONS` | `{ externalPublications: row.cur_externalPublications \|\| null }` |
| `SHOPIFY_COLLECTIONS` | `{ shopifyCollections: row.cur_shopifyCollections \|\| null }` |
| `AMAZON_CONDITION_TYPE` | `{ conditionType: row.cur_conditionType \|\| null }` |
| `AMAZON_SHIPPING_TEMPLATE` | `{ shippingTemplate: row.cur_shippingTemplate \|\| null }` |
| `AMAZON_CATEGORY` | `{ amazonCategory: { categoryType, itemTypeKeyword: isITK ? cur_itemTypeKeyword : null, amazonCategory: isITK ? null : cur_amazonCategoryObj \|\| null } }` |
| fallback | `{}` |

---

### Response Shape (TypeScript Interfaces)

```typescript
// ── Shared ─────────────────────────────────────────────────────────────────

interface AttributeItem { id: number; name: string; label: string; }

// ── Attributes ──────────────────────────────────────────────────────────────

interface AttributeDirectRecord {
  baseline: { id: number; changeType: string; value: any; attribute: AttributeItem };
  current: { value: any };
  // typeKey is NOT in the response — used only internally in SQL
}
interface AttributeHierarchyGroup {
  hierarchy: AttributeItem;
  children: { baseline: { id: number; changeType: string; value: any; attribute: AttributeItem }; current: { value: any } }[];
}
interface AttributesResult {
  direct: AttributeDirectRecord[];
  hierarchies: AttributeHierarchyGroup[];
  changeCount: number;
  selectableCount: number;
}

// ── Pricing ─────────────────────────────────────────────────────────────────

interface PricingRootRecord {
  baseline: { id: number; priceType: string; priceData: any };
  current: { priceData: any };
}
interface PricingResult {
  root: PricingRootRecord[];
  attachedAttributes: AttributeDirectRecord[];
  changeCount: number;
  selectableCount: number;
}

// ── Media ───────────────────────────────────────────────────────────────────

interface MediaDigitalAsset {
  id: number; type: string; fileOriginalName: string; filePath: string;
  fileUniqueName: string; sequence: number; fileType: string;
}
interface MediaItem { isMainImage: boolean; digitalAsset: MediaDigitalAsset; }
interface MediaResult {
  baseline: { id: number; media: MediaItem[] };
  current: { media: MediaItem[] };
  changeCount: number;
  selectableCount: number;
}

// ── Variant Structure ────────────────────────────────────────────────────────

interface VariantStructureResult {
  baseline: { id: number; metaData: any; attributes: AttributeItem[] };
  current: {
    attributes: AttributeItem[];
    wrapper: { id: number; sku: string; asin: string; amazonCurrentVariationTheme: string; associatedVariants: Array<{ id: number; sku: string }> } | null;
  };
  changeCount: number;
  selectableCount: number;
}

// ── Variant Attributes ───────────────────────────────────────────────────────

interface VariantAttributeRecord {
  baseline: { id: number; value: any[]; attribute: AttributeItem };
  current: { value: any[]; label: string[] };
}
interface VariantAttributesResult {
  records: VariantAttributeRecord[];
  changeCount: number;
  selectableCount: number;
}

// ── Misc ────────────────────────────────────────────────────────────────────

interface MiscEnrichedFields {
  tags?: string | null;
  shopifyCategory?: { id: number; name: string; fullName: string } | null;
  themeTemplate?: string | null;
  externalPublications?: object[] | null;
  shopifyCollections?: Array<{ id: number; label: string }> | null;
  conditionType?: { value: string; label: string } | null;
  shippingTemplate?: { value: string; label: string } | null;
  amazonCategory?: { categoryType: string; itemTypeKeyword?: string | null; amazonCategory?: { id: number; name: string; hierarchy: string } | null } | null;
}
interface MiscRecord {
  baseline: { id: number; baselineType: string; value: any; enriched: MiscEnrichedFields };
  current: MiscEnrichedFields;
}
interface MiscResult { records: MiscRecord[]; changeCount: number; selectableCount: number; }

// ── Top-level ────────────────────────────────────────────────────────────────

interface EntityGroups {
  attributes?: AttributesResult;
  pricing?: PricingResult;
  media?: MediaResult;
  variantStructure?: VariantStructureResult;   // variant-level only
  variantAttributes?: VariantAttributesResult; // variant-level only
  misc?: MiscResult;
}
interface ChangedEntityChange {
  variantId: number | null;   // null = product-level
  groups: EntityGroups;
  changeCount: number;
  selectableCount: number;
}
interface MarketplaceChange {
  amazonMarketplaceId: number | null;
  changes: ChangedEntityChange[];
  changeCount: number;
  selectableCount: number;
}
```

---

### Key Rules

| Rule | Detail |
|------|--------|
| Count field name | `changeCount` (not `changesCount`) |
| `typeKey` | Computed in SQL only; not present in any response field |
| `currency` | Not resolved or returned |
| `variantIds` in SQL | Inline string interpolation, not a bound parameter |
| Media scope | No `amazon_marketplace_id` filter — `queryMedia` runs once per `buildMarketplaceChange` call and its result is shared across all marketplaces |
| Pricing-owned attrs | `hKey IN ('purchasable_offer', 'list_price')` → `pricing.attachedAttributes`; excluded from `attributes` |
| Group key omitted | When a group has no baseline rows for that entity, the key is not set on `groups` |
| Empty marketplace | Omitted when `changeCount === 0` — both Amazon (`.filter`) and Shopify (ternary `[]`) |
| Entity sort | `null` first, then ascending `variantId` |
| `attrLabel` fallback | interfaceLanguage → languageCode (no raw `name` in SQL) |
| `hLabel` fallback | interfaceLanguage → languageCode → `h.title` |

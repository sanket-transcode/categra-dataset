# Readiness — How It Is Actually Calculated Today

> Source of truth: `api/apps/api-main/src/modules/app/catalog/products/listings/completeness/calculate-completeness.service.ts`
> Storage: `tbl_readiness_values` · Config: `tbl_readiness_configurations`

---

## 1. The Universal Formula

```
                    ┌──────────────────────────────────────┐
                    │  readinessValue =                    │
                    │      totalFilledItems                │
                    │      ──────────────── × 100          │
                    │        totalItems                    │
                    │      (2 decimals, 0 if totalItems=0) │
                    └──────────────────────────────────────┘
```

`totalItems` / `totalFilledItems` are accumulated from **4 buckets**:

```
  ┌─ BUCKET A ─────────────┐ ┌─ BUCKET B ────────────┐ ┌─ BUCKET C ──────────┐ ┌─ BUCKET D ─────────┐
  │ Structured attributes  │ │ Unevaluated configs   │ │ Stock + Price       │ │ Primary Media      │
  │ (with a value row)     │ │ (no product_attribute)│ │ (operational)       │ │ (countCheck)       │
  ├────────────────────────┤ ├───────────────────────┤ ├─────────────────────┤ ├────────────────────┤
  │ +1 total per config    │ │ +1 total ONLY if attr │ │ +1 total per config │ │ +1 total if config │
  │ +1 filled if isValid   │ │    ∈ product structure│ │ +1 filled if isValid│ │ +1 filled if valid │
  │                        │ │ +0 filled (always)    │ │ SKIPPED for parent  │ │                    │
  │                        │ │ else NOT_APPLICABLE   │ │   wrapper products  │ │                    │
  └────────────────────────┘ └───────────────────────┘ └─────────────────────┘ └────────────────────┘
```

---

## 2. Storage Key — One Row Per Scope

`tbl_readiness_values` unique scope:

```
  ( product_id , variant_id , channel_id , language_code , account_id )  →  readiness_value FLOAT
       │             │            │             │
       │             │            │             └─ 'en_US', 'de_DE', ...
       │             │            └─ NULL = MASTER · <id> = Amazon/Shopify channel
       │             └─ NULL = parent/product level · <id> = variant level
       └─ always set
```

> `tbl_product_channel_marketplace_readiness_values` exists as an entity but its write path is
> **commented out** — Amazon marketplaces land in `tbl_readiness_values` keyed by `language_code`.

---

## 3. Entry Point Fan-Out — `calculateCompleteness()`

```
calculateCompleteness({ req, productId, channelId? })
        │
        ├── load: productChannels · productVariants(ids) · baseLanguage · accountLanguages
        │
        ├─► FOR EACH active productChannel (filtered by channelId if passed)
        │     │
        │     ├── channelType = AMAZON ──► processChannelAmazon()
        │     │        ├─ for each ProductChannelMarketplace
        │     │        │     calculateCompletenessForMarketPlace1(productId, variantId=NULL,
        │     │        │        { channelId, marketplaceId, languageCode: acm.language })
        │     │        └─ for each variantId (concurrency 5)
        │     │              for each VariantMarketplace(status=true)
        │     │                 calculateCompletenessForMarketPlace1(productId, variantId, {...})
        │     │
        │     └── channelType = SHOPIFY ─► processChannelShopify()
        │              langs = channel.shopifyLanguageMappings[].language
        │              ├─ calculateCompletenessForNewProduct(productId, NULL, langs, {channelId})
        │              └─ for each variantId (concurrency 5)
        │                    calculateCompletenessForNewProduct(productId, vId, langs, {channelId})
        │
        └─► MASTER  (channel = null)
              langs = unique( accountLanguages ∪ baseLanguage )
              ├─ calculateCompletenessForNewProduct(productId, NULL, langs, channel=null)
              └─ for each variantId (concurrency 5)
                    calculateCompletenessForNewProduct(productId, vId, langs, channel=null)
```

**Rows written per product** (V = variant count, L = languages, M = Amazon marketplaces):

```
  MASTER   :  (1 + V) × L_master
  SHOPIFY  :  (1 + V) × L_shopify                 per shopify channel
  AMAZON   :  1 × M  +  Σ variant marketplaces    per amazon channel
```

---

## 4. Config Selection — What Counts As An "Item"

```
tbl_readiness_configurations  WHERE account_id = :acct AND status = TRUE
                                AND language_code = <scope language>
                                AND channel_id    = <scope channel | NULL>
        │
        ├── de-duplicated by  getReadinessConfigKey():
        │      attributeId | channelId | languageCode | criteria | JSON(configurations) | JSON(metaData)
        │
        ├── SPLIT ─────────────────────────────────────────────────────────────┐
        │                                                                       │
   name NOT IN (Stock, Price, Primary Media)      name IN (Stock, Price)   name = Primary Media
        → BUCKET A / B (structured)                → BUCKET C              → BUCKET D
                                                    (base language only     (criteria='countCheck',
                                                     in master/shopify)      findOne → single config)
```

> **Master/Shopify path only:** Bucket C is gated by `if (baseLanguage === languageCode)`.
> Non-base languages therefore get **no Stock/Price items at all**.
> **Amazon path:** Bucket C runs for **every** marketplace language (no base-language gate).

---

## 5. Attribute Value Resolution — 3-Step Fallback Ladder

`tbl_product_attributes` lookup, always scoped `variant_id = <scope variant> | NULL`.

### MASTER / SHOPIFY — `calculateCompletenessForNewProduct`

```
 STEP 2  ┌────────────────────────────────────────────────────────────────┐
  direct │ language_code = L · channel_id = NULL                          │
         │ + is_inherited = FALSE  (only when L ≠ baseLanguage)           │
         └────────────────────────────────────────────────────────────────┘
                │ attributes still missing ↓
 STEP 3  ┌────────────────────────────────────────────────────────────────┐
  lang   │ language_code = L · channel_id = NULL · is_inherited = FALSE   │
         └────────────────────────────────────────────────────────────────┘
                │ still missing ↓
 STEP 4  ┌────────────────────────────────────────────────────────────────┐
  master │ language_code = baseLanguage · channel_id = NULL               │
         └────────────────────────────────────────────────────────────────┘
                ↓
 STEP 5  merge + de-dup by  attributeId|variantId|languageCode|channelId|valueGroupId
```

### AMAZON — `calculateCompletenessForMarketPlace1`

```
 STEP 2  language_code = marketplace.language · channel_id = :channelId · is_inherited = FALSE
 STEP 3  language_code = marketplace.language · channel_id = NULL       · is_inherited = FALSE
 STEP 4  language_code = baseLanguage         · channel_id = NULL
```

### Then the `countInherited` gate — `getEffectiveReadinessAttributeValue()`

```
  languageMatches = (row.language_code == requestedLanguage)
  channelMatches  = (row.channel_id    == requestedChannel)
  fallbackUsed    = !languageMatches || !channelMatches

  ┌──────────────┬─────────────────┬──────────────────────────────────────┐
  │ fallbackUsed │ config.count_   │ resulting value                      │
  │              │ inherited       │                                      │
  ├──────────────┼─────────────────┼──────────────────────────────────────┤
  │ false        │  any            │ actual value  → source ".direct"     │
  │ true         │  TRUE           │ actual value  → "*_fallback" source  │
  │ true         │  FALSE          │ ''  (EMPTY → counts as NOT filled)   │
  └──────────────┴─────────────────┴──────────────────────────────────────┘

  valueSource when fallbackUsed:  !channel && !lang → channel_language_fallback
                                  !channel         → channel_fallback
                                  else             → language_fallback
```

### Value column picked by `attributeType`

```
  text                → value_varchar  ||  value_array[0]
  MultiText/textarea  → value_text
  everything else     → value_varchar ?? value_text ?? value_decimal
                         ?? value_int ?? value_boolean ?? value_array[0]

  normalize: strip <tags>, &nbsp; → ' ', trim   →  if empty after strip ⇒ ''
```

---

## 6. Criteria Evaluation — `evaluateReadinessCriteria()`

```
  criteria = 'presenceCheck'
      isValid = !!value.trim()                                  reason: 'missing'

  criteria = 'lengthCheck' | 'countCheck'      →  length = value.length
  criteria = 'valueCheck'                      →  length = Number(value) || 0

      min & max set  →  isValid = min ≤ length ≤ max      reason 'min' | 'max'
      min only       →  isValid = length ≥ min
      max only       →  isValid = length ≤ max
      neither        →  isValid = !!length                reason 'missing'

  default (unknown criteria) →  same as presenceCheck
```

---

## 7. Bucket B — Configs With No Attribute Row

`getUnevaluatedReadinessConfigResults()`

```
  for each readinessConfig NOT present in the evaluated (flattened) set:

        attributeId ∈ structuredAttributeIds ?
                (= linked via tbl_attribute_groups_and_attribute_linkers
                   to an ACTIVE tbl_product_attribute_groups row of this product)
        │
        ├── YES ──► totalItems += 1 · filled += 0
        │           { isValid:false, reason:'missing' }
        │
        └── NO  ──► NOT COUNTED AT ALL
                    { notApplicable:true, readinessStatus:'NOT_APPLICABLE',
                      reason:'not_applicable',
                      notApplicableReason:'Attribute is not in this product structure.' }
```

---

## 8. Bucket C — Stock & Price (Operational)

### Parent-wrapper gate — `isParentWrapperProduct()`

```
  scope has variantId ?  ──YES──► isParentWrapperScope = FALSE
           │NO
           ▼
  SELECT COUNT(*) FROM tbl_product_variant_attributes pva
    JOIN tbl_attributes attr ON attr.id = pva.attribute_id
   WHERE pva.product_id = :pid AND pva.account_id = :acct
     AND pva.is_active = TRUE AND attr.is_variant_key = TRUE
     AND COALESCE(attr.is_deleted,false) = FALSE
           │
           └─ count > 0  ⇒  isParentWrapperScope = TRUE
```

```
  isParentWrapperScope = TRUE  ⇒  Stock & Price EXCLUDED from denominator
     MASTER/SHOPIFY : pushed as NOT_APPLICABLE result, totalItems NOT incremented
     AMAZON         : returns null, skipped entirely
```

### Stock resolution — `getOperationalStockValue()` (first non-NULL wins)

```
  channelId && variantId
      1. tbl_variant_marketplaces.amazon_stock        (vm.status=true, acm.language=L)  → variant_marketplace.amazon_stock
      2. tbl_product_variant_channels.external_stock  (is_active=true)                  → variant_channel.external_stock

  channelId && !variantId
      3. tbl_product_channel_marketplaces.amazon_stock (pcm.status=true, acm.language=L)→ product_channel_marketplace.amazon_stock
      4. tbl_product_channels.external_stock          (is_active=true)                  → product_channel.external_stock

  ALWAYS (fallback tail)
      5. tbl_product_variant_stocks.in_hand_qty                                         → product_variant_stock.in_hand_qty
             WHERE product_id AND (variant_id = :vid | IS NULL)
               AND (channel_id = :cid OR channel_id IS NULL)   [master: channel_id IS NULL]
             ORDER BY (channel_id IS NULL) ASC   ← channel-specific row preferred
      6. SUM(tbl_warehouse_products.total_quantity)                                     → warehouse_product.total_quantity
```

### Price resolution — `getOperationalPriceValue()`

```
  amazonMarketplaceId = lookup(product_channels ▸ pcm ▸ acm WHERE acm.language = L)   [NULL if no channel]

  SCOPED QUERY on tbl_product_variant_prices:
      COALESCE( MAX(pricing_details->'offerPrice'->>'price'::numeric), MAX(price) )
      WHERE account_id · product_id
        AND (variant_id            = :vid | IS NULL)
        AND (channel_id            = :cid | IS NULL)
        AND (amazon_marketplace_id = :mid | IS NULL)
              │
              ├── value > 0                       ⇒ USE IT  (product_variant_price.scoped)
              ├── !countInherited || !channelId   ⇒ USE scoped as-is (may be NULL / 0)
              └── else → MASTER FALLBACK: same query with channel_id IS NULL
                                                AND amazon_marketplace_id IS NULL
                                          (product_variant_price.master_fallback)
```

### Operational criteria — `evaluateOperationalPrimaryCriteria()`

```
  presenceCheck        →  isValid = (value !== null && value > 0)
  value === null       →  isValid = false, reason 'missing'
  min set & value<min  →  false / 'min'
  max set & value>max  →  false / 'max'
  no min AND no max    →  isValid = value > 0
  otherwise            →  true

  ⚠ hasOperationalValue() requires  > 0  — a stock or price of exactly 0 is NEVER valid.
```

---

## 9. Bucket D — Primary Media

```
  config = findOne(status=true, criteria='countCheck', language=L, channel=<scope>, attr.name='Primary Media')
  if found  →  totalItems += 1   (unconditionally, even for a parent wrapper)

  COUNT tbl_product_media WHERE account_id · product_id
        · variant_id = <scope|NULL> · channel_id = <scope|NULL>
        · (language_code IN [L, UPPER(L), lower(L)]  OR  language_code IS NULL)
          │
          ├── count > 0  OR  !countInherited  ⇒ done
          └── countInherited && count = 0  ⇒ walk fallback scopes IN ORDER:
                     if channelId:            ( variant , NULL    )   ← drop channel
                     if variantId:            ( NULL    , channel )   ← drop variant
                     if variantId:            ( NULL    , NULL    )   ← master
                (first scope with count > 0 wins, fallbackUsed = true)

  valueSource:  variant+channel → product_media.variant_channel
                variant only    → product_media.variant_master
                channel only    → product_media.parent_channel
                neither         → product_media.parent_master

  evaluatePrimaryMediaCount:  min>count→'min' · max<count→'max' · no min/max→count>0
```

---

## 10. Scenario Matrix — Exact Parameters Per Case

| # | Scenario | Method | `variantId` | `channelId` | `languageCode` | Buckets counted |
|---|---|---|---|---|---|---|
| 1 | **Master · standalone product** (no variants) | `calculateCompletenessForNewProduct` | `NULL` | `NULL` | each of `accountLangs ∪ base` | A, B, **C (base lang only)**, D |
| 2 | **Master · parent WITH variants** | `calculateCompletenessForNewProduct` | `NULL` | `NULL` | each master lang | A, B, ~~C~~ *(NOT_APPLICABLE)*, D |
| 3 | **Master · individual variant** | `calculateCompletenessForNewProduct` | `<vid>` | `NULL` | each master lang | A, B, **C (base lang only)**, D |
| 4 | **Shopify · parent / standalone** | `calculateCompletenessForNewProduct` | `NULL` | `<shopifyChannelId>` | each `shopifyLanguageMappings.language` | A, B, C\*, D |
| 5 | **Shopify · variant** | `calculateCompletenessForNewProduct` | `<vid>` | `<shopifyChannelId>` | each shopify lang | A, B, C\*, D |
| 6 | **Amazon · parent / standalone** (per marketplace) | `calculateCompletenessForMarketPlace1` | `NULL` | `<amazonChannelId>` | `amazonChannelMarketplace.language` | A, B, C **(every lang)**, D |
| 7 | **Amazon · variant** (per marketplace) | `calculateCompletenessForMarketPlace1` | `<vid>` | `<amazonChannelId>` | `vm ▸ pcm ▸ acm.language` ∥ base | A, B, C **(every lang)**, D |

\* C only when `languageCode === baseLanguage`; and skipped entirely for a parent-wrapper scope
(scenarios 2 / 4 when the parent carries variant-key attributes).

### Key behavioural differences at a glance

```
                          MASTER / SHOPIFY                    AMAZON
  attribute step-2 scope   channel_id = NULL                  channel_id = :channelId
  step-2 is_inherited      FALSE only if L ≠ baseLanguage     always FALSE
  Stock/Price bucket       only when L == baseLanguage        every marketplace language
  parent-wrapper S/P       pushed as NOT_APPLICABLE result    silently dropped (null)
  per-attribute detail     collected in readinessResults[]    NOT collected (counts only)
  row written              (p, v, NULL|shopifyCh, L)          (p, v, amazonCh, acm.language)
```

---

## 11. Parent With Variants — Where Aggregation Actually Happens

The calculator **never averages**. Each scope gets its own independent row.
Averaging is purely a **read-side** concern in the product list.

`api/apps/api-main/src/modules/app/catalog/products/productList/productListQueryBuilder.ts`

```
  readiness_agg CTE:
        SELECT trv.product_id,
               ROUND(AVG(COALESCE(trv.readiness_value,0)),2)  AS readiness_value,
               JSONB_AGG({languageCode, readinessValue})      AS readiness_values
        FROM tbl_readiness_values trv
        WHERE trv.variant_id IS NULL          ← ⚠ PARENT ROWS ONLY, variant rows excluded
          AND trv.account_id = :accountId
          AND <channel filter>
        GROUP BY trv.product_id               ← the average is ACROSS LANGUAGES

  buildReadinessChannelFilter():
        readinessLanguage set  →  channel_id IS NULL AND language_code = :readinessLanguage
        else channelId set     →  channel_id = :channelId
        else                   →  channel_id IS NULL
        (+ always  AND variant_id IS NULL)
```

**The readiness min/max filter** (`buildReadinessProductFilter`) DOES branch on variants:

```
   product has NO active variants  ─► AVG(product-level rows)  BETWEEN min AND max
   product HAS active variants     ─► AVG(its VARIANT rows)    BETWEEN min AND max
   product has NO readiness rows   ─► included only if  min ≤ 0 ≤ max
```

Variant pre-filter CTE (`variant_readiness_filtered`, built only when `hasReadinessFilter && channelId && isActive`):

```
   SELECT variant_id FROM tbl_readiness_values
    WHERE account_id · channel_id = :channelId · language_code = :languageCode · variant_id IS NOT NULL
    GROUP BY variant_id HAVING ROUND(AVG(readiness_value),2) BETWEEN :readinessMin AND :readinessMax
   UNION
   variants with NO readiness row at all   (when min ≤ 0 ≤ max)
```

---

## 12. Read APIs — What Each Returns

| API | Returns | Scope filter |
|---|---|---|
| `getProductChannelWiseReadinessValue` | **stored** `{languageCode, readinessValue}[]` | no channel → `channel_id IS NULL`; Amazon → `channel_id=:cid`, sorted by `amazon_marketplace_id`; other → `channel_id=:cid, variant_id IS NULL` |
| `getReadinessView` / `getReadinessViewBatch` | **live** per-attribute breakdown, master scope | `productId, languageCode, variantId?` |
| `getProductWiseReadinessView` | **live** per-attribute breakdown | `productId, languageCode, variantId?` |
| `getMarketplaceWiseReadinessView` | **live** breakdown, channel-scoped | `productId, channelId, languageCode, variantId?` |
| `getAmazonProductWiseReadinessView` | **live** breakdown, marketplace-scoped | resolves `languageCode` from `productChannelMarketplaceId` |

> The view APIs **recompute** with the same ladder; they do not read `readiness_value`.
> `calculateCompletenessByProductIdAndChannelId` is a **separate legacy path** writing
> `tbl_product_channels.completeness` (denominator starts at 1, only `text`/`MultiText` count) — not readiness.

---

## 13. Recalculation Triggers

`operational-readiness-recalculation.service.ts` → `recalculateForMutation({ productId, channelId?, variantId?, source })`

```
  STOCK_UPDATED · WAREHOUSE_STOCK_UPDATED · AMAZON_STOCK_UPDATED · SHOPIFY_STOCK_UPDATED
  PRICE_UPDATED · AMAZON_PRICE_UPDATED · SHOPIFY_PRICE_UPDATED · BULK_PRICE_UPDATED · BULK_STOCK_UPDATED
  PRODUCT_CONTENT_UPDATED · PRODUCT_LOCALIZED_CONTENT_UPDATED · PRODUCT_STRUCTURE_UPDATED
  PRODUCT_VARIANT_CONTENT_UPDATED · PRODUCT_VARIANT_STRUCTURE_UPDATED
  SHOPIFY_METAFIELD_UPDATED · PRODUCT_TRANSLATION_UPDATED
  PRODUCT_MEDIA_{UPDATED,DELETED,REORDERED,MAIN_IMAGE_UPDATED,INHERITANCE_UPDATED}
  PRODUCT_VARIANT_MEDIA_UPDATED · CHANNEL_MEDIA_UPDATED · SYNC_MEDIA_IMPORTED · DAM_MEDIA_ASSIGNED
                                  │
                                  └──► calculateCompleteness() → full fan-out (§3) → socket emit
```

---

## 14. Worked Example

```
  Product P (parent, 2 variants V1/V2) · base = en_US · account langs = [en_US, de_DE]
  Amazon channel A with marketplaces: en_US, de_DE
  Configs (channel=NULL, en_US): ProductName presenceCheck · Description lengthCheck(min=50)
                                 Stock presenceCheck · Price presenceCheck · PrimaryMedia countCheck(min=1)

  ── MASTER / P / variant=NULL / en_US ──────────────────────────────
     ProductName   "Blue Shirt"                          valid   ✓
     Description   38 chars < 50                 reason 'min'    ✗
     Stock         PARENT WRAPPER → NOT_APPLICABLE   (not counted)
     Price         PARENT WRAPPER → NOT_APPLICABLE   (not counted)
     PrimaryMedia  2 images ≥ 1                          valid   ✓
                                                  ───────────────
                     totalItems = 3 · filled = 2  →  66.67

  ── MASTER / P / variant=V1 / en_US ────────────────────────────────
     ProductName   inherited from parent, countInherited=TRUE → "Blue Shirt"   ✓
     Description   inherited, countInherited=FALSE            → ''  'missing'  ✗
     Stock         in_hand_qty = 12  > 0                                       ✓
     Price         scoped price 29.99 > 0                                      ✓
     PrimaryMedia  0 at (V1, master) → fallback (NULL,NULL) → 2                ✓
                                                  ───────────────
                     totalItems = 5 · filled = 4  →  80.00

  ── MASTER / P / variant=NULL / de_DE ──────────────────────────────
     Stock/Price bucket SKIPPED (de_DE ≠ baseLanguage)
     ProductName / Description / PrimaryMedia only
                     totalItems = 3

  ── AMAZON A / P / variant=V1 / de_DE ──────────────────────────────
     Stock/Price bucket RUNS (no base-language gate on the Amazon path)
     Stock = tbl_variant_marketplaces.amazon_stock where acm.language = 'de_DE'
                     totalItems = 5

  ── PRODUCT LIST (no channel, no readinessLanguage) ────────────────
     readiness_agg → AVG over P's rows with variant_id IS NULL, channel_id IS NULL
                   = AVG(66.67 en_US , X de_DE)   ← variant scores are NOT included
```

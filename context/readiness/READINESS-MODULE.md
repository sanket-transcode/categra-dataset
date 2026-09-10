# Readiness (Product Completeness) — How it works today

_Scope: `api/` only. Describes the code paths that are actually wired and executing today.
Dead/legacy code is listed at the very end so it can be ignored._

---

## 1. One-paragraph summary

Readiness = "what % of the configured content rules does this product satisfy, in this
language, on this channel". Rules ("readiness configurations") are rows in
`tbl_readiness_configurations`, each one attached to **one attribute × one channel × one
language**, carrying **one criteria** (`presenceCheck` / `lengthCheck` / `countCheck` /
`valueCheck`) and a `{min, max}` JSON config. The engine walks every enabled rule in the
scope, evaluates it, and stores a single percentage per
(product, variant, channel, language) in `tbl_readiness_values`.

There is **no rule DSL, no conditional/dependency rules, no regex, no cross-attribute rules**.
"Dynamic" means: the user can attach any of the 4 criteria + min/max to any attribute, per
channel, and toggle it on/off — nothing more.

---

## 2. Where the code lives

```
api/apps/api-main/src/
├── entities/
│   ├── readinessConfigurations.entity.ts        → tbl_readiness_configurations   (THE RULES)
│   ├── readinessValue.entity.ts                 → tbl_readiness_values           (THE SCORES)
│   └── marketplaceReadinessValue.entity.ts      → tbl_product_channel_marketplace_readiness_values
│                                                   (declared, write path commented out — unused)
│
├── modules/app/catalog/attributes/attribute-core/
│   ├── attribute.service.ts                     → createAttributeReadiness()  (user-defined rules)
│   │                                               updateAttributeReadinessConfiguration()
│   └── readinessConfigurations/
│       └── readiness-configurations.service.ts  → seeds the 5 predefined rules per language/channel,
│                                                   lists rules, removes them, re-triggers recalc
│
├── modules/app/catalog/products/listings/completeness/
│   ├── calculate-completeness.service.ts        → ★ THE ENGINE (evaluation + scoring + read APIs)
│   ├── calculate-completeness.controller.ts     → /product-completeness/*
│   └── operational-readiness-recalculation.service.ts → event fan-in ("something changed → recalc")
│
└── modules/app/system/operations/readiness/
    ├── completeness/                            → per-channel attribute on/off flag
    │                                               (tbl_attribute_channels.count_against_completeness)
    └── readiness-diagnostics/                   → read-only "shadow evaluator" / audit APIs
```

---

## 3. Data model

```
                 tbl_attributes                       tbl_channels
                        │                                   │
                        │  attribute_id (nullable)          │ channel_id (NULL = "Master")
                        ▼                                   ▼
        ┌────────────────────────────────────────────────────────────┐
        │            tbl_readiness_configurations   (A RULE)         │
        ├────────────────────────────────────────────────────────────┤
        │ account_id                                                 │
        │ attribute_id      → which field the rule is about          │
        │ channel_id        → NULL = master scope, else per channel  │
        │ language_code     → NOT NULL, always a language-scoped row │
        │ criteria          → presenceCheck|lengthCheck|countCheck|  │
        │                     valueCheck   (free-form STRING column) │
        │ configurations    → JSON { min, max }  (stored as strings) │
        │ status            → rule on/off                            │
        │ count_inherited   → may a fallback/inherited value satisfy │
        │                     this rule?                             │
        │ is_amazon_channel / meta_data → Amazon-only extras         │
        └────────────────────────────────────────────────────────────┘
                                   │  evaluated by the engine
                                   ▼
        ┌────────────────────────────────────────────────────────────┐
        │              tbl_readiness_values   (THE SCORE)            │
        │  product_id, variant_id (NULL = parent), channel_id (NULL  │
        │  = master), language_code, account_id → readiness_value %  │
        └────────────────────────────────────────────────────────────┘
```

One score row per **product/variant × channel × language**. Amazon marketplace results are
written into this same table, keyed by the marketplace's language.

---

## 4. What rule types are supported today

### 4.1 The 4 criteria (this is the whole rule engine)

`evaluateReadinessCriteria()` in `calculate-completeness.service.ts`:

| criteria | what it measures | uses min/max | typical use |
|---|---|---|---|
| `presenceCheck` | value non-empty after trim | no | Price, "just fill it in" |
| `lengthCheck` | **string length** of the value | yes | Product Name 50–200 chars, Description ≥ 300 |
| `valueCheck` | **numeric value** of the value | yes | Stock ≥ 1 |
| `countCheck` | a real count only for Primary Media; on a normal attribute it degrades to *string length* | yes | Primary Media 1–9 |
| _anything else_ | falls to `default:` → treated as `presenceCheck` | no | an unknown criteria never errors |

min/max semantics: both set → `min ≤ x ≤ max`; only one set → that bound only; neither set →
"non-empty" (`!!length`). Failure reason is reported as `'missing' | 'min' | 'max'`.

### 4.2 The 5 predefined (system-seeded) rules

Seeded by `createPreDefinedReadinessConfigurations()` whenever a language is added, a channel is
connected, or an Amazon/Shopify channel is set up. Seeded for `channel_id = NULL` **and** for the
connected channel, for every account language:

| Attribute | criteria | default config | value comes from |
|---|---|---|---|
| Product Name | `lengthCheck` | min 50, max 200 | `tbl_product_attributes` |
| Product Description | `lengthCheck` | min 300 | `tbl_product_attributes` |
| Primary Media | `countCheck` | min 1, max 9 | `COUNT(tbl_product_media_linkers)` |
| Stock | `valueCheck` | min 1 | stock tables (see §8) |
| Price | `presenceCheck` | — | price tables (see §8) |

Stock and Price are seeded **only for the base language** in master scope; for a channel they are
seeded for every language of that channel.

### 4.3 User-defined ("dynamic") rules — yes, supported

`POST /attribute/create-attribute-readiness` → `attribute.service.createAttributeReadiness()`

- Pick any attribute(s), a `criteria`, a `{min,max}`, `countInherited`, and a channel (or master).
- The row is **fanned out to every account language** (base + all additional), so a rule created
  once exists once per language.
- Duplicate guard: same attribute + channel + **same criteria** is rejected.
  → **different criteria on the same attribute is allowed**, i.e. an attribute can carry several
  rules (e.g. `presenceCheck` *and* `lengthCheck`); the engine "flattens" them so **each rule is
  its own denominator item**.
- `PUT /attribute/update-attribute-readiness-configuration` edits `status`, `criteria`,
  `configurations`, `countInherited` (bulk array).
- Both endpoints then fire `updateReadinessToAllAffectedProduct()`, which re-scores every product
  in the account (or in that channel) in batches of 50, fire-and-forget.

**Not supported:** conditional rules, rules depending on another attribute's value, regex/pattern,
enum/allowed-value checks, category-specific rules, marketplace-specific rules (channel + language
is the finest scope), weights/priority (every rule counts exactly 1), severity levels, or a
blocking-vs-warning distinction.

### 4.4 Amazon bullet-point rules — seeded but dead

For Amazon channels two extra rows are seeded with `attribute_id = NULL`,
`is_amazon_channel = true`, `meta_data = { bulletPoints: true }` (a `countCheck` min 3 and a
`lengthCheck` min 50). Every evaluation query joins `Attribute` with `required: true`, so
**rows with `attribute_id = NULL` are never evaluated**. They are only returned by the
"predefined attributes list" API for display. Treat as not implemented.

### 4.5 The other "completeness" switch

`tbl_attribute_channels.count_against_completeness` (module
`system/operations/readiness/completeness`, `/completeness/*`) is a separate per-channel
per-attribute boolean. The **current** engine does not consult it — the recalculation calls behind
it are commented out and only the legacy `calculateCompletenessByProductIdAndChannelId()` reads
it. Today it is UI state only.

---

## 5. Media — is it there, and how are rules attached?

**Yes, but only as one dedicated rule: "Primary Media".**

```
Primary Media rule  (attribute "Primary Media", criteria countCheck, {min,max})
        │
        ▼   resolvePrimaryMediaCount()
COUNT(*) tbl_product_media_linkers WHERE account, product,
        variant_id = <scope>, channel_id = <scope>,
        AND (language_code IN [lang, LANG, lang-lower] OR language_code IS NULL)
        │
        ├── count > 0                      → use it
        └── count = 0 AND count_inherited  → fallback chain, first hit wins:
                 (variant, master channel) → (parent, same channel) → (parent, master)
        │
        ▼
evaluatePrimaryMediaCount(count, {min,max})   →  PASS / FAIL(min|max|missing)
```

- The rule is fetched with `criteria: 'countCheck'` hard-coded in the query, so changing that
  rule's criteria in the UI silently drops it out of scoring.
- It contributes exactly **1** item to the denominator, no matter how many images exist.
- It is scoped per channel and per variant, and semi-per-language (language-tagged media **and**
  language-agnostic media both count).

**Media-typed attributes** (`media`, `Attachment`) can have rules attached like any other
attribute, but the value extractor has no array/asset handling for them — the value is read as a
string, so a `countCheck` on a media attribute measures **string length**, not asset count. The
diagnostics module flags exactly this (`COUNT_CHECK_NON_MEDIA_ATTRIBUTE`,
`CURRENT_EVALUATOR_TEXT_ONLY_RISK`). Known gap, not a feature.

---

## 6. Language binding — what is and isn't language-bound

Every rule **row** is language-bound (`language_code` is NOT NULL). What differs is how the
*value* behind the rule is resolved:

| Rule | Row per language? | Value language-bound? | Notes |
|---|---|---|---|
| Content attributes (Name, Description, any custom attribute) | yes | **yes** | value read from `tbl_product_attributes` for that `language_code`, with the 3-step fallback below |
| **Primary Media** | yes | partly | media rows matching the language **or** with `language_code = NULL` both count |
| **Stock** | yes (a row exists per language) | **no** | value comes from stock tables that have no language. In master scope it is only evaluated when `languageCode === baseLanguage`, so it is counted once, not once per language |
| **Price** | yes (a row exists per language) | **no** | same as Stock. On Amazon the marketplace's language selects the marketplace, not a translated price |
| Amazon marketplace scope | yes | yes | the marketplace's `language` (from `tbl_amazon_channel_marketplaces`) is the language of the whole evaluation |

Value fallback chain for content attributes — `countInherited` decides whether a fallback value is
allowed to *satisfy* the rule:

```
1. exact scope       : language = requested, channel = requested
2. language fallback : same language, channel = NULL (master value)
3. master fallback   : base language,  channel = NULL
        │
        ▼
getEffectiveReadinessAttributeValue()
   if the value came from a different language/channel AND count_inherited = false
        → value is discarded (treated as empty → rule fails)
   else → value is used, flagged fallbackUsed + valueSource
```

---

## 7. Scoring & where it is written

```
totalItems       = # of enabled, applicable rules in scope
totalFilledItems = # of those that passed
readinessValue   = totalItems === 0 ? 0 : round2(totalFilledItems / totalItems * 100)
```

Denominator details:

- Each *rule* counts 1 (an attribute with 2 rules counts 2).
- A rule whose attribute is in the product's structure but has no value → counts 1, fails
  (`reason: 'missing'`).
- A rule whose attribute is **not part of this product's attribute groups** → `NOT_APPLICABLE`,
  excluded from the denominator.
- Stock/Price on a **parent wrapper product** (a product that has variant-key attributes) →
  `NOT_APPLICABLE`, excluded; they only count on variants and standalone products.
- Stock/Price are added only in the base-language pass (master) or the marketplace-language pass
  (Amazon).
- Primary Media adds 1 whenever its rule exists.

Persisted with `findOrCreate` + update into `tbl_readiness_values`, keyed by
(product, variant, channel, language, account).

---

## 8. Calculation flow (the live path)

```
                       any product mutation
  (content, translation, price, stock, warehouse stock, media add/delete/reorder,
   DAM assign, Shopify metafield, variant structure, bulk updates, action-center)
                                   │
                                   ▼
        OperationalReadinessRecalculationService.recalculateForMutation()
        / .recalculateUniqueTargets()   ── fire-and-forget, concurrency 3
                                   │  + socket emit { readinessRefresh: true }
                                   ▼
        CalculateCompletenessService.calculateCompleteness({ req, productId, channelId? })
                                   │
        ┌──────────────────────────┼───────────────────────────┐
        ▼                          ▼                           ▼
  AMAZON channels           SHOPIFY channels            MASTER (channel = NULL)
  per marketplace           per shopify language        per account language
        │                          │                           │
        ▼                          ▼                           ▼
 calculateCompletenessForMarketPlace1()   calculateCompletenessForNewProduct()
        │                                      │
        └──────────────► same rule loop ◄──────┘

   1. load enabled rules for (account, channel, language), excluding Stock/Price/Primary Media
   2. load product attribute values (+ language fallback + master fallback)
   3. flatten: one entry per (attribute value × rule)
   4. evaluateReadinessCriteria() per entry              → pass / fail
   5. add rules that matched no value at all             → fail, or NOT_APPLICABLE
   6. Stock & Price (base language / marketplace language only) via SQL value resolvers
   7. Primary Media via media COUNT
   8. score → tbl_readiness_values
```

Run per product **and** per variant: the parent (`variant_id = NULL`) and every variant each get
their own score row, at concurrency 5.

Stock value resolution order (`getOperationalStockValue`):
`variant_marketplaces.amazon_stock` → `product_variant_channels.external_stock` →
`product_channel_marketplaces.amazon_stock` → `product_channels.external_stock` →
`product_variant_stocks.in_hand_qty` → `SUM(warehouse_products.total_quantity)`.

Price (`getOperationalPriceValue`): scoped `product_variant_prices` (channel + Amazon
marketplace), using `pricing_details.offerPrice.price` else `price`; if empty and
`count_inherited` is on → master price (channel NULL, marketplace NULL).

---

## 9. Read APIs

| Endpoint | Purpose |
|---|---|
| `POST /product-completeness/product-channel-wise-readiness-value` | stored % per language / channel / marketplace for a product or variant |
| `POST /product-completeness/product-wise-readiness-view` | live rule-by-rule breakdown, master scope (batch variant when `languageCodes` is sent) |
| `POST /product-completeness/marketplace-wise-readiness-view` | same breakdown for an Amazon marketplace scope |
| `POST /product-completeness/test-calculation` | force a recalculation |
| `POST /attribute/get-attribute-readiness-list`, `.../get-predefined-attribute-readiness-list` | rule listing for the settings UI |
| `POST /attribute/create-attribute-readiness`, `PUT /attribute/update-attribute-readiness-configuration` | rule CRUD |
| `POST /readiness-diagnostics/{product, rule, rules-summary, account-summary, collision-report, snapshot, snapshots}` | audit only |

The "view" endpoints re-evaluate live and return, per rule: `attributeName, criteria,
configuration, inherited, isValid, reason, currentValue, valueSource, fallbackUsed,
notApplicableReason`.

---

## 10. Diagnostics module (read-only)

`system/operations/readiness/readiness-diagnostics` runs every rule twice — once with the
**current** evaluator, once with a **shadow** (type-aware) evaluator — and reports the drift:
denominator drift, score drift, `UNSUPPORTED_CRITERIA`, `AMBIGUOUS_SCOPE`, `COLLISION_RISK`, plus
risk-scored snapshots persisted to `logs/readiness-diagnostics-snapshots.json`. It never writes
readiness values. It is effectively the documented list of known engine gaps:

- `LENGTH_CHECK_NON_TEXT_ATTRIBUTE`, `VALUE_CHECK_NON_NUMERIC_ATTRIBUTE`,
  `COUNT_CHECK_NON_MEDIA_ATTRIBUTE` — a criteria applied to the wrong attribute type.
- `CURRENT_EVALUATOR_TEXT_ONLY_RISK` — number/boolean/date/multiselect attributes are evaluated as
  strings by the live engine.
- `SAME_LANGUAGE_MARKETPLACE_COLLISION` — two Amazon marketplaces sharing one language write to
  the same `tbl_readiness_values` row (last write wins).
- `CURRENT_MASTER_SHOPIFY_MEDIA_IGNORES_VARIANT_ID`.

---

## 11. Legacy / dead code — ignore when reasoning about behaviour

| Item | Why |
|---|---|
| `calculateCompletenessForNewProduct1()` | superseded by `calculateCompletenessForNewProduct()`; no callers |
| `getAmazonProductWiseReadinessView()` | no callers |
| `calculateCompletenessByProductIdAndChannelId()` / `calculateProductCompletenessByChannelId()` | old flag-based completeness writing `tbl_product_channels.completeness`; only reachable from itself |
| `ProductChannelMarketPlaceReadinessValue` entity | its `findOrCreate` is commented out; Amazon scores go to `tbl_readiness_values` |
| Amazon `bulletPoints` readiness rows | seeded, never evaluated (§4.4) |
| `count_against_completeness` recalculation calls in `completeness.service.ts` | commented out |
| `ReadinessValueService` | thin `BaseService`, used only to delete rows during product-structure merges |

# Readiness Module — Restructure Plan

> **Scope:** `api/` (plus the webapp call sites that break when routes are renamed).
> **Deliverable of this document:** the target structure and the complete rename ledger.
> **No code has been changed yet.**
>
> **Hard constraints honoured here**
> 1. No database schema changes — no migration is produced or required by this plan.
> 2. No service *logic* changes — every method body moves verbatim; only names, files,
>    module wiring and route strings change.
> 3. Entity TypeScript property names are **not** touched in this pass. They are collected in
>    §9 as a separate, later execution item.
> 4. Renamed HTTP routes are updated on the webapp side in the same pass (§6).

---

## 1. The terminology decision

Three different things in this codebase are currently called "completeness". They are **not** the
same thing, and collapsing them all into "readiness" would be wrong. The rule applied throughout
this plan:

| # | What it actually is | Today's name | Decision |
|---|---|---|---|
| **A** | The rule-driven % stored in `tbl_readiness_values` (the real feature) | `completeness`, `Completeness`, `CalculateCompletenessService`, `/product-completeness/*` | **Becomes `readiness`.** This is the only thing allowed to carry the term. |
| **B** | Variant-combination coverage % written to `tbl_product_variant_channels.completeness` / `tbl_product_channels.completeness` by `ProductVariantService.calculateCompleteness()` | `completeness` | **Not readiness.** Renamed to `variantCombinationCoverage` so it stops competing for the term. Entity property rename deferred (§9). |
| **C** | Action-center sync-snapshot fidelity (`FULL_PROVIDER_SNAPSHOT` / `EVENT_DELTA` / …) | `snapshotCompleteness` | **Not readiness, not product content.** Renamed to `snapshotCoverage` so no "completeness" identifier survives anywhere. |

Prose occurrences of the English word "completeness" inside unrelated documentation strings
(`core/constants/amazon-events/amazon-events.catalog.ts` ×4,
`actionEngineSettings.registry.ts:110`) are reworded to "coverage" / "fully populated" as part of
the sweep, since the requirement is that the word does not appear at all.

**Vocabulary after the restructure** — use exactly these nouns:

| Term | Means |
|---|---|
| **readiness** | the feature as a whole |
| **readiness rule** | one row of `tbl_readiness_configurations` (currently called "readiness configuration") |
| **readiness criteria** | `presenceCheck` / `lengthCheck` / `countCheck` / `valueCheck` |
| **readiness score / readiness value** | the persisted % in `tbl_readiness_values` |
| **readiness view** | the live, rule-by-rule breakdown returned to the UI |
| **readiness recalculation** | the event-driven "something changed → rescore" fan-in |

---

## 2. Where the code lives today

```
api/apps/api-main/src/
├── entities/
│   ├── readinessConfigurations.entity.ts                       (rules)
│   ├── readinessValue.entity.ts                                (scores)
│   └── marketplaceReadinessValue.entity.ts                     (declared, unused)
│
├── modules/app/catalog/products/listings/
│   ├── completeness/
│   │   ├── calculate-completeness.service.ts        3332 lines  ← the whole engine
│   │   ├── calculate-completeness.controller.ts       74 lines
│   │   ├── operational-readiness-recalculation.service.ts 258 lines
│   │   └── product-completeness.module.ts             31 lines
│   └── marketplace/readiness/
│       ├── readiness-value.service.ts                 10 lines  (thin BaseService)
│       └── readiness-value.module.ts                   9 lines
│
└── modules/app/catalog/attributes/attribute-core/
    ├── attribute.controller.ts     → 5 readiness routes bolted onto /attribute
    ├── attribute.service.ts        → createAttributeReadiness(), updateAttributeReadinessConfiguration(),
    │                                 getAttributesListForReadiness() + 5 private readiness helpers
    ├── dto/create-attribute-readiness.dto.ts
    ├── dto/update-readiness-configuration.dto.ts
    └── readinessConfigurations/
        ├── readiness-configurations.service.ts       628 lines  (seeding, listing, removal, recalc trigger)
        └── dto/rediness-config.dto.ts                          (note: filename is misspelled)
```

Three problems this plan fixes:

1. **Readiness is spread across three unrelated homes** — `products/listings/completeness`,
   `products/listings/marketplace/readiness`, and `attributes/attribute-core`. There is no single
   module you can point at.
2. **One 3332-line god service** that is simultaneously the evaluator, the value resolver, the
   SQL layer, the persistence layer and the read API.
3. **Rule CRUD lives in the attribute module** and is exposed on `/attribute/*`, so readiness rules
   look like an attribute feature rather than a readiness feature.

---

## 3. Target structure

One absolute module, `ReadinessModule`, at `catalog/products/readiness/`, composed of five
sub-modules plus shared `dto/`, `types/` and `constants/`.

```
api/apps/api-main/src/modules/app/catalog/products/readiness/
│
├── readiness.module.ts                     ← THE single aggregator; the only thing app.module.ts
│                                             and every consumer imports
├── constants/
│   └── readiness.constants.ts              ← criteria keys, concurrency limits, predefined-rule
│                                             defaults, primary-attribute names
├── types/
│   └── readiness.types.ts                  ← ReadinessScope, ReadinessRuleResult,
│                                             ReadinessEvaluation, ReadinessRecalculationSource…
├── dto/
│   ├── index.ts
│   ├── readiness-rule-list.dto.ts
│   ├── create-readiness-rule.dto.ts
│   ├── update-readiness-rule.dto.ts
│   ├── seed-readiness-rules.dto.ts
│   ├── readiness-value.dto.ts
│   ├── readiness-view.dto.ts
│   └── marketplace-readiness-view.dto.ts
│
├── rules/                                  ← WHAT is measured (config surface)
│   ├── readiness-rules.module.ts
│   ├── readiness-rules.controller.ts       ← /readiness/rules/*
│   ├── readiness-rules.service.ts          ← list, create, update, delete, duplicate guard
│   └── readiness-rules-seeder.service.ts   ← predefined rules per language/channel/marketplace
│
├── engine/                                 ← HOW it is measured (evaluation)
│   ├── readiness-engine.module.ts
│   ├── readiness-engine.service.ts         ← the orchestrator: calculateReadiness()
│   ├── readiness-scope.resolver.ts         ← which (channel × language × marketplace × variant)
│   │                                         scopes exist for a product
│   ├── readiness-criteria.evaluator.ts     ← the 4 criteria + min/max semantics
│   ├── readiness-attribute-value.resolver.ts   ← attribute value + inheritance fallback chain
│   ├── readiness-operational-value.resolver.ts ← stock & price SQL resolution
│   └── readiness-media.resolver.ts         ← primary-media count + media fallback chain
│
├── scoring/                                ← the NUMBER and its storage
│   ├── readiness-scoring.module.ts
│   ├── readiness-score.calculator.ts       ← totals → percentage (pure, no I/O)
│   └── readiness-value.service.ts          ← tbl_readiness_values persistence (BaseService)
│
├── recalculation/                          ← WHEN it is measured
│   ├── readiness-recalculation.module.ts
│   └── readiness-recalculation.service.ts  ← mutation fan-in, concurrency, socket refresh
│
└── views/                                  ← READ side
    ├── readiness-views.module.ts
    ├── readiness.controller.ts             ← /readiness/*
    └── readiness-view.service.ts           ← live rule-by-rule breakdowns
```

### Dependency direction (no new cycles inside the module)

```
                       readiness.module.ts
                              │
        ┌──────────┬──────────┼───────────┬────────────┐
        ▼          ▼          ▼           ▼            ▼
     rules/     engine/    scoring/  recalculation/  views/
        │          │          ▲           │            │
        │          └──────────┘           │            │
        │       engine persists via       │            │
        │            scoring              │            │
        └──────────────► engine ◄─────────┴────────────┘
             rules feed engine;      recalculation and views
             engine never imports    both call engine
             the rules controller
```

- `scoring/` depends on nothing inside the module (leaf).
- `engine/` depends on `scoring/` + `rules/` (read-only rule fetch).
- `recalculation/` and `views/` depend on `engine/`.
- `rules/` depends on `recalculation/` **only** for the "rule changed → rescore affected products"
  trigger — expressed as a `forwardRef` inside the module. That is the one intra-module cycle and
  it already exists today (`ReadinessConfigurationsService` ⇄ `CalculateCompletenessService`), so
  removing it would be a logic change and is out of scope.

### Why the rule surface moves out of `attribute-core`

Rule CRUD currently sits in `AttributeService` purely because rules point at attributes. After the
move, `attribute-core` keeps only what it owns (attributes), and `ReadinessRulesService` queries
attributes through the existing `AttributeService` — a normal cross-module dependency in the same
direction the engine already uses. `attribute.controller.ts` loses 5 routes; `attribute.service.ts`
loses 3 public methods and 5 private helpers.

---

## 4. File-by-file migration map

### 4.1 Files that move / split

| Today | Becomes | Notes |
|---|---|---|
| `listings/completeness/calculate-completeness.service.ts` (3332) | split across `engine/*` + `views/*` + `scoring/*` | see §4.2 |
| `listings/completeness/calculate-completeness.controller.ts` (74) | `views/readiness.controller.ts` | routes renamed (§6) |
| `listings/completeness/operational-readiness-recalculation.service.ts` (258) | `recalculation/readiness-recalculation.service.ts` | bodies verbatim; identifiers renamed |
| `listings/completeness/product-completeness.module.ts` (31) | `readiness.module.ts` + 5 sub-module files | |
| `listings/marketplace/readiness/readiness-value.service.ts` (10) | `scoring/readiness-value.service.ts` | verbatim |
| `listings/marketplace/readiness/readiness-value.module.ts` (9) | folded into `scoring/readiness-scoring.module.ts` | |
| `attribute-core/readinessConfigurations/readiness-configurations.service.ts` (628) | split: `rules/readiness-rules.service.ts` + `rules/readiness-rules-seeder.service.ts` | see §4.3 |
| `attribute-core/readinessConfigurations/dto/rediness-config.dto.ts` | `readiness/dto/readiness-rule-list.dto.ts` + `readiness/dto/seed-readiness-rules.dto.ts` | fixes the `rediness` typo |
| `attribute-core/dto/create-attribute-readiness.dto.ts` | `readiness/dto/create-readiness-rule.dto.ts` | drop from `attribute-core/dto/index.ts` |
| `attribute-core/dto/update-readiness-configuration.dto.ts` | `readiness/dto/update-readiness-rule.dto.ts` | drop from `attribute-core/dto/index.ts` |

**Folders deleted after the move:** `listings/completeness/`,
`listings/marketplace/readiness/`, `attribute-core/readinessConfigurations/`.

### 4.2 Splitting `calculate-completeness.service.ts`

Every member below moves **verbatim** (body unchanged); only its owning class and, where noted,
its name change. Line numbers refer to the current file.

| Lines | Current member | New home | New name |
|---|---|---|---|
| 74–78 | `getReadinessConfigAttributeName`, `getReadinessConfigKey` | `engine/readiness-engine.service.ts` | `getReadinessRuleAttributeName`, `getReadinessRuleKey` |
| 89 | `getUniqueReadinessConfigurations` | engine | `getUniqueReadinessRules` |
| 99–109 | `getProductAttributeKey`, `getUniqueReadinessProductAttributes` | `engine/readiness-attribute-value.resolver.ts` | unchanged |
| 119 | `getStructuredAttributeIdsForProductGroups` | `engine/readiness-scope.resolver.ts` | unchanged |
| 144–176 | `normalizeReadinessValue`, `getReadinessAttributeValue`, `normalizeNullableNumber` | `readiness-attribute-value.resolver.ts` | unchanged |
| 182–206 | `getProductAttributeScope`, `getProductAttributeValueSource`, `getEffectiveReadinessAttributeValue` | `readiness-attribute-value.resolver.ts` | unchanged |
| 231 | `evaluateReadinessCriteria` | `engine/readiness-criteria.evaluator.ts` | unchanged |
| 280 | `isParentWrapperProduct` | `readiness-scope.resolver.ts` | unchanged |
| 307–336 | `isOperationalPrimaryAttribute`, `getOperationalPrimaryNotApplicableResult`, `getUnevaluatedReadinessConfigResults` | engine | last one → `getUnevaluatedReadinessRuleResults` |
| 392–418 | `toNumberOrNull`, `hasOperationalValue`, `queryNumericValue`, `getVariantPredicate` | `readiness-operational-value.resolver.ts` | unchanged |
| 420 | `getAmazonMarketplaceIdForReadinessScope` | `readiness-scope.resolver.ts` | unchanged |
| 455–581 | `getOperationalStockValue`, `getOperationalPriceValue` | `readiness-operational-value.resolver.ts` | unchanged |
| 651–677 | `evaluateOperationalPrimaryCriteria`, `evaluateOperationalPrimaryReadinessConfig` | `readiness-operational-value.resolver.ts` | last one → `evaluateOperationalPrimaryReadinessRule` |
| 716–884 | `getPrimaryMediaCondition`, `countPrimaryMediaForReadiness`, `getPrimaryMediaSource`, `getPrimaryMediaFallbackScopes`, `resolvePrimaryMediaCount`, `evaluatePrimaryMediaCount`, `evaluatePrimaryMediaReadinessConfig` | `engine/readiness-media.resolver.ts` | last one → `evaluatePrimaryMediaReadinessRule` |
| 884–900 | `READINESS_CONCURRENCY_LIMIT`, `mapWithConcurrency` | `constants/readiness.constants.ts` + engine util | |
| 902 | `calculateCompleteness` | `engine/readiness-engine.service.ts` | **`calculateReadiness`** |
| 1107–1144 | `getProductChannels`, `getChannelAttributes` | `readiness-scope.resolver.ts` | unchanged |
| 1187 | `getProductChannelWiseReadinessValue` | `views/readiness-view.service.ts` | `getProductReadinessValues` |
| 1393 | `getProductWiseReadinessView` | `views/readiness-view.service.ts` | `getProductReadinessBreakdown` |
| 1680 | `calculateCompletenessForNewProduct` | `engine/readiness-engine.service.ts` | **`calculateReadinessForScope`** |
| 2067 | `calculateCompletenessForMarketPlace1` | `engine/readiness-engine.service.ts` | **`calculateReadinessForMarketplace`** (the `1` suffix goes — its dead twin `…ForNewProduct1` and `getAmazonProductWiseReadinessView` are already removed) |
| 2436 | `getReadinessView` | `views/readiness-view.service.ts` | unchanged |
| 2687 | `getReadinessViewBatch` | `views/readiness-view.service.ts` | unchanged |
| 2976 | `getMarketplaceWiseReadinessView` | `views/readiness-view.service.ts` | `getMarketplaceReadinessBreakdown` |

The `findOrCreate` + update against `tbl_readiness_values` that currently happens inline inside
`calculateCompletenessForNewProduct` / `…ForMarketPlace1` is lifted — unchanged — into
`scoring/readiness-value.service.ts` as `persistReadinessScore()`, and the
`totalFilledItems / totalItems × 100` arithmetic into `scoring/readiness-score.calculator.ts` as
`calculateReadinessPercentage()`. Both are pure moves, called from the same place in the same
order, with the same rounding.

### 4.3 Splitting `readiness-configurations.service.ts`

| Lines | Current member | New home | New name |
|---|---|---|---|
| 33 | `getAttributeReadinessList` | `rules/readiness-rules.service.ts` | `getReadinessRules` |
| 126 | `getPreDefinedAttributesList` | `rules/readiness-rules.service.ts` | `getPredefinedReadinessRules` |
| 231 | `createPreDefinedReadinessConfigurations` | `rules/readiness-rules-seeder.service.ts` | `seedPredefinedReadinessRules` |
| 482 | `removeReadinessConfigurations` | `rules/readiness-rules-seeder.service.ts` | `removeReadinessRules` |
| 515 | `updateReadinessToAllAffectedProduct` | `rules/readiness-rules.service.ts` | `recalculateReadinessForRuleChange` |
| 588 | `getRuleChangeAffectedProductIds` | `rules/readiness-rules.service.ts` | unchanged (private) |

Moved out of `attribute.service.ts` into `rules/readiness-rules.service.ts`:

| Line | Member | New name |
|---|---|---|
| 429 | `createAttributeReadiness` | `createReadinessRules` |
| 1147 | `updateAttributeReadinessConfiguration` | `updateReadinessRules` |
| 2194 | `getAttributesListForReadiness` | `getAttributesAvailableForReadinessRules` |
| 2096 | `escapeReadinessLikePattern` | unchanged (private) |
| 2100 | `getReadinessAttributeSourceProvider` | unchanged (private) |
| 2124 | `getReadinessSourceFilterCondition` | unchanged (private) |
| 2157 | `getReadinessTypeFilterValues` | unchanged (private) |
| 2176 | `getReadinessSearchCondition` | unchanged (private) |

`attribute.service.ts:1012` (delete a deleted attribute's rules) and `:232` (seed predefined rules
when an attribute is created) stay where they are and simply call the renamed services.

---

## 5. Symbol rename ledger (api)

### 5.1 Classes / providers / modules

| Old | New | Declared in |
|---|---|---|
| `CalculateCompletenessService` | `ReadinessEngineService` | `engine/readiness-engine.service.ts` |
| `CalculateCompletenessModule` | `ReadinessModule` | `readiness.module.ts` |
| `ProductCompleteNessController` | `ReadinessController` | `views/readiness.controller.ts` |
| `OperationalReadinessRecalculationService` | `ReadinessRecalculationService` | `recalculation/readiness-recalculation.service.ts` |
| `ReadinessConfigurationsService` | `ReadinessRulesService` (+ new `ReadinessRulesSeederService`) | `rules/` |
| `ReadinessValueService` | `ReadinessValueService` *(name kept, new home)* | `scoring/readiness-value.service.ts` |
| `ReadinessValueModule` | folded into `ReadinessScoringModule` | `scoring/readiness-scoring.module.ts` |
| — | new: `ReadinessCriteriaEvaluator`, `ReadinessAttributeValueResolver`, `ReadinessOperationalValueResolver`, `ReadinessMediaResolver`, `ReadinessScopeResolver`, `ReadinessScoreCalculator`, `ReadinessViewService`, `ReadinessRulesController` | |

### 5.2 Injected field names

| Old | New |
|---|---|
| `_calculateCompletenessService` | `_readinessEngineService` |
| `_operationalReadinessRecalculationService` | `_readinessRecalculationService` |
| `_readinessConfigurationsService` | `_readinessRulesService` / `_readinessRulesSeederService` |
| `_productAttributeAttachmentService` (controller field, misnamed) | `_readinessViewService` |

Consumer files touched by §5.1–5.3 (35):

```
app.module.ts
modules/app/catalog/catalog.module.ts
modules/app/catalog/attributes/attribute-core/attribute.module.ts
modules/app/catalog/attributes/attribute-core/attribute.service.ts
modules/app/catalog/attributes/attribute-core/attribute.controller.ts
modules/app/catalog/channel/channel.service.ts
modules/app/catalog/channel/amazonChannel/amazon-channel.service.ts
modules/app/catalog/channel/shopifyChannel/shopify-channel.service.ts
modules/app/catalog/products/product.module.ts
modules/app/catalog/products/product.service.ts
modules/app/catalog/products/lifecycle/update-details.service.ts
modules/app/catalog/products/listings/product-channel.module.ts
modules/app/catalog/products/listings/product-channel.service.ts
modules/app/catalog/products/pricing/product-pricing.module.ts
modules/app/catalog/products/pricing/product-pricing.service.ts
modules/app/catalog/products/stock/product-stock.module.ts
modules/app/catalog/products/stock/product-stock.service.ts
modules/app/catalog/products/variants/product-variant.module.ts
modules/app/catalog/products/variants/product-variant.service.ts
modules/app/catalog/products/variants/product-variants-flow.service.ts
modules/app/catalog/products/shopifyProduct/shopify-product.module.ts
modules/app/catalog/products/shopifyProduct/shopify-product.service.ts
modules/app/commerce/billing/customer-subscription/customer-subscription.module.ts
modules/app/identity/accounts/account/account.module.ts
modules/app/identity/accounts/account/accountConfiguration/account-configuration.service.ts
modules/app/inventory/warehouses/warehouse/warehouse.module.ts
modules/app/inventory/warehouses/warehouse-stock/warehouse-stock.module.ts
modules/app/inventory/warehouses/warehouse-stock/warehouse-stock.service.ts
modules/app/platforms/amazon/channel-config/marketplace/amazon-channel-marketplace.service.ts
modules/app/sync/sync-orchestration.module.ts
modules/app/sync/productSyncing/amazon/backwardSync/amazon-product-bs.service.ts
modules/app/sync/productSyncing/shopify/backwardSync/shopify-product-bsq3.service.ts
modules/app/system/operations/action-center/action-center.module.ts
modules/app/system/operations/action-center/action-center.service.ts
modules/app/system/operations/monitoring/live-insight/live-insight.module.ts
```

### 5.3 Method-call renames at call sites

`calculateCompleteness({ req, productId, channelId })` becomes `calculateReadiness({ ... })` — 8 sites:

```
readiness-configurations.service.ts:559            (moves into rules/)
operational-readiness-recalculation.service.ts:102 (moves into recalculation/)
calculate-completeness.controller.ts:21            (moves into views/)
product-channel.service.ts:5768, 5995
product.service.ts:1888
account-configuration.service.ts:1231
amazon-product-bs.service.ts:4509
shopify-product-bsq3.service.ts:2759
```

`getProductChannelWiseReadinessValue()` becomes `getProductReadinessValues()` —
`product-channel.service.ts:313`, `:3191`.

`createPreDefinedReadinessConfigurations()` becomes `seedPredefinedReadinessRules()` — 7 sites:
`attribute.service.ts:232`, `amazon-channel.service.ts:449`, `channel.service.ts:878`,
`shopify-channel.service.ts:1184` and `:1297`, `account-configuration.service.ts:360`,
`amazon-channel-marketplace.service.ts:293`.

`removeReadinessConfigurations()` becomes `removeReadinessRules()` — 3 sites:
`channel.service.ts:882`, `account-configuration.service.ts:364`,
`amazon-channel-marketplace.service.ts:303`.

`updateReadinessToAllAffectedProduct()` becomes `recalculateReadinessForRuleChange()` — 2 sites:
`attribute.service.ts:531`, `:1199` (both move into `rules/`).

### 5.4 Type renames

| Old | New | File |
|---|---|---|
| `OperationalReadinessRecalculationSource` | `ReadinessRecalculationSource` | `types/readiness.types.ts` |
| `OperationalReadinessRecalculationTarget` | `ReadinessRecalculationTarget` | idem |
| `OperationalReadinessRecalculationOptions` | `ReadinessRecalculationOptions` | idem |
| `GetReadinessListDTO` | `ReadinessRuleListDto` | `dto/readiness-rule-list.dto.ts` |
| `CreatePreDefinedReadinessConfigurationsDTO` | `SeedReadinessRulesDto` | `dto/seed-readiness-rules.dto.ts` |
| `CreateAttributeReadinessDTO` | `CreateReadinessRuleDto` | `dto/create-readiness-rule.dto.ts` |
| `UpdateReadinessConfigurationDto` | `UpdateReadinessRuleDto` | `dto/update-readiness-rule.dto.ts` |
| `ActionCenterSyncEvaluationSnapshotCompleteness` | `ActionCenterSyncEvaluationSnapshotCoverage` | `action-center-sync-evaluation.service.ts` (category C) |

The 32 `ReadinessRecalculationSource` string literals (`'STOCK_UPDATED'`,
`'PRODUCT_MEDIA_REORDERED'`, ...) keep their values — they are logged, not persisted, and renaming
them buys nothing.

### 5.5 Structured-log event names

Emitted by the recalculation service; log payloads only, no consumer contract:

| Old | New |
|---|---|
| `operational_readiness_recalculation_started` | `readiness_recalculation_started` |
| `operational_readiness_recalculation_completed` | `readiness_recalculation_completed` |
| `operational_readiness_recalculation_failed` | `readiness_recalculation_failed` |
| `operational_readiness_recalculation_skipped` | `readiness_recalculation_skipped` |
| `operational_readiness_recalculation_batch_started` | `readiness_recalculation_batch_started` |
| `operational_readiness_refresh_emit_failed` | `readiness_refresh_emit_failed` |

### 5.6 Category B — variant combination coverage (`ProductVariantService`)

| Old | New |
|---|---|
| `calculateVariantAttributeCompleteness()` | `calculateVariantCombinationCoverage()` |
| `calculateCompleteness(req, variantIds, isCountCompleteness, data)` | `calculateCombinationCoverage(req, variantIds, isChannelScoped, data)` |
| local `completenessValue` | `coverageValue` |
| the `console.error` label mentioning the old method name | updated to the new name |

Call sites: `product-variant.service.ts:257`, `:641`, `:1209`. The `{ completeness: ... }` update
payloads keep the **entity property name** for now (§9).

### 5.7 Category C — action-center snapshot coverage

`snapshotCompleteness` becomes `snapshotCoverage` across
`action-center-sync-evaluation.service.ts` (24 occurrences), `action-center.controller.ts:458`,
`amazon-product-fsq3.service.ts:700` and `:746`,
`amazon-events-action-center-materialization.service.ts:280`, plus the mirrored FE type at
`webapp/src/lib/action-center-remediation-navigation.ts:563`. The enum **values**
(`'SCOPED_PROVIDER_SNAPSHOT'`, `'FULL_PROVIDER_SNAPSHOT'`, `'DETECTOR_PROJECTION'`,
`'EVENT_DELTA'`, `'UNKNOWN'`) are unchanged — they are persisted inside action-center snapshot
JSON.

---

## 6. Route surface

Two controllers, one prefix. HTTP verbs and request/response bodies are unchanged — this pass is a
rename only. (The "minimal, consistent GET endpoints" idea from `readiness-issues.md` is a
separate, later task; converting POST to GET here would force a second FE rewrite.)

### 6.1 Readiness values and views — `ReadinessController` (`@Controller('readiness')`)

| Old route | New route | Handler |
|---|---|---|
| `POST /product-completeness/product-channel-wise-readiness-value` | `POST /readiness/product-values` | `getProductReadinessValues` |
| `POST /product-completeness/product-wise-readiness-view` | `POST /readiness/product-view` | `getProductReadinessBreakdown` / `getReadinessViewBatch` (the `languageCodes`-present branch stays inside the handler, unchanged) |
| `POST /product-completeness/marketplace-wise-readiness-view` | `POST /readiness/marketplace-view` | `getMarketplaceReadinessBreakdown` |
| `POST /product-completeness/test-calculation` | `POST /readiness/recalculate` | `calculateReadiness` |

### 6.2 Readiness rules — `ReadinessRulesController` (`@Controller('readiness/rules')`)

| Old route | New route | Handler |
|---|---|---|
| `POST /attribute/get-attribute-readiness-list` | `POST /readiness/rules/list` | `getReadinessRules` |
| `POST /attribute/get-predefined-attribute-readiness-list` | `POST /readiness/rules/predefined-list` | `getPredefinedReadinessRules` |
| `POST /attribute/create-attribute-readiness` | `POST /readiness/rules/create` | `createReadinessRules` |
| `PUT /attribute/update-attribute-readiness-configuration` | `PUT /readiness/rules/update` | `updateReadinessRules` |
| `POST /attribute/get-attributes-list-for-readiness` | `POST /readiness/rules/available-attributes` | `getAttributesAvailableForReadinessRules` |

### 6.3 Deliberately left alone

`POST /products/product-readiness-value` (`products.controller.ts:1162` →
`ProductService.productReadinessValue`) already reads unambiguously and belongs to the product
list/summary surface rather than the readiness module. It is **not** moved or renamed. Flagged
here so the decision is explicit rather than an oversight — see §12.

### 6.4 Webapp call sites to update in the same pass

| File | Line | Change |
|---|---|---|
| `webapp/src/lib/api.ts` | 80 | `GET_PRODUCT_READINESS_VALUES: 'product-completeness/product-channel-wise-readiness-value'` becomes `'readiness/product-values'` |
| `webapp/src/lib/api.ts` | 146 | `GET_ATTRIBUTES_LIST_FOR_READINESS: 'attribute/get-attributes-list-for-readiness'` becomes `'readiness/rules/available-attributes'` |
| `webapp/src/lib/api.ts` | 147 | `CREATE_ATTRIBUTE_READINESS: 'attribute/create-attribute-readiness'` becomes `'readiness/rules/create'` |
| `webapp/src/lib/api.ts` | — | **add** `GET_READINESS_RULES`, `GET_PREDEFINED_READINESS_RULES`, `UPDATE_READINESS_RULES`, `GET_PRODUCT_READINESS_VIEW`, `GET_MARKETPLACE_READINESS_VIEW` |
| `.../products/[product]/_components/readiness/_components/readinessDetailDialog.tsx` | 372, 421, 452 | three hardcoded `product-completeness/*` strings become the new `api.ts` builders (no hardcoded URLs in components, per the root CLAUDE.md contract) |
| `.../products/[product]/_components/action-center/ProductActionCenterIssueDrawer.tsx` | 14839 | hardcoded `'product-completeness/product-wise-readiness-view'` becomes a builder |
| `.../readiness/_components/list/List.tsx` | 284, 363, 498 | three hardcoded `attribute/*-readiness-*` strings become builders |
| `.../readiness/_components/list/_components/addCriteriaDialog.tsx` | 440, 501 | already uses builders — nothing beyond the `api.ts` value swap |

Unchanged FE call sites (they hit routes that are not renamed): `readiness-tab.tsx:168`,
`readinessTooltip.tsx:66`, `ProductActionCenterIssueDrawer.tsx:14811`.

### 6.5 Permissions registry — `common/security/permissions.data.ts`

| Line | Change |
|---|---|
| 279–281 | the three `/product-completeness/*` route strings become the new `/readiness/*` strings |
| 282 | `'/completeness/get-completeness-list'` — **delete**. Dead: the `/completeness` controller (the `count_against_completeness` per-channel flag module) no longer exists in the codebase. |
| 1425–1434 (`productReadiness.*`) | `/attribute/create-attribute-readiness`, `/attribute/update-attribute-readiness-configuration`, `/attribute/get-attributes-list-for-readiness` become the new `/readiness/rules/*` strings; add the two list routes |
| 1799–1802 | the `{ permissions: ['completeness.*', 'completeness.list'], path: 'completeness' }` entry — **delete** (dead route group; the live one is `productReadiness.*` → `readiness` at 1803–1806) |

Mirror in `webapp/src/lib/authenticated-route-inventory.ts:52–53` — delete the same
`completeness` entry.

---

## 7. i18n keys

### 7.1 Delete — orphaned by the removed diagnostics module

In `core/i18n/english.ts` and its four siblings (`french`, `german`, `spanish`, `roman`):

```
readiness_product_diagnostics_fetched
readiness_account_summary_fetched
readiness_rules_summary_fetched
readiness_collision_report_fetched
readiness_snapshot_created
readiness_snapshots_fetched
```

`readiness_rules_fetched` is **kept** — it becomes the success key for `POST /readiness/rules/list`.

### 7.2 Rename — category A

Applies identically to all five language files; only the key changes, and the translated string is
reworded from Completeness / Complétude / Vollständigkeit / Completitud / Completitudine to the
readiness wording already present in the same file.

| Old key | New key | en line |
|---|---|---|
| `completeness` | `readiness` | 246 |
| `completeness_fetched` | `readiness_fetched` | 374 |
| `completeness_updated` | `readiness_updated` | 375 |
| `completeness_fetch_failed` | `readiness_fetch_failed` | 939 |
| `completeness_update_failed` | `readiness_update_failed` | 940 |
| `completeness_calculation_failed` | `readiness_calculation_failed` | 976 |

Same six lines per sibling: french 254/380/381/948/949/985 · german 253/383/384/955/956/992 ·
spanish 256/384/385/949/950/986 · roman 253/381/382/950/951/987.
The `// Completeness messages` section comments (en 373 and 938, plus siblings) become
`// Readiness messages`.

### 7.3 `core/i18n/translationData/*.ts`

| Key | Change | Files |
|---|---|---|
| `countAgainstCompleteness` | becomes `countAgainstReadiness` | en:56, de:58, es:56, fr:57, ro:56 — **this key mirrors the entity property, so it moves with §9, not in this pass.** Listed here so it is not lost. |
| `isPublishingEnabled` value `'Publishing allowed only for products with completeness 100%'` | reworded to `'... with readiness 100%'` | en:65 and siblings — value only, safe now |

### 7.4 Renamed message keys used by the moved handlers

```
readiness_configurations_created          becomes  readiness_rules_created
readiness_configurations_fetched          becomes  readiness_rules_list_fetched
attribute_readiness_configuration_updated becomes  readiness_rules_updated
attribute_readiness_not_found             becomes  readiness_rule_not_found
```

`readiness_attributes_fetched` (en:310) and `readiness_view_fetched` (en:427) are already
correctly named and stay as they are.

---

## 8. Sockets

`modules/app/system/socket/socket.config.ts:52–53`

```ts
// Completeness value on Product Detail screen
PRODUCT_COMPLETENESS: 'PRODUCT_COMPLETENESS',
```

This constant has **zero emitters and zero listeners in `api/`** — the live refresh signal is
actually the `readinessRefresh: true` flag inside `emitProductErrorSocket`, consumed at
`webapp/src/app/(index)/(menu-layout)/products/[product]/page.tsx:689`. Two options; the plan takes
the first:

1. **Delete it** from `socket.config.ts` and from the FE mirror
   `webapp/src/app/(index)/(menu-layout)/_components/socket/socket.ts:59`. Nothing breaks.
2. If it is reserved for planned work, rename it to `PRODUCT_READINESS: 'PRODUCT_READINESS'` in
   both files instead.

---

## 9. Deferred — database fields and entity TypeScript keys

**Not executed in this pass.** These need a migration and/or an API-payload contract change, both
out of scope here. Recorded so the terminology cleanup can be finished later in one shot.

### 9.1 Database columns still carrying the word

| Table | Column | Category | Proposed |
|---|---|---|---|
| `tbl_attribute_channels` | `count_against_completeness` | A | `count_against_readiness` |
| `tbl_product_channels` | `completeness` | B | `variant_combination_coverage` |
| `tbl_product_variant_channels` | `completeness` | B | `variant_combination_coverage` |

### 9.2 Entity TypeScript properties still carrying the word

| Entity | Property | Proposed |
|---|---|---|
| `entities/attribtueChannels.entity.ts:38` | `countAgainstCompleteness` | `countAgainstReadiness` |
| `entities/productChannel.entity.ts:158` | `completeness` | `variantCombinationCoverage` |
| `entities/productVariantChannels.entity.ts:54` | `completeness` | `variantCombinationCoverage` |

The entity filename `attribtueChannels.entity.ts` is also misspelled — worth fixing in the same
later pass.

### 9.3 Blast radius when §9 is executed

These property names travel straight out into API response bodies, so the following must change
together:

**api — `countAgainstCompleteness` (13 sites):**
`attribute.service.ts` 285, 958, 1248, 1308, 1498, 1513, 1572, 1944, 2758 ·
`product-variant.service.ts` 1240 · `entities/attribtueChannels.entity.ts` 38 ·
`core/i18n/translationData/{en,de,es,fr,ro}.ts`

**api — `completeness` column reads/writes (14 sites):**
`product-channel.service.ts` 299, 325, 1193, 1209, 1298, 1314, 3095 ·
`product-overview.service.ts` 447 · `product.service.ts` 6407, 16451 ·
`product-variants-flow.service.ts` 2405 (raw SQL `jsonb_build_object`) ·
`live-insight.service.ts` 344 (raw SQL) · `product-variant.service.ts` 1301, 1322

**webapp (payload and table-config keys):**
`attributes/_components/list/List.tsx:648` ·
`products/[product]/hooks/useOverviewData.tsx` 121–145, 308–309 ·
`products/_components/gridList/_components/gridItem.tsx:498` · `gridFooter.tsx:114` ·
`lib/globalVariables.ts` 4064, 4127, 4184 (`product_completeness` column key) ·
`lib/tableConfigurationsConstants.ts` 511–518 (`defaultCompleteNessTableSettings`,
`countedForCompleteNess`) · `store/completeness.store.ts` (whole file, becomes
`readiness-settings.store.ts`) · `types/index.types.ts:198` and `lib/constants.ts:1269`
(`productCompletenessChannels`) ·
`readiness/_components/list/_components/completeness-tooltip.tsx` (`CompletenessTooltip`) ·
the `completeness: 'all'` filter key in `product.store.ts`, `product-variant.store.ts`,
`digitalAssets.store.ts`, `product-file-attachment.store.ts`, `readiness.store.ts`

---

## 10. Execution order

Each step compiles on its own and is independently revertible.

| # | Step | Touches |
|---|---|---|
| 1 | Create the `readiness/` skeleton: `constants/`, `types/`, `dto/` (move the 4 DTO files, fix the `rediness` typo, rename the classes, drop them from `attribute-core/dto/index.ts`) | new files + 4 deletions |
| 2 | Move `scoring/`: `ReadinessValueService` verbatim; add `ReadinessScoreCalculator` and `persistReadinessScore()` lifted out of the engine | 3 files |
| 3 | Split the engine into the 6 `engine/*` files per §4.2, moving bodies verbatim. Keep `CalculateCompletenessService` as a temporary re-export shim so nothing else breaks yet | new files |
| 4 | Move `recalculation/`; rename its class, types and log events | 1 file |
| 5 | Move `views/`: the view methods plus the controller with the **new** routes | 2 files |
| 6 | Move `rules/`: split `readiness-configurations.service.ts`, pull the 3 public + 5 private members out of `attribute.service.ts`, create `ReadinessRulesController` | ~5 new files + `attribute.service.ts`, `attribute.controller.ts` |
| 7 | Write `readiness.module.ts` and the 5 sub-module files; register in `app.module.ts` and `catalog.module.ts`; delete `product-completeness.module.ts` and `readiness-value.module.ts` | ~8 files |
| 8 | Sweep consumers: `CalculateCompletenessModule`/`Service` become `ReadinessModule`/`ReadinessEngineService`, field names, method names (§5.2, §5.3); **remove the shim from step 3** | 35 files |
| 9 | Permissions and socket config (§6.5, §8) | 2 api files, 2 webapp files |
| 10 | i18n: delete orphans, rename keys, reword strings (§7) | 10 api files |
| 11 | Category B rename in `ProductVariantService` (§5.6) | 1 file |
| 12 | Category C `snapshotCompleteness` rename (§5.7) | 4 api files, 1 webapp file |
| 13 | Reword the prose "completeness" in `amazon-events.catalog.ts` (135, 359, 493, 551) and `actionEngineSettings.registry.ts` (110) | 2 files |
| 14 | Webapp: `lib/api.ts` builders and de-hardcoding the 7 inline URL strings (§6.4) | 4 files |
| 15 | Delete the emptied folders `listings/completeness/`, `listings/marketplace/readiness/`, `attribute-core/readinessConfigurations/` | — |

---

## 11. Verification

```bash
# 1. no identifier or route carries the word any more
cd api && grep -rniE "completeness" apps/api-main/src --include=*.ts

# expected surviving hits after this pass, and nothing else:
#   entities/attribtueChannels.entity.ts        countAgainstCompleteness + its field mapping
#   entities/productChannel.entity.ts           completeness
#   entities/productVariantChannels.entity.ts   completeness
#   the ~27 read/write sites of those three properties listed in 9.3
#   core/i18n/translationData/*.ts              countAgainstCompleteness

# 2. no folder or file name carries the word
find apps/api-main/src -iname "*complete*"      # must be empty

# 3. webapp: no stale route strings
cd ../webapp && grep -rniE "product-completeness|attribute/(get|create|update)-[a-z-]*readiness" src

# 4. build + lint both repos
cd ../api && pnpm build:all && pnpm lint
cd ../webapp && pnpm build && pnpm lint
```

Behavioural check — no logic changed, so the numbers must be identical. Pick a product with a
non-trivial score, call `POST /readiness/recalculate`, and confirm `tbl_readiness_values` holds the
same rows and the same percentages as before the restructure, for master, Shopify-language and
Amazon-marketplace scopes, on the parent and on every variant.

---

## 12. Open items

1. **`POST /products/product-readiness-value`** (§6.3) — left on the product controller. Move it
   under `/readiness` as `product-summary` too, or keep it as a product-list concern?
2. **`PRODUCT_COMPLETENESS` socket constant** (§8) — the plan deletes it as dead. Confirm it is not
   reserved for planned work; if it is, it becomes `PRODUCT_READINESS` instead.
3. **`entities/marketplaceReadinessValue.entity.ts`** — still declared, still unused (its write
   path was commented out and is now gone). Not touched by this plan; delete it in the §9 pass?

---

## 13. Implementation notes (executed)

This plan has been executed. Three things differ from §3–§9 as written; each is recorded here
rather than silently absorbed.

### 13.1 A seventh engine file was needed: `ReadinessRuleUtils`

§4.2 put the shared low-level helpers on `ReadinessEngineService`. Doing that produced two
constructor cycles:

```
readiness-scope.resolver  --queryNumericValue-->  operational-value.resolver
operational-value.resolver --getAmazonMarketplaceIdForReadinessScope--> scope.resolver   ← cycle

operational-value.resolver --getReadinessRuleAttributeName--> engine.service
engine.service            --evaluateOperationalPrimaryReadinessRule--> operational-value  ← cycle
```

They were broken by extracting a dependency-free leaf, `engine/readiness-rule.utils.ts`
(`ReadinessRuleUtils`), holding: `getReadinessRuleAttributeName`, `getReadinessRuleKey`,
`getUniqueReadinessRules`, `isOperationalPrimaryAttribute`,
`getOperationalPrimaryNotApplicableResult`, `getUnevaluatedReadinessRuleResults`,
`mapWithConcurrency`, `toNumberOrNull`, `hasOperationalValue`, `queryNumericValue` and
`getVariantPredicate`. Its only dependency is the Sequelize instance.

The resulting graph is acyclic in one direction:

```
rule.utils · attribute-value.resolver · criteria.evaluator      (leaves)
      └─> scope.resolver ─> operational-value.resolver
                  media.resolver ─┘
                        └─> engine.service        └─> view.service
```

`ReadinessViewService` turned out not to need `ReadinessEngineService` at all — it shares the
resolvers, not the orchestrator. Only `ReadinessController` injects the engine (for
`POST /readiness/recalculate`).

The concurrency limits moved to `constants/readiness.constants.ts` as
`READINESS_CONCURRENCY_LIMIT` (5) and `READINESS_RECALCULATION_CONCURRENCY` (3).

### 13.2 Two modules needed a new import

`ChannelModule` and `AmazonChannelMarketplaceModule` previously received
`ReadinessConfigurationsService` transitively through `AttributeModule`. Since the rule services
left `attribute-core`, both now import `ReadinessModule` directly. Every other consumer already
imported the old `CalculateCompletenessModule` and needed only the rename.

### 13.3 The deferred list is smaller than §9.3 predicted

The webapp half of §9.3 was mostly renameable after all, because those symbols turned out not to
mirror the entity keys:

| Item | Outcome |
|---|---|
| `store/completeness.store.ts` | **deleted** — zero importers, a leftover of the removed `/completeness` page |
| `readiness/.../completeness-tooltip.tsx` (`CompletenessTooltip`) | **deleted** — zero importers |
| `defaultCompleteNessTableSettings` / `countedForCompleteNess` | **deleted** — only the dead store used them |
| `productCompletenessChannels` | renamed to `productReadinessChannels` (no API ever sends this key) |
| `completeness: 'all'` filter key in 5 stores | renamed to `readiness` (spread into list payloads, but no API reads it) |
| `calculateCompleteness()` in `useOverviewData.tsx` | renamed to `calculateChannelCoverage()` (category B) |
| FE translation keys `Completeness`, `Countedforcompleteness`, `Product_Completeness` | renamed to `Readiness`, `Countedforreadiness`, `Product_Readiness`; the last collided with an existing `Product_Readiness` key, so the duplicate was dropped |

**What actually remains deferred** — DB columns, entity TS properties, and one persisted key:

| Kind | Symbol | Where |
|---|---|---|
| DB column + entity key | `count_against_completeness` / `countAgainstCompleteness` | `attribtueChannels.entity.ts:38`; read/written in `attribute.service.ts` (9), `product-channel.service.ts` (3), `product-variant.service.ts` (2), `amazon-channel-product-type-schema.service.ts` (3), `amazon-product-bs.service.ts` (3, raw SQL), `shopify-product-bsq3.service.ts`, `amazon-attribute-mapping.service.ts`, `shopify-product.service.ts`, `readiness-scope.resolver.ts` (2), `core/types/sync/amazonSync.interface.ts`, `core/i18n/translationData/*.ts` (5), `webapp attributes/List.tsx` |
| DB column + entity key | `completeness` (category B, variant combination coverage) | `productChannel.entity.ts:158`, `productVariantChannels.entity.ts:54`; read/written in `product-channel.service.ts` (7), `product.service.ts` (2), `product-variant.service.ts` (2), `product-variants-flow.service.ts` (raw SQL), `live-insight.service.ts` (raw SQL), `product-overview.service.ts`, `webapp useOverviewData.tsx` (6) |
| Persisted table-column key | `product_completeness` | `webapp lib/globalVariables.ts` (3), `gridItem.tsx`, `gridFooter.tsx` — saved in each user's table configuration, so renaming needs a data migration |

### 13.4 Verification actually run

```
api      tsc -p apps/api-main/tsconfig.app.json --noEmit     clean
api      tsc -p apps/api-worker/tsconfig.app.json --noEmit   clean
api      pnpm build:all                                      clean
api      pnpm lint                                           clean (2 pre-existing warnings
                                                             in productListQueryBuilder.ts)
api      pnpm format                                         applied
webapp   tsc --noEmit                                        clean
webapp   pnpm lint / pnpm format                             clean / applied
find apps/api-main/src -iname "*complete*"                    no matches
grep webapp for product-completeness / attribute/*-readiness  no matches
```

Nest DI wiring was verified statically (every provider's constructor dependency is global,
declared in its own sub-module, or exported by an imported module). **The behavioural check from
§11 has not been run** — it needs a live database: recalculate a product with a non-trivial score
and confirm `tbl_readiness_values` holds identical rows and percentages for master,
Shopify-language and Amazon-marketplace scopes, on the parent and every variant.

### 13.5 Open items from §12, as implemented

1. `POST /products/product-readiness-value` — left on the product controller, unchanged.
2. `PRODUCT_COMPLETENESS` socket constant — deleted from `socket.config.ts` and from the webapp
   mirror; it had no emitter or listener.
3. `entities/marketplaceReadinessValue.entity.ts` — left in place, untouched.

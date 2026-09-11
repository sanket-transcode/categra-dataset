# Readiness — Implementation Plan: Triggers, Scoring, Storage

> **Scope:** `api/` only. This plan overrides the current engine (`catalog/products/readiness/engine/*`),
> the recalculation fan-in (`recalculation/*`) and every trigger call site. Rules CRUD (`rules/*`),
> the seeder and the stored-value read API are touched only where noted.
>
> **Builds on:** `READINESS-TRIGGERS.md` Part B (scope contract, decisions D1–D4) and the
> restructure already executed (`READINESS-MODULE-RESTRUCTURE.md` §13). Nothing in those two is
> re-decided here; this document is the *how*.
>
> **Decisions confirmed with the product owner for this plan**
>
> | # | Question | Decision |
> |---|---|---|
> | E1 | Where the executor runs | **BullMQ queue in api-worker** (`readiness-recalculation`), one job per merged scope, jobId = scope key + short coalescing delay |
> | E2 | When the parent scope stops counting Stock/Price | When the **scope's channel has at least one active variant attribute**: master → `tbl_product_variant_attributes`, non-Amazon → `tbl_product_variant_attribute_channels`, Amazon → `tbl_wrapper_variant_attributes` (via `tbl_wrappers`) |
> | E3 | Variant media on a channel | Inheritance flag decides master-vs-channel media; **no parent fallback** |
> | E4 | Master-scope price currency | **Account base currency only** |
>
> No schema change is required. No migration is produced by this plan.

---

## 1. Target architecture in one picture

```
  ANY mutation site (stock, price, content, media, structure, sync, rules, channel config)
        │
        │   ReadinessTriggerService.trigger({ req, source, scopes: ReadinessScope[] })
        │   ── the ONLY entry point; every old method is deleted ──
        ▼
  ┌──────────────────────────── api-main ────────────────────────────┐
  │  ReadinessTriggerService                                         │
  │    1. validate + normalise every scope   (§3.2)                  │
  │    2. merge scopes with the same key     (§3.3)                  │
  │    3. enqueue one job per merged scope   (§3.4)                  │
  │       jobId = hash(scope) · delay = READINESS_COALESCE_DELAY_MS   │
  │       → a burst of identical triggers collapses into ONE job     │
  └──────────────────────────────────────────────────────────────────┘
        │  BullMQ  `readiness-recalculation`
        ▼
  ┌──────────────────────────── api-worker ──────────────────────────┐
  │  ReadinessRecalculationProcessor  →  ReadinessExecutor.run(job)  │
  │    A. ReadinessScopePlanner   resolve scope → concrete targets   │
  │       (product × entity × channel × language) honouring EVERY    │
  │       active/deleted flag (§4)                                   │
  │    B. ReadinessScopeEvaluator score each target (§5)             │
  │       ← the SAME evaluator the live "view" endpoints use         │
  │    C. ReadinessValueService   bulk upsert + delete stale rows    │
  │    D. socket `readinessRefresh` (relayed to api-main)            │
  └──────────────────────────────────────────────────────────────────┘
```

Three properties the current code does not have and this plan guarantees:

1. **One entry point.** `recalculateForMutation`, `recalculateUniqueTargets` and every direct
   `calculateReadiness()` call disappear. Nothing outside the module can reach the engine.
2. **One evaluator.** The stored score and the live breakdown come from the same
   `ReadinessScopeEvaluator.evaluate()`; the view service stops re-implementing the ladder.
3. **Flags are checked before work starts, not after.** A scope that is inactive/deleted at any
   level produces *no* row and *deletes* the stale one.

---

## 2. File map after this plan

```
catalog/products/readiness/
├── readiness.module.ts                          (unchanged aggregator)
├── constants/readiness.constants.ts             + queue name, coalesce delay, batch sizes
├── types/
│   ├── readiness.types.ts                       ReadinessScope, ReadinessTriggerRequest,
│   │                                            ReadinessRecalculationSource (+5 new values)
│   └── readiness-evaluation.types.ts            NEW: ReadinessTarget, ReadinessRuleResult,
│                                                ReadinessScopeEvaluation, ReadinessScopeContext
│
├── trigger/                                     NEW  — "WHEN" (replaces recalculation/)
│   ├── readiness-trigger.module.ts
│   ├── readiness-trigger.service.ts             trigger(), normalise, merge, enqueue
│   └── readiness-scope.merger.ts                pure: merge + key
│
├── execution/                                   NEW  — "WHAT EXACTLY" (worker side)
│   ├── readiness-execution.module.ts
│   ├── readiness-executor.service.ts            run(job): plan → evaluate → persist → emit
│   ├── readiness-scope.planner.ts               scope → ReadinessTarget[] + stale keys (§4)
│   └── readiness-run.context.ts                 per-run memo (rules, attributes, languages)
│
├── engine/                                      "HOW" — value resolvers, all rewritten
│   ├── readiness-engine.module.ts
│   ├── readiness-scope.evaluator.ts             NEW: evaluate(target, ctx) → evaluation (§5)
│   ├── readiness-criteria.evaluator.ts          kept (4 criteria + min/max) + count on arrays
│   ├── readiness-attribute-value.resolver.ts    rewritten: 3-level ladder (§5.2)
│   ├── readiness-media.resolver.ts              rewritten: inheritance flag, no language (§5.4)
│   ├── readiness-stock.resolver.ts              NEW (split from operational): §5.5
│   ├── readiness-price.resolver.ts              NEW (split from operational): §5.6
│   ├── readiness-bullet-point.resolver.ts       NEW: §5.7
│   ├── readiness-attribute-range.resolver.ts    NEW: in-range attribute ids + variant-attribute
│   │                                            ids per scope (§5.3, E2)
│   └── readiness-rule.utils.ts                  trimmed to what is still used
│   (deleted: readiness-engine.service.ts, readiness-operational-value.resolver.ts,
│             readiness-scope.resolver.ts — their responsibilities move to the files above)
│
├── scoring/                                     unchanged except:
│   ├── readiness-value.service.ts               + deleteReadinessValuesNotIn(), deleteForEntity()
│   ├── readiness-value.collector.ts             unchanged
│   └── readiness-score.calculator.ts            unchanged
│
├── rules/                                       unchanged except:
│   ├── readiness-rules.service.ts               recalculateReadinessForRuleChange → trigger()
│   └── readiness-rules-seeder.service.ts        seed/remove → trigger() / delete rows
│
└── views/
    ├── readiness-view.service.ts                rewritten on top of ReadinessScopeEvaluator
    └── readiness.controller.ts                  POST /readiness/recalculate takes a ReadinessScope

sync/queue/processors/readiness.processor.ts     NEW: @Processor('readiness-recalculation')
core/constants/global-enums.ts                   QueueNames.READINESS_RECALCULATION
core/constants/queue-concurrency.ts              concurrency for the new queue
```

Dependency direction (no cycles):

```
scoring ◄── engine ◄── execution ◄── processor (worker)
             ▲            ▲
           views        rules ──► trigger ◄── every mutation site (api-main)
```

`trigger/` depends only on `QueueService` + types. `execution/` depends on `engine/` + `scoring/`.
`rules/` keeps its existing `forwardRef` to the module but now points at `ReadinessTriggerService`.

---

## 3. The trigger side (api-main)

### 3.1 Public API — the only one

```ts
// types/readiness.types.ts
export type ReadinessChannelSelector = 'ALL' | 'MASTER' | number;
export type ReadinessRuleType = 'stock' | 'price' | 'bullet_point';        // GlobalEnums.ReadinessRuleTypes

export interface ReadinessScope {
	channel: ReadinessChannelSelector;     // REQUIRED
	productIds?: number[];                 // omit = every non-deleted product of the account
	variantIds?: number[];                 // only with exactly one productId; omit = parent + all variants
	languageCodes?: string[];              // omit = every active language of each resolved channel scope
	attributeIds?: number[];               // D2 gate
	ruleTypes?: ReadinessRuleType[];       // D2 gate
}

export interface ReadinessTriggerRequest {
	req: AuthenticatedRequest;             // accountId + userId are taken from here, never from the scope
	source: ReadinessRecalculationSource;
	scopes: ReadinessScope[];
	/** Wait for the job to finish (manual endpoint, tests). Default false. */
	awaitCompletion?: boolean;
}

// trigger/readiness-trigger.service.ts
@Injectable()
export class ReadinessTriggerService {
	async trigger(request: ReadinessTriggerRequest): Promise<{ jobIds: string[] }>;
	/** Cleanup-only: delete stored rows for entities that ceased to exist. */
	async remove(request: ReadinessRemovalRequest): Promise<void>;
}

export interface ReadinessRemovalRequest {
	req: AuthenticatedRequest;
	source: 'ENTITY_REMOVED' | 'CHANNEL_SCOPE_CHANGED' | 'ACCOUNT_LANGUAGES_CHANGED';
	productIds?: number[];  variantIds?: number[];  channel?: ReadinessChannelSelector;  languageCodes?: string[];
}
```

`ReadinessRecalculationSource` gains the five values from `READINESS-TRIGGERS.md` B.6
(`READINESS_RULE_CHANGED`, `CHANNEL_SCOPE_CHANGED`, `ACCOUNT_LANGUAGES_CHANGED`, `PRODUCT_IMPORTED`,
`ENTITY_REMOVED`). `PRODUCT_LOCALIZED_CONTENT_UPDATED` starts being emitted.

### 3.2 Normalisation (rejects, never throws to the caller)

| Check | Action |
|---|---|
| `req.user.accountId` missing | log `readiness_trigger_skipped reason=no_account`, return |
| `channel` not `'ALL' \| 'MASTER' \| positive int` | skip that scope |
| `variantIds` set with `productIds.length !== 1` | skip that scope, log `reason=variant_ids_need_single_product` |
| empty arrays | treated as omitted |
| ids | `Number()`, drop non-finite / ≤ 0, de-dupe |
| `languageCodes` | trimmed, de-duped, kept as given (language codes are compared case-sensitively everywhere else in the code base) |

### 3.3 Merge (pure, `readiness-scope.merger.ts`)

Merge key = `${accountId}|${channel}|${sorted productIds ?? '*'}`.
Rules exactly as `READINESS-TRIGGERS.md` B.4: union of each narrower list, *omitted wins* (omitted
means "all"). `channel:'ALL'` for a product set additionally absorbs `'MASTER'` and any `<id>`
scope with the same product set.

### 3.4 Enqueue

```ts
await this._queueService.addJob(
	{
		type: GlobalEnums.QueueNames.READINESS_RECALCULATION,
		accountId: String(accountId),
		userId: String(userId),
		data: { req: { user: { id, accountId } }, source, scope },        // same sanitizedReq shape the other producers use
	},
	{
		jobId: `readiness:${scopeHash}`,      // sha1 of the merge key + narrower lists, stable across processes
		delay: READINESS_COALESCE_DELAY_MS,   // 1500 ms — a burst of identical scopes becomes one job
		attempts: 3,
		backoff: { type: 'exponential', delay: 5000 },
		removeOnComplete: true,
		removeOnFail: 200,
	},
);
```

Why the delay: `updateProductDetails` fires 3–4 triggers for one save, sync imports fire one per
product, action-center remediation fires one per variant. BullMQ ignores an `add` whose `jobId`
already exists in `delayed`/`waiting`, so identical scopes inside the window are free. Different
scopes for the same product (e.g. `attributeIds:[1]` then `attributeIds:[2]`) are *not* merged
across processes — that is fine, both jobs are gate-narrowed and cheap.

`awaitCompletion: true` uses `job.waitUntilFinished(queueEvents)` with a 60 s cap; only the manual
endpoint and tests use it.

### 3.5 Removal (no recompute)

`remove()` does not enqueue. It runs `ReadinessValueService.deleteForScope()` inline (a single
`DELETE`), because the cases are all synchronous deletions of something that no longer exists:
product/variant deleted, product-channel deactivated, marketplace disabled, Shopify language
mapping removed, account language removed.

---

## 4. The planner (worker) — scope → concrete targets

`ReadinessScopePlanner.plan(scope, ctx)` returns:

```ts
interface ReadinessTarget {
	productId: number;
	variantId: number | null;            // null = parent / standalone
	channelId: number | null;            // null = master
	channelType: 'MASTER' | 'AMAZON' | 'SHOPIFY';
	languageCode: string;
	// Amazon only — the marketplace behind this language
	amazon?: { productChannelId: number; productChannelMarketplaceId: number; variantMarketplaceId: number | null;
	           amazonChannelMarketplaceId: number; amazonMarketplaceId: number; currency: string; productTypeName: string };
	shopify?: { productChannelId: number; defaultCurrency: string };
}

interface ReadinessPlan {
	targets: ReadinessTarget[];          // what to (re)compute — narrowed by languageCodes / variantIds / gate
	activeKeys: Set<string>;             // EVERY currently-valid (product,variant,channel,language) of the
	                                     // resolved (product × channel) pairs, ignoring narrowing —
	                                     // used to delete stale rows (§6.2)
}
```

### 4.1 Flag matrix — a target exists only if every row on its path passes

| Level | Table | Condition |
|---|---|---|
| Product | `tbl_products` | `account_id = :acct AND is_deleted = false` (`is_archive` is *not* a readiness flag — archived products keep their scores) |
| Variant | `tbl_product_variants` | `is_deleted = false` |
| Channel | `tbl_channels` | `is_deleted = false` |
| Product ↔ channel | `tbl_product_channels pc` | `is_active = true AND is_deleted = false` |
| Variant ↔ channel (Shopify) | `tbl_product_variant_channels pvc` | `is_active = true AND is_deleted = false` |
| Marketplace on channel (Amazon) | `tbl_amazon_channel_marketplaces acm` | `status = true` |
| Product ↔ marketplace (Amazon) | `tbl_product_channel_marketplaces pcm` | `status = true AND is_deleted = false` |
| Variant ↔ marketplace (Amazon) | `tbl_variant_marketplaces vm` | `status = true AND is_deleted = false` |
| Shopify language | `tbl_shopify_language_mappings` | `is_deleted = false` |
| Master language | `tbl_account_languages ∪ account base language` | — |

### 4.2 Resolution, per channel selector

```
products   = scope.productIds ?? SELECT id FROM tbl_products WHERE account_id AND is_deleted = false
             (a productIds entry that fails the product flag is dropped silently)

for product P:
  entities = scope.variantIds ? those ids ∩ non-deleted variants of P          -- parent NOT added (D4)
                              : [null] + every non-deleted variant of P

  channelScopes =
    'MASTER' → [master]
    'ALL'    → [master] + every pc of P passing the pc + channel flags
    <id>     → the pc of P for that channel, if it passes; else nothing (and its keys are NOT in activeKeys,
               so its rows are deleted in §6.2)

  for channelScope C:
    MASTER:  languages = account languages ∪ base language
             targets   = entities × languages

    SHOPIFY: languages = shopify_language_mappings(C, is_deleted=false)
             parent    → languages
             variant V → languages, only if pvc(V, C) is active
             -- a Shopify variant scope has NO custom attributes (§5.3), but it still has
             --   name/description/media/stock/price rows, so the target exists.

    AMAZON:  parent    → one target per pcm of pc(P,C) passing pcm + acm flags; language = acm.language
             variant V → one target per vm(V) whose pcm passes; language = acm.language
             -- two marketplaces sharing a language collide on the row key; the planner keeps the
             --   one with the lowest acm.id and logs `readiness_marketplace_language_collision`.

  narrowing (targets only, never activeKeys):
    languageCodes set → keep targets whose languageCode ∈ languageCodes
    D2 gate: if attributeIds or ruleTypes set → keep a target only if the active rule set for
             (C, language) (§4.3) contains a rule with attribute_id ∈ attributeIds
             or configurations.type ∈ ruleTypes
```

### 4.3 Per-run memo (`ReadinessRunContext`)

Loaded once per job, never per target:

| Memo | Key | Source |
|---|---|---|
| base language, account languages | account | `AccountConfiguration`, `AccountLanguage` |
| base currency id | account | `tbl_currencies WHERE account_id AND currency = account_configuration.base_currency` |
| active rules | `channelId\|languageCode` | `tbl_readiness_configurations WHERE account_id AND status = true` — one query for the whole run, grouped in memory |
| attribute metadata (`id, name, attributeType, isPrimaryAttribute, isVariantKey`) | attribute id | one `IN (...)` query for every attribute referenced by a loaded rule + the 5 primary attributes |
| Amazon product-type name per product channel | pc.id | `tbl_product_channels.amazon_product_type_id → tbl_amazon_product_types.name` |
| bullet-point attribute id | productTypeName | `tbl_attributes WHERE account_id AND name = 'AM_' \|\| UPPER(name) \|\| '_bullet_point_value'` |
| in-range attribute ids | `productId\|channelType\|channelId` | §5.3 query |
| variant-attribute ids | `productId\|channelType\|channelId\|marketplaceId` | §5.3 query |

No Redis. The memo lives for one job; the persisted table *is* the cross-request cache. A Redis
layer would have to be invalidated on every mutation that already triggers a recalculation, i.e. it
would be invalidated exactly as often as it is filled.

---

## 5. The evaluator — `ReadinessScopeEvaluator.evaluate(target, ctx)`

```ts
interface ReadinessScopeEvaluation {
	target: ReadinessTarget;
	totalItems: number;
	totalFilledItems: number;
	readinessValue: number;            // ReadinessScoreCalculator.calculateReadinessPercentage()
	results: ReadinessRuleResult[];    // one per rule, incl. NOT_APPLICABLE ones — the view endpoints return these
}

interface ReadinessRuleResult {
	ruleId: number; attributeId: number | null; attributeName: string; ruleType: ReadinessRuleType | null;
	criteria: string; configuration: { min?: string | number; max?: string | number };
	isApplicable: boolean; isValid: boolean; reason: 'missing' | 'min' | 'max' | 'not_applicable' | null;
	currentValue: string | number | null; valueSource: string | null; fallbackUsed: boolean;
	notApplicableReason?: string;
}
```

Every rule of the `(channelId, languageCode)` rule set is routed by **what it is**, not by attribute
name string matching:

```
rule.configurations.type = 'stock'         → §5.5   (skipped for wrapper parent, E2)
rule.configurations.type = 'price'         → §5.6   (skipped for wrapper parent, E2)
rule.configurations.type = 'bullet_point'  → §5.7   (Amazon targets only; NOT_APPLICABLE elsewhere)
rule.attribute.name = 'Primary Media'      → §5.4
rule.attribute.name ∈ {Product Name, Product Description}  → §5.2 (always in range)
any other attribute                        → §5.3 range check, then §5.2 value ladder
```

`Score = filled / applicable × 100`, two decimals, 0 when nothing applies. Each rule counts 1; an
attribute with two rules counts 2 (unchanged).

### 5.1 One candidate query per target

All attribute-backed rules of a target read `tbl_product_attributes` **once**:

```sql
SELECT pa.attribute_id, pa.variant_id, pa.channel_id, pa.language_code, pa.is_inherited,
       pa.value_varchar, pa.value_text, pa.value_array, pa.value_decimal, pa.value_int, pa.value_bool
FROM tbl_product_attributes pa
JOIN tbl_product_attribute_groups pag ON pag.id = pa.product_attribute_group_id
WHERE pag.product_id = :productId AND pag.is_active = true
  AND pa.account_id = :accountId
  AND pa.attribute_id IN (:ruleAttributeIds)          -- rules of this (channel, language) + bullet point attr
  AND pa.language_code IN (:languageCode, :baseLanguageCode)
  AND (pa.channel_id IS NULL OR pa.channel_id = :channelId)      -- master target: IS NULL only
  AND (pa.variant_id IS NULL OR pa.variant_id = :variantId)      -- parent target: IS NULL only
```

Master-channel and Shopify targets of the same entity share languages ⇒ the executor evaluates
targets grouped by `(productId, variantId, channelId)` and issues this query with
`language_code IN (all target languages, base)` once per group.

### 5.2 Name / Description / any attribute value — the 3-level ladder

Mirrors what every product/variant list and detail query does (`productListQueryBuilder.ts:188`,
`product-variants-flow.service.ts:1986`, `product-stock.service.ts:2344`, `product-overview.service.ts:279`):

```
level 1  channel_id = :channelId   AND language_code = :lang  AND is_inherited = false   (channel targets only)
level 2  channel_id IS NULL        AND language_code = :lang  AND is_inherited = false
level 3  channel_id IS NULL        AND language_code = :base                              (is_inherited ignored)
first level with a NON-EMPTY value wins; none → ''  (reason 'missing')
```

- The row set is always filtered on the target's own `variant_id` (`= :variantId` or `IS NULL`).
  Primary attributes never inherit from the parent (`isInheritFromParentAllowed` = false); the
  variant's own Name/Description rows are what the variant screens show.
- Custom attributes on a **variant** additionally walk the parent, exactly as `pickAttributeValue()`
  in `common/lib/utils.ts:1526` orders them: `variant/channel → variant/master → parent/channel →
  parent/master → variant/base → parent/base`. This keeps readiness equal to the value the variant
  screen presents. (No change from the current resolver's chain; only the `is_inherited` handling
  on levels 1–2 is aligned with the SQL above.)
- Value column by `attributeType` and normalisation (strip tags, `&nbsp;`, trim) stay as they are in
  `getReadinessAttributeValue()`.
- Criteria: `presenceCheck` / `lengthCheck` / `valueCheck` as today. `countCheck` on an
  attribute-backed rule now counts **non-empty entries of `value_array`** when the value column is
  an array, string length otherwise (fixes the diagnostics gap `COUNT_CHECK_NON_MEDIA_ATTRIBUTE`).
- `valueSource` = `product_attribute.channel` / `.master_language` / `.master_base` /
  `.parent_*`, `fallbackUsed` = level > 1.

### 5.3 Custom attributes — in range or not counted (and the wrapper rule, E2)

A custom-attribute rule is **applicable** for a target iff the attribute is in range **and** is not
a variant attribute for the target's channel. Otherwise the result is `NOT_APPLICABLE` and excluded
from the denominator.

**In range** (`readiness-attribute-range.resolver.ts`, one query per `(product, channelType, channelId)`):

```sql
-- returns the attribute ids that count for this product on this channel
SELECT DISTINCT pa.attribute_id
FROM tbl_product_attribute_groups pag
JOIN tbl_attribute_groups ag ON ag.id = pag.attribute_group_id AND ag.is_deleted = false
JOIN tbl_product_attributes pa ON pa.product_attribute_group_id = pag.id
     AND pa.account_id = :accountId AND pa.variant_id IS NULL AND pa.channel_id IS NULL
     AND pa.language_code = :baseLanguageCode
{ MASTER  : LEFT JOIN nothing }                                       -- ag.type IS NULL
{ SHOPIFY : JOIN tbl_attribute_groups_channels agc
              ON agc.attribute_group_id = ag.id AND agc.channel_id = :channelId }   -- ag.type = 'SHOPIFY'
{ AMAZON  : -- ag.type = 'AMAZON' AND ag.name = generateAmazonAttributeGroupName(productTypeName) }
WHERE pag.product_id = :productId AND pag.is_active = true
  AND <type predicate above>
```

Product Name / Product Description / Primary Media are always in range (product-structure group,
`isPrimaryAttribute`). **Shopify variant targets have no custom attributes at all** — for them the
custom-attribute rules are `NOT_APPLICABLE` without running the query.

**Variant attributes** (excluded for *every* target of that scope, and the E2 wrapper signal):

```sql
-- MASTER
SELECT pva.attribute_id FROM tbl_product_variant_attributes pva
WHERE pva.product_id = :productId AND pva.account_id = :accountId AND pva.is_active = true
-- NON-AMAZON channel
SELECT COALESCE(pvac.mapped_attribute_id, pva.attribute_id) AS attribute_id
FROM tbl_product_variant_attributes pva
JOIN tbl_product_variant_attribute_channels pvac ON pvac.product_variant_attribute_id = pva.id AND pvac.channel_id = :channelId
WHERE pva.product_id = :productId AND pva.is_active = true
-- AMAZON parent: wrapper of (product, channel, marketplace)
SELECT wva.attribute_id FROM tbl_wrappers w
JOIN tbl_wrapper_variant_attributes wva ON wva.wrapper_id = w.id AND wva.is_active = true
WHERE w.product_id = :productId AND w.channel_id = :channelId AND w.amazon_marketplace_id = :amazonMarketplaceId
-- AMAZON variant: the wrapper the variant marketplace points at
SELECT wva.attribute_id FROM tbl_variant_marketplaces vm
JOIN tbl_wrapper_variant_attributes wva ON wva.wrapper_id = vm.wrapper_id AND wva.is_active = true
WHERE vm.id = :variantMarketplaceId
```

`isWrapperParent(target) = target.variantId === null && variantAttributeIds(target).size > 0`
(E2). Same set answers both questions; it is memoised per `(product, channelType, channelId, marketplaceId)`.

### 5.4 Primary Media — channel-bound, language-free

```
media rows: tbl_product_media_linkers WHERE account_id AND product_id AND variant_id IS NOT DISTINCT FROM :variantId
            (language_code / channel_marketplace_id are never written — ignored)

MASTER target      → COUNT(channel_id IS NULL)
channel target     → inherited = EXISTS tbl_product_attributes pa
                        JOIN tbl_attributes a ON a.id = pa.attribute_id AND a.name = 'Primary Media'
                        WHERE pa.account_id AND pa.channel_id = :channelId
                          AND pa.variant_id IS NOT DISTINCT FROM :variantId AND pa.is_inherited = true
                        (same predicate as computeMediaLinkersWithInheritance, product.service.ts:22637)
                     inherited      → COUNT(channel_id IS NULL)          valueSource 'product_media.master_inherited'
                     not inherited  → COUNT(channel_id = :channelId)     valueSource 'product_media.channel'
no parent fallback (E3). count → evaluatePrimaryMediaCount(min,max) unchanged.
```

Because the count does not depend on language, the executor computes it **once per
`(product, variant, channel)`** and reuses it for every language target of that group. The rule row
is still fetched per language (rules are language-scoped), only the value is shared.

Denominator: the Primary Media rule counts 1 whenever it exists in the rule set — for wrapper parents
too (unchanged).

### 5.5 Stock — "the figure the stock screen shows"

Resolution mirrors `product-stock.service.ts` (`getMasterChannelData`, `getAmazonChannelData`,
`getVariantStockByAmazonChannel`, `getShopifyStockData`):

| Target | Final stock | Source string |
|---|---|---|
| MASTER, standalone or variant | `SUM(tbl_warehouse_products.total_quantity)` for `(product, variant_id IS NOT DISTINCT FROM :variantId)`; **NULL when no warehouse row** (the stock screen shows `not_defined`). `tbl_product_variant_stocks.in_hand_qty` is a mirror kept by the warehouse service and is *not* read. | `warehouse_product.total_quantity` |
| SHOPIFY, parent/standalone | `tbl_product_channels.external_stock` of the pc | `product_channel.external_stock` |
| SHOPIFY, variant | `tbl_product_variant_channels.external_stock` of the pvc | `variant_channel.external_stock` |
| AMAZON, parent/standalone | `fba ? fbaInventory.fulfillable_quantity : pcm.amazon_stock` | `amazon_fba_inventory.fulfillable_quantity` / `product_channel_marketplace.amazon_stock` |
| AMAZON, variant | `fba ? fbaInventory.fulfillable_quantity : vm.amazon_stock` | idem / `variant_marketplace.amazon_stock` |

`fba` for an Amazon target = `pcm.is_fba` (parent) / `vm.is_fba` (variant) **OR** any
`tbl_channel_product_warehouses` row for `(product, variant, channel, amazon_marketplace_id)`
whose warehouse has `amazon_fulfillment_center = true` — the same two signals the stock screen ORs
together. `fbaInventory` = `tbl_amazon_fba_inventory WHERE product_id AND variant_id IS NOT
DISTINCT FROM :variantId AND channel_id AND marketplace_id`; missing row → NULL. FBM stock — whether
Categra-managed warehouses or externally managed — is what Amazon currently holds
(`amazon_stock`); Categra's own warehouse total only matters for the SYNC_GAP status, which is not a
readiness input.

Evaluation (`valueCheck` min 1 by default, `presenceCheck` allowed): NULL → `missing`; min/max as
today; no min & no max → `> 0`. **A stock of exactly 0 fails** (unchanged, intentional).

Stock/Price for **MASTER and SHOPIFY** targets are evaluated once per `(product, variant, channel)`
and reused across that group's languages — the value is language-independent. For **AMAZON** each
language *is* a marketplace and is resolved separately. This also removes the current
"base language only" gate in master scope: the Stock/Price rules are seeded per language already
(seeder: master gets them for the base language only, channels for every language), so the rule set
decides, and the value is simply shared.

### 5.6 Price — `offerPrice` only, inheritance flag respected

```
row(P, V, C, M, currency) := tbl_product_variant_prices WHERE account_id AND product_id
                              AND variant_id IS NOT DISTINCT FROM :variantId
                              AND channel_id IS NOT DISTINCT FROM :channelId
                              AND amazon_marketplace_id IS NOT DISTINCT FROM :marketplaceId
                              AND currency_id = :currencyId
offer(row) := NULLIF(pricing_details->'offerPrice'->>'price','')::numeric        -- nothing else is read

MASTER target   currency = account base currency (E4)
                value = offer(row(P,V,NULL,NULL,base))                              source 'product_variant_price.master'
SHOPIFY target  currency = tbl_shopify_channels.default_currency of the channel
                sp = row(P,V,C,NULL,cur)
                value = sp exists AND COALESCE(sp.pricing_details->'offerPrice'->>'isInherited','true') = 'false'
                          ? offer(sp)                                               source '.channel'
                          : offer(row(P,V,NULL,NULL,cur))                           source '.master_inherited'
AMAZON target   currency = acm.currency of the marketplace; marketplaceId = acm.amazon_marketplace_id
                same rule as Shopify with M = marketplaceId
```

This is the `resolved_prices` CTE of `product-pricing.service.ts:3505` verbatim. Note the
inheritance flag lives **inside** `pricing_details.offerPrice.isInherited`, not in the row's
`is_inherited` column; absent flag = inherited (the CTE's `COALESCE(..., 'true')`).

Evaluation: `presenceCheck` (default) → `value > 0`; `valueCheck` with min/max as today. Wrapper
parents (E2) → `NOT_APPLICABLE`.

### 5.7 Bullet points — Amazon only, always counted

```
applies      : target.channelType === 'AMAZON'  (any other target → NOT_APPLICABLE, excluded)
attribute    : tbl_attributes WHERE account_id AND name = `AM_${productTypeName.toUpperCase()}_bullet_point_value`
               productTypeName = tbl_amazon_product_types.name via tbl_product_channels.amazon_product_type_id
               (the same shape product.service.ts:19372 builds: `AM_${productType}_${path.join('_')}`
                with CONSTANTS.BULLET_POINT + CONSTANTS.VALUE)
in range     : NOT checked — the rule is counted for every Amazon target (owner decision)
               (no product type on the pc, or the attribute does not exist → value '' → fails 'missing')
value        : §5.2 ladder on that attribute id; bullet points are stored as value_array
               (getAttributeValueForAttributeType → { valueArray }), so
               countCheck → number of non-empty entries, lengthCheck → every entry ≥ min (all must pass)
wrapper rule : NOT applied — bullet points are content, not operational; a wrapper parent still needs them
```

The rule row is `attribute_id = NULL, configurations.type = 'bullet_point'` (seeded for Amazon
channels only, `countCheck min 3`); its attribute id is resolved at evaluation time from the
product type, which is why it cannot be a normal attribute-backed rule.

### 5.8 Language independence, summarised

| Rule family | Varies by language? | Evaluated once per… |
|---|---|---|
| Name, Description, custom attributes, bullet points | yes | target |
| Primary Media | **no** | `(product, variant, channel)` |
| Stock, Price on MASTER / SHOPIFY | **no** | `(product, variant, channel)` |
| Stock, Price on AMAZON | yes (language = marketplace) | target |

---

## 6. Storage

### 6.1 Writes

Unchanged mechanism: `ReadinessValueCollector` → `ReadinessValueService.upsertReadinessValues()`
(UNNEST update-then-insert, `READINESS_VALUE_UPSERT_COUNT = 500`, update only when the value moved).
One collector per job, flushed at the end; the whole job runs in **one transaction** so a failure
leaves the previous scores intact and BullMQ retries.

Row key stays `(account_id, product_id, variant_id, channel_id, language_code)`. Amazon marketplaces
remain keyed by language (owner statement: "each language is treated as a separate marketplace").

### 6.2 Deleting stale rows (fixes G2–G5, G8 without a new trigger for every case)

After evaluation, for every `(productId, channelId)` pair the plan resolved:

```sql
DELETE FROM tbl_readiness_values
WHERE account_id = :accountId AND product_id = :productId
  AND channel_id IS NOT DISTINCT FROM :channelId
  AND (variant_id, language_code) NOT IN (:activeKeysOfThisPair)     -- from ReadinessPlan.activeKeys
```

`activeKeys` is computed from the flag matrix (§4.1) **before narrowing**, so a job narrowed to
`languageCodes:['de_DE']` still deletes the `fr_FR` row of a marketplace that was disabled last week.
For `channel: <id>` where the pc failed its flags, `activeKeys` for that pair is empty ⇒ all its rows
go. For `channel: 'ALL'`, channels that are no longer linked are also swept:
`DELETE … WHERE product_id AND channel_id IS NOT NULL AND channel_id NOT IN (:resolvedChannelIds)`.

`remove()` (§3.5) covers the cases where no recalculation is wanted at all.

### 6.3 What is *not* stored

Per-rule results are not persisted (D2). The view endpoints re-run `evaluate()` live, which is now
the same code path, so "stored score ≠ breakdown" can no longer happen.

---

## 7. Views and the manual endpoint

| Endpoint | Change |
|---|---|
| `POST /readiness/product-view` | `ReadinessViewService` builds a `ReadinessTarget` (master, `variantId?`, `languageCode`/`languageCodes`) through the planner (so the flags apply — an inactive scope returns `not_applicable`) and returns `evaluate().results` + `readinessValue`. |
| `POST /readiness/marketplace-view` | same with an Amazon/Shopify target; `productChannelMarketplaceId` → target via planner. |
| `POST /readiness/product-values` | unchanged (reads stored rows). |
| `POST /readiness/recalculate` | body is a `ReadinessScope`; `accountId` from JWT; calls `trigger({ awaitCompletion: true })`. Validated by a new `RecalculateReadinessDto` (`channel`, optional id arrays). |

`getScopedReadinessRuleResults()` and the duplicated rule loading in the view service are deleted.

---

## 8. Call-site migration

Every site in `READINESS-TRIGGERS.md` A.3 / A.4 moves to `ReadinessTriggerService.trigger()` with
the scope from B.5. Mechanically:

| Today | Target |
|---|---|
| `_readinessRecalculationService.recalculateForMutation({ req, productId, channelId?, variantId?, source })` | `_readinessTriggerService.trigger({ req, source, scopes: [{ channel, productIds: [productId], variantIds?, languageCodes?, attributeIds?, ruleTypes? }] })` |
| `recalculateUniqueTargets(req, source, targets[])` | `trigger({ req, source, scopes: targets.map(toScope) })` — merging is inside |
| direct `_readinessEngineService.calculateReadiness({ req, productId, channelId? })` (8 sites) | `trigger(...)` with an explicit `channel` (`'ALL'` where nothing was passed) and the B.6 source |
| private wrappers (`recalculateOperationalReadiness`, `recalculateReadinessAfterMediaMutation`, `recalculateReadinessAfterContentMutation`, `recalculateVariantContentReadiness`, `recalculateShopifyMetafieldReadiness`, `recalculateReadinessAfterTranslation`, `recalculateGroupedReadinessTargets`, `recalculateUniqueOperationalReadinessTargets`) | kept as thin adapters where they add `transaction.afterCommit` deferral (media path); otherwise inlined. Each becomes ≤ 5 lines: build scope → `trigger()`. |

Concrete scope per source is the B.5 table; two refinements the deeper investigation adds:

- **Stock sources** pass `ruleTypes:['stock']` **and**, for Amazon, `languageCodes:[acm.language]`
  — every Amazon stock site has the marketplace in hand (`fetchAmazonMarketplaceStock`,
  `updateAmazonStock`, `switchToFba/Fbm`, FBA inventory refresh). A new trigger is added in
  `amazonFbaInventory` after an inventory pull (`AMAZON_STOCK_UPDATED`) — FBA stock changes without
  any `amazon_stock` write today.
- **Media inheritance toggle** (`updatePrimaryMediaInheritance`) passes `channel: C` (the toggle is
  channel-specific; `'ALL'` was wasteful).

New triggers (the G-list), each one line at the mutation:

| Gap | Site | Call |
|---|---|---|
| G1 order stock deduction | `amazon-order.service.ts`, `shopify-order-bsq5.service.ts`, `warehouse-product-location.service.ts`, `warehouse-location.service.ts` | `trigger({ source:'WAREHOUSE_STOCK_UPDATED', scopes: perAffected(P,V) → { channel:'ALL', productIds:[P], variantIds:[V], ruleTypes:['stock'] } })` |
| G2 marketplace enabled / disabled | `amazon-channel-marketplace.service.ts:293-302` | enabled → `trigger({ source:'CHANNEL_SCOPE_CHANGED', scopes:[{ channel:C, languageCodes:[acm.language] }] })`; disabled → `remove({ channel:C, languageCodes:[acm.language] })` |
| G3 product / variant listed or unlisted on a marketplace | wherever `pcm.status` / `vm.status` is written (`product-channel.service`, `variant-marketplace.service`) | listed → `trigger({ channel:C, productIds:[P], variantIds?, languageCodes:[lang] })`; unlisted → `remove(...)` |
| G4 Shopify language mapping added / removed | `shopify-channel.service.ts:1240, :1353` | as G2 |
| G5 product-channel (de)activated | `product-channel.service` `is_active` writes | activate → `trigger({ channel:C, productIds:[P] })`; deactivate → `remove({ channel:C, productIds:[P] })` |
| G6 rules seeded | `readiness-rules-seeder.service.ts` end of `seedPredefinedReadinessRules` | `trigger({ source:'READINESS_RULE_CHANGED', scopes: seeded (channel, language) pairs })` — replaces the "write 0 rows for every product" loop, which is deleted |
| G7 attribute group assigned / unassigned | `product-attribute-group` writes | `trigger({ source:'PRODUCT_STRUCTURE_UPDATED', scopes:[{ channel:'ALL', productIds:[P] }] })` |
| G8 product / variant deleted | `product-deletion.service.ts`, variant delete path | `remove({ source:'ENTITY_REMOVED', productIds / variantIds })` |

`recalculateReadinessForRuleChange` keeps computing `getRuleChangeAffectedProductIds()` and passes
them as `productIds` with `channel: rule.channelId ?? 'MASTER'`, `languageCodes:[rule.languageCode]`
and **no** gate (a rule turning off changes the denominator).

`calculateAllProductReadiness` (account languages changed) becomes
`trigger({ source:'ACCOUNT_LANGUAGES_CHANGED', scopes:[{ channel:'ALL', languageCodes: added }] })` —
`productIds` omitted means every product; the planner chunks products into batches of
`READINESS_PRODUCTS_PER_JOB = 50` and re-enqueues the remainder as follow-up jobs so no single job
holds a transaction over thousands of products.

---

## 9. Worker processor

```ts
// sync/queue/processors/readiness.processor.ts
@Processor(GlobalEnums.QueueNames.READINESS_RECALCULATION, {
	concurrency: getQueueConcurrency(GlobalEnums.QueueNames.READINESS_RECALCULATION),   // 4
	...WORKER_OPTIONS,
})
export class ReadinessRecalculationProcessor extends BaseQueueProcessor {
	protected async doProcess(job: Job): Promise<void> {
		await this._readinessExecutor.run(job.data.data.req, job.data.data.source, job.data.data.scope);
	}
}
```

Registered in `SyncConsumersModule`; `ReadinessExecutionModule` is imported there. Structured logs
(`readiness_run_started` / `_completed` with `targets`, `rowsWritten`, `rowsDeleted`, `durationMs` /
`_failed`) replace the current per-target console noise. `preloadQueues()` already covers stalled
recovery for every queue.

`SocketService.emitToAgainstAccountId` transparently relays through `JOB_STATUS_UPDATE` when no
gateway is present, so `ProductSocketService.emitProductErrorSocket(req, { productId, variantId,
channelId, readinessRefresh: true, reason: source })` is called from the executor unchanged — once
per product in the job, after commit.

---

## 10. Concurrency inside a job

- Targets are grouped by `(productId, variantId, channelId)`; groups run with
  `READINESS_CONCURRENCY_LIMIT = 5` (`mapWithConcurrency`), languages inside a group run
  sequentially (they share the candidate row set and the language-free values).
- All lookups inside a group are plain indexed selects; the only aggregate is the warehouse SUM,
  which already has `20260717120000-add-warehouse-products-readiness-index`.
- Queue concurrency 4 × 5 groups = at most 20 concurrent groups per worker process, well inside
  `DB_POOL_MAX_WORKER`.

---

## 11. Constants added

```ts
// core/constants/global-enums.ts  QueueNames
READINESS_RECALCULATION: 'readiness-recalculation',

// core/constants/queue-concurrency.ts
[GlobalEnums.QueueNames.READINESS_RECALCULATION]: 4,

// readiness/constants/readiness.constants.ts
export const READINESS_COALESCE_DELAY_MS = 1500;
export const READINESS_PRODUCTS_PER_JOB = 50;
export const READINESS_AWAIT_COMPLETION_TIMEOUT_MS = 60_000;
export const AMAZON_BULLET_POINT_ATTRIBUTE_SUFFIX = `_${CONSTANTS.BULLET_POINT}_${CONSTANTS.VALUE}`;   // '_bullet_point_value'
```

`READINESS_RECALCULATION_CONCURRENCY` (the old in-process pool) is deleted.

---

## 12. Execution order

Each step compiles and can ship alone; behaviour changes land in step 5.

| # | Step | Files |
|---|---|---|
| 1 | Types: `ReadinessScope`, `ReadinessTriggerRequest`, evaluation types, new source values, constants, queue name + concurrency | 4 |
| 2 | `trigger/`: service + merger + module, unit tests for normalise/merge/key | 4 |
| 3 | `execution/`: planner (+ `activeKeys`), run context, executor shell that still calls the *old* engine; processor + `SyncConsumersModule` wiring; `ReadinessValueService.deleteReadinessValuesNotIn/deleteForScope` | 6 |
| 4 | Swap all call sites to `trigger()` / `remove()`, add the G1–G8 triggers, delete `recalculation/` and the private wrappers that became one-liners | ~25 |
| 5 | Engine rewrite: `ReadinessScopeEvaluator`, range resolver (E2), 3-level attribute ladder, media (E3), stock, price (E4), bullet points; executor switched to the new evaluator; delete `readiness-engine.service.ts`, `readiness-operational-value.resolver.ts`, `readiness-scope.resolver.ts` | 10 |
| 6 | Views on the evaluator; `POST /readiness/recalculate` DTO | 3 |
| 7 | Seeder: drop the zero-row loop, trigger `READINESS_RULE_CHANGED`; rules service → `trigger()` | 2 |
| 8 | Remove dead code: `marketplaceReadinessValue.entity.ts` (already dropped by migration `20260911043949`), leftover `ReadinessRuleUtils` members, `OPERATIONAL_READINESS_RULE_CONDITION` | 3 |

---

## 13. Verification

**Unit (`*.spec.ts`, colocated):** merger (union/omitted-wins/`ALL` absorption), planner flag matrix
(one fixture per row of §4.1 — each flag flipped must drop the target *and* its activeKey), attribute
ladder (levels 1–3, `is_inherited` on level 2, parent chain for variant custom attributes), media
inheritance flag, stock table per §5.5, price table per §5.6, bullet-point name + array count,
range/wrapper resolver per channel type.

**Integration (needs DB):** one product with 2 variants, Amazon channel (2 marketplaces, one FBA),
Shopify channel (2 languages), account languages `en_US` + `de_DE`. Run
`POST /readiness/recalculate { channel:'ALL', productIds:[P] }` and assert the exact row set
`(variant, channel, language)` and that every row equals `product-view` / `marketplace-view` for the
same target. Then: disable one marketplace → its rows are gone after the next trigger; unlist a
variant → its row gone; toggle media inheritance → channel score moves, master does not; set FBA
fulfillable to 0 → Amazon stock fails, master unaffected.

**Static:** `pnpm build:all`, `pnpm lint`, `pnpm format`; `grep -rn "calculateReadiness\|recalculateForMutation\|recalculateUniqueTargets" apps/` returns only the engine's own definitions.

---

## 14. Remaining clarifications (not blocking — defaults stated, change on request)

| # | Question | Default taken in this plan |
|---|---|---|
| C1 | Two Amazon marketplaces of one channel sharing a language (`SAME_LANGUAGE_MARKETPLACE_COLLISION`) | lowest `acm.id` wins the row; logged. A per-marketplace row key would need a schema change, which is out of scope. |
| C2 | Master Stock with no warehouse row at all | NULL → `missing` (stock screen shows `not_defined`). If a legacy `in_hand_qty` should count when no warehouse rows exist, add it as a fallback in §5.5 — one line. |
| C3 | Bullet-point `lengthCheck` semantics when a second bullet-point rule is added by a user | "every entry ≥ min"; alternative is "longest entry". |
| C4 | `is_archive` products | still scored (only `is_deleted` is a flag, per the task list). |
| C5 | Trial/subscription capability gating on the new queue (`queue.subscription-capability.ts`) | none — readiness is not a billable action. |

---

## 15. Implementation notes (executed 2026-09-11)

The plan has been implemented in `api/`. Verified: `tsc --noEmit` for api-main and api-worker,
`pnpm lint`, `pnpm format`, and 21 colocated unit tests (`trigger/readiness-scope.merger.spec.ts`,
`engine/readiness-attribute-value.resolver.spec.ts`, `engine/readiness-criteria.evaluator.spec.ts`).
The DB-backed integration check of §13 has **not** been run - it needs a live database and Redis.

Where the code differs from §2-§12 as written:

| # | Plan | Implemented | Why |
|---|---|---|---|
| N1 | `readiness-scope.resolver.ts` deleted, planner in `execution/` | as planned; the per-run memo is `ReadinessRunContext` (a plain class built by `ReadinessExecutorService.createContext()`), not a Nest provider | it carries the transaction and per-job state, so one instance per run is the correct lifetime |
| N2 | Stock triggers pass `languageCodes:[acm.language]` for Amazon | only the order-driven Amazon path (`amazon-order.service.ts`) passes the marketplace language; the stock-service sites pass `ruleTypes:['stock']` + `channel:C` without a language | the marketplace language is not in hand at those sites (only ids are); the job is already gate-narrowed, so the saving would be one marketplace per channel |
| N3 | Media triggers pass `attributeIds:[PRIMARY_MEDIA]` | media triggers pass no attribute gate | the attribute id would need a lookup per trigger; a media change recomputes the whole (product, channel) scope, which is cheap |
| N4 | `product-view` / `marketplace-view` return `results + readinessValue` | they keep returning the results array (batch: `Record<language, results[]>`) | keeps the webapp contract untouched; the score is derivable and the FE never used it |
| N5 | `POST /readiness/recalculate` body = `ReadinessScope` | as planned (`RecalculateReadinessDto`, `awaitCompletion` defaults to true, source `MANUAL`) | the old `{ productId, channelId }` body is gone; the webapp does not call this route |
| N6 | Job id = scope hash | job id = `readiness:<hash>:<time bucket>` (bucket = coalesce window) | an identical scope triggered while its job is already *active* would otherwise be swallowed and its change lost; the bucket makes it land in the next job |
| N7 | Account-wide scopes re-enqueue remaining product batches | as planned, through `ReadinessTriggerService.trigger()` from inside the worker (`READINESS_PRODUCTS_PER_JOB = 50`) | — |
| N8 | Seeder triggers `READINESS_RULE_CHANGED` for seeded slices | as planned, deferred with `transaction.afterCommit()` when the seeder runs inside a caller's transaction; the "write 0 rows for every product" loop is gone | the worker must see the committed rules |
| N9 | `removeReadinessRules` (channel, languages) | now deletes scores for that channel + languages only (`trigger.remove`) | the old code deleted the languages' rows across **every** channel |
| N10 | G1 order stock | trigger placed in `AmazonOrderService.syncChannelInventoryFromWarehouses` (shared by the Amazon and Shopify order paths) + the Amazon marketplace-only branch | one place covers both order sources |
| N11 | G2 marketplace enabled | explicit `CHANNEL_SCOPE_CHANGED` trigger for newly enabled marketplace languages in `amazon-channel-marketplace.service.ts` (the seeder only fires when rules were actually missing) | — |
| N12 | G3 listing toggles | variant listing (`changeVariantMarketplaceStatus`) triggers / removes explicitly; product-level marketplace toggles rely on the existing `UpdateMarketplaceDetail` trigger + the executor's stale-row sweep | pcm status is written inside the same handler that already triggers the channel scope |
| N13 | G5 product-channel (de)activation | `updateProductChannelsStatus` triggers / removes; `createOrUpdateProductChannel` removes for unlinked channels (`afterCommit`) | activation through `createOrUpdateProductChannel` is covered by the caller's `'ALL'` trigger |
| N14 | G7 attribute group assigned / unassigned | covered by `update-details.service.ts`: any group change disables the attribute gate and recomputes the whole scope | no separate hook needed |
| N15 | G8 product / variant deleted | `softDeleteProducts` calls `remove()` for products, variants, channel scopes and marketplace languages, inside the deletion transaction; the deletion cleanup queue still hard-deletes rows later | — |
| N16 | Unit tests import `src/core` | specs mock the `src/core` barrel with `src/core/constants/global-enums` | the barrel loads the ORM and cannot be required under Jest |

Trigger-site deviation worth knowing when reading logs: sources are unchanged strings; the two new
ones actually emitted are `PRODUCT_IMPORTED` (create product, Amazon Q4 / ASIN import, Shopify Q3),
`CHANNEL_SCOPE_CHANGED` (marketplace / listing / product-channel / channel-detail changes),
`READINESS_RULE_CHANGED`, `ACCOUNT_LANGUAGES_CHANGED`, `ENTITY_REMOVED` (removal only) and `MANUAL`.

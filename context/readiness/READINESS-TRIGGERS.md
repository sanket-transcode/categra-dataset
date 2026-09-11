# Readiness — Triggers & Requested Scope

> **Scope of this document:** `api/apps/api-main/src/modules/app/catalog/products/readiness/` and every
> call site outside it that starts a readiness (re)calculation.
> Part A is an inventory of what exists today. Part B is the target trigger → scope contract.
> No code has been changed.

Decisions confirmed with the product owner before Part B was written:

| # | Question | Decision |
|---|---|---|
| D1 | What makes an Amazon marketplace language "active" for readiness | Marketplace enabled on channel (`tbl_amazon_channel_marketplaces.status = true`) **and** the product is listed on it (`tbl_product_channel_marketplaces.status = true AND is_deleted = false`); for variants additionally `tbl_variant_marketplaces.status = true AND is_deleted = false` |
| D2 | What `attributeIds` / `ruleTypes` narrowing means | **Gate only.** They decide which scopes are *skipped* (no active rule references them). Any scope that passes the gate is fully recomputed. No per-rule result storage, no migration |
| D3 | What a master-scope mutation requests | `channel: 'ALL'` — master + every active channel, because master stock / price / content fall back into channel scopes |
| D4 | Does a variant-only trigger recalculate the parent | **No.** Parent rows are computed from the parent's own attributes / media (operational rules are N/A for a parent wrapper), so a variant change cannot move the parent score |

---

## Part A — What triggers readiness today

### A.1 The three entry points

```
ReadinessRecalculationService.recalculateForMutation({ req, productId, channelId?, variantId?, source })
        └─► ReadinessEngineService.calculateReadiness({ req, productId, channelId? })   ← variantId is IGNORED
                └─► socket emit  (readinessRefresh: true)

ReadinessRecalculationService.recalculateUniqueTargets(req, source, targets[])
        └─► dedupes by `${productId}:${channelId ?? 'all'}`  (variantId collapses to null on conflict)
        └─► calculateReadiness() per unique target, concurrency 3

ReadinessEngineService.calculateReadiness({ req, productId, channelId? })   ← called directly by 8 sites
```

`readiness-recalculation.service.ts:29-113`, `:150-229`; `readiness-engine.service.ts:76-301`.

### A.2 What `calculateReadiness()` fans out to today

For **one product** (`readiness-engine.service.ts:76-301`):

| Step | Scope produced | Language source | Active filter applied today |
|---|---|---|---|
| 1 | Every `tbl_product_channels` row of the product (`channelId` narrows to one) | — | `pc.is_active = true` only (`readiness-scope.resolver.ts:121`) |
| 2a | Amazon: parent × every `tbl_product_channel_marketplaces` of that product channel | `tbl_amazon_channel_marketplaces.language` | **none** — `pcm.status`, `pcm.is_deleted`, `acm.status` are *not* checked (`:113-141`) |
| 2b | Amazon: every variant × its `tbl_variant_marketplaces` | same | `vm.status = true` only; `vm.is_deleted`, `pcm.status`, `acm.status` *not* checked (`:143-166`) |
| 3 | Shopify: parent + every variant × every `ShopifyLanguageMapping.language` of the channel | `tbl_shopify_language_mappings` | `is_deleted` *not* checked |
| 4 | Master: parent + every variant × (`tbl_account_languages` ∪ base language) | account languages | — |

Observations that matter for Part B:

- `channelId` is the **only** narrowing the engine supports. `variantId`, `languageCode`, `attributeId`, rule type are all accepted upstream and dropped.
- `channelId` given → master scope is **still** recomputed (step 4 is unconditional).
- Variants are loaded with `findAll({ productId })` — no `isActive` / `isDeleted` filter on the variant itself.
- The engine writes one `tbl_readiness_values` row per `(product, variant, channel, language)`; Amazon marketplace identity is collapsed into `language`.
- Every trigger is fire-and-forget (`.catch(() => {})`), so no caller sees failures.

### A.3 Inventory of every trigger call site

Legend for the "Scope passed" column: `P` = productId, `V` = variantId, `C` = channelId, `∅` = not passed / null.
Paths are relative to `api/apps/api-main/src/modules/app/catalog/products/` unless they start with another module.

#### A.3.1 Operational — stock

| # | File : line | Method | Source | Scope passed |
|---|---|---|---|---|
| 1 | `stock/product-stock.service.ts:203` | `addOrUpdateStock` | `STOCK_UPDATED` | P, V?, C=∅ |
| 2 | `:357, :380, :635` | `syncAmazonStock` | `AMAZON_STOCK_UPDATED` | P, V / ∅, C |
| 3 | `:943` | `syncCategraStockOnShopify` | `SHOPIFY_STOCK_UPDATED` | P, V, C |
| 4 | `:1045` | `syncShopifyStock` | `SHOPIFY_STOCK_UPDATED` | P, V, C |
| 5 | `:4306, :4355` | `updateAmazonStock` | `AMAZON_STOCK_UPDATED` | P, V, C |
| 6 | `:4531` | `fetchAmazonMarketplaceStock` | `AMAZON_STOCK_UPDATED` | P, V?, C |
| 7 | `:4564, :4591` | `updateShopifyStock` | `SHOPIFY_STOCK_UPDATED` | P, V / ∅, C |
| 8 | `:4832, :4861, :4993` | `switchToFba` | `AMAZON_STOCK_UPDATED` | P, V / ∅, C |
| 9 | `:5062, :5081, :5227` | `switchToFbm` | `AMAZON_STOCK_UPDATED` | P, V / ∅, C |
| 10 | `inventory/warehouses/warehouse-stock/warehouse-stock.service.ts:1014, :1036` | `createWarehouseStock` | `WAREHOUSE_STOCK_UPDATED` | P, V?, C=∅ |
| 11 | `:2122` | `addWarehouseTransferStock` | `WAREHOUSE_STOCK_UPDATED` | P, V?, C=∅ |
| 12 | `:3444` | `syncAmazonStockDetect` (batch) | `AMAZON_STOCK_UPDATED` | targets[] P, V, C |
| 13 | `:3476, :3553` | `updateAmazonStock` | `AMAZON_STOCK_UPDATED` | P, V?, C (resolved from marketplace) |
| 14 | `variants/product-variant.service.ts:2506` | `updateAmazonChannelMarketplaceDefaultWarehouse` (batch) | `AMAZON_STOCK_UPDATED` / `SHOPIFY_STOCK_UPDATED` | targets[] P, V, C |
| 15 | `system/operations/action-center/action-center.service.ts:7965` | `saveRemediationVariantOperationalValues` | `BULK_STOCK_UPDATED` / `BULK_PRICE_UPDATED` | P, C, V only if exactly one saved |

#### A.3.2 Operational — price

| # | File : line | Method | Source | Scope passed |
|---|---|---|---|---|
| 16 | `pricing/product-pricing.service.ts:1015` | `updateProductVariantsPricingForMaster` | `BULK_PRICE_UPDATED` | P, V=∅, C=∅ |
| 17 | `:6142` | `updatePricingForMaster` | `PRICE_UPDATED` | P, V?, C=∅ |
| 18 | `:7508` | `updatePricingForAmazonMarketplace` | `AMAZON_PRICE_UPDATED` | P, V?, C |
| 19 | `:8122` | `updatePricingForShopify` | `SHOPIFY_PRICE_UPDATED` | P, V?, C |
| 20 | `:772, :1379, :1526` | bulk pricing paths (batch) | `BULK_PRICE_UPDATED` | targets[] |

#### A.3.3 Content — attributes / structure / translation

| # | File : line | Method | Source | Scope passed |
|---|---|---|---|---|
| 21 | `lifecycle/update-details.service.ts:525` | `updateProductGroupAttributes` | `PRODUCT_CONTENT_UPDATED` | P, V?, C (single channel derived from attribute rows, else ∅) |
| 22 | `:923` | `updateProductEssentials` | `PRODUCT_CONTENT_UPDATED` | P, V?, C? |
| 23 | `:1335` | `updateProductPrimaryAttributes` | `PRODUCT_STRUCTURE_UPDATED` | P, V?, C? |
| 24 | `:1695` | `updateShopifyAttributeMapping` | `SHOPIFY_METAFIELD_UPDATED` | P, C |
| 25 | `:287` | `updateProductDetails` (groups 21–24 into one `recalculateUniqueTargets`) | grouped: `PRODUCT_STRUCTURE_UPDATED` > `SHOPIFY_METAFIELD_UPDATED` > `PRODUCT_CONTENT_UPDATED` | targets[] |
| 26 | `variants/product-variants-flow.service.ts:3607` | `createNewVariant` | `PRODUCT_VARIANT_STRUCTURE_UPDATED` | P, V, C? |
| 27 | `:4332` | `updateProductVariant` | `PRODUCT_VARIANT_CONTENT_UPDATED` | P, V, C=∅ |
| 28 | `:4827` | `updateProductVariantForChannel` | `PRODUCT_VARIANT_CONTENT_UPDATED` | P, V, C |
| 29 | `:7955` | `updateShopifyProductVariantAttributeMappings` | `SHOPIFY_METAFIELD_UPDATED` | P, V, C |
| 30 | `shopifyProduct/shopify-product.service.ts:1791` | `addAttributesToMetafieldsGroup` (batch) | `SHOPIFY_METAFIELD_UPDATED` | targets[] P, C |
| 31 | `:2087` | `removeAttributesFromMetafieldsGroup` (batch) | `SHOPIFY_METAFIELD_UPDATED` | targets[] P, C |
| 32 | `:2412` | `deleteAttributeFromProduct` (batch) | `SHOPIFY_METAFIELD_UPDATED` | targets[] P, C |
| 33 | `product.service.ts:16861` | `aiTranslationForProductCreation` | `PRODUCT_TRANSLATION_UPDATED` | P, V=∅, C=∅ |
| 34 | `:17206` | `processTranslationUpdates` | `PRODUCT_TRANSLATION_UPDATED` | P, V=∅, C=∅ |

`PRODUCT_LOCALIZED_CONTENT_UPDATED` is declared in the source union but **never emitted**.

#### A.3.4 Media

All go through `product.service.ts:667-757` (`runOrScheduleMediaReadinessRecalculation`, deferred to `transaction.afterCommit` when a transaction is present).

| # | File : line | Method | Source | Scope passed |
|---|---|---|---|---|
| 35 | `product.service.ts:8406` | `upsertProductMedia` | derived: `PRODUCT_VARIANT_MEDIA_UPDATED` / `CHANNEL_MEDIA_UPDATED` / `PRODUCT_MEDIA_UPDATED` | P, V?, C? |
| 36 | `:8528` | `setMainImage` | `PRODUCT_MEDIA_MAIN_IMAGE_UPDATED` | P, V?, C? |
| 37 | `:8707` | `deleteProductMedia` | `PRODUCT_MEDIA_DELETED` | P, V?, C? |
| 38 | `:8751` | `deleteImage` | `PRODUCT_MEDIA_DELETED` | P, V?, C? (from the image row) |
| 39 | `:8929` | `overrideMissingMediaScope` | `CHANNEL_MEDIA_UPDATED` | P, V?, C |
| 40 | `:9297` | `updateImageSequences` | `PRODUCT_MEDIA_REORDERED` | P only |
| 41 | `:9453` | `updateImageSequenceScope` | `PRODUCT_MEDIA_REORDERED` | P, V?, C? |
| 42 | `:12530` | `uploadChunkImages` | derived (as 35) | P, V?, C? |
| 43 | `:12714` | `addDigitalAssestsInProduct` | `DAM_MEDIA_ASSIGNED` | P, V?, C? |
| 44 | `:17363` | `updatePrimaryMediaInheritance` | `PRODUCT_MEDIA_INHERITANCE_UPDATED` | P, V?, C? |
| 45 | `variants/product-variants-flow.service.ts:3274` | `updateShopifyProductVariantImg` | `PRODUCT_MEDIA_MAIN_IMAGE_UPDATED` | P, V, C |
| 46 | `:5292` | `handelVariantImagesCreate` | `PRODUCT_VARIANT_MEDIA_UPDATED` | P, V, C? |
| 47 | `sync/productSyncing/amazon/backwardSync/amazon-product-bs.service.ts:2319` | `attachAmazonImages` | `SYNC_MEDIA_IMPORTED` | targets[] from media-linker rows (P, V?, C?) |
| 48 | `system/media/digital-assets/digital-assets.service.ts:2378` | `downloadAndAttachShopifyImagesInBulk` | `SYNC_MEDIA_IMPORTED` | targets[] from media-linker rows |

#### A.3.5 Direct `calculateReadiness()` calls (bypass the recalculation service — no socket emit, no source)

| # | File : line | Method | When | Scope passed |
|---|---|---|---|---|
| 49 | `readiness/views/readiness.controller.ts:27` | `POST /readiness/recalculate` | manual UI request | P, C? |
| 50 | `readiness/rules/readiness-rules.service.ts:464` | `recalculateReadinessForRuleChange` | after `createReadinessRules` (`:649`) and `updateReadinessRules` (`:739`) | every affected P (SQL), C from the rule |
| 51 | `product.service.ts:1885` | `createProduct` (`isCreatedInCategra`) | product created in Categra | P, C? |
| 52 | `listings/product-channel.service.ts:5582` | `UpdateMarketplaceDetail` | Amazon marketplace detail saved | P only (all channels) |
| 53 | `:5809` | `updatetShopifyProducChanelDetails` | Shopify channel detail saved | P, C |
| 54 | `identity/accounts/account/accountConfiguration/account-configuration.service.ts:1122` | `calculateAllProductReadiness` ← `updateAccountConfiguration:363` | account languages changed | every product of the account, all channels |
| 55 | `sync/productSyncing/amazon/backwardSync/amazon-product-bs.service.ts:4507` | `calculateImportedProductsReadiness` ← `amazon-product-bsq4.service.ts:328`, `listings/productSearchByAsin/amazon-product-cbaq4.service.ts:256` | Amazon import batch done / ASIN search import | each imported P, all channels |
| 56 | `sync/productSyncing/shopify/backwardSync/shopify-product-bsq3.service.ts:2758` | `calculateReadinessForProducts` ← `:284` | Shopify import batch done | each imported P, C |

Commented-out: `sync/productSyncing/productStructureManagement/product-structure-management.service.ts:601` (`recalculateReadinessValues`) — structural change relies on the later Q4 import trigger (#55).

### A.4 Mutations that change readiness but have **no trigger today**

| Gap | Where the mutation happens | Why it matters |
|---|---|---|
| G1 | Order-driven stock deduction: `commerce/orders/order/amazon/amazon-order.service.ts`, `sync/productSyncing/shopify/backwardSync/shopify-order-bsq5.service.ts`, `inventory/.../warehouse-product-location.service.ts`, `inventory/.../warehouse-location.service.ts` | Stock rule value changes silently |
| G2 | Amazon marketplace enabled / disabled on channel (`platforms/amazon/channel-config/marketplace/amazon-channel-marketplace.service.ts:293-302` seeds/removes rules, never recalculates) | New language scope appears with no rows; disabled scope keeps stale rows |
| G3 | Product listed / unlisted on a marketplace (`pcm.status`), variant listed / unlisted (`vm.status`) | Same as G2 at product level |
| G4 | Shopify language mapping added / removed (`catalog/channel/shopifyChannel/shopify-channel.service.ts:1240, :1353` seeds rules only) | Same as G2 for Shopify |
| G5 | Product channel activated / deactivated (`pc.is_active`) | Scope appears / disappears |
| G6 | `seedPredefinedReadinessRules` / `removeReadinessRules` (attribute create `catalog/attributes/attribute-core/attribute.service.ts:220`, channel create `catalog/channel/channel.service.ts:878`, language add `identity/.../account-language.service.ts:125`) | Rules change → denominators change, no recompute |
| G7 | Product attribute group assigned / unassigned (`tbl_product_attribute_groups`) | Changes which attributes the content rules see |
| G8 | Variant / product deleted or deactivated | Stale `tbl_readiness_values` rows are never removed |

---

## Part B — Target trigger contract

### B.1 The scope object

Every readiness run is described by exactly one `ReadinessScope`. Narrower keys are optional; omitting a key means "everything under the keys above it".

```ts
type ReadinessChannelSelector =
	| 'ALL'          // master + every active channel of the product(s)
	| 'MASTER'       // master scope only (channel_id IS NULL rows)
	| number;        // exactly this tbl_channels.id (Amazon or Shopify)

type ReadinessRuleType = 'stock' | 'price' | 'bullet_point';   // GlobalEnums.ReadinessRuleTypes

interface ReadinessScope {
	accountId: number;                    // REQUIRED — from req.user.accountId; never trusted from the client
	channel: ReadinessChannelSelector;    // REQUIRED — singular (see B.2)
	languageCodes?: string[];             // omit = every ACTIVE language of each resolved channel scope
	productIds?: number[];                // omit = every product of the account (rule / language / channel-level events only)
	variantIds?: number[];                // omit = parent + every active variant; set = ONLY those variants (D4)
	attributeIds?: number[];              // omit = all attribute-backed rules are in range
	ruleTypes?: ReadinessRuleType[];      // omit = all type-backed rules (stock / price / bullet_point) are in range
}

interface ReadinessRecalculationRequest {
	req: AuthenticatedRequest;
	source: ReadinessRecalculationSource;
	scopes: ReadinessScope[];             // batch API; the service merges scopes (B.4)
}
```

### B.2 Singular vs plural — final decision

| Key | Cardinality | Why |
|---|---|---|
| `accountId` | singular, required | A request is always executed as one tenant. |
| `channel` | **singular** | Every real mutation happens in exactly one channel context (an Amazon marketplace update, a Shopify price update, a master edit). The only multi-channel case is "everything that inherits from master", and that is exactly what `'ALL'` means (D3). A plural `channelIds[]` would only ever be used as `'ALL'` in practice and would force callers to enumerate channels they do not know about. Cross-channel batches (sync imports, rule changes) send one scope per channel. |
| `languageCodes` | **plural**, optional | Translation updates (`processTranslationUpdates`) and localized edits write several languages in one commit. Amazon marketplaces are addressed by language too (readiness rows have no marketplace id), so "en + de marketplaces of channel 12" is `channel: 12, languageCodes: ['en','de']`. |
| `productIds` | **plural**, optional | Sync imports, metafield-group changes and rule changes touch many products with the same narrower keys. Omitting it is the only sane way to express "all products of the account" for rule / account-language / channel-level events. |
| `variantIds` | **plural**, optional | Bulk stock / price / remediation saves touch several variants of one product in one commit. **Constraint:** `variantIds` may only be set when `productIds` has exactly one entry; the service rejects (logs + skips) a scope that violates this — a variant list across several products is ambiguous and must be sent as one scope per product. |
| `attributeIds` | **plural**, optional | `updateProductGroupAttributes` and metafield-group add/remove change several attributes at once. |
| `ruleTypes` | **plural**, optional | Symmetric with `attributeIds` — both are rule gates (D2). A single mutation rarely touches two types, but an import or a remediation can (stock + price). Plural costs nothing. |

An empty array (`variantIds: []` etc.) is treated as omitted.

### B.3 How the engine resolves a scope (replaces the `calculateReadiness()` fan-out)

```
resolve(scope):
  products  = scope.productIds ?? all products of accountId (is_deleted = false)
  for each product:
    entities = scope.variantIds
                 ? those variants (must belong to product, is_active = true)             -- D4: parent NOT added
                 : [parent] + every active variant of the product
    channelScopes =
      scope.channel === 'MASTER' → [master]
      scope.channel === 'ALL'    → [master] + every tbl_product_channels row with pc.is_active = true AND pc.is_deleted = false
      scope.channel === <id>     → that product-channel row only (skip if inactive / deleted / not linked)

    for each channelScope:
      activeLanguages =
        master  → tbl_account_languages ∪ base language
        Shopify → tbl_shopify_language_mappings WHERE channel_id = c AND is_deleted = false
        Amazon  → (D1)  SELECT acm.language
                        FROM tbl_product_channel_marketplaces pcm
                        JOIN tbl_amazon_channel_marketplaces acm ON acm.id = pcm.amazon_channel_marketplace_id
                        WHERE pcm.product_channel_id = pc.id
                          AND pcm.status = true AND pcm.is_deleted = false
                          AND acm.status = true
                        -- variant entity: additionally JOIN tbl_variant_marketplaces vm
                        --   ON vm.product_channel_marketplace_id = pcm.id
                        --   AND vm.product_variant_id = :variantId
                        --   AND vm.status = true AND vm.is_deleted = false
      languages = scope.languageCodes ? activeLanguages ∩ scope.languageCodes : activeLanguages

      for each (entity, language):
        rules = active tbl_readiness_configurations for (accountId, channel_id, language)
        if scope.attributeIds or scope.ruleTypes is set:                        -- D2 gate
          if no rule in `rules` has attribute_id ∈ attributeIds
             and no rule has configurations.type ∈ ruleTypes → SKIP this (entity, language)
        recompute the FULL score for (product, entity, channel, language) and upsert tbl_readiness_values
```

Consequences of D1 / D4 that differ from today:

- Amazon parent scope now checks `pcm.status`, `pcm.is_deleted`, `acm.status` (today: none).
- Amazon variant scope now checks `vm.is_deleted`, `pcm.status`, `pcm.is_deleted`, `acm.status` (today: `vm.status` only).
- Shopify now checks `ShopifyLanguageMapping.is_deleted`.
- `channel: <id>` no longer silently recomputes master as well.
- `variantIds` is honoured; parent is not touched.
- Rows for a `(product, variant, channel, language)` that is no longer active are **deleted**, not left stale (fixes G2–G5, G8).

### B.4 Merging rules for a batch (`scopes[]`)

Two scopes merge when `accountId`, `channel`, and `productIds` (as a set) are equal:

| Key | Merge result |
|---|---|
| `languageCodes` | union; if either side omitted → omitted |
| `variantIds` | union; if either side omitted → omitted (parent + all variants) |
| `attributeIds` | union; if either side omitted → omitted (gate off) |
| `ruleTypes` | union; if either side omitted → omitted (gate off) |

`channel: 'ALL'` absorbs `'MASTER'` and any `<id>` for the same product set. This replaces today's `${productId}:${channelId ?? 'all'}` key, which drops `variantId` on the first conflict.

### B.5 Genuine triggers and the scope each must request

`accountId` is always present and omitted from the table. Keys not listed are omitted.

#### B.5.1 Stock

| Trigger (source) | Originates today at | Requested scope |
|---|---|---|
| Master stock edit (`STOCK_UPDATED`) — `addOrUpdateStock` | #1 | `channel:'ALL'`, `productIds:[P]`, `variantIds:[V]` if given, `ruleTypes:['stock']` — `'ALL'` because channel scopes fall back to master stock (D3) |
| Warehouse stock created / transferred (`WAREHOUSE_STOCK_UPDATED`) | #10, #11 | same as above |
| Order-driven stock deduction (**new**, `WAREHOUSE_STOCK_UPDATED`) | G1 | same as above, one scope per affected `(product, variant)`; batch per order |
| Amazon stock pushed / pulled / detected (`AMAZON_STOCK_UPDATED`) | #2, #5, #6, #12, #13 | `channel:C`, `productIds:[P]`, `variantIds:[V…]`, `languageCodes:[marketplace language]`, `ruleTypes:['stock']` — the marketplace is known at every site, so the language is known; omit `languageCodes` only when the site really touches every marketplace of the channel |
| Switch to FBA / FBM (`AMAZON_STOCK_UPDATED`) | #8, #9 | `channel:C`, `productIds:[P]`, `variantIds:[V]` or omitted, `languageCodes:[marketplace language]`, `ruleTypes:['stock']` |
| Shopify stock pushed / pulled (`SHOPIFY_STOCK_UPDATED`) | #3, #4, #7 | `channel:C`, `productIds:[P]`, `variantIds:[V]` or omitted, `ruleTypes:['stock']` — Shopify stock is not per language, so all active languages of the channel |
| Default warehouse for a marketplace / channel changed (`AMAZON_STOCK_UPDATED` / `SHOPIFY_STOCK_UPDATED`) | #14 | `channel:C`, `productIds:[P]`, `variantIds:[V…]`, `ruleTypes:['stock']` (+ `languageCodes` for Amazon) |
| Bulk remediation save, stock (`BULK_STOCK_UPDATED`) | #15 | `channel:C`, `productIds:[P]`, `variantIds:[every saved V]` (today collapses to null when > 1), `ruleTypes:['stock']` |

#### B.5.2 Price

| Trigger (source) | Originates today at | Requested scope |
|---|---|---|
| Master price edit (`PRICE_UPDATED`) | #17 | `channel:'ALL'`, `productIds:[P]`, `variantIds:[V]` if given, `ruleTypes:['price']` |
| Master bulk price (`BULK_PRICE_UPDATED`) | #16, #20 | `channel:'ALL'`, `productIds:[P]`, `variantIds:[touched V…]`, `ruleTypes:['price']` |
| Amazon marketplace price (`AMAZON_PRICE_UPDATED`) | #18 | `channel:C`, `productIds:[P]`, `variantIds:[V]` if given, `languageCodes:[marketplace language]`, `ruleTypes:['price']` |
| Shopify price (`SHOPIFY_PRICE_UPDATED`) | #19 | `channel:C`, `productIds:[P]`, `variantIds:[V]` if given, `ruleTypes:['price']` |
| Bulk remediation save, price (`BULK_PRICE_UPDATED`) | #15 | `channel:C`, `productIds:[P]`, `variantIds:[every saved V]`, `ruleTypes:['price']` |

#### B.5.3 Content (attribute values)

| Trigger (source) | Originates today at | Requested scope |
|---|---|---|
| Master attribute values edited (`PRODUCT_CONTENT_UPDATED`) | #21, #22 (channel ∅) | `channel:'ALL'`, `productIds:[P]`, `variantIds:[V]` if given, `languageCodes:[edited languages]`, `attributeIds:[changed attribute ids]` — `'ALL'` because channel values fall back to master (`count_inherited`) |
| Channel attribute values edited (`PRODUCT_CONTENT_UPDATED`) | #21, #22 (channel set), #28 | `channel:C`, `productIds:[P]`, `variantIds:[V]` if given, `languageCodes:[edited languages]`, `attributeIds:[changed]` |
| Localized content edited (`PRODUCT_LOCALIZED_CONTENT_UPDATED` — currently never emitted) | — | as the two rows above with `languageCodes` always set; emit it instead of `PRODUCT_CONTENT_UPDATED` when the edit is language-specific |
| Primary attributes / structure changed (`PRODUCT_STRUCTURE_UPDATED`) | #23 | `channel:'ALL'` or `C` as passed, `productIds:[P]`, `variantIds:[V]` if given — no `attributeIds` (structure changes can alter which rules apply) |
| Variant created (`PRODUCT_VARIANT_STRUCTURE_UPDATED`) | #26 | `channel:'ALL'`, `productIds:[P]`, `variantIds:[new V]` |
| Variant content edited (`PRODUCT_VARIANT_CONTENT_UPDATED`) | #27 | `channel:'ALL'`, `productIds:[P]`, `variantIds:[V]`, `languageCodes`, `attributeIds` as known |
| Shopify metafield mapping changed (`SHOPIFY_METAFIELD_UPDATED`) | #24, #29, #30–#32 | `channel:C`, `productIds:[P…]`, `variantIds:[V]` if given, `attributeIds:[mapped attribute ids]` |
| AI translation written (`PRODUCT_TRANSLATION_UPDATED`) | #33, #34 | `channel:'ALL'`, `productIds:[P]`, `languageCodes:[target languages]` — translations are master-scope values that channels inherit |
| Product created in Categra | #51 | `channel:'ALL'`, `productIds:[P]` (`source: 'PRODUCT_IMPORTED'`, see B.6) |
| Product attribute group assigned / unassigned (**new**, `PRODUCT_STRUCTURE_UPDATED`) | G7 | `channel:'ALL'`, `productIds:[P]` |

#### B.5.4 Media

| Trigger (source) | Originates today at | Requested scope |
|---|---|---|
| Any product-level media mutation (`PRODUCT_MEDIA_*`, `DAM_MEDIA_ASSIGNED`) with channel ∅ | #35–#44 | `channel:'ALL'`, `productIds:[P]`, `variantIds:[V]` if given, `languageCodes:[media language]` if the media row has one (omit when `language_code IS NULL` — those rows count for every language), `attributeIds:[PRIMARY_MEDIA attribute id]` |
| Channel-scoped media mutation (`CHANNEL_MEDIA_UPDATED`, `PRODUCT_MEDIA_MAIN_IMAGE_UPDATED` with channel) | #39, #45 | `channel:C`, same narrowing as above |
| Variant media (`PRODUCT_VARIANT_MEDIA_UPDATED`) | #46 | `channel:'ALL'` or `C` as passed, `productIds:[P]`, `variantIds:[V]`, `attributeIds:[PRIMARY_MEDIA]` |
| Image sequence reordered (`PRODUCT_MEDIA_REORDERED`) | #40, #41 | same as product-level; only the main-image rule can change |
| Media imported by sync (`SYNC_MEDIA_IMPORTED`) | #47, #48 | one scope per distinct `(product, channel)` in the linker rows: `channel:C or 'ALL'`, `productIds:[P]`, `variantIds:[V…]`, `attributeIds:[PRIMARY_MEDIA]` |
| Primary-media inheritance toggled (`PRODUCT_MEDIA_INHERITANCE_UPDATED`) | #44 | `channel:'ALL'`, `productIds:[P]`, `variantIds:[V]` if given, `attributeIds:[PRIMARY_MEDIA]` |

#### B.5.5 Rules and account / channel configuration (recompute-everything events — `productIds` omitted)

| Trigger | Originates today at | Requested scope |
|---|---|---|
| Readiness rule created / updated / enabled / disabled (`READINESS_RULE_CHANGED`) | #50 | `channel: rule.channel_id ?? 'MASTER'`, `languageCodes:[rule.language_code]`, no `attributeIds` / `ruleTypes` — a rule turning *off* still changes the denominator, and the D2 gate would look at the post-change rule set and skip. So rule changes always recompute the whole `(channel, language)` slice. Product list stays what `getRuleChangeAffectedProductIds` computes today → pass it as `productIds` |
| Rules seeded for new attribute / channel / marketplace / language (**new**, `READINESS_RULE_CHANGED`) | G6 | same as above, one scope per seeded `(channel, language)` |
| Account languages changed (`ACCOUNT_LANGUAGES_CHANGED`) | #54 | `channel:'ALL'`, `languageCodes:[added languages]`; removed languages → delete rows instead of recompute |
| Amazon marketplace enabled on channel (**new**, `CHANNEL_SCOPE_CHANGED`) | G2 | `channel:C`, `languageCodes:[marketplace language]` |
| Amazon marketplace disabled on channel (**new**) | G2 | delete rows `channel_id = C AND language_code = L` for every product; no recompute |
| Shopify language mapping added / removed (**new**) | G4 | as G2 |
| Product listed / unlisted on a marketplace, variant listed / unlisted (**new**) | G3 | `channel:C`, `productIds:[P]`, `variantIds:[V]` if variant-level, `languageCodes:[marketplace language]`; unlisting → delete rows |
| Product channel activated / deactivated (**new**) | G5 | `channel:C`, `productIds:[P]`; deactivation → delete rows |
| Amazon marketplace detail saved | #52 | `channel:C` (today passes nothing → all channels), `productIds:[P]`, `languageCodes:[marketplace language]` |
| Shopify channel detail saved | #53 | `channel:C`, `productIds:[P]` |
| Sync import batch completed (Amazon Q4 / ASIN search / Shopify Q3) (`PRODUCT_IMPORTED`) | #55, #56 | `channel:C`, `productIds:[batch P…]` — today Amazon passes no channel (all channels recomputed for every imported product) |
| Manual `POST /readiness/recalculate` | #49 | accept the full `ReadinessScope` from the body (with `accountId` overwritten from the JWT) |
| Variant / product deleted or deactivated (**new**, `ENTITY_REMOVED`) | G8 | no recompute — delete `tbl_readiness_values` rows for the entity |

### B.6 Source enum additions

Add to `ReadinessRecalculationSource` (`types/readiness.types.ts`):

```
'READINESS_RULE_CHANGED'          // B.5.5 rule create / update / seed
'CHANNEL_SCOPE_CHANGED'           // G2–G5 marketplace / language / listing / product-channel status
'ACCOUNT_LANGUAGES_CHANGED'       // #54
'PRODUCT_IMPORTED'                // #51, #55, #56
'ENTITY_REMOVED'                  // G8 cleanup-only
```

`PRODUCT_LOCALIZED_CONTENT_UPDATED` already exists; start emitting it. G1 reuses `WAREHOUSE_STOCK_UPDATED`.

### B.7 Call-site migration summary

| Today | Target |
|---|---|
| `recalculateForMutation({ productId, channelId?, variantId?, source })` | `recalculate({ req, source, scopes: [scope] })` |
| `recalculateUniqueTargets(req, source, targets[])` | `recalculate({ req, source, scopes })` — merge per B.4 |
| direct `calculateReadiness({ productId, channelId? })` (#49–#56) | route through the recalculation service so every run has a `source`, a socket emit and a log line |
| `channelId: null` / `undefined` meaning "all" | explicit `channel: 'ALL'` / `'MASTER'` / `<id>` — no more null-vs-undefined |

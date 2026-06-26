look at file context\delta-sync-review.html, it is the initial prototype for the below feature:

Now similar UI (with changes if needed) will be implemented when someone tries to sync a product/variant or bunch of products or variants, user can select whatever changed points he needs to be synced with external channel (Amazon catalog or Shopify catalog) instead of pushing all the changes, but it is not that simple, it has some dependency rules and constraints, so for every type of baseline the selection rules and constraints are listed below:

1. Attributes
    1. Amazon: Selection will be hierarchy based: ex. item_package_dimensions has package length and package unit, so both cannot be selected independently
    2. Shopify: An individual attribute can be selected, no problem at all
2. Pricing - Single Select
    1. One select as a whole to push whole pricing structure at all or not
    2. For amazon: attributes come under “purchasable_offer” & “list_price” will also be the part of pricing group logically
    3. Reason:
        1. Amazon doesn’t allow a partial pricing update, the whole pricing structure must be updated
3. Media - Single Select
    1. One select as a whole to push changed media snapshot at all or not
    2. Reason:
        1. It will be way much complex for individual changes selection as images can be added, deleted and sequences can be changed
4. Variant Structure
    1. Amazon: Variant wise one select to push the current structure for the variant itself
    2. Shopify: one common select that will affect every published variant for the product
5. Variant Attributes
    1. One select as a whole to push all variant attribute value changes at all or not
    2. Reason:
        1. We can’t put inconsistent variant combinations: ex. Color + Size both have changes but if we push only for color then at external platform, the variant will be evaluated as per (new Color + old Size)
6. Miscellaneous
    1. Individual select for below fields
        1. TAGS - tags
        2. SHOPIFY_CATEGORY - category
        3. SHOPIFY_THEME_TEMPLATE - templateSuffix
        4. SHOPIFY_EXTERNAL_PUBLICATIONS - publications
        5. SHOPIFY_COLLECTIONS - collections
        6. AMAZON_CONDITION_TYPE - condition_type
        7. AMAZON_SHIPPING_TEMPLATE - merchant_shipping_group
        8. AMAZON_CATEGORY - recommended_browse_nodes or item_type_keyword

**Constraints:**

- Variant attributes dependency on variant structure
    - Amazon:
        - For a market if a structure change is selected for a variant, all its sibling variants within the common wrapper will also get selected for structure
        - If user tries to select a variant for variant attributes then if structure change is also present for that variant, it will also get selected along with it
    - Shopify: If user tries to select a variant for variant attributes, If variant structure is changed, all the variants variant attributes will be selected with the variant structure itself

---

The actual entities are:

src\entities\attributeSyncBaseline.entity.ts
src\entities\priceSyncBaseline.entity.ts
src\entities\productMediaSyncBaseline.entity.ts
src\entities\variantStructureBaseline.entity.ts
src\entities\variantAttributeSyncBaseline.entity.ts
src\entities\miscSyncBaseline.entity.ts

---

After reviewing all the context, Now, I am going to ask you some questions or clarifications one by one, answer them accordingly

====================================================================

# SELECTIVE SYNC — FE & BE CONSIDERATIONS + PAYLOAD FORMAT

====================================================================

## 0. Scoping reference (drives everything)

A sync operation runs in one **scope** = one product × one channel × (one marketplace for Amazon | none for Shopify). Within a scope there are two levels: **product** (variantId = null) and **variant** (per variantId).

Each baseline keys on a different set of dimensions — this is the single most important fact:

| Group              | Keyed by                                                                      | Marketplace dim?          | Level             | Selection unit                                                                                |
| ------------------ | ----------------------------------------------------------------------------- | ------------------------- | ----------------- | --------------------------------------------------------------------------------------------- |
| Attributes         | product/variant, channel, **marketplace**, attributeId, **hierarchyId**       | Amazon: yes / Shopify: no | product + variant | Amazon: `hierarchyId` (grouped) **+** `attributeId` (direct/primary) · Shopify: `attributeId` |
| Pricing            | product/variant, channel, **marketplace**, priceType                          | yes                       | product + variant | one flag                                                                                      |
| **Media**          | product/variant, **channel**                                                  | **NO (channel-level)**    | product + variant | one flag                                                                                      |
| Variant structure  | **variantId (required)**, channel, **marketplace**                            | yes                       | variant only      | per-variant flag → group effect                                                               |
| Variant attributes | **variantId (required)**, channel, **marketplace**, productVariantAttributeId | yes                       | variant only      | per-variant flag (all-or-nothing)                                                             |
| Misc               | product/variant, channel, **marketplace**, baselineType                       | yes                       | product + variant | per `baselineType`                                                                            |

Two structural facts: **(a) Media is the only channel-level baseline** (no `amazon_marketplace_id`); **(b) Variant structure & variant attributes never exist at product level.**

Clarifications locked:

- **Amazon attributes are NOT 100% hierarchy-based.** They split into two sub-groups in the SAME Attributes group: **(1) direct** — primary attributes / attributes whose rules allow independent selection (selected per `attributeId`, like Shopify); **(2) hierarchy-grouped** — selected as a whole root (`hierarchyId`). Both can appear at product and variant level.
- An attribute never maps to more than one group/hierarchy per channel + marketplace → no FE disambiguation needed.
- Media is synced for **every** marketplace regardless (same media sent to all); FE shows the common media once per channel with a "common to all marketplaces" indication.

---

## 1. FRONTEND considerations (per case)

### A. Channel type (Amazon vs Shopify)

1. **Attributes — Amazon**: render in ONE Attributes group but as two kinds of unit — **direct** attributes (primary / rule-based) get an independent checkbox each (like Shopify); **hierarchy-grouped** attributes get one checkbox per root, with composite roots (e.g. `item_package_dimensions` = length + unit) rendering children as **read-only detail under the single parent checkbox**. Only the grouped ones carry the hierarchy badge; direct ones look like a normal attribute.
2. **Attributes — Shopify**: flat list, one independent checkbox per attribute (all direct).
3. **Pricing-owned attributes (Amazon)**: roots `purchasable_offer` / `list_price` are **excluded from Attributes** and shown inside the Pricing card. (Shopify: no exclusion.)
4. **Misc field set depends on channel**: Shopify → TAGS, SHOPIFY_CATEGORY, SHOPIFY_THEME_TEMPLATE, SHOPIFY_EXTERNAL_PUBLICATIONS, SHOPIFY_COLLECTIONS. Amazon → AMAZON_CONDITION_TYPE, AMAZON_SHIPPING_TEMPLATE, AMAZON_CATEGORY.
5. **Card bodies differ**: Amazon pricing shows list/offer/sale/business/qty-discounts; Shopify list/offer only. Amazon structure shows the parent **wrapper** block (parent SKU, ASIN, variation theme); Shopify shows axes only.

### B. Marketplace dimension (Amazon)

6. Marketplace chips render **only for Amazon**; Shopify has none.
7. Switching marketplace re-renders marketplace-scoped groups (attributes, pricing, structure, variant attributes, misc) — different change sets per marketplace; selection state is per (channel, marketplace).
8. Media does **not** re-render on marketplace switch (see C).

### C. Media — channel-common special case

9. Same media shown in **every** marketplace view of an Amazon channel.
10. Media selection state is **shared across marketplaces** (selecting in US shows selected in CA).
11. Show a "Channel-wide" indication so users know media syncs to all marketplaces of the channel. (Same for variant media.)
12. Metrics must count media **once per channel**, not once per marketplace.
13. After a sync including media, media disappears from **all** marketplace views of that channel (baseline cleared channel-wide).

### D. Cascade / auto-select (constraints — UX side; backend enforces truth)

14. **Amazon structure cascade**: selecting structure for variant V auto-selects + marks structure for **all sibling variants in V's wrapper** (that marketplace). Deselect cascades off too.
15. **Shopify structure cascade**: selecting any variant's structure auto-selects **all** variants' structure.
16. **Variant attributes → structure**: selecting a variant's attributes, if that variant's structure changed, auto-selects its structure (Amazon → triggers #14 siblings); Shopify → selects **all** variants' attributes + structure.
17. Auto-pulled items are visually marked ("Auto") and **locked** while the dependency holds, so an inconsistent selection can't be made.
18. **Variant attributes is single-flag**: individual changed attrs (Color, Size) render **read-only** under one checkbox.

### E. Notes / warnings

19. Structure (Amazon): "Selecting structure also syncs sibling variants in the same wrapper: SKU-A, SKU-B."
20. Structure (Shopify): "Structure is shared across the product — selecting it syncs all variants."
21. Variant attributes (Amazon): "Selecting attributes also selects this variant's structure (and siblings) if structure changed."
22. Variant attributes (Shopify): "Selecting any variant's attributes selects all variants' attributes + structure if structure changed."
23. Pricing (Amazon): "Amazon requires the whole pricing block; partial pricing can't be synced."

### F. Block / structure

24. Product block shows: Attributes, Pricing, Media, Misc (no Structure / Variant Attributes).
25. Variant block shows all six groups.
26. Amazon variant blocks should display/group by **wrapper** membership (needed for #14/#19).

### G. Change-type & empty states

27. Added / Removed / Updated render differently (chips for attributes; NEW/DEL badges + reordering for media; axis add/remove for structure).
28. Empty scope → "No changes for {scope}" state.
29. Partially-changed composite hierarchy (Amazon) → show the parent group with changed child highlighted + unchanged siblings as context (full top-level object is what syncs).

---

## 2. BACKEND considerations (per case)

### A. Baseline write (already implemented for attributes)

- `attributeSyncBaseline.hierarchy_id` stores the **ROOT** ancestor hierarchy node id (Amazon only; Shopify null). Resolved in `processAttributeOutOfSync` per (amazonChannelId, amazonMarketplaceId, amazonProductTypeId) via `AttributeAndAmazonAttributeLinker → AmazonAttribute → AmazonAttributeHierarchy`, root = `parent_hierarchy[0]` (root-first) or self.
- Rows stay **per-attribute**; diff/revert are per-attribute. Hierarchy grouping is a selection-time concern only.

### B. Selection resolution (read → push)

- Each selector resolves to baseline rows by `(productId, variantId?, channelId, amazonMarketplaceId?)` + group key:
    - Amazon attributes → rows whose root `hierarchyId` ∈ selected `hierarchyIds` (grouped) **OR** whose `attributeId` ∈ selected `attributeIds` (direct/primary).
    - Shopify attributes → rows with selected `attributeId`.
    - Pricing → all PriceSyncBaseline rows in scope + (Amazon) attribute rows whose root hierarchy key ∈ {purchasable_offer, list_price}.
    - Media → the single channel-level ProductMediaSyncBaseline row(s) (no marketplace).
    - Structure → VariantStructureBaseline row for the variant (+ expansion, see C).
    - Variant attributes → all VariantAttributeSyncBaseline rows for the variant.
    - Misc → MiscSyncBaseline row for the selected `baselineType`.

### C. Constraint expansion — BACKEND IS SOURCE OF TRUTH (FE only warns)

- **Pricing ↔ price-attributes (Amazon)**: `pricing=true` also selects attribute baselines whose root hierarchy ∈ {purchasable_offer, list_price}.
- **Structure (Amazon)**: structure selected for V → expand to all sibling variants in V's **wrapper** within (channel, marketplace). Wrapper/sibling membership resolved from `variantStructureBaseline.previousMetaData` / current variant structure.
- **Structure (Shopify)**: structure selected for any variant → all variants' structure.
- **Variant attributes ↔ structure**: variantAttributes selected for V + V structure changed → also select V's structure (Amazon → siblings); Shopify → all variants' variantAttributes + structure.

### D. Sync execution

- On sync of a selected hierarchy/pricing group, **rebuild the full current top-level object from live data** (changed children + unchanged siblings at current values) and push — Amazon accepts whole top-level properties only.
- **Media**: always pushed to **every** marketplace of the channel (channel-level); clear the media baseline channel-wide after success.
- After a successful push, **delete/clear the corresponding baselines** for what was synced (per the existing delete/restore controllers).

### E. Misc

- Idempotent media selection across marketplaces — dedupe so the same channel media isn't pushed multiple times within one batch.
- No attribute group ambiguity (clarification): an `attributeId` maps to exactly one group/hierarchy per channel + marketplace, so selection by id is unambiguous.

---

## 3. PAYLOAD FORMAT (sync selected)

One unit = one product within one scope (channel + optional marketplace). Batch = array of these. Selection carries **raw user intent**; the backend expands constraints (section 2C).

```ts
// POST /sync/selected   body: SyncSelectionPayload[]
interface SyncSelectionPayload {
	productId: number;
	channelId: number;
	channelType: 'AMAZON' | 'SHOPIFY';
	amazonMarketplaceId?: number | null; // required for AMAZON, null/omit for SHOPIFY

	product: LevelSelection; // baselines with variantId = null
	variants: VariantSelection[]; // one per published variant in scope
}

interface LevelSelection {
	attributes?: {
		attributeIds?: number[]; // directly-selectable attributes — Shopify (all) + Amazon (direct/primary)
		hierarchyIds?: number[]; // AMAZON only: selected ROOT hierarchy ids (grouped attributes)
	};
	pricing?: boolean; // single flag (Amazon also pulls purchasable_offer/list_price)
	media?: boolean; // single flag — resolves at CHANNEL level (all marketplaces)
	miscTypes?: string[]; // selected baselineType values (e.g. ['TAGS','AMAZON_CONDITION_TYPE'])
}

interface VariantSelection extends LevelSelection {
	variantId: number;
	structure?: boolean; // variant-level only; backend expands to wrapper siblings / all variants
	variantAttributes?: boolean; // variant-level only; all-or-nothing for the variant
}
```

### Example

```jsonc
[
	{
		"productId": 101,
		"channelId": 7,
		"channelType": "AMAZON",
		"amazonMarketplaceId": 3,
		"product": {
			"attributes": {
				"attributeIds": [12, 18], // direct/primary attrs (Brand, Material)
				"hierarchyIds": [90], // grouped root (item_package_dimensions = length + height)
			},
			"pricing": true, // also pulls purchasable_offer/list_price
			"media": true, // syncs to all Amazon marketplaces
			"miscTypes": ["AMAZON_CONDITION_TYPE", "AMAZON_SHIPPING_TEMPLATE", "AMAZON_CATEGORY"],
		},
		"variants": [
			{
				"variantId": 5001,
				"attributes": { "attributeIds": [40], "hierarchyIds": [77] },
				"pricing": true, // variant-level pricing (this marketplace)
				"media": true, // variant media — channel-level, syncs to all marketplaces
				"variantAttributes": true,
				"structure": true, // backend expands to wrapper siblings
			},
			{ "variantId": 5002 },
		],
	},
	{
		"productId": 101,
		"channelId": 9,
		"channelType": "SHOPIFY",
		"product": {
			"attributes": { "attributeIds": [12, 34] },
			"media": true,
			"miscTypes": [
				"TAGS",
				"SHOPIFY_CATEGORY",
				"SHOPIFY_THEME_TEMPLATE",
				"SHOPIFY_EXTERNAL_PUBLICATIONS",
				"SHOPIFY_COLLECTIONS",
			],
		},
		"variants": [
			{ "variantId": 5001, "variantAttributes": true }, // Shopify: backend pulls all variants + structure
		],
	},
]
```

### Notes

- **Amazon attributes use BOTH** `attributeIds` (direct/primary) and `hierarchyIds` (grouped) in the same `attributes` object. Shopify uses only `attributeIds`.
- `hierarchyIds` exclude pricing roots — those are owned by the `pricing` flag.
- `media` may be sent once per channel; backend treats it channel-wide regardless of which marketplace scope sent it.
- `structure` / `variantAttributes` only appear on `variants[]`, never on `product`.

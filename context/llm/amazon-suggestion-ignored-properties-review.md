# Amazon AI suggestion: which top-level schema properties to ignore

Review of `IGNORED_AMAZON_SCHEMA_SUGGESTION_PROPERTIES` (`api/apps/api-main/src/core/constants/constants.ts`). This is a proposal only; the constant has not been changed.

## 1. Scope and method

- **Input:** the 33 full product-type schemas in `dataset/schema` (17 product types, marketplaces US, CA, UK, DE, FR, MX, IN). The `*-allof`, `*-properties`, `*-validation` and single-property files were skipped, because they repeat parts of the full schemas.
- **Properties:** 400 distinct top-level properties. 88 of them match the current ignore list.
- **Size:** each property was reduced the same way `reduceSchemaForAISuggestions` reduces it (title, first sentence of the description, type, enum, nested properties). "Avg chars" below is that reduced size, averaged over the schemas that contain the property. "Schemas" is how many of the 33 contain it.
- **Decisions you already made:**
  - Offer and operational fields are ignored, except `condition_type` (it stays suggested for variants and is skipped for parents with variants, as today).
  - Compliance and measurement fields: decided here, with the reasoning below.

## 2. Decision rules

A property is **ignored** when any of these holds:

| Rule | Why |
|---|---|
| R1 Seller- or brand-owned fact | The correct value lives outside the product content (brand, identifiers, model names, dates, contacts). The AI can only invent it, and an invented value looks plausible, which is worse than an empty field. |
| R2 Legal attestation or regulatory declaration | The seller certifies it and is liable for it (safety, hazmat, chemicals, age restrictions, origin, certifications). A suggestion that the seller clicks through becomes a false declaration. |
| R3 Offer, pricing, fulfilment or logistics | Set per offer, not per product; comes from the seller's operations or a dedicated flow (pricing has its own prompt). |
| R4 Media, links or system-managed structure | Images, URLs, parentage and variation structure are filled by the system, not by text generation. |
| R5 Large and low value | Costs a lot of input tokens in every call and rarely helps (only when the reason is stated explicitly). |

A property is **kept** when the value can be derived from the product details, title, description or images, even if the seller normally confirms it. A suggestion helps the seller decide (the task's constraint).

## 3. Prerequisite: the ignore list must not remove variation-theme properties

`reduceSchemaForAISuggestions` checks the ignore list **before** it adds the variant's `variantAttributeProperties`:

```ts
if (isIgnored || isVariantSpecificIgnored) { continue; }
```

So any ignored property that is also a variation-theme attribute disappears from the variant prompt too. The schemas contain themes such as `ITEM_DISPLAY_LENGTH`, `ITEM_DISPLAY_WEIGHT`, `ITEM_WEIGHT`, `MODEL`, `MODEL_NAME`, `MODEL_NUMBER`, `PART_NUMBER`, `TEAM_NAME`, `ATHLETE`, `EDITION` and `DENOMINATION/DESIGN_NAME`.

- **Already affected today:** `item_weight`, `model_number` and `team_name` are ignored, so variants whose theme uses them never get a suggestion for their defining value.
- **Recommended fix (one line):** skip the ignore check for keys in `variantAttributeProperties` on variant calls, i.e. `const isIgnored = !(isVariant && variantAttributeProperties.includes(key)) && IGNORED_…some(…)`. The parent call already removes the variation-theme properties separately, so the parent is unaffected.
- Several proposals in §6 (`item_display_*`, `model_name`, `part_number`, `athlete`, `denomination`) assume this fix. Without it they would break variant suggestions for those themes.

## 4. Currently ignored and should stay ignored

| Property (pattern) | Schemas | Avg chars | Reason |
|---|---|---|---|
| `parentage_level` | 32 | 271 | R4. Parent/child is set by Categra's variant model. |
| `child_parent_sku_relationship` | 32 | 428 | R4. Parent SKU link comes from the variant structure. |
| `variation_theme` | 32 | 3,804 | R4. Chosen by the wrapper; also large. |
| `brand` | 33 | 225 | R1. Strict seller value (the task's example). |
| `externally_assigned_product_identifier` | 33 | 539 | R1. GTIN/UPC/EAN barcode. |
| `merchant_suggested_asin` | 33 | 267 | R1. ASIN identifier. |
| `product_site_launch_date` | 32 | 391 | R1/R3. Seller's launch date. |
| `main_product_image_locator` | 33 | 311 | R4. Images come from media. |
| `other_product_image_locator_1..8` | 33 | ~293 each | R4. Images come from media. |
| `main_offer_image_locator` | 33 | 283 | R4. Images come from media. |
| `other_offer_image_locator_1..5` | 33 | ~262 each | R4. Images come from media. |
| `swatch_product_image_locator` | 33 | 314 | R4. Images come from media. |
| `purchasable_offer` | 33 | 7,032 | R3. Pricing; separate dedicated prompt. |
| `list_price` | 32 | 430 | R3. Pricing; separate dedicated prompt. |
| `skip_offer` | 33 | 351 | R3. Offer switch. |
| `fulfillment_availability` | 33 | 1,177 | R3. Fulfilment channel and stock. |
| `team_name` | 31 | 33,169 | R5 + R2. The largest property in the dataset (1,778 enum values, ~8k tokens) and a licensed-merchandise claim: a wrong team is a trademark problem. Keep ignored, but see §3 for variants whose theme is `TEAM_NAME`. |
| `compatible_cellular_phone_models` | 0 | – | Not present in these schemas; keep ignored (same huge-enum pattern as `compatible_phone_models`, see §7). |
| `gpsr_manufacturer_reference` | 33 | 457 | R1/R2. EU GPSR manufacturer contact. |
| `model_number` | 32 | 266 | R1. Manufacturer's code. See §3 for the `MODEL_NUMBER` theme. |
| `manufacturer` | 28 | 242 | R1. Manufacturer/publisher name. |
| `manufacturer_contact_information` | 9 | 457 | R1/R2. Legal contact data. |
| `country_of_origin` | 33 | 1,586 | R2. Customs and labelling declaration; 268-value enum. |
| `hazmat` | 28 | 513 | R2. Hazmat classification. |
| `item_dimensions` | 23 | 1,416 | Measurement, see §8. |
| `item_weight` | 33 | 515 | Measurement, see §8. See §3 for the `ITEM_WEIGHT` theme. |
| `item_package_dimensions` | 32 | 1,445 | R3. Shipping and fee data. |
| `item_package_weight` | 33 | 456 | R3. Shipping and fee data. |
| `product_tax_code` | 33 | 1,985 | R2/R3. Tax classification. |
| `battery` | 27 | 2,589 | R2. Battery transport data (cell composition, IEC code). |
| `battery_contains_free_unabsorbed_liquid` | 23 | 503 | R2. Transport certification. |
| `battery_installation_device_type` | 23 | 385 | R2. Dangerous-goods classification input. |
| `contains_battery_or_cell` | 23 | 444 | R2. Dangerous-goods classification input. |
| `has_multiple_battery_powered_components` | 23 | 378 | R2. Dangerous-goods classification input. |
| `has_replaceable_battery` | 23 | 293 | R2. Dangerous-goods classification input. |
| `is_battery_non_spillable` | 23 | 483 | R2. Transport certification. |
| `lithium_battery` | 27 | 1,757 | R2. Lithium energy, packaging and weight. |
| `non_lithium_battery_energy_content` | 23 | 674 | R2. Transport data. |
| `non_lithium_battery_packaging` | 23 | 564 | R2. Transport data. |
| `num_batteries` | 27 | 547 | R2. Battery quantity and type used for DG review. |
| `number_of_lithium_metal_cells` | 27 | 440 | R2. Transport data. |
| `number_of_lithium_ion_cells` | 27 | 411 | R2. Transport data. |
| `compliance_media` | 33 | 2,171 | R4/R2. Uploaded compliance documents (URLs). |
| `compliance_age_range` | 9 | 392 | R2. Toy safety age grading. |
| `compliance_recommended_age` | 1 | 421 | R2. Toy safety age grading. |
| `compliance_printing_and_publication_country` | 1 | 387 | R2. Country declaration, like `country_of_origin`. |
| `baa_taa_compliance_acknowledgement` | 6 | 484 | R2. Legal acknowledgement. |
| `baa_taa_regulation_compliance` | 6 | 541 | R2. US government procurement compliance. |
| `fcc_radio_frequency_emission_compliance` | 11 | 1,701 | R2. FCC registration data. |
| `regulatory_compliance_certification` | 28 | 854 | R2. Certification numbers. |

## 5. Currently ignored but should be suggested

These are caught only by the broad `compliance` and `batteries` regexes. They describe the product itself, in the same way `material` or `closure` do, which the AI already fills. The "compliance" label means the marketplace (mostly Amazon.in) uses them for classification; the value is still a product description the seller would otherwise type by hand. All have small enums (2 to 16 values), so together they cost about 1.7k chars per schema on average, and each appears only in the product types it applies to.

| Property | Schemas | Avg chars | Why it is ignored today | Why it should be suggested |
|---|---|---|---|---|
| `compliance_animal_leather` | 5 | 484 | Matches `compliance` | Type of leather is visible in the details (genuine, faux, none). |
| `compliance_book_type` | 1 | 595 | Matches `compliance` | Intended use of the book, derivable from its content. |
| `compliance_chest_size` | 3 | 411 | Matches `compliance` | Two-value size class from the apparel details. |
| `compliance_collar_type` | 3 | 369 | Matches `compliance` | Same fact as `collar_style`, which is already suggested. |
| `compliance_construction_type` | 5 | 352 | Matches `compliance` | Construction method (e.g. woven/knitted) from the details. |
| `compliance_cork_type` | 2 | 333 | Matches `compliance` | Material composition. |
| `compliance_cover_type` | 1 | 373 | Matches `compliance` | Hard or soft cover. |
| `compliance_covering_level` | 2 | 407 | Matches `compliance` | Coverage of the garment. |
| `compliance_handbag_type` | 5 | 351 | Matches `compliance` | Bag style, same as the item type. |
| `compliance_is_handmade` | 10 | 352 | Matches `compliance` | Yes/no that the description usually states. |
| `compliance_joining_method` | 1 | 359 | Matches `compliance` | Book binding, same as `binding`. |
| `compliance_operation_mode` | 1 | 406 | Matches `compliance` | Energy source (manual, battery…), same as `power_source_type`. |
| `compliance_other_material_additions` | 2 | 526 | Matches `compliance` | Material list. |
| `compliance_outer_surface_material` | 7 | 542 | Matches `compliance` | Outer material, same as `material`. |
| `compliance_page_count` | 1 | 366 | Matches `compliance` | Page-count class. |
| `compliance_plastic_sheeting_type` | 2 | 428 | Matches `compliance` | Material type. |
| `compliance_primary_function` | 2 | 359 | Matches `compliance` | Primary function of the item. |
| `compliance_printing_method` | 3 | 407 | Matches `compliance` | Print style on apparel. |
| `compliance_rubber_type` | 2 | 365 | Matches `compliance` | Same as `rubber_type`, already suggested. |
| `compliance_shirt_type` | 3 | 425 | Matches `compliance` | Occasion/use of the shirt. |
| `compliance_t_shirt_design` | 3 | 448 | Matches `compliance` | Design (plain, graphic…). |
| `compliance_toy_material` | 1 | 459 | Matches `compliance` | Primary material. |
| `compliance_toy_type` | 1 | 483 | Matches `compliance` | Toy category. |
| `compliance_warp_or_filling_coloring` | 3 | 434 | Matches `compliance` | Number of colours in the fabric. |
| `compliance_weave_type` | 5 | 379 | Matches `compliance` | Same as `weave_type`, already suggested. |
| `compliance_wood_type` | 2 | 546 | Matches `compliance` | Same as `wood_type`, already suggested. |
| `batteries_required` | 27 | 381 | Matches `batteries` | Whether the product needs batteries is obvious from the product (a tote bag doesn't; a lit mirror does). It does not declare battery chemistry or transport data. |
| `batteries_included` | 27 | 499 | Matches `batteries` | Same reasoning; the seller confirms. |

Age grading, country and document fields in the `compliance_*` family stay ignored (§4).

## 6. Not ignored today but should be

| Property | Schemas | Avg chars | Rule | Reason |
|---|---|---|---|---|
| `california_proposition_65` | 13 | 762 | R2 | Legal warning declaration. |
| `contains_pfas` | 7 | 444 | R2 | Chemical declaration under EU/US state law. |
| `ghs` | 25 | 668 | R2 | Chemical hazard classification. |
| `ghs_chemical_h_code` | 33 | 1,174 | R2 | Hazard codes; 100-value enum in every schema. |
| `gpsr_safety_attestation` | 33 | 460 | R2 | Seller's EU safety attestation ("yes" = no warnings needed). |
| `supplier_declared_dg_hz_regulation` | 28 | 550 | R2 | Dangerous-goods regulation; required in 28 schemas, so a guessed "not applicable" is a false declaration. |
| `safety_data_sheet_url` | 28 | 352 | R2/R4 | URL of the SDS. |
| `pesticide_marking` | 10 | 808 | R2 | Pesticide registration data. |
| `has_less_than_30_percent_state_of_charge` | 23 | 401 | R2 | Lithium shipping condition. Missed today because the name has no "battery". |
| `cpsia_cautionary_statement` | 2 | 675 | R2 | Legally required toy warning. |
| `is_this_product_subject_to_buyer_age_restrictions` | 33 | 381 | R2 | Legal restriction flag. |
| `manufacturer_minimum_age` | 1 | 311 | R1/R2 | Manufacturer's safety age recommendation (same as `compliance_age_range`). |
| `manufacturer_maximum_age` | 1 | 311 | R1/R2 | Same. |
| `is_oem_sourced_product` | 23 | 372 | R1 | Sourcing fact the content doesn't show. |
| `supplier_declared_has_product_identifier_exemption` | 33 | 356 | R1/R2 | GTIN exemption is granted by Amazon to the brand. |
| `import_designation` | 8 | 532 | R2 | Import/origin classification, same as `country_of_origin`. |
| `taa_compliant_country` | 6 | 1,909 | R2 | Procurement compliance; 127-value enum. |
| `government_contract_information` | 6 | 497 | R1 | Seller's contract name and number. |
| `is_green_purchasing_law_compliant` | 1 | 452 | R2 | Japanese legal compliance flag. |
| `epr_product_packaging` | 8 | 2,957 | R2/R3 | Packaging materials and recycled share for EPR fees. |
| `epr_eco_fee_eubr` | 3 | 428 | R2/R3 | EPR battery fee (amount + currency). |
| `dsa_responsible_party_address` | 33 | 471 | R1/R2 | EU responsible person's contact. |
| `importer_contact_information` | 1 | 363 | R1/R2 | Legal contact data. |
| `packer_contact_information` | 1 | 357 | R1/R2 | Legal contact data. |
| `rtip_manufacturer_contact_information` | 1 | 369 | R1/R2 | Legal contact data. |
| `image_locator_ps01` .. `image_locator_ps06` | 4 | 341 each | R4 | Product-safety image locations. |
| `external_product_information` | 1 | 562 | R1/R4 | External system key/value store. |
| `package_contains_sku` | 9 | 506 | R3 | Child SKU/quantity of the next package level. |
| `package_level` | 9 | 257 | R3 | Unit/case/pallet level. |
| `master_pack_layers_per_pallet_quantity` | 13 | 400 | R3 | Pallet logistics. |
| `master_packs_per_layer_quantity` | 13 | 370 | R3 | Pallet logistics. |
| `number_of_boxes` | 2 | 277 | R3 | Shipping boxes. |
| `part_number` | 33 | 270 | R1 | Manufacturer part number, same as `model_number`. See §3 (`PART_NUMBER` themes). |
| `national_stock_number` | 32 | 239 | R1 | NSN identifier. |
| `unspsc_code` | 32 | 282 | R1 | 8-digit taxonomy code, free text with no enum, so the AI can only invent one. |
| `model_name` | 28 | 360 | R1 | "As defined by the manufacturer"; strict seller value like `brand`/`model_number`. See §3 (`MODEL`, `MODEL_NAME` themes). |
| `sub_brand` | 1 | 405 | R1 | Brand-like. |
| `designer` | 1 | 211 | R1 | Brand-like. |
| `manufacture_year` | 4 | 242 | R1 | Seller fact. |
| `model_year` | 1 | 258 | R1 | Seller fact. |
| `athlete` | 26 | 228 | R1/R2 | Licensed name, same reasoning as `team_name`. See §3 (`ATHLETE` themes). |
| `league_name` | 31 | 1,081 | R2/R5 | Licensed league; 48-value enum sent in 31 schemas for a field that is empty for nearly all products. |
| `garment_size_country` | 6 | 1,617 | R1/R5 | "Country where the garment's brand originates"; 266-value enum. |
| `version_for_country` | 4 | 1,637 | R1/R5 | Regional version of the product; 266-value enum. |
| `condition_note` | 33 | 304 | R3 | Offer field (your decision). |
| `merchant_shipping_group` | 32 | 312 | R3 | Offer field (your decision). |
| `max_order_quantity` | 33 | 327 | R3 | Offer field (your decision). |
| `merchant_release_date` | 33 | 359 | R3 | Offer field (your decision). |
| `map_policy` | 13 | 244 | R3 | Offer field (your decision). |
| `gift_options` | 33 | 456 | R3 | Offer field (your decision). |
| `ships_globally` | 33 | 296 | R3 | Offer field (your decision). |
| `uvp_list_price` | 3 | 1,424 | R3 | Price (DE list price); belongs to the pricing prompt. |
| `supplemental_condition_information` | 33 | 1,784 | R3 | Condition details of a used offer. |
| `street_date` | 3 | 409 | R3 | First ship date. |
| `denomination` | 5 | 215 | R3 | Gift-card amount. See §3 (`DENOMINATION` theme). |
| `warranty_description` | 11 | 252 | R1 | Seller's warranty policy. |
| `warranty_type` | 6 | 345 | R1 | Seller's warranty policy. |
| `publication_date` | 1 | 279 | R1 | Book fact; a date can't be inferred. |
| `reprint_date` | 1 | 409 | R1 | Book fact. |
| `copyright` | 1 | 401 | R1 | Copyright year. |
| `item_display_dimensions` | 31 | 2,559 | Measurement, see §8 | |
| `item_display_weight` | 17 | 463 | Measurement, see §8 | |
| `item_length_width_height` | 5 | 2,011 | Measurement, see §8 | |
| `item_depth_width_height` | 1 | 1,755 | Measurement, see §8 | |
| `item_dimensions_fraction` | 2 | 2,065 | Measurement, see §8 | |
| `item_length_width` | 1 | 1,122 | Measurement, see §8 | |
| `item_width_height` | 5 | 1,135 | Measurement, see §8 | |

`condition_type` stays suggested, as you decided.

## 7. Large properties that are kept on purpose

| Property | Schemas | Avg chars | Why it stays |
|---|---|---|---|
| `compatible_phone_models` | 2 (phone case) | 26,811 | The largest kept property (1,360-value enum, ~6.7k tokens). For a phone case it is the core attribute, and the title almost always names the model, so the suggestion is genuinely useful. If cost matters more, it is the first candidate to ignore under R5. |
| `apparel_size`, `shirt_size`, `headwear_size`, `maximum_size` | 1–4 | 11k–17k | Size systems; these are variation-theme properties for apparel and must reach variant prompts. Parents already drop them via the variation-theme removal. |
| `stones`, `pearl`, `stone`, `metals` | 4 | 2.8k–7.2k | Core jewellery descriptions, derivable from the details. |
| `language` | 8 | 6,873 | Needed for books (544-value enum). For other types it is usually left empty; a per-type rule would be needed to drop it, which the global list can't express. |
| `chain_length` | 3 | 4,170 | Jewellery spec, stated in the title ("18 inch chain"), and a variation theme. |
| `recommended_browse_nodes` | 19 | 1,051 | Kept by earlier decision (enum + `enumNames`). |

## 8. Measurements (decision and reasoning)

- **Rule:** ignore the generic dimension blocks and weights; keep single, named product measurements.
- **Ignore** `item_dimensions`, `item_weight` (already ignored), and add `item_display_dimensions`, `item_display_weight`, `item_length_width_height`, `item_depth_width_height`, `item_dimensions_fraction`, `item_length_width` and `item_width_height`. Package dimensions and weight stay ignored (R3).
  - They are seller-measured facts. An image gives no absolute scale, and weight can't be seen at all, so without a stated value the AI can only guess.
  - Up to four of these blocks describe the same L×W×H in different formats in one schema, so the AI invents the same guess several times.
  - They are large: 1.1k–2.6k chars each, and `item_display_dimensions` alone is in 31 of 33 schemas.
  - When the product text does state the size, the Amazon import usually has the value already (catalog-sourced), which improvement #4 now skips anyway.
- **Keep** single measurements that are named product specs and usually appear in the title or bullets: `capacity`, `volume_capacity_name`, `chain_length`, `strap_length`, `earring_length`, `brim_width`, `item_length`, `item_width`, `item_diameter`, `outside_diameter`, `item_thickness`, `dial_size`, `shoulder_to_bottom_hem_length`, `water_resistance_depth`, `total_diamond_weight`, `total_gem_weight`, `size_per_pearl`, `line_size`, `paper_size`. They are small (≈250–650 chars).
- Several dimension/weight properties are variation themes (`ITEM_DISPLAY_LENGTH`, `ITEM_DISPLAY_WEIGHT`, `ITEM_WEIGHT`…). The §3 fix keeps them in variant prompts, where the value defines the variant.

## 9. Proposed constant

```ts
export const IGNORED_AMAZON_SCHEMA_SUGGESTION_PROPERTIES = [
	// Variation structure, identifiers, brand (R1/R4)
	'^parentage_level$',
	'^child_parent_sku_relationship$',
	'^variation_theme$',
	'^brand$',
	'^sub_brand$',
	'^designer$',
	'^manufacturer$',
	'^model_number$',
	'^model_name$',
	'^part_number$',
	'^externally_assigned_product_identifier$',
	'^merchant_suggested_asin$',
	'^national_stock_number$',
	'^unspsc_code$',
	'^supplier_declared_has_product_identifier_exemption$',
	'^external_product_information$',
	'^manufacture_year$',
	'^model_year$',
	'^publication_date$',
	'^reprint_date$',
	'^copyright$',
	'^warranty_description$',
	'^warranty_type$',
	'^government_contract_information$',
	'^is_oem_sourced_product$',
	'^garment_size_country$',
	'^version_for_country$',
	// Licensed names (R2/R5)
	'^team_name$',
	'^athlete$',
	'^league_name$',
	// Media and links (R4)
	'^main_product_image_locator$',
	'^other_product_image_locator',
	'^main_offer_image_locator',
	'^other_offer_image_locator',
	'^swatch_product_image_locator$',
	'^image_locator_ps',
	'^safety_data_sheet_url$',
	// Offer, pricing, fulfilment, logistics (R3)
	'^purchasable_offer$',
	'^list_price$',
	'^uvp_list_price$',
	'^skip_offer$',
	'^fulfillment_availability$',
	'^product_site_launch_date$',
	'^merchant_release_date$',
	'^street_date$',
	'^merchant_shipping_group$',
	'^max_order_quantity$',
	'^map_policy$',
	'^gift_options$',
	'^ships_globally$',
	'^condition_note$',
	'^supplemental_condition_information$',
	'^denomination$',
	'^product_tax_code$',
	'^package_contains_sku$',
	'^package_level$',
	'^master_pack',
	'^number_of_boxes$',
	// Measurements (see §8)
	'^item_dimensions$',
	'^item_weight$',
	'^item_package_dimensions$',
	'^item_package_weight$',
	'^item_display_dimensions$',
	'^item_display_weight$',
	'^item_length_width_height$',
	'^item_depth_width_height$',
	'^item_dimensions_fraction$',
	'^item_length_width$',
	'^item_width_height$',
	// Legal, safety and regulatory declarations (R2)
	'^compatible_cellular_phone_models$',
	'^gpsr_manufacturer_reference$',
	'^gpsr_safety_attestation$',
	'^manufacturer_contact_information$',
	'^rtip_manufacturer_contact_information$',
	'^importer_contact_information$',
	'^packer_contact_information$',
	'^dsa_responsible_party_address$',
	'^country_of_origin$',
	'^import_designation$',
	'^taa_compliant_country$',
	'^baa_taa_',
	'^hazmat$',
	'^supplier_declared_dg_hz_regulation$',
	'^ghs',
	'^california_proposition_65$',
	'^contains_pfas$',
	'^pesticide_marking$',
	'^cpsia_cautionary_statement$',
	'^is_this_product_subject_to_buyer_age_restrictions$',
	'^manufacturer_(minimum|maximum)_age$',
	'^is_green_purchasing_law_compliant$',
	'^epr_',
	'^fcc_radio_frequency_emission_compliance$',
	'^regulatory_compliance_certification$',
	'^compliance_media$',
	'^compliance_age_range$',
	'^compliance_recommended_age$',
	'^compliance_printing_and_publication_country$',
	// Battery transport data (R2); batteries_required / batteries_included stay suggested
	'battery',
	'^num_batteries$',
	'^number_of_lithium_(metal|ion)_cells$',
	'^has_less_than_30_percent_state_of_charge$',
];
```

Pattern changes worth noting:

- The broad `compliance` regex is replaced by explicit entries, so the descriptive `compliance_*` fields in §5 become suggestable.
- `batteries` is replaced by `^num_batteries$`. `battery` does not match "batteries", so `batteries_required` and `batteries_included` become suggestable.
- The patterns are case-insensitive (`new RegExp(pattern, 'i')`) and unanchored ones match anywhere in the key.
- `^compatible_cellular_phone_models$` is kept although it isn't in these schemas.

## 10. Size impact

Measured over the 33 full schemas, per parent call with the whole reduced schema:

| | Avg chars per schema |
|---|---|
| Sent today (all non-ignored top-level properties) | 51,464 |
| Removed by §6 | −17,972 |
| Added back by §5 | +1,677 |
| **Proposed** | **≈ 35,169 (−31.7%)** |

These are schema-only figures for a full parent schema. Variant calls send only the variation-theme and variant-specific keys, so they barely change. Catalog-sourced skipping (#4) overlaps with some of these keys, so the real saving on top of the current code is lower.

## 11. What is not decided here

- Applying §9 to `constants.ts` and the §3 code fix; both are waiting for your approval.
- Per-product-type rules (e.g. keep `language` only for books). The current constant is global, so this review stays global.

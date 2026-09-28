# Amazon BS Q5: plan to cut LLM input tokens

_Scope: `api/` only. Recommendations, not implementation. Written 2026-09-24._

**Bottom line:** the run sends **15.31M** input tokens in 1,558 calls. Stacking the 8 improvements below brings that to about **3.05M tokens in about 460 calls (−80%)**. Most of the saving comes from four places:
- the channel scope repeats every master call with a byte-identical prompt (−48%);
- one suggestion call per variant (−10% more);
- dropdown option lists sent with every localization call (−8% more);
- asking again for values that were already imported (about −6% more).

---

## 1. Baseline and data source

The figures come from `tbl_llm_call_logs`, run **taskQueueId 3490** (2026-09-23 13:22–13:31). It matches the task figures exactly:

| source | calls | distinct prompts | input tokens | avg / call | model |
|---|---|---|---|---|---|
| AMAZON_BS_SUGGESTION | 910 | 464 | 11,723,348 | 12,883 | gemini-2.5-flash |
| AMAZON_BS_GROUP_LOCALIZATION | 350 | 182 | 2,852,847 | 8,151 | gemini-2.5-flash-lite |
| AMAZON_BS_PRIMARY_LOCALIZATION | 298 | 198 | 734,775 | 2,466 | gemini-2.5-flash-lite |
| **Total** | **1,558** | **844** | **15,310,970** | | |

Dataset: 65 parents and 187 variants took part in the calls (251 entities synced). The base language is `en_US`; the other languages are `en_CA` and `es_MX`. There are 33 product types, led by SWEATSHIRT, CELLULAR_PHONE_CASE, TOTE_BAG and DRINKING_CUP.

> **Caveat: all 1,558 calls failed.** Every row has `status = PROVIDER_ERROR` with `429 SPEND_CAP_EXCEEDED`. So `input_tokens_estimated = true` everywhere: tokens are `ceil(chars / 4) + 258 per image` (`estimateLlmInputTokens`). The prompts are the real ones, so the ratios below hold. Absolute Gemini counts for JSON-heavy prompts are usually 10–30% higher than chars/4. See §6 for side effects of this failure mode.

### Where the tokens sit

| call type | calls | avg tokens | what fills it (avg chars) |
|---|---|---|---|
| Suggestion, **parent** | 196 | 32,987 | schema **119,716** (enum lists 64%, descriptions 21%) · system 8,317 · details 1,925 · image ~0.9 |
| Suggestion, **variant** | 714 | 7,364 | schema **16,700** (`purchasable_offer` 35%, `apparel_size` enum 33%) · system 8,317 · details 2,339 · image ~1 |
| Group localization | 350 | 8,151 | attributeMetadata **22,504** (97% `allowedOptions` lists; max 292k) · sourceData 4,350 · system 4,454 |
| Primary localization | 298 | 2,466 | system 4,454 · sourceData 3,954 · metadata 159 |

---

## 2. Round-trip analysis (why so many calls)

Per parent `P` in `amazon-product-bsq5.service.ts:132-301`:

| step | call site | scopes (targets) | calls per parent |
|---|---|---|---|
| Primary localization | `bsq5:215` → `ProductService.localizeProductAttributeValues` (`product.service.ts:17515`) | `null × accountEnabledLanguages` **+** `channel × productSyncLanguages` | (L_acc + L_p) × ⌈keys/100⌉ |
| Group localization | `bsq5:241` → same | `null × L_p` **+** `channel × L_p` | 2·L_p × ⌈keys/100⌉ |
| Suggestion, parent | `bsq5:285` → `suggestAmazonGroupAttributeValues` (`product.service.ts:18107`) | `null × L_p` **+** `channel × L_p` | 2·L_p |
| Suggestion, variants | `bsq5:292-300` (one call per variant) | same | 2·L_p·V |

Findings from the prompt bodies:

1. **The channel scope repeats the master call.** Every channel-scope suggestion prompt is byte-identical to its master prompt: 446 of 455 are duplicates, worth 5.81M tokens. So are 168 of 175 group-localization prompts (1.32M tokens) and all 100 channel primary prompts (0.27M).
   - Why: the source query keeps only `isInherited = false` rows or base-language master rows. BS channel rows are inherited, so no channel candidate survives, `selectSourceCandidatesForTarget` picks the same values, and `resolveTargetSchema` resolves the same schema. The model is paid twice for one answer.
   - The writer already supports fan-out: `applyAISuggestedAmazonAttributeValues` takes `scopes: [...]`.
2. **Variants each re-send the system prompt and schema.** A variant call is 72% fixed overhead (system prompt plus a shared variant schema), and its per-variant payload is only 2.3k chars. The 357 master variant calls cover 79 (parent, language) groups, averaging 4.5 variants each and up to 56.
3. **Localization metadata isn't scoped to the batch.** `buildAttributeLocalizationMetadata` (`product.service.ts:585`) is built once from **all** attributes of the group and sent with every batch, including full `allowedOptions` lists:
   - About 26% of the metadata chars belong to attributes that aren't in that call.
   - Many dropdown source values are **already a valid option in the target language**. The prompt tells the model to return that exact option, so the call adds nothing.
4. **Localization re-translates values that already exist.**
   - Of the keys sent for master `en_US`, 91.5% (group) and 99.8% (primary) already have their own non-inherited `en_US` value. For `en_CA` the figures are 19% (group) and 56% (primary).
   - `selectSourceCandidatesForTarget` drops target-language candidates whenever another language exists, so the call "translates" `en_CA → en_US`.
   - Master rows are insert-only in `upsertProductAttributes` (`updatableRecords` excludes master unless `isForceUpdateMaster`). The result is thrown away for product attributes and survives only as an `attribute_suggested_values` row.
5. **Oversized fixed text.** The suggestion system prompt is 8.3k chars (about 2.1k tokens). It restates "traverse / preserve hierarchy / don't flatten" about six times and has four examples. That's 16% of every suggestion call. The localization system prompt plus user boilerplate is about 5.7k chars.
6. **Offer data the model cannot know.** `purchasable_offer`, `list_price` and `condition_type` average about 6.4–6.9k chars in *every* parent and variant call. They're in `VARIANT_SPECIFIC_AMAZON_PROPERTIES`, and the system prompt even shows a made-up `value_with_tax: 19.99`. Prices come from the Amazon listing import, not from inference.
7. **Suggestions for values that are already imported.** Of the parent schema keys, 17.5% already hold an imported `AM_*` value. Many are technical keys the product already has. They're also already stored as catalog-sourced suggestions by `storeCatalogAttributeSuggestedValues`.

---

## 3. Improvements, largest impact first

The **marginal** saving is measured on top of every improvement above it, so the figures add up. The **standalone** saving is what the change alone would save against the 15.31M baseline. Ranges show how sensitive the estimate is to the tuning noted. Marginals marked **≈** are derived rather than re-simulated (see §8).

Flags:
- 🟢 **output-equivalent**: the same persisted data is expected.
- 🟠 **behaviour change**: it deliberately stops producing something.

### #1 One LLM call per distinct prompt across scopes 🟢
- **Change:**
  - `suggestAmazonGroupAttributeValues`: group `targetScopes` by `(resolved amazonChannelProductType.id, languageCode, hash(productAttributesPayload), hash(variantAttributeProperties))`. Call the LLM once per group and pass the whole group as `scopes` to `applyAISuggestedAmazonAttributeValues`.
  - `localizeProductAttributeValues` (loop at `product.service.ts:17806`): group scopes by `(languageCode, hash(sourceData))` per batch. Call once and fan the response out per scope, writing `productAttributeRecords` for the master scope only, as today.
  - Optional safety net: a per-`taskQueueId` in-memory memo in `LlmOrchestratorService.execute`, keyed by `hash(system + user + images)`.
- **Estimate:** −7.39M (−48.3%). Measured: these are exactly the duplicate-prompt tokens. Suggestion −5.81M, group −1.32M, primary −0.27M.
- **Result:** 15.31M → **7.92M**. Calls 1,558 → **844**. Range 7.2–7.4M.
- **Risk:** low. Today the two scopes get two independent samples of the same prompt; afterwards they share one answer.

### #2 Batch variants into one suggestion call per parent and language (≤5 per call) 🟢
- **Change:**
  - Replace the per-variant loop (`bsq5:292-300`) with `suggestAmazonGroupAttributeValues({ variantIds: [...] })`.
  - Group the variants by wrapper/variation theme (same `variantAttributeProperties`, so same reduced schema). Send the system prompt and schema once, plus `variants: { <variantId>: <details> }`, and have the response keyed by variantId.
  - Attach images in order and label them per variant in the text.
  - Cap at 5 variants per call, because `LLM_SUGGESTION_MAX_OUTPUT_TOKENS = 8192`.
  - This needs a new prompt builder next to `buildSuggestionPrompt`.
- **Estimate:**
  - 357 → 123 master variant calls.
  - Marginal −1.59M. Result 7.92M → **6.33M (cum −58.7%)**. Calls 844 → **610**.
  - Standalone −3.15M (20.6%). Range −1.34M (cap 3) to −1.77M (cap 10).
- **Risk:** medium. There's a larger output per call and a quality drift risk. Validate with `parsed_keys / requested_keys` per variant.

### #3 Localization: copy exact dropdown matches, scope metadata to the call 🟢
- **Change** in `localizeProductAttributeValues`:
  - (a) Before batching: if an item has `allowedOptions` and **every** source value is (case-insensitively) one of the target-language options, write it directly with the existing record builders, with no LLM call.
  - (b) Build `attributeMetadata` per batch: only attributeIds present in that batch's `sourceData`, and only entries that have `allowedOptions`. Text is already the documented default.
- **Estimate:**
  - Group localization 1.53M → 0.33M; 27 master calls vanish entirely.
  - Marginal −1.21M. Result 6.33M → **5.12M (cum −66.6%)**. Calls 610 → **583**.
  - Standalone −2.25M (14.7%). Range −1.0M to −1.25M (exact-case matching → lower end).
- **Note:** (a) and (b) only pay off together. Alone, (a) saves −0.45M and (b) saves −0.26M on the master scope; combined they save −1.13M.

### #4 Skip schema keys the product already has, except content keys 🟠
- **Change:** in `suggestAmazonGroupAttributeValues`, collect the top-level schema keys whose `AM_<PT>_<key>_*` attributes already have a non-empty value for the scope, product/variant and language. Pass them as `excludeKeys` to `reduceSchemaForAISuggestions`. Always keep the content keys `item_name`, `bullet_point`, `product_description` and `generic_keyword`, where AI rewrites are the point.
- **Estimate:** marginal **≈ −0.85M**. Result 5.12M → **≈ 4.27M (cum −72.1%)**. Standalone −3.14M (20.5%). Range −0.7M to −1.0M.
- **Behaviour:** no AI alternative is offered for technical attributes that were imported from Amazon. The catalog-sourced suggestions stay.

### #5 Compress the system prompts and user boilerplate 🟢
- **Change:**
  - `SUGGESTION_SYSTEM_PROMPT`: 8,317 → about 2,400 chars, with one example and each rule stated once.
  - `ATTRIBUTE_LOCALIZATION_SYSTEM_PROMPT`: 4,454 → about 1,600 chars.
  - Trim the user-prompt banners and the "IMPORTANT / Processing steps" blocks to about 350 chars.
  - Use Gemini `responseMimeType: application/json` (already `format: 'json'`) instead of the prose "no markdown / no code fences" rules.
- **Estimate:** marginal −0.71M. Result ≈ 4.27M → **≈ 3.56M (cum −76.7%)**. Standalone −2.14M (14.0%). Range −0.55M to −0.80M (target prompt of 1.8k–3.5k chars).
- **Risk:** medium. This needs an eval pass on a fixed product sample before rollout.

### #6 Localization: skip keys whose target scope already has its own value 🟠
- **Change:** in `localizeProductAttributeValues`, drop an item for a target scope when `item.candidates` already contains a candidate with the same `channelId` and `languageCode`. The data is already loaded, so this needs no extra query.
- **Estimate:**
  - Master `en_US` localization disappears almost entirely. Localization calls 346 → 223.
  - Marginal −0.22M. Result ≈ 3.56M → **≈ 3.35M (cum −78.1%)**. Calls 583 → **460**.
  - Standalone −1.05M (6.9%). Range −0.15M to −0.30M (channel-scope existence was approximated from master).
- **Behaviour:** no translated `attribute_suggested_values` row where a target-language value already exists. The product-attribute row isn't touched today either.

### #7 Remove offer, price and condition from AI suggestion 🟠
- **Change:** add `^purchasable_offer$`, `^list_price$` and `^condition_type$` to `IGNORED_AMAZON_SCHEMA_SUGGESTION_PROPERTIES`. It's used only by `reduceSchemaForAISuggestions`. Drop the price example from the system prompt.
- **Estimate:** marginal **≈ −0.17M**. Result ≈ 3.35M → **≈ 3.18M (cum −79.3%)**. Standalone −1.58M (10.3%). Range −0.14M to −0.20M.
- **Behaviour:** no AI-invented prices or conditions. Real values still come from the listing import.

### #8 Trim schema descriptions 🟢
- **Change:** in `reduceProperty`, keep each `description` to its first sentence, at most 120 chars.
- **Estimate:** marginal **≈ −0.13M**. Result ≈ 3.18M → **≈ 3.05M (cum −80.1%)**. Standalone −0.51M (3.3%). Range −0.08M to −0.18M.

---

## 4. Summary

Baseline: **15,310,970 input tokens in 1,558 calls** (run 3490, estimated at chars/4).

| # | Improvement | Ops affected | Flag | Standalone saving (vs baseline) | Marginal saving (in order) | Range (marginal) | Input after | Cum. reduction | Calls after | Effort / risk | Verify in `tbl_llm_call_logs` |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | One LLM call per distinct prompt across scopes | SUG, GROUP, PRIM | 🟢 | 7.39M (48.3%) | −7.39M | 7.2–7.4M | 7.92M | −48.3% | 844 | S / low | `count(*) = count(DISTINCT user_prompt_hash)` per task |
| 2 | Batch variants per (parent, language), ≤5 per call | SUG | 🟢 | 3.15M (20.6%) | −1.59M | 1.34–1.77M | 6.33M | −58.7% | 610 | M–L / medium | variant-source calls ≈ ⌈V/5⌉ per parent-language; `parsed_keys/requested_keys` |
| 3 | Copy exact dropdown matches, scope metadata to the call | GROUP (PRIM) | 🟢 | 2.25M (14.7%) | −1.21M | 1.0–1.25M | 5.12M | −66.6% | 583 | S / low | avg `attributeMetadata` section ≤ 3k chars |
| 4 | Skip schema keys already filled, except content keys | SUG | 🟠 | 3.14M (20.5%) | ≈ −0.85M | 0.7–1.0M | ≈ 4.27M | ≈ −72.1% | 583 | M / low | parent `requested_keys` drops ~15–20% |
| 5 | Compress system prompts and user boilerplate | SUG, GROUP, PRIM | 🟢 | 2.14M (14.0%) | −0.71M | 0.55–0.80M | ≈ 3.56M | ≈ −76.7% | 583 | S / medium (eval) | `system_prompt_chars` SUG ≈ 2.4k, LOC ≈ 1.6k; parse-fail rate unchanged |
| 6 | Skip localization keys that already have a target value | GROUP, PRIM | 🟠 | 1.05M (6.9%) | −0.22M | 0.15–0.30M | ≈ 3.35M | ≈ −78.1% | 460 | S / low | ~no `en_US` master localization calls |
| 7 | Remove offer, price and condition from AI suggestion | SUG | 🟠 | 1.58M (10.3%) | ≈ −0.17M | 0.14–0.20M | ≈ 3.18M | ≈ −79.3% | 460 | XS / low | no `purchasable_offer` in prompts |
| 8 | Trim schema descriptions (first sentence, ≤120 chars) | SUG | 🟢 | 0.51M (3.3%) | ≈ −0.13M | 0.08–0.18M | **≈ 3.05M** | **≈ −80.1%** | **~460** | XS / low | schema chars −5–10% |

End state by operation:
- **Suggestion:** 11.72M → about 2.81M (−76%), 910 → about 230 calls.
- **Group localization:** 2.85M → about 0.10M (−96.5%).
- **Primary localization:** 0.73M → about 0.14M (−81%).
- **Localization calls:** 648 → about 230.

**Output-equivalent items only (#1, #2, #3, #5, #8)** give about **4.24M (−72%)**. The 🟠 items #4, #6 and #7 add the last ~8 points.

---

## 5. Other callers affected by shared-path changes

Approved: shared paths may change. The callers are:
- `localizeProductAttributeValues`: `products.controller.ts:1356` (UI), `product.service.ts:16589` and `:23029`, `amazon-product-cbaq5.service.ts:170,195` (catalog-by-ASIN Q5).
- `suggestAmazonGroupAttributeValues`: `products.controller.ts:1372` (UI), `amazon-product-cbaq5.service.ts:238,246` (same per-variant loop, so #2 applies there too), `amazon-channel-product-type-schema.service.ts:500`.
- `reduceSchemaForAISuggestions` and prompt builders: only via the two methods above.

The UI single-scope calls gain from #3–#8. #1 and #2 are no-ops for them.

---

## 6. Side findings (not token reductions, but fix them)

1. **No circuit breaker on quota errors.** Q5 kept sending for about 8 minutes, 1,558 calls, after Gemini returned `429 SPEND_CAP_EXCEEDED` on the first one (`retryable = false`). Abort the remaining AI steps of the task on a non-retryable `QUOTA_EXCEEDED`.
2. **Failed localizations are marked as done.** `localizeProductAttributeValues` upserts `ProductTranslationStatus.isTranslated = true` for every group and scope (`product.service.ts:18037-18068`) even when every batch failed. The next run filters those keys out (`translatedKeys`, `:17800`), so run 3490's products will never be localized unless the status is cleared. Write the status only for scopes whose batches succeeded.
3. **The figures are estimates.** Output tokens are NULL and input is chars/4. Re-measure after a funded run, where `input_tokens_estimated = false`, before and after each rollout.

## 7. Considered, not ranked

- **Merging `en_US` and `en_CA` suggestion calls** (schema sent once, answers keyed by language). Schemas and enums are per marketplace, so they'd need a union or diff. Revisit after #2 lands.
- **Gemini `mediaResolution` low for attached images.** In the estimate, the 860 images are 0.22M at baseline and about 0.1M after #1/#2. It affects real billed tokens more than this estimate does.
- **Provider caching (billed cost only, not sent tokens).** Gemini implicit caching discounts repeated prompt prefixes. Keep the static system prompt first and the per-product data last, as today. After #5 the system prompt may fall below the caching minimum, which doesn't matter for the sent-token metric.

## 8. Method (reproducible)

- **Data:** read-only SELECTs on `tbl_llm_call_logs` (task 3490), `tbl_product_attributes` and `tbl_attributes`.
- **Simulation:** a scratch simulator re-parsed all 828 distinct master prompts (455 suggestion, 373 localization), applied each transformation to the actual JSON, and re-costed with the same estimator (chars/4 + 258 per image).
  - The remaining ~16 non-duplicate channel prompts were scaled with their operation.
  - Variant batching assumed 200 chars of batch framing plus 60 chars per variant.
  - #5 assumed the target prompt sizes stated above.
- **Ordering:** greedy. At each step the change with the largest remaining saving was applied, and each marginal figure includes all earlier ones.
- **Derived figures (≈):** #1, #2, #3, #5 and #6 are exact simulator results; they don't interact with the schema-level items. The marginals for #4, #7 and #8 were derived from their simulated standalone savings and batching ratios, not re-simulated in this order, so they carry roughly ±20%.

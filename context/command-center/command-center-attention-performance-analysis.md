# Command Center `POST /command-center/attention` — calculation & response-time analysis

Date: 2026-10-14
Scope: `api/` (read path), `db-migrations/` (index proposals), `webapp/` (call site only). **Nothing was changed** — analysis only.
Measured against: account **329** (2 channels, 367 published listing keys), read-only transaction, `EXPLAIN (ANALYZE, BUFFERS)`.

---

## 1. TL;DR

| | |
|---|---|
| Symptom | ~30 s response (measured here: **39 s** cold) |
| Root cause | One SQL query — `buildAttentionKeySignalsQuery` — does a **sequential scan of `tbl_product_media_linkers` (490,573 rows, 9,472 pages) once per listing key** (367 times = **3.45 M page reads**). |
| Everything else | The other four parallel signals take **0.4–0.5 s each** (≈ 0.15 s of that is network round trip to the DB host). Node-side calculation is negligible. |
| Fix #1 | Pre-select the account's main-image linkers once (435 rows) and join against that. **39.1 s → 0.44 s, byte-identical output** (verified on account 329). |
| Expected result | Cold request ≈ **1–1.5 s**; cached (90 s TTL) ≈ a few ms. |

---

## 2. Request flow

`webapp command-center/main.tsx` → `useSectionRequest(GET_COMMAND_CENTER_ATTENTION)` → `POST command-center/attention`
→ `CommandCenterController.getAttention` → `CommandCenterAttentionService.getAttention` (`command-center-attention.service.ts`)

```
getAttention
 ├─ ChannelService.getUserWiseChannels        (Redis-cached role/channel scope)
 ├─ Redis cache lookup  (key: account:user:scope:channel:marketplace, TTL 90 s)   ← miss = everything below
 └─ Promise.all (5 parallel)
     ├─ inventory   buildInventoryKeysQuery      (pulse.sql)
     ├─ pricing     buildPricingKeysQuery        (pulse.sql)
     ├─ listing     buildListingIssuesQuery      (pulse.sql, LIMIT 5000)
     ├─ keySignals  buildAttentionKeySignalsQuery (attention.sql)   ← 39 s
     └─ health      ChannelHealthService.getHealth (5 sequential DB round trips)
 └─ buildAttentionItems (in-memory) → orderAttentionItems → resolveNextActions → cache set
```

All four SQL signals start from the same `pulse_keys` CTE (`buildPulseKeysCteSql`): scoped channels → active product channels → active variant channels → UNION ALL of 4 branches (Amazon product×marketplace, Amazon variant×marketplace, non-Amazon product, non-Amazon variant). Measured size for acct 329: 370 keys, 367 published (`SUCCESS`/`OUT_OF_SYNC`), 2 channels. This CTE is **re-evaluated in each of the four queries**.

Total wall time ≈ the slowest branch, because they run in parallel → the 39 s query alone defines the response time.

---

## 3. Calculation behind every attention aspect

Every card is a *family*; each family collapses units (deduped by `buildPublishedUnitKey`) into one card with scope chips + a sub-type breakdown. Order: `orderAttentionItems`; actions: `resolveNextActions(countsByFamily)`.

| Family | Source signal | Calculation | DB cost (acct 329) |
|---|---|---|---|
| `OUT_OF_STOCK` / `LOW_STOCK` / `STOCK_SYNC` | inventory query | Per published key: FBA effective (`is_fba` or FC warehouse mapping) → `fbaFulfillable`; else mapped non-FC warehouses → `warehouseStock`; else `remoteStock`. Threshold only for mapped non-FBA keys. `hasSyncIssue` = stock sync failure row OR (`mapped>0` and remote≠warehouse). `classifyInventoryState` → OOS/LOW/sync. Per key: 3 LATERAL subqueries (warehouse mapping+stock, FBA inventory, shared-pool fan-out) + `EXISTS` on `tbl_product_sync_failures`. | **395 ms** (367 rows) |
| `MISSING_PRICE` | pricing query | Per published key: latest exact-scope price row, latest master row (`channel_id IS NULL`), offer/sale baseline rows. JS `resolvePublishedEffectivePrice` follows baseline → non-inherited scope → master, active sale else offer, legacy `price` fallback; `null` ⇒ missing. | **493 ms** (367 rows) |
| `AMAZON_LISTING_BLOCKER` / `AMAZON_LISTING_WARNING` | listing query | `tbl_channel_product_errors` (197,107 rows) joined to published Amazon keys, `LIMIT 5000`. JS: if any non-warning exists, only blockers are counted (else warnings). Bucket via `classifyAmazonListingIssue` (rule table, substring match). | **489 ms** (1,734 rows) |
| `AMAZON_NOT_SEARCHABLE` / `SHOPIFY_NOT_ACTIVE` | key-signal query | JS `resolveDiscoverabilitySignal` on `marketplace_external_status` (Amazon JSON text) / `channel_external_status` (Shopify). Pure in-memory. | (same query as below) |
| `MAIN_IMAGE` | key-signal query | Published main-image `file_type`: media sync baseline snapshot (`previous_sync_value` JSON → first `isMainImage` → `tbl_digital_assets`) wins; else live `tbl_product_media_linkers` main image (channel-specific first, then channel `NULL`). JS `resolveMainImageState` runs for **Amazon rows only**. | **38,830 ms** ← the problem |
| `PRODUCT_TYPE_CHANGED` | key-signal query | `pc.suggested_product_attribute_group_id IS NOT NULL` (PK join on `tbl_product_channels`). | negligible |
| `CONNECTION`, `ACCOUNT_HEALTH`, `PERMISSION_REQUIRED`, `MARKETPLACE_HEALTH` | `ChannelHealthService.getHealth` | Reason-code checks on already-derived channel/marketplace health (`deriveChannelHealth`, `deriveMarketplaceHealth`). In-memory after load. | ≈ 5 round trips |
| `DATA_TRUST` | `getHealth().dataTrust` | `computeDataTrust`: stale inventory/orders sync per channel/marketplace; one unit per issue, `affectedCount` as count. In-memory. | included above |

In-memory work (`buildAttentionItems`, grouping, sorting) is O(rows) over a few hundred to a few thousand rows — **< 10 ms**; it is not a factor.

---

## 4. Measurements

Network note: the DB host is remote; a trivial query costs **~145–150 ms** round trip from the analysis machine, so each figure below includes that floor. In production (API next to the DB) the floor is much lower, which makes the 39 s problem even more dominant.

| Signal | Time | Rows |
|---|---|---|
| inventory | 395 ms | 367 |
| pricing | 493 ms | 367 |
| listing issues | 489 ms | 1,734 |
| `pulse_keys` count alone | 177 ms | 1 |
| **key signals (as shipped)** | **38,830 ms** (EXPLAIN: 39,205 ms) | 367 |
| health: channels / marketplaces / failures / stock failures / recent orders | ~150 ms each (≈ RTT) | 2 / 3 / 1 / 0 / 0 |
| key signals, **rewritten** (§5.1) | **435 ms** | 367 (identical) |

### EXPLAIN evidence (key-signals query)

```
->  Limit  (actual rows=1 loops=367)
      Buffers: shared hit=26080 read=3453837
      ->  Seq Scan on tbl_product_media_linkers pml   (actual rows=2 loops=367)
            Filter: is_main_image IS TRUE AND NOT (variant_id IS DISTINCT FROM NULL)
                    AND (channel_id = apc.channel_id OR channel_id IS NULL)
                    AND account_id = 329 AND product_id = ...
            Rows Removed by Filter: 490450          ← per loop
Execution Time: 39204.992 ms
```

Why no index is used:
- Existing indexes on the table: `pkey`, `idx_fk_..._channel_marketplace_id`, `idx_tpml_variant_channel_marketplace_lang (variant_id, channel_id, channel_marketplace_id, language_code)`. **None starts with `product_id` or `account_id`.**
- Product-level keys have `variant_id = NULL`; `variant_id IS NOT DISTINCT FROM pk.variant_id` cannot use the `variant_id`-leading btree for the NULL case, and `(channel_id = X OR channel_id IS NULL)` blocks further narrowing.
- The account has only **435 main-image linker rows out of 490,573** — a tiny set hidden behind a full scan, repeated per key.

Cost scales as `published keys × linker-table pages`, i.e. it gets **worse quadratically** as accounts and the shared table grow; large accounts can exceed any gateway timeout.

---

## 5. Improvements, sorted by impact

### 5.1 (Impact: ~99 % of the latency) Stop scanning `tbl_product_media_linkers` per key

**Change:** materialise the account's main-image linkers once and `LATERAL` over that small set. Semantics are unchanged (channel-specific first, then channel `NULL`, then `sequence`).

```sql
WITH <pulse_keys CTEs>,
main_linkers AS MATERIALIZED (
	SELECT pml.product_id, pml.variant_id, pml.channel_id, pml.sequence, da.file_type
	FROM tbl_product_media_linkers pml
	JOIN tbl_digital_assets da ON da.id = pml.digital_assets_id
	WHERE pml.account_id = :accountId AND pml.is_main_image IS TRUE
),
published AS ( ... )
SELECT ...
LEFT JOIN LATERAL (
	SELECT ml.file_type
	FROM main_linkers ml
	WHERE ml.product_id = pk.product_id
		AND ml.variant_id IS NOT DISTINCT FROM pk.variant_id
		AND (ml.channel_id = pk.channel_id OR ml.channel_id IS NULL)
	ORDER BY CASE WHEN ml.channel_id = pk.channel_id THEN 0 ELSE 1 END, ml.sequence
	LIMIT 1
) mi ON TRUE
```

**Measured:** 39,092 ms → 435 ms, row-for-row identical output on account 329.
**Where:** `command-center-attention.sql.ts` only (lines 42–53).
**Risk:** very low. The CTE still costs one pass over the linker table (~350 ms here) until 5.2 is added.

### 5.2 (Impact: removes the remaining linker scan; keeps cost flat as data grows) Add a partial index for main-image linkers

New migration in `db-migrations/` (via `/new-migration`; do not hand-name):

```sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_tpml_account_main_image
ON tbl_product_media_linkers (account_id, product_id, variant_id, channel_id)
WHERE is_main_image IS TRUE;
```

With it, the 5.1 CTE becomes an index scan over only the account's main-image rows (435 of 490k → a few ms) instead of a 9.4k-page seq scan. The same index also helps the other readers of this table (`product.service.ts`, `productListQueryBuilder.ts`, `readiness-media.resolver.ts`) if they filter on account/product + main image.
**Not measured:** the DB session was read-only, so the index could not be created; benefit is derived from the plan (435-row selectivity), not benchmarked. Verify with `EXPLAIN` after creating it on staging.

### 5.3 (Impact: small today, protects the future) Restrict the media work to Amazon rows

`collectKeySignalGroups` only evaluates `resolveMainImageState` when `channelType === 'AMAZON'`, yet the SQL runs the baseline + linker lookups for **every** published key (Shopify included) and discards the result. Guard the two `LEFT JOIN LATERAL ... mb/mib/mi` with `AND pk.channel_type = 'AMAZON'` (or compute media in a separate Amazon-only query). Saves the lookup for all Shopify keys.

### 5.4 (Impact: ~0.3–1 s, less DB/connection pressure) Run one SQL pass instead of four

Inventory, pricing, listing-issues and key-signals each rebuild `scoped_channels → active_* → pulse_keys` (`pulse_keys` alone is ~177 ms incl. RTT) and each takes a pooled connection; the Pulse endpoint (loaded by the same page) repeats the same CTE again. Options:
1. Materialise `pulse_keys` once into a temp CTE and return the four signals as separate result sets via a single multi-statement call (or one query with `jsonb_agg` blocks), or
2. Cache the `pulse_keys` rows per `(account, scope)` in Redis for the 90 s window and let pulse + attention both read them.
Gain grows with account size and with simultaneous section loads sharing `DB_POOL_MAX_MAIN`.

### 5.5 (Impact: repeat loads and concurrency) Make the cache cover the cold path

Currently `getKey` → miss → compute → `setKey` (TTL `COMMAND_CENTER_ATTENTION_CACHE_TTL_SECONDS = 90`):
- Key includes `user:${req.user.id}` although the scope is already expressed by `scope:<channelIds>` — two users with the same channel access recompute independently. Drop `user:` from the key.
- No single-flight: concurrent refreshes (tab reload, two users) all run the full computation. Add a short Redis lock / in-process promise map per key.
- Consider stale-while-revalidate (serve the cached payload past TTL, refresh in background) so users never wait on a cold compute. Optionally warm after channel syncs.

### 5.6 (Impact: ~0.3–0.5 s) Channel health: parallelise its serial round trips

`ChannelHealthService.loadChannelInputs` runs: channels → `Promise.all(marketplaces, failures, stockFailures)` → `loadPublishedListingCounts` → `loadRecentOrderMarketplaces`. The last two depend only on earlier results but run one after another; `loadPublishedListingCounts` is skipped when there are no stock failures. Start `loadRecentOrderMarketplaces` as soon as marketplaces are known, in parallel with listing counts. Also `getUserWiseChannels` is resolved twice per request (service + health); pass the already-resolved scope.

### 5.7 (Impact: low now, risk later) Missing supporting indexes on the other lateral lookups

| Table | Rows now | Existing indexes | Note |
|---|---|---|---|
| `tbl_product_media_sync_baselines` | 20 | **PK only** | Seq-scanned once per key (loops=367) — free today (2 pages), becomes a per-key full scan if populated. Add `(product_id, channel_id)`. |
| `tbl_amazon_fba_inventories` | 2,705 | **PK only** | Inventory lateral does `product_id, variant_id, channel_id, marketplace_id`; add `(product_id, channel_id, marketplace_id, variant_id)`. |
| `tbl_channel_product_warehouses` | 119 | PK only | Tiny; index `(product_id, channel_id)` only if it grows. |
| `tbl_price_sync_baselines` | 204 | `product_id` | OK. |
| `tbl_product_variant_prices` | 256,389 | variant-leading indexes + `(product_id, product_channel_id)` | Pricing query 493 ms; watch item. |

### 5.8 (Correctness note, not latency) Listing-issue `LIMIT :maxScanned` has no `ORDER BY`

`buildListingIssuesQuery` caps at 5,000 rows without an ordering, and `collectListingGroups` slices again. For accounts with more than 5,000 issue rows the surviving subset is arbitrary (could drop blockers while keeping warnings and switch the card from "blocker" to "warning"). Order by severity (errors first) before the limit, or aggregate in SQL instead of capping.

---

## 6. Expected outcome

| Step | Cold response (acct 329) |
|---|---|
| Today | ~39 s measured (~30 s observed) |
| + 5.1 | ≈ 0.5–1 s (slowest signal 0.49 s + health + overhead) |
| + 5.2 | same, but flat as the table grows |
| + 5.4 / 5.6 | trims ~0.3–1 s more; fewer DB connections |
| + 5.5 | repeat loads served from cache; no stampede |

Recommended order: **5.1 → 5.2 → 5.5 → 5.3 → 5.6 → 5.4 → 5.7**.

---

## 7. Method, limits & caveats

- Queries were generated by importing the real builders (`command-center-pulse.sql.ts`, `command-center-attention.sql.ts`) and run with `BEGIN READ ONLY` + `ROLLBACK`; no writes, no DDL.
- One account (329; small: 367 keys). Accounts with more keys scale the broken query proportionally to keys × table size.
- Timings include ~145 ms RTT per query from the analysis machine; the 39 s figure is dominated by server-side scanning, not latency.
- Index proposals (5.2, 5.7) are reasoned from plans and table sizes; they were not created or benchmarked.
- `getHealth` was timed query-by-query (not the service end-to-end); its in-memory derivation was not profiled (small data, expected negligible).
- The `~30 s` reported by the user vs `39 s` measured differs by cache state/hardware; the plan clearly attributes the whole difference to the linker seq scan.

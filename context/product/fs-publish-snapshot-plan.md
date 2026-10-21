# Plan — Forward Sync (publish) consistency with sync baselines (api + db-migrations)

Status: **plan only, nothing implemented yet.** Written 2026-10-20.
Paths are relative to `api/apps/api-main/src/` unless stated. Line numbers are as of today. Re-check them before editing.

---

## 0. Decisions taken with the user

| #   | Topic                          | Decision                                                                                                                                                                                                                    |
| --- | ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | Approach                       | **Shadow T0 snapshot table** (user's solution 2). Nothing is blocked. The first edit/revert after T0 on an in-flight scope keeps the T0 value; the payload reads snapshot-else-live; baselines are re-derived from the snapshot after FS |
| D2  | Baseline groups                | **All six**: attributes, variant attributes, media, pricing, misc, variant structure                                                                                                                                        |
| D3  | Channels                       | **Amazon + Shopify** FS                                                                                                                                                                                                     |
| D4  | BS overlapping an in-flight FS | **Out of scope**. BS behaviour is unchanged                                                                                                                                                                              |
| D5  | Overlapping publish            | **Rejected server-side** when any requested scope already has an in-flight task                                                                                                                                              |
| D6  | What is captured               | **Only fields in the publish selection.** A partial publish captures only its selected fields; a full publish captures everything. Other fields follow the normal baseline flow                                          |
| D7  | Webapp                         | **No change**                                                                                                                                                                                                                |
| D8  | Stuck tasks                    | Finalise as **failure** from the existing task fail/orphan paths. No new cron                                                                                                                                                |
| D9  | "Is it being published?" check | Decided by the existing **`sync_in_progress` flag** on the scope entity: Amazon → PCM (parent/standalone) / VM (variant); Shopify → PC (covers its variants, since Shopify publishes the whole product). Used both for overlap rejection and for the capture guard (§4.1, §4.2) |
| D10 | Read order during payload build | **Snapshot is read after the live rows** at every read point (Gap 1, §4.3). The other race gaps discussed (edit concurrent with the click; write paths that bypass capture) are **not** addressed in this revision |

"T0" = the moment the publish task is created (`tbl_task_queues.created_at`). "In-flight scope" = a PC/VC/PCM/VM that a not-yet-finalised FS task will publish.

---

## High-level walkthrough of the solution

Requirement for Forward Sync (FS) (Publish):

- It forwards the value state as it was when the publish task was created (T0)
- Changes are allowed while FS is in progress, but they never go into the payload
- Revert and field modification are **not blocked**, not even for fields that had baselines before the task started

We maintain an extra value state, the "publish snapshot". It stores the T0 value of anything changed while FS is in progress for that product.

When the publish task is created

- If any product / variant in the request is already being published into that channel / Amazon marketplace (its `syncInProgress` flag is already true), reject the new publish
- Otherwise set `syncInProgress` = true on everything being published (as today)
- Record which product / variant / channel / marketplace the task is going to publish, and which fields (the whole product for a full publish, only the selected changes for a partial publish)

When someone changes a value (or reverts a baseline) while FS is in progress

- "In progress" is decided by the `syncInProgress` flag of that product / variant in that channel / Amazon marketplace
- If the flag is true and the field is part of what is being published
  - Before the change is saved, store the value it had at T0 in the snapshot
  - Only the first change is stored. Later changes (Blue → Red → Yellow) do not overwrite it, so the snapshot always holds the T0 value
- The change itself is saved to the normal tables as usual
- Baselines keep working exactly as today. Until the publish finishes, the channel still has the old value, so the baselines are still correct
- If the field is not part of the publish (e.g. a partial publish of pricing only and an attribute is edited), nothing is stored and it follows the normal baseline flow

While FS is building the payload

- First read the values from the normal tables, then read the snapshot values (always in this order)
- If a field has a snapshot value, pick the snapshot value
- Otherwise, pick from the normal tables (the field hasn't changed since T0, so it is the T0 value)
- Why this order: a change saves its new value and its snapshot value together. So if the normal-table read already saw the new value, the snapshot read that comes after it is sure to see the snapshot value. In the reverse order, a change saved between the two reads would put its new value into the payload

In case of FS succeed

- Clear the baselines of what was published (as today, but for all of them, not only those created before the task)
- Set the status to In Sync
- Then, for every field that has a snapshot value
  - If the current value is still the same as the snapshot (published) value, do nothing
  - If the current value differs, create the baseline with the snapshot value as the previous sync value
  - The status becomes Out of Sync
- This also applies when the product was Not Published. Changes made during the first publish become baselines, and the status becomes Out of Sync

In case of FS failure

- Do not touch baselines or status
  - For a published product, the baselines created during the publish already describe what the channel really has (the publish never reached it)
  - For a Not Published product, no baselines were created during the publish, so it simply stays Not Published and there is nothing to delete

When the task is finished (success, failure, or stuck/orphaned)

- Set `syncInProgress` back to false (as today) and delete the snapshot values and the "being published" records of that task

Examples

- Green → Blue (baseline Green) → publish task starts → revert to Green
  - Snapshot stores Blue, so the payload sends Blue
  - Succeed: channel has Blue, local Green, so a baseline is created with Blue and the status is Out of Sync
  - Failure: channel still has Green, local Green, so the status is In Sync
- Green → Blue (baseline Green) → publish task starts → Blue → Red
  - Snapshot stores Blue, so the payload sends Blue
  - Revert during the publish goes back to Green (the channel still has Green at that moment)
  - Succeed: the baseline becomes Blue (local Red or Green ≠ Blue), so the status is Out of Sync and a revert after this goes to Blue
- Product Not Published (Blue) → publish task starts → Blue → Red
  - Payload sends Blue
  - Succeed: baseline Blue is created and the status is Out of Sync
  - Failure: stays Not Published

---

## 1. Problem: what is wrong today

### 1.1 Baseline semantics (the invariant we must keep)

A baseline row (`tbl_*_sync_baselines`) means: *"the channel currently holds `previous_sync_value`, and the local value differs from it."*
So for every published scope and field: **channel value = baseline.previous_sync_value if a baseline exists, else the local value.**
`SyncBaselineService.reconcileSyncStatus` (`modules/app/sync/baselines/sync-baseline/sync-baseline.service.ts:300`) derives SUCCESS / OUT_OF_SYNC from whether any baseline is left.

### 1.2 Why FS breaks it

FS runs in three timelines:

| Moment                        | Where                                                                                                                                                                                                                                                                                       |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **T0** task created           | api-main: `AmazonProductFSService.publishProducts` (`…/amazon/forwardSync/amazon-product-fs.service.ts:157`, task at :310 / `syncStandAloneProduct` :601 / `syncVariantOnAmazon` :1097); `ShopifyProductFSService.publishProducts` (`…/shopify/forwardSync/shopify-product-fs.service.ts:214`). Only SKU/scope metadata is stored in Redis (`feedData`). **No values are read.** |
| **T1** payload built (per product, batch by batch) | api-worker: Amazon `AmazonProductFSQ1Service.processParentProductData` (:560), the standalone path (:2571+), `AmazonProductVariantFSService` (:130/:638), `AmazonProductFSService.getPrimaryAttributesData` (:2371); Shopify `ShopifyProductFSQueueService.collect*` / `build*` (:1494–:3116). **Reads live tables.** |
| **T2** channel accepted       | Amazon `AmazonProductFSQ3Service` (feed result, :470–:580) / FSQ1 single-item (:3584); Shopify `ShopifyProductFSQueueService` (:749). Baselines are cleared with `before: taskQueue.createdAt` and status is reconciled. |

Defects (matching the task description):

1. **Payload is not the T0 state.** Any edit between T0 and T1 leaks into the payload. For a bulk publish, T1 for the last product can be many minutes after T0.
2. **Revert during FS.** Green→Blue (baseline prev=Green, created < T0) → task → revert. The revert writes Green and deletes the baseline. If T1 already read Blue, the channel gets Blue while local is Green and no baseline exists: **shown as in sync, but it isn't.**
3. **Edit of an already-baselined field during FS.** Green→Blue (< T0), Blue→Red (> T0). The baseline keeps prev=Green and the payload sends Red or Blue depending on timing. After T2 the baseline (created < T0) is cleared, so if Blue was sent, local Red ≠ channel Blue with no baseline.
4. **Edit back to the baseline value during FS.** Green→Blue (< T0), task, Blue→Green (> T0). `getBaselineMetrices` deletes the baseline (`attribute-sync-baseline.service.ts:1223`, `isValueUnchangedFromBaseline`). If Blue was sent, local Green ≠ channel Blue with no baseline.
5. **First publish (NOT_PUBLISHED).** All six processors create baselines only for `isEntityPublished` scopes (`attribute-sync-baseline.service.ts:507/526/567/589`, `price-sync-baseline.service.ts:59–70`, …). An edit during the first publish records nothing. After T2 the status becomes SUCCESS while local ≠ channel.
6. **The `before` filter is unreliable.** Baselines are now written by the async `sync-baseline-out-of-sync` queue, so `created_at` is the worker time, not the edit time. Also, an existing baseline that is *updated* during flight keeps its old `created_at`.
7. **Media deletions.** `removeDeletedImagesFromAmazon` (`amazon-product-fsq1.service.ts:4078`, called at FSQ1 :3544 / FSQ3 :509/:547) removes every recorded deletion. That includes images deleted after T0 that the T0 payload still sent.
8. **No server-side overlap guard.** Only the webapp disables the publish button (`syncInProgress`). Two tasks can run on one scope.

---

## 2. Target design (one picture)

```
T0  publishProducts (api-main)
      ├─ reject if any scope already has sync_in_progress = true    (D5, D9)
      ├─ create task
      └─ SET sync_in_progress = true on PCM/VM (Amazon) or PC (Shopify)
         + INSERT tbl_publish_inflight_scopes (task, scope, selection)  ← same tx

T0…T2  any edit / revert on that product (api-main, request tx)
      └─ PublishSnapshotService.capture(group, oldRows, tx)
            • only if the scope has sync_in_progress = true                              (D9)
              and its in-flight task's selection covers the field                        (D6)
            • INSERT … ON CONFLICT DO NOTHING  → first capture wins = T0 value
      └─ normal baseline trigger (unchanged, async)

T1  payload build (api-worker), per read point:
      └─ 1. read live rows   2. THEN read snapshot rows of this task               (D10)
         live rows  ──overlay(snapshot rows)──▶  T0 rows  ──▶ payload

T2  per scope result
      ├─ success → clear selected baselines of the scope (no `before`)
      │            → reconcileSyncStatus(skipPublishedStatusFilter)          (NOT_PUBLISHED → SUCCESS)
      │            → REPLAY snapshot: old = snapshot row, new = current live,
      │              restricted to this scope + selection, inline (not queued)
      │              ⇒ baseline prev = T0 value where local ≠ T0, status → OUT_OF_SYNC
      └─ failure → nothing (baselines written during flight are already correct
                   relative to the untouched channel; NOT_PUBLISHED stays NOT_PUBLISHED)

task end / fail / orphan  → sync_in_progress = false (existing writes)
                            + DELETE snapshot + inflight rows of the task
```

Why this satisfies every rule in the task:

- **Payload = T0 state.** Every field changed after T0 has a snapshot row holding its T0 value. Unchanged fields are still at their T0 value in the live tables.
- **Changes during FS are allowed** and are never in the payload.
- **Revert and edit stay enabled.** The snapshot is separate from `previous_sync_value`, so a revert cannot destroy the T0 value. The baseline keeps describing the channel's *pre-publish* state until T2, which is what the channel really holds until then.
- **NOT_PUBLISHED.** Changes made during the first publish materialise as baselines + OUT_OF_SYNC at success. On failure there was nothing to create or delete.

---

## 3. Data model (db-migrations)

Create with `cd db-migrations && npm run migrate:create -- --name publish-snapshots` (one migration, two tables). Add the entities in `entities/` and the `index.ts` barrel.

### 3.1 `tbl_publish_inflight_scopes`. One row per scope a task will publish

The `sync_in_progress` flag on `tbl_product_channel_marketplaces` / `tbl_variant_marketplaces` (Amazon) and `tbl_product_channels` (Shopify) is the **authority** on whether a scope is being published (D9). This table does not decide that. It only adds what the flag cannot hold: **which task** owns the scope (for snapshot keying) and **which fields** that task publishes (`selection`, for D6). The flag and the row are always written and cleared together (§4.1, §4.5).

| column                  | type        | note                                                                                                     |
| ----------------------- | ----------- | -------------------------------------------------------------------------------------------------------- |
| `id`                    | bigserial   |                                                                                                          |
| `task_queue_id`         | bigint      | FK `tbl_task_queues` ON DELETE CASCADE                                                                    |
| `product_id`            | bigint      |                                                                                                          |
| `variant_id`            | bigint null | null = parent/standalone scope                                                                           |
| `channel_id`            | bigint      |                                                                                                          |
| `amazon_marketplace_id` | bigint null | null for Shopify                                                                                         |
| `selection`             | jsonb null  | `PublishCaptureSelection` (below); **null = full publish**                                                 |

`selection` shape: `PublishCaptureSelection = { attributeIds: number[]; variantAttributes: boolean; variantStructure: boolean; media: boolean; pricing: boolean; miscTypes: string[] }`.

- It is derived from `BaselineSelectionFilter` (`incremental/interface/incrementalSync.interface.ts`) **once, at registration**, with Amazon `hierarchyIds` already resolved to the `attributeIds` they cover (merged into `attributeIds`). It carries no `hierarchyIds`.
- **Why:** the edit-time coverage check (§4.2) is then plain flag / set membership. It needs no hierarchy lookup inside the user's edit transaction, and it uses the same resolution the payload builder used when the publish was created.
- Not indexed: it is only read for scopes already found through the flag, and only in app code.

Indexes: `(product_id)`, `(task_queue_id)`. No unique index: the flag check (§4.1) and the frontend already guarantee one task per scope.

Not stored on purpose: `account_id` (rows are found by `product_id` or `task_queue_id`, both already account-specific) and `started_at` (T0 = `tbl_task_queues.created_at`, reached through `task_queue_id`; the finalisation and media-deletion filters already load the task).

### 3.2 `tbl_publish_snapshots`. T0 image of every row changed after T0

| column          | type        | note                                                                                                                              |
| --------------- | ----------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `id`            | bigserial   |                                                                                                                                   |
| `task_queue_id` | bigint      | FK `tbl_task_queues` ON DELETE CASCADE                                                                                             |
| `product_id`    | bigint      |                                                                                                                                   |
| `group`         | text        | one of `GlobalEnums.SYNC_BASELINE_GROUPS`                                                                                          |
| `row_key`       | text        | deterministic identity of the captured row/value per group (§5)                                                                    |
| `variant_id`            | bigint null | scope the value belongs to (same meaning as in `tbl_publish_inflight_scopes`); null = parent/standalone |
| `channel_id`            | bigint null | null = master (media only)                                                                        |
| `amazon_marketplace_id` | bigint null | null for Shopify, and for media (no marketplace level)                                            |
| `publish_value` | jsonb null  | the value this publish sends (as at T0), in the per-group shape below. **SQL NULL only for an attribute / variant-attribute row that did not exist at T0** (an add after T0) |

The scope columns are filled for the groups captured per scope (variant attributes, variant structure, misc, pricing, media). They stay **all null for attribute rows**: a captured attribute row is a product-level source row that feeds many scopes through inheritance, so its identity lives only in `row_key`.

`publish_value` format rules:

1. **One typed shape per group.** A TypeScript discriminated union `PublishSnapshotValue` (keyed by `group`) in `publish-snapshot/` types. Capture, overlay and the success-step replay all use it; no untyped `any`.
2. **Same shape as the trigger's old values.** Each group stores exactly what its baseline trigger already passes as the old value. The replay (§4.4) can then hand `publish_value` straight back to the processors, whose equality checks (`isEqualValue`, `compareChangedData`) already handle that shape: prices as strings, dates as ISO strings. This avoids fake baselines from type mismatches such as `12.5` vs `'12.50'`.

   | Group | `publish_value` |
   |---|---|
   | ATTRIBUTES | trimmed `oldAttributes` row, or SQL NULL if the row didn't exist at T0 |
   | VARIANT_ATTRIBUTES | the `oldRecords` item (`VariantAttributeSyncOutOfSyncRecord` value part), or SQL NULL if it didn't exist at T0 |
   | VARIANT_STRUCTURE | `{ oldVariantAttributeIds, oldMetaData }` |
   | MEDIA | trimmed `oldMediaRecords` array plus `wasInherited`; an empty set is `[]` |
   | MISC | `oldValueObj` (`MiscSyncBaselinePreviousSyncValue`) |
   | PRICING | the old `pricingDetails` object |

3. **Null means only "did not exist".** For variant structure, media, misc and pricing, `publish_value` is **never** SQL NULL. It is always the group's object or array, even when the fields inside it are empty (e.g. `{ conditionType: null }`, `[]`). Never rely on `'null'::jsonb` vs SQL NULL, because Sequelize blurs the two.
4. **Trimmed rows.** Attribute and media rows keep only what the overlay and replay need: identity columns, value columns, `isInherited`, `sequence` (media: `digitalAssetsId`, `isMainImage`, `sequence`, `amazonUrl`, `shopifyExternalId`, as in `ProductMediaSnapshot`). No `id`, no `createdAt` / `updatedAt`, no joined objects.

Unique `(task_queue_id, group, row_key)`. `row_key` still carries the full identity, including variant / channel / marketplace, so uniqueness needs no null-handling on the scope columns. Captures use `INSERT … ON CONFLICT DO NOTHING`, so the **first** capture after T0 wins and later edits never overwrite it.

Index `(task_queue_id, product_id, "group", variant_id, channel_id, amazon_marketplace_id)`. It serves the overlay read ("this task, this product, this group, this scope"), `finalizeScope` (same filter), and per-product release (prefix `task_queue_id, product_id`). Release by task uses the leading `task_queue_id`.

Both tables are transient. Rows are deleted at task finalisation (§7) and cascade if a task row is deleted.

---

## 4. Lifecycle in detail

### 4.1 Register (T0, api-main)

New `PublishSnapshotService` in `modules/app/sync/baselines/publish-snapshot/` (module + service + types), exported to the catalog/sync modules that need it.

- **Which flag marks which scope (D9)**, as written today:
  - Amazon parent / standalone → `tbl_product_channel_marketplaces.sync_in_progress` (`amazon-product-fs.service.ts:828`, :1853, :2072).
  - Amazon variant → `tbl_variant_marketplaces.sync_in_progress` (`buildFeedDataForMarketplaces` :2014).
  - Shopify → `tbl_product_channels.sync_in_progress` (`shopify-product-fs.service.ts:330`). Shopify publishes the whole product, so this one flag also covers its variants. `tbl_product_variant_channels.sync_in_progress` is not set by Shopify FS today and is not needed for this.
- `registerInflightScopes({ req, taskQueueId, scopes: { productId, variantId, channelId, amazonMarketplaceId, selection }[], transaction })`
  - Called exactly where the flags above are set to true, and in the **same transaction** as that flag update. The flag and the registry row then become visible together. Today those flag updates run without a transaction, so each site gets one wrapping the flag update + registry insert.
  - Amazon selection comes from `ProductPublishPlan` → `MarketplacePublishPlan` (already in hand there) mapped through `SyncSelectionService.toBaselineFilter` (`incremental/sync-selection.service.ts`). It is then converted to `PublishCaptureSelection` (§3.1) by resolving `hierarchyIds` → `attributeIds` in one query per registration (not per scope).
  - Shopify selection = null (Shopify carries no partial logic yet, per [incremental-publish-decisions]).
- **Overlap rejection (D5, D9).** Before creating the task, `assertNoInflightScopes(scopes)` checks the **flags**: any requested PCM / VM (Amazon) or PC (Shopify) with `sync_in_progress = true` means the scope is being published. The API then returns `publish_already_in_progress` (new i18n key) and no task is created. Today the webapp disables the button from these same flags; this moves the same rule to the server.
- A flag left `true` by a crashed task blocks publishing that scope until the existing orphan paths reset it (§4.5). That is the same as the webapp button today.
- **Partial overlap in a bulk request.** The whole request is rejected, which keeps it simple and predictable. *(Not asked. See open item O1.)*

### 4.2 Capture (T0…T2, api-main, inside the edit's transaction)

`PublishSnapshotService.capture({ req, group, items, transaction })`:

1. Cheap guard on the **flags** (D9): one query over the edited products' PCM / VM / PC rows `WHERE sync_in_progress = true`. If none, return immediately. This is the path almost every edit takes. (Optional later: a Redis set `FS:inflight:{accountId}` to skip even this query.)
2. For the flagged scopes only, join `tbl_publish_inflight_scopes` on the scope to get the owning `task_queue_id` and `selection`. Keep only items the task's `selection` covers (D6, §5 per group). `selection = null` covers everything. A flagged scope with no registry row (flag set by a code path that doesn't register, or a leftover from before this change) cannot be keyed to a task. It is logged as a warning and skipped.
3. `INSERT INTO tbl_publish_snapshots … ON CONFLICT (task_queue_id, group, row_key) DO NOTHING` with the **old** (pre-edit) image, inside `transaction`.

**Race rule (critical).** The capture must commit **no later than the live write**. Otherwise the worker can read the new live value at T1 before the snapshot exists. So:

- Call sites whose write runs in a transaction pass that transaction to the capture, and the capture runs **before commit**.
- Call sites that write without a transaction call `capture` **before** the write (the old values are already computed there).

**Where capture is wired.**

1. `SyncBaselineTriggerService.trigger` (`baselines/sync-baseline/trigger/sync-baseline-trigger.service.ts`) calls `capture` synchronously *before* it schedules the enqueue. Its payload already carries the old values for all six groups (`sync-baseline-out-of-sync.types.ts`: `oldAttributes`, `oldRecords`, `oldMediaRecords`, misc `oldValueObj`, structure `oldVariantAttributeIds/oldMetaData`, price `pricingDetails`). The trigger becomes `async capture → enqueue`, and still never throws: a capture failure is logged and must **fail the edit** (rethrow inside the caller's tx) rather than silently lose the T0 value. *(See O2.)*
2. 37 trigger call sites today (product.service 16, product-variants-flow 6, update-details 5, product-channel 4, product-pricing 4, shopify-product 1, amazon-channel-product-type 1). About 10 already pass a transaction. **Every other site must move its trigger call before `commit()` and pass `transaction`, or (no-tx writes) call `capture` before the write.** The call-site audit table in `out-of-sync-queue-plan.md` §1.1 lists them by TX-open / after-commit / no-tx. Re-verify each line, then update that table with the new column "capture position".
3. **Revert endpoints** write live values directly, without the trigger. Add `capture` inside their transactions before the live write:
   - `AttributeSyncBaselineService.restoreAttributeSyncBaseline` (`attribute-sync-baseline.service.ts:1572`). Capture the current row image before `upsertProductAttributes` / `delete`.
   - `VariantAttributeSyncBaselineService`, `ProductMediaSyncBaselineService`, `MiscSyncBaselineService`, `PriceSyncBaselineService`, `VariantStructureBaselineService` restore methods (their `restore-*` DTOs/controllers under `baselines/*`).

### 4.3 Overlay (T1, api-worker)

`PublishSnapshotService.loadForTask(taskQueueId, productIds, groups)` returns snapshot rows grouped by product and group. Pure overlay helpers (one per group, §5) take *live rows + snapshot rows* and return *T0 rows*. Each payload read point applies the matching helper to its rows before building values. No other builder logic changes.

**Read-order rule (D10, Gap 1). At every read point: read the live rows first, then load the snapshot rows for that group, then overlay.** Never load snapshots up front and reuse them for later live reads.

- **Why:** a change writes its new live value and its snapshot row in the same transaction (§4.2). Snapshot rows are insert-only and are not deleted before the task is released. So if a live read already sees a changed value, any snapshot read that starts after it is guaranteed to see that change's snapshot row.
- **The reverse order is unsafe:** (1) snapshot loaded and empty, (2) a change commits, (3) the live read returns the new value, so the new value goes into the payload.
- **Cost:** one extra indexed query per read point (`task_queue_id, product_id, group`). For read points that run inside one product loop, the snapshot query can be shared only if it runs **after the last live read** it serves.
- **Shopify:** the `collect*` steps write intermediate data to Redis (`productKey`). The overlay is applied in each `collect*` step right after its own live query, before writing to Redis, not later at the `build*` step.

Read points:

| Group                         | Amazon read points                                                                                                                                                                                                                      | Shopify read points                                                                     |
| ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Attributes                    | `AmazonProductFSService.getPrimaryAttributesData` (:2371, primary attr groups :2394), `AmazonProductFSQ1Service.extractProductAndVariantData` (:134, :150/:216), `processParentProductData` attribute reads, standalone path (:3464), `AmazonProductVariantFSService.getPrimaryAttributesData` (:213) | `collectPrimaryDataOfproduct` (:1668), `collectShopifyAttributeGroups` (:2095), `buildMetafields` (:2749) |
| Variant attributes            | `extractProductAndVariantData`, `AmazonProductVariantFSService.syncSingleVariantOnAmazon` (:638)                                                                                                                                         | `collectVariantOptions` (:1809), `getVariantsInBatch` (:1899)                            |
| Pricing                       | `getPrimaryAttributesData` price read (:2442–:2511)                                                                                                                                                                                      | `getVariantsInBatch` price fields                                                        |
| Media                         | `processParentProductData` (:821), standalone (:2788)                                                                                                                                                                                    | `buildMedia` (:3116)                                                                     |
| Misc (condition, shipping, category, item type) | `getPrimaryAttributesData` (`condition_type`, `recommended_browse_nodes` from PCM), shipping template lookups (fsq1 :710 / :2675), product type (:627 / :2607)                                         | `collectCoreProduct` (:1589: tags, category, theme template, publications, collections)  |
| Variant structure             | wrapper / theme reads (`processParentProductData` :749 / :900 / :1789, `getWrapperSku` variant-fs :2060)                                                                                                                                 | `collectVariantOptions`                                                                  |

**Media deletions (defect 7).** `removeDeletedImagesFromAmazon` and the `MediaDeletions.findAll` reads (fsq1 :1558 / :2424 / :3538) only take deletions with `createdAt < task.createdAt`. A later deletion belongs to the next publish. (Check that `tbl_media_deletions` has `created_at`; if not, add it in the same migration.)

### 4.4 Finalise per scope (T2, api-worker)

New `PublishSnapshotService.finalizeScope({ req, taskQueueId, scope, outcome: 'SUCCESS' | 'FAILED', selection })`. **On SUCCESS**, at the three clear sites (Amazon fsq1 :3584, fsq3 :501 / :539; Shopify fs-queue :749):

1. `clearSyncBaselines` for the scope + selection, **without `before`** (it is replaced by step 3). Selection and `restrictToParentEntity` stay as today.
2. `reconcileSyncStatus({ skipPublishedStatusFilter: true })` as today. NOT_PUBLISHED / OUT_OF_SYNC becomes SUCCESS.
3. **Replay.** For this task's snapshot rows that the scope + selection cover, read the **current** live values and call the existing processors inline (not through the queue) with `old = publish_value`, `new = current`, restricted to this scope:
   - `AttributeSyncBaselineService.processAttributeOutOfSync`, `VariantAttributeSyncBaselineService.processVariantAttributeOutOfSync`, `VariantStructureBaselineService.processVariantStructureOutOfSync`, `ProductMediaSyncBaselineService.processProductMediaOutOfSync`, `MiscSyncBaselineService.processMiscSyncOutOfSync`, `PriceSyncBaselineService.processPriceOutOfSync`.
   - Each gains an optional `restrictToScopes?: SyncBaselineScope[]` (and `restrictToSelection?: BaselineSelectionFilter`) argument that filters the PC/VC/PCM/VM fan-out. **Why:** an attribute row fans out to every marketplace. Without the filter, a marketplace whose publish failed would get prev = T0, which is wrong for it.
   - The processors already produce exactly the right result: no existing baseline + value changed → baseline with `previous_sync_value = old` (the T0 value), and status flips to OUT_OF_SYNC (step 2 made it SUCCESS, which `isEligibleForOutOfSync` accepts). Equal values create nothing.
   - Route the call through `SyncBaselineOutOfSyncExecutorService.run` (`baselines/sync-baseline/execution/`) so the six-way dispatch exists only once.

**On FAILED** for a scope (Amazon message with issues / feed failure / single-item error; Shopify product error): **no baseline or status change.** During flight the normal baseline flow kept describing the channel's untouched pre-publish state. For a NOT_PUBLISHED scope it created nothing (the existing `isEntityPublished` gate), so the scope stays NOT_PUBLISHED.

### 4.5 Release

When a task reaches a terminal state, `releaseTask(taskQueueId)` deletes its `tbl_publish_inflight_scopes` and `tbl_publish_snapshots` rows. This runs after every scope's finalisation, **in the same transaction as the existing `syncInProgress: false` writes**, so the flag (D9) and the registry never disagree. Existing reset points: `queue.service.ts` :1538 / :1548 (Shopify failure), :1757 / :1766 (orphan), :2614 (task data failure), plus the FS success paths (fsq1 :3607, Shopify fs-queue :747). The terminal paths are:

- Success / complete: `QueueService.processAmazonSyncForwardComplete` (`sync/queue/queue.service.ts:2801`), `finalizeStandaloneForwardSync` (:2272), Shopify FS completion in `shopify-product-fs-queue.service.ts`.
- Failure (D8): `handleAmazonSyncForwardFailure` (:2073), `handleStandaloneForwardSyncFailure` (:2232), `failAllTaskQueueRawData` (:2667), `updateTaskQueueData` (:2567), `handleShopifySyncForwardFailure` (:1437), the request-side catch in `publishProducts` (amazon-product-fs :585), and the orphan paths `failOrphanedTask` (:1593) / `checkAndMarkOrphanedTasks` (:1801). In these paths, every scope not yet finalised is treated as **FAILED** (no-op) and then released.

A scope can also be released individually, at its own finalisation (where its flag is set back to false today), so that a bulk task does not keep a finished product locked while other batches run. Delete only that scope's inflight row, together with its flag reset. Snapshot rows of that product are deleted when the product has no inflight scope left in the task.

---

## 5. Per-group capture key, selection coverage, overlay

| Group                                  | Captured from (trigger payload)                                                                | `row_key`                                                                                     | Selection covers it when                                                                     | Overlay                                                                                     |
| -------------------------------------- | ---------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| ATTRIBUTES                             | `oldAttributes` (ProductAttribute rows; absent row ⇒ `publish_value = null`)                       | `pag:{pagId}\|attr:{attributeId}\|v:{variantId?}\|ch:{channelId?}\|lang:{languageCode}`         | `selection` null, or `attributeId ∈ selection.attributeIds` (hierarchies already resolved at registration) | replace / remove / add `ProductAttribute` rows by key; the existing inheritance logic then resolves T0 effective values |
| VARIANT_ATTRIBUTES                     | `oldRecords` (per scope + `productVariantAttributeId` + language)                               | `pva:{id}\|v:{variantId}\|ch\|mp\|lang`                                                        | `selection.variantAttributes`                                                                 | replace the value for that scope                                                            |
| VARIANT_STRUCTURE                      | `oldVariantAttributeIds` + `oldMetaData` per scope                                              | `v:{variantId}\|ch\|mp`                                                                        | `selection.variantStructure`                                                                  | replace variant-attribute ids + wrapper/theme metadata for that scope                       |
| MEDIA                                  | `oldMediaRecords` (+ `wasInherited`) per product/variant/channel                                | `v:{variantId?}\|ch:{channelId?}` (whole media set)                                            | `selection.media`                                                                             | replace the scope's whole media linker set                                                  |
| MISC                                   | `oldValueObj` per scope + `baselineType`                                                        | `misc:{baselineType}\|v\|ch\|mp`                                                               | `baselineType ∈ selection.miscTypes`                                                          | replace those fields on PC/PCM/VC/VM read results                                           |
| PRICING                                | `pricingDetails` (old) per scope + `priceType`                                                  | `price:{priceType}\|v\|ch\|mp`                                                                 | `selection.pricing`                                                                           | replace `pricingDetails` used for `purchasable_offer` / `list_price` / Shopify price       |

**Phase-0 verification** (before coding the capture): for every call site, confirm that the old payload is a **complete** pre-edit image for the overlay (identity + all value columns + `isInherited`/`sequence`) and that deletions/additions are represented. Where it is not complete, read the live row(s) inside the transaction **before** the write and capture that instead, converted to the §3.2 `publish_value` shape. Price sites also pass ORM instances today. Use ids only (as the queue plan already requires).

---

## 6. Scenario walkthroughs (from the task)

| # | Story | Payload | After success | After failure |
|---|---|---|---|---|
| S1 | In Sync, Green. Task. Green→Blue during FS | Green (snapshot) | baseline prev=Green, OUT_OF_SYNC | normal flow already made baseline prev=Green, OUT_OF_SYNC. Correct (channel Green) |
| S2 | Green→Blue (< T0, baseline prev=Green). Task. Revert during FS | Blue (snapshot captured Blue at revert) | channel Blue, local Green ⇒ baseline prev=Blue, OUT_OF_SYNC | revert deleted baseline, local Green = channel Green ⇒ SUCCESS |
| S3 | Green→Blue (< T0). Task. Blue→Red during FS | Blue | baseline prev=Blue (local Red), OUT_OF_SYNC; a revert then goes to Blue | baseline still prev=Green (correct, channel Green); revert → Green |
| S3b | S3, then revert *during* FS | Blue | local Green ≠ Blue ⇒ baseline prev=Blue | local Green = channel Green ⇒ no baseline |
| S4 | Green→Blue (< T0). Task. Blue→Green during FS | Blue | baseline prev=Blue, OUT_OF_SYNC | baseline deleted by normal flow; local Green = channel Green |
| S5 | NOT_PUBLISHED, Blue. Task. Blue→Red during FS | Blue | SUCCESS, then replay ⇒ baseline prev=Blue, OUT_OF_SYNC | stays NOT_PUBLISHED, nothing to delete |
| S6 | Partial publish (pricing only). Attribute edited during FS | attribute not sent | attribute not captured (D6), its normal baseline untouched | same |
| S7 | Second publish on an in-flight scope | rejected `publish_already_in_progress` | — | — |

The user's solution-1 rules map onto these as follows. "Create baseline + Out of Sync" for NOT_PUBLISHED = S5 success. "On failure reset to NOT_PUBLISHED + delete baselines" = S5 failure (none are ever created). "Pick baseline value if created after the task" = the snapshot overlay. No field needs blocking.

---

## 7. Implementation phases

1. **P0, verification (no code):** the §5 completeness check per call site; `tbl_media_deletions.created_at`; confirm every Amazon/Shopify FS terminal path in §4.5; refresh the call-site table.
2. **P1, schema:** migration with both tables (+ media deletions `created_at` if missing); entities `PublishInflightScope`, `PublishSnapshot`; barrel export.
3. **P2, service:** `publish-snapshot/` module with `assertNoInflightScopes`, `registerInflightScopes`, `capture`, `loadForTask`, overlay helpers (pure functions), `finalizeScope`, `releaseTask`. JSDoc on all functions (api rule).
4. **P3, T0 wiring:** overlap rejection by `sync_in_progress` flag + registration in the same transaction as the flag update, in Amazon (`publishProducts`, `syncStandAloneProduct`, `syncVariantOnAmazon`) and Shopify `publishProducts`. Add the i18n key `publish_already_in_progress`.
5. **P4, capture wiring:** trigger service capture-before-enqueue; move after-commit triggers before commit / add pre-write capture for no-tx sites; six restore endpoints.
6. **P5, overlay wiring:** the §4.3 read points for Amazon (bulk, variant, standalone) and Shopify, each following the live-then-snapshot read order; media deletion `createdAt < T0` filter.
7. **P6, finalisation:** replace `before` at the four clear sites with `finalizeScope`; add `restrictToScopes` / `restrictToSelection` to the six processors; `releaseTask` in all §4.5 paths, in the same transaction as the `syncInProgress: false` resets.
8. **No test spec files.** Do not add or change any `*.spec.ts` / e2e test files for this work. Verify scenarios S1–S6 (§6) manually against a running app instead.

No webapp changes (D7). Errors from the overlap rejection surface through the existing publish error toast.

---

## 8. Risks and notes

- **Uncaptured write paths.** Any write that does not go through the baseline trigger or a restore endpoint (e.g. a bulk import or AI apply path that skips the trigger) leaks into the payload. Those paths already fail to create baselines today, so this is not a new gap. P0 should list them.
- **Async baseline jobs vs. finalisation.** A normal `sync-baseline-out-of-sync` job from an in-flight edit can run *after* `finalizeScope`. Because the replay already left the baseline at prev=T0, the processors' existing-baseline branch keeps prev=T0 and the result converges. Ordering between two jobs for the same field is still unguarded (queue plan D3), which is unchanged.
- **Variant structure** is the riskiest group to overlay: wrapper/theme decisions are spread across `processParentProductData`. Implement it last.
- **Performance.** The capture guard is one indexed query per edit. Snapshot rows exist only for products that are actually in flight *and* edited.
- **Shopify FS** runs in one transaction per product (`shopify-product-fs-queue.service.ts` :749 commits after clear + reconcile). The replay should run after that commit, or inside the same transaction if the processors are given the transaction (they open their own today).

## 9. Open items (not decided yet; ask before implementing)

- **O1:** a bulk publish where only some products are in flight. Reject the whole request (current plan) or publish only the free products and report the skipped ones?
- **O2:** when `capture` itself fails, should the user's edit fail (current plan, keeps consistency) or succeed with an error logged?

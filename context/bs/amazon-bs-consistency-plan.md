# Plan — Amazon Backward Sync (Import): consistent & atomic batch lifecycle

Status: **plan only, nothing implemented yet.** Written 2026-09-19. Scope: `api/` only.

Files in scope (all under `api/apps/api-main/src/`):

| Alias | File |
| --- | --- |
| **BS** | `modules/app/sync/productSyncing/amazon/backwardSync/amazon-product-bs.service.ts` |
| **Q1…Q6** | `…/backwardSync/amazon-product-bsq1.service.ts` … `amazon-product-bsq6.service.ts` |
| **PROC** | `modules/app/sync/queue/base-queue.processor.ts`, `modules/app/sync/queue/processors/amazon-backward-sync.processors.ts` |
| **QS** | `modules/app/sync/queue/queue.service.ts` |
| **TQS** | `modules/app/sync/queue/task-queue/task-queue.service.ts` |
| **CH** | `modules/app/catalog/channel/channel.service.ts` |
| **CACHE** | `core/cache/cacheBase.service.ts` (+ new `core/cache/redis-errors.ts`) |
| **UTILS** | `common/lib/utils.ts` |
| **TYPES** | `core/types/sync/amazonSync.interface.ts` |

Line numbers below refer to the files as read on 2026-09-19.

---

## 0. Decisions already taken (with the user)

| # | Decision | Choice |
| --- | --- | --- |
| D1 | `syncedKeys` / `failedKeys` exclusivity | **Move + cleanup.** Adding SKUs to one set removes them from the other in the *same* SQL statement. When a previously-failed batch succeeds on re-run, the batch-scoped CATEGRA errors of the moved SKUs (`taskQueueId` + `metaData.currentQueue` + `sku`) are deleted and Q2's `importFailedCount` is decremented by the moved non-imported SKUs. A batch whose SKUs are already all in `syncedKeys[queue]` is skipped on re-run (index still advanced). |
| D2 | Redis data unavailable | **Classify.** Connection/timeout failures → transient (`RedisUnavailableError`) → job is delayed and retried like DB-connection errors in `BaseQueueProcessor`. Genuinely missing key → `UnrecoverableError(TASK_DATA_WAS_REMOVED)` → task fails immediately (manual retry restarts from Q1 via the existing `isPreviousSyncDataAvailable` path). Batch cleanup/failure handlers use a `skuMap` captured **before** the batch starts; they never re-read Redis to decide what to track. |
| D3 | Shared helpers | **May be modified** (TQS, CH, CACHE, PROC, UTILS) as long as callers that don't pass a transaction keep today's behaviour. |
| D4 | Where tracking writes live | **Tail of the batch transaction.** Success tracking (`syncedKeys` move, stale-error cleanup, channel sellable counts, master socket data) is the last statement group of the batch transaction; sockets are emitted in `transaction.afterCommit`. Failure tracking runs in its own short transaction after rollback. Q2's `setProductChannelMarketplaceLoading(true)` joins the batch transaction so its socket is emitted after commit and strictly before `productSuccess`. |
| D5 | Behaviour changes | End behaviour is preserved except for the items tagged **[FIX]** in §2 (things that are wrong today). Items tagged **[ASK]** are behaviour changes that need the user's go-ahead before implementation (listed again in §9). |

---

## 1. Current state

### 1.1 Pipeline

```
importProducts (BS:321)                      ── creates TaskQueue, initialises raw_data[queue].status, adds Q1 job
 └─ Q1  sync-amazon-fetch-products           ── fetch listings → parent/child/CP lists in Redis (no DB batches)
     └─ Q2  sync-amazon-create-base-product  ── CP batches: create products/variants/PCM/VM/pricing   (sequential)
          ├─ Q3  sync-amazon-process-media   ── CP batches: R2 upload + media linkers                 ┐
          ├─ Q4  sync-amazon-variant-schema-handler ── CP batches: product types, attributes, variation themes ┤ run IN PARALLEL
          │    └─ Q5  sync-amazon-translation ── CP batches: localisation + AI suggestions             │
          └─ Q6  sync-amazon-orders (optional) ── orders + order items (own Redis list `AM:order-*`)  ┘
```

Every processor is a `BaseQueueProcessor` subclass (PROC:12-130). `process()` (PROC:55-88):
`enforceTrialCountedJobActiveCap → handleJobActive → doProcess → handleJobCompleted`; on throw: DB-connection errors are delayed (`moveToDelayed`, max `DB_CONNECTION_MAX_RETRIES`), otherwise `handleJobFailed` → if attempts are exhausted (`DEFAULT_JOB_OPTIONS.attempts = 3`, exponential backoff 5 s; `shouldHandleFailedJobImmediately` QS:117) or the error is an `UnrecoverableError` → `QS.handleAmazonSyncImportFailure` (QS:1248) marks the task/queue FAILED (or `toBeFailed` when a parallel queue is running), stops loaders, starts deferred tasks.

`handleJobCompleted` → `QS.processAmazonSyncImportComplete` (QS:937): stops loaders when the "necessary" queues are done, and — for the last queues (`AMAZON_SYNC_IMPORT_LAST_QUEUES`) — reads `AM:task-*.productTracker`, writes `rawData.failedProducts`, stops channel processing, deletes Redis keys.

Batch state lives in two places:

| Store | Key / column | Written by |
| --- | --- | --- |
| Redis (RedisJSON) | `AM:task-{accountId}-{taskQueueId}` = `{ channelId, status, currentQueue, sellableCount, productTracker, [queueName]: { status, currentBatchIndex, batchSize } }` | `updateTaskInRedis` (BS:1538, whole-document `JSON.SET $`), `fetchParentSKUsBatch` (BS:4578), `updateBatchIndexInRedis` (BS:1554), `addTrackingProductsAtomic` (BS:1624, WATCH/MULTI on the whole key), `completeTaskQueueJob` (QS:739), Q6 `initializeTaskStatusInRedis` / `fetchBatchFromRedis` / `handleOrderSyncFailure` |
| Redis | `AM:cp-…`, `AM:child-…`, `AM:parent-…`, `AM:sku-…-*`, `AM:sync-disabled-entities-…`, `AM:tracking-…` (written once in Q1:208, never read → dead) | Q1, Q2 (`updateProductIdInRedis` BS/Q2:765 whole list; `processVariantPrimaryAttributes` Q2:1991 whole child list **inside the DB tx**) |
| Postgres `tbl_task_queues.raw_data` | `[queueName].{status,startedAt,completedAt,total,syncedKeys,failedKeys,retryCount}`, root `importFailedCount`, `toBeFailed`, `toBeStarted`, `failedProducts` | `TQS.addKeysToRawDataInnerObjectSetWithSocketTrigger` (TQS:663, set-union, **no removal from the sibling set**), `incrementRawDataRootKeyWithSocketTrigger` (TQS:722), `updateRawDataInnerObject*` |

### 1.2 Per-queue lifecycle today

| Step | Q2 (`processAllAmazonParentProducts` Q2:327-763) | Q3 (Q3:69-255) | Q4 (Q4:168-430) | Q5 (Q5:103-381) |
| --- | --- | --- | --- | --- |
| Batch fetch | `fetchParentSKUsBatch` (throws `TASK_DATA_WAS_REMOVED` on null) | same | same | same |
| Pre-tx work | — | R2 downloads/uploads + `attachAmazonImages` **commits its own tx** (BS:2220-2311) | `getProductTypeDataForMarketplaces` (own tx), `fetchMissingProductTypeSchemas` (own tx per type, Amazon API) | `startProductTranslationInProgress` (swallows) |
| Batch tx | products, wrappers, variants, PCM/VM, pricing, inheritance, `clearSyncBaselines`, `subtractUncreatedSellableCount`, `commit` | `clearSyncBaselines(MEDIA)`, `commit` | variation themes, attributes, `createMissingAttributeOptions`, `populateCategraErrorsWithIds`, `clearSyncBaselines`, **`safeCommitTransaction`** (swallows commit errors) | no tx (per-product service calls) |
| Success cleanup (order as coded) | `updateProductIdInRedis` (throws) → `Promise.allSettled([ addSellableKeys SYNCED (+channel socket, +master socket), setProductChannelMarketplaceLoading(true) ])` → `tryCatchWrapper( productSuccess → skuMap → tracker )` | `addSellableKeys SYNCED` (unwrapped) → `tryCatchWrapper( skuMap → tracker )` | `tryCatchWrapper(refreshAllSellableEntities)` → `calculateImportedProductsReadiness` (fire-and-forget, unhandled rejection) → `addSellableKeys SYNCED (+channel socket)` (unwrapped) → `tryCatchWrapper(skuMap → tracker)` | `stopProductTranslationInProgress` (not awaited) → `addSellableKeys SYNCED` (unwrapped) → `tryCatchWrapper(skuMap → tracker)` |
| Failure cleanup | `tryCatchWrapper( rollback → skuMap → tracker(hasErrors) → importFailedCount += nonImported → addSellableKeys FAILED (includeNonImported, +channel socket) → createCategraErrors(all SKUs) )` | `tryCatchWrapper( rollback → skuMap → tracker → FAILED → createCategraErrors(imported only) )` | `rollback` → `tryCatchWrapper( skuMap → tracker → FAILED (+channel socket) → createCategraErrors(imported only) )` | `tryCatchWrapper( skuMap → tracker → FAILED → createCategraErrors(imported only) )` |
| Finally | `updateBatchIndexInRedis` (throws) | `updateBatchIndexInRedis` (throws) | `setProductChannelMarketplaceLoading(false)` (**throws**) → `updateBatchIndexInRedis` (throws) | `updateBatchIndexInRedis` (throws) |
| Queue end | `setImportedProductsConciliation` → no-imported check (`UnrecoverableError`) → `completeTaskQueueJob` → status QUEUED for Q3/Q4 (awaited) & Q6 (not awaited) → `addJob` ×3 (not awaited) | `completeTaskQueueJob` | `completeTaskQueueJob` → Q5 QUEUED → `addJob` (not awaited) | `completeTaskQueueJob` |

Q1 has no CP batches (single flow, own tx only in `reconcileMissingImportedEntities` Q1:2822-3237). Q6 iterates `AM:order-*` in batches of 5 with one tx per order (Q6:649-724) and whole-document `updateTaskInRedis` writes (Q6:329, 807).

---

## 2. Flaw catalogue

Each flaw has an id (F#), the evidence, the effect, and the section that fixes it.

### F1 — Lost updates on `AM:task-*` between parallel queues → batch index rollback → **SKUs in both key sets** (root cause of the reported blocker) **[FIX]**

`updateTaskInRedis` (BS:1538-1552) reads the whole document, spreads new data, and rewrites it with `JSON.SET key $`. `fetchParentSKUsBatch` (BS:4611-4622), `updateBatchIndexInRedis` (BS:1567-1577), `completeTaskQueueJob` (QS:791-800), Q6 `initializeTaskStatusInRedis` (Q6:323-335), Q6 `fetchBatchFromRedis` (Q6:798-807) and `handleOrderSyncFailure` (Q6:229-241) all do the same. Q3, Q4 and Q6 are started together by Q2 (Q2:282-318) and run concurrently in api-worker, so this interleaving is routine:

```
Q3: read task {Q3.idx=7, Q4.idx=3}     Q4: read task {Q3.idx=7, Q4.idx=3}
Q3: write     {Q3.idx=8, Q4.idx=3}
                                       Q4: write     {Q3.idx=7, Q4.idx=4}   ← Q3's advance is lost
Q3: next loop fetches batch 7 again → re-runs it → if it now fails → batch 7 SKUs are in syncedKeys AND failedKeys
```

The same double-processing happens when (a) the processor throws after the SYNCED write but before the index advance (Q4's `finally` `setProductChannelMarketplaceLoading` rethrows on error, BS:1852-1856; `updateBatchIndexInRedis` throws on a null read) and BullMQ re-runs the job from the stale index (`attempts: 3`), or (b) `CacheBaseService.setKey` swallows a Redis error (CACHE:136-138) inside `updateBatchIndexInRedis`, so the index never advances and the `while (true)` loop re-processes the same batch until something fails. Fix: §3.2, §3.3, §3.4.

### F2 — `syncedKeys` / `failedKeys` are append-only sets **[FIX]**

`addKeysToRawDataInnerObjectSetWithSocketTrigger` (TQS:663-720) only unions. Nothing ever removes a SKU from the sibling set, so any re-run with a different outcome leaves the SKU in both. `importFailedCount` (Q2:729-734) is also only ever incremented, so a re-run that fails again double-counts. Fix: §3.3 (D1).

### F3 — Redis outage is indistinguishable from "key removed" **[FIX]**

`CacheBaseService.getKey` returns `null` on *any* error (CACHE:147-156), `getJsonArrayRange` likewise, `setKey`/`setJsonPartial`/`setJsonArrayRange` swallow errors. Every `if (!data) throw new Error(TASK_DATA_WAS_REMOVED)` in BS/Q2-Q6 therefore fires on a 1-second Redis hiccup (`REDIS_READY_TIMEOUT_MS = 1000`) and the job burns its 3 attempts on a transient problem, while a genuinely deleted key is retried 3× pointlessly. Inside batch failure handlers `getSellableEntitiesByCP` re-reads `AM:child-*` (BS:4640-4645); if Redis is the reason the batch failed, the handler throws inside `tryCatchWrapper` and **nothing** gets tracked (no FAILED keys, no errors). Fix: §3.5 (D2), §3.1 (pre-captured skuMap).

### F4 — Sockets are emitted before commit / inside transactions **[FIX]**

- `CH.updateChannelWithSocketTrigger(req, id, fields, transaction)` (CH:931-954) updates in the tx and emits immediately — used with a tx by `addSellableKeysToTaskQueue` (BS:4894-4899).
- `TQS.addKeysToRawDataInnerObjectSetWithSocketTrigger(…, transaction)` (TQS:713), `updateRawDataInnerObjectWithSocketTrigger(…, transaction)` (TQS:620), `incrementRawDataInnerObjectValueWithSocketTrigger` (TQS:654) emit immediately even when a tx is passed.
- `masterChannelSocketTrigger(req, transaction)` (CH:956) same.
- BS `recalculateReadinessAfterProductMediaRecordsBestEffort(…, null, …)` (BS:2305) passes `null` although `product.service.ts:698-715` already supports `afterCommit`.

`transaction.afterCommit` is not used anywhere in BS. Fix: §3.6.

### F5 — Q2: Redis/DB coupling and post-commit ordering **[FIX]**

- `processVariantPrimaryAttributes` writes `variantId`s into `AM:child-*` at Q2:1977-1991 **while the tx is open**. On rollback Redis keeps variant ids that don't exist; Q3/Q4 then read them via `getSellableEntitiesByCP`.
- `updateProductIdInRedis` (Q2:765-795) runs after commit and **throws** on a null read; the catch then rolls back a committed tx (no-op), counts the batch as non-imported (`importFailedCount` grows), adds FAILED keys and CATEGRA errors — for products that exist. It also rewrites the whole CP list (`setKey(cpKey, parentArray)`), racing with `fetchParentSKUsBatch`'s per-item writes.
- `createMainImageAttachmentError` (Q2:853-870) is called without the tx → survives rollback, duplicated on re-run.
- Ordering the user requires (loader before `productSuccess`) is only true by accident: loader is inside `Promise.allSettled` (Q2:658-677), `productSuccess` after; a slow socket relay can still deliver `PRODUCT_CREATED` before `SYNC_LOADING`.
- Non-imported SKUs: success path adds only imported SKUs to SYNCED (`includeNonImported` default false) while the failure path adds all; `convertSKUMapToArray(skuMap)` (Q2:753) creates errors for all SKUs while Q3-Q5 use `skipUnimported = true`. Intentional (a Q2 failure means the SKU was never created), keep — but make it explicit in the lifecycle options instead of implicit defaults.

### F6 — Q3: media commit and baseline clear are two transactions **[FIX]**

`attachAmazonImages` commits its own tx (BS:2309-2311) and only afterwards Q3 opens a second tx for `clearSyncBaselines` (Q3:154-166). A failure in the second tx marks the batch FAILED while media linkers are already persisted; R2 objects from the first tx are never cleaned. `attachAmazonImages` already accepts `externalTransaction` — Q3 doesn't pass one. Readiness recalculation (BS:2305) runs inside the tx instead of after commit.

### F7 — Q4: commit errors swallowed; `finally` throws **[FIX]**

`safeCommitTransaction` (Q4:319) logs and continues on commit failure → SYNCED keys, tracker FINISHED and readiness are written for data that was **not** persisted. The `finally` block calls `setProductChannelMarketplaceLoading(false)` which rethrows (BS:1855) → the whole processor fails *after* the batch was tracked, without advancing the index (see F1). `calculateImportedProductsReadiness` (Q4:328) is not awaited and not caught → unhandled rejection. `getProductTypeDataForMarketplaces`/`fetchMissingProductTypeSchemas` use their own transactions on purpose (Amazon API calls in between) — keep, but they must run **before** the batch tx is opened (they do today).

### F8 — Q5: loaders not stopped on failure, unawaited stop

`stopProductTranslationInProgress` (Q5:292) is not awaited (its socket may arrive after the SYNCED socket). On batch failure it's not called at all, so `translationInProgress` stays `true` until `processAmazonSyncImportComplete` stops all loaders at the very end. **[ASK-1]** whether to stop the translation loader per failed batch (proposed: yes, in the failure tx).

### F9 — Inconsistent cleanup order and options across queues

See §1.2 table: the order of tracker/keys/sockets differs per queue; `triggerChannelSocket` is true in Q2/Q4 and false in Q3/Q5 (channel `syncedCount` etc. change in every queue → the UI channel card lags behind in Q3/Q5); `tryCatchWrapper` wraps whole sequences so the first failing step silently skips the rest (e.g. `addTrackingProductsAtomic` throwing skips FAILED keys and errors in Q3/Q5 failure paths).

### F10 — `addTrackingProductsAtomic` is not robust

BS:1640-1680: WATCH on the *whole* task key (invalidated by any concurrent path write from the sibling queue), only 3 attempts, no backoff, and `catch → break` swallows errors — under Q3/Q4 contention tracker updates are silently lost, and `processAmazonSyncImportComplete` then computes `failedProducts`/`succeedProducts` from an incomplete tracker.

### F11 — Processor-level inconsistencies

- `completeTaskStep` (TQS:1009-1029) swallows errors and returns `{ updated: false }` → `completeTaskQueueJob` silently skips the Redis status update, `isSyncing=false` and deferred-task start.
- Q2 (Q2:271-318) and Q4 (Q4:456-466) enqueue the next jobs without `await`; Q2 updates the ORDERS status without `await`. A worker crash between `completeTaskQueueJob` and `addJob` leaves the task PROCESSING with no job (orphan → auto-retry path). Q1 (Q1:474-485) passes the **full** `req` and the access token into job data (others pass `sanitizedReq`).
- `QS.processAmazonSyncImportComplete` runs `stopProductsLoading` without `await` (QS:970-983) and swallows everything.
- `subtractUncreatedSellableCount` (BS:4915) is inside the batch tx (correct) but `addSellableKeysToTaskQueue` recomputes channel counts in a separate tx started *after* the batch tx — two round trips and a window where counts are inconsistent.

### F12 — Minor

- `AM:tracking-*` is set to `{}` in Q1:208 and never read; only deleted by pattern. Remove the write (keep the delete pattern harmless).
- `fetchParentSKUsBatch` sets `currentQueueStatus` on each CP item via `setJsonArrayRange` — Q3 and Q4 overwrite each other's value; informational only, keep but document.
- `addTrackingProductsAtomic` throws `TASK_DATA_WAS_REMOVED` before its loop (BS:1636-1638) — that one *does* propagate out of `tryCatchWrapper`'s inner sequence and skips subsequent steps (F9).

---

## 3. Target design

### 3.1 One batch lifecycle for Q2–Q5 (`runCPBatch`)

New method on **BS** (all queue services already inject `AmazonProductBSService`):

```ts
interface RunCPBatchOptions<TCtx> {
	req: AuthenticatedRequest;
	channel: Channel;
	metaData: AmazonBackwardSyncJobMetaData;
	queueName: string;                      // GlobalEnums.QueueNames.AMAZON_SYNC_*
	batch: { batchCPProducts: CPListItem[]; batchIndex: number; batchSize: number };
	failure: {                              // queue-specific error descriptor
		errorKeyword: string;                 // e.g. PRODUCT_IMPORT_FROM_AMAZON_ERROR.code
		errorMessage: string;
		trackNonImportedSkus: boolean;        // Q2: true (FAILED keys + importFailedCount + errors include non-imported); Q3-5: false
	};
	channelSocket: boolean;                 // emit channel sellable counts after commit (see F9 – proposed true for all)
	/** Non-transactional preparation (Amazon API, R2 uploads, loaders that must be ON before work). Errors → batch failure. */
	prepare?: (ctx: BatchContext) => Promise<TCtx>;
	/** All DB work. Must use ctx.transaction for every write. Errors → batch failure (rollback). */
	work: (ctx: BatchContext, prepared: TCtx) => Promise<BatchWorkResult>;
	/** Extra tx-bound success writes that must be visible atomically with the batch (e.g. loaders). Runs before the tracking tail. */
	onSuccessInTransaction?: (ctx: BatchContext, result: BatchWorkResult) => Promise<void>;
	/** Post-commit essential Redis writes (Q2: productIds + variantIds). Retried; failure → transient error (job retried, batch re-run is idempotent). */
	onCommittedEssential?: (ctx: BatchContext, result: BatchWorkResult) => Promise<void>;
	/** Post-commit best-effort side effects in declared order (readiness, offers refresh). Each wrapped individually. */
	onCommittedBestEffort?: Array<{ name: string; run: (ctx: BatchContext, result: BatchWorkResult) => Promise<void> }>;
	/** Extra writes inside the failure transaction (e.g. loaders OFF). */
	onFailureInTransaction?: (ctx: BatchContext, error: unknown) => Promise<void>;
	/** Cleanup of non-transactional resources created in prepare/work (R2 objects). Best effort. */
	onFailureCleanup?: (ctx: BatchContext, prepared: TCtx | undefined, error: unknown) => Promise<void>;
}

interface BatchContext {
	req; channel; metaData; queueName; taskQueueId; batchIndex; batchSize; batchCPProducts;
	skuMap: SKUMap;                          // captured ONCE before prepare (strict Redis read)
	imported: CategraTrackingRecord[];       // partitionProductsByImport(skuMap)
	nonImported: CategraTrackingRecord[];
	transaction: Transaction;                // set for work/onSuccessInTransaction; a NEW tx for onFailureInTransaction
	afterCommit: (name: string, fn: () => Promise<void>) => void;   // ordered, best-effort, registered on the current tx
}

interface BatchWorkResult { productIds: number[]; newProductIds?: number[]; [k: string]: unknown }
```

Execution order (this is the contract every queue follows; nothing else may reorder it):

```
0. ctx.skuMap = getSellableEntitiesByCP(strict)            ← Redis read; missing → UnrecoverableError, down → RedisUnavailableError (rethrown, no tracking)
1. if isBatchAlreadySynced(queueName, ctx) → log SKIPPED_SYNCED_BATCH, goto 9                     (D1)
2. prepared = prepare?(ctx)                                 ← outside tx (Amazon API, R2, loaders ON)
3. tx = sequelize.transaction(); ctx.transaction = tx
4. result = work(ctx, prepared)
5. onSuccessInTransaction?(ctx, result)                     ← e.g. Q2 loaders ON, Q4/Q5 loaders OFF (row updates only)
6. tracking tail (same tx, row lock on tbl_task_queues held for ms):
   6a. moved = TQS.moveKeysBetweenRawDataInnerObjectSets(taskQueueId, queueName, SYNCED_KEYS_KEY, FAILED_KEYS_KEY, importedSkus, tx)
   6b. if moved.length: ChannelProductErrorService.delete({ taskQueueId, type: CATEGRA, sku IN moved, metaData->>'currentQueue' = queueName }, tx)
       if queueName === Q2 && moved∩nonImportedBefore: TQS.incrementRawDataRootKey(IMPORT_FAILED_COUNT_KEY, -n, tx)      (D1)
   6c. counts = ProductService.getChannelsSellableCount({ req, channelId, transaction: tx });  CH.updateById(channel, populateChannelSellableCounts(counts), tx)
   6d. if result.newProductIds?.length: CH.masterChannelSocketTrigger(req, tx)   → its socket deferred to afterCommit (§3.6)
   6e. register afterCommit emitters, in this fixed order:
       [task-queue socket] → [channel socket + channels list] → [master channel socket] → queue-specific (Q2: SYNC_LOADING then PRODUCT_CREATED; Q4/Q5: SYNC_LOADING/TRANSLATION_LOADING off)
7. tx.commit()                                              ← commit error → goes to 8 (real failure path, not swallowed)
   afterCommit callbacks run here (each individually try/caught, logged with consoleError + name)
8. post-commit:
   8a. onCommittedEssential?(ctx, result) with retry(3, 250ms·n) — failure → throw RedisUnavailableError (job delayed & retried; step 1 makes the re-run idempotent)
   8b. addTrackingProductsAtomic({ hasErrors:false, skuMap, trackingMetadata: finished-statuses-up-to-queueName })   (best effort, §3.4)
   8c. onCommittedBestEffort in order, each wrapped                                                      (best effort)
9. advance index: setBatchIndex(queueName, batchIndex + 1)  ← path-level JSON.SET, explicit value (idempotent); down → RedisUnavailableError
```

Failure path (any throw in 2–7):

```
F0. if isTransientInfraError(error) || error instanceof UnrecoverableError:
       safeRollbackTransaction(tx); onFailureCleanup?; rethrow                      ← NOT tracked as a batch failure; job retries / task fails
F1. safeRollbackTransaction(tx)
F2. onFailureCleanup?(ctx, prepared, error)                                          (best effort; R2 objects)
F3. failure tx (own try/catch → consoleError, never throws):
    ftx = sequelize.transaction()
    keys = failure.trackNonImportedSkus ? all skus : imported skus
    TQS.moveKeysBetweenRawDataInnerObjectSets(…, FAILED_KEYS_KEY, SYNCED_KEYS_KEY, keys, ftx)          ← exclusive (D1)
    if queueName === Q2: incrementRawDataRootKey(IMPORT_FAILED_COUNT_KEY, +nonImported.length − alreadyCountedFromPreviousFailedRun, ftx)
    createCategraErrors({ records: keys-as-records, error, errorKeyword, metaData:{currentQueue,batchIndex,batchSize,stack}, transaction: ftx })
    onFailureInTransaction?(ctx, error)                                              (loaders OFF for Q4/Q5)
    channel sellable counts (6c) if channelSocket
    afterCommit: task-queue socket → channel socket → queue-specific loader sockets
    ftx.commit()
F4. addTrackingProductsAtomic({ hasErrors: true, skuMap })                          (best effort)
F5. goto 9 (advance index — the batch outcome is recorded; never re-run it)
```

`isBatchAlreadySynced`: `SELECT raw_data->queueName->'syncedKeys'` for the task (one query, no lock) and check that every key of `importedSkus` (Q3–Q5) / all SKUs (Q2) is present. For Q2 the check additionally requires every `batchCPProducts[i].productId` to be set in the CP list (otherwise `onCommittedEssential` was lost and the batch must run again — Q2's work is idempotent after §3.7).

`tryCatchWrapper` policy (applies to the whole BS module): use it **only** for independent best-effort side effects, one call per side effect, always with an `onError` that calls `consoleError(<OPERATION_NAME>, { taskQueueId, queueName, batchIndex }, error)`. Never wrap a sequence, never wrap the failure transaction (it has its own try/catch that logs and continues), never wrap anything whose failure must fail the job (`work`, `commit`, `onCommittedEssential`, index advance). A small helper `runBestEffort(name, ctx, fn)` in BS enforces this shape.

Parallelism policy: `Promise.all` only for independent *reads* or independent status writes on different rows/keys (e.g. initial queue status updates in `importProducts`). Never for tracking writes, never `Promise.allSettled` to hide failures.

### 3.2 Redis task-meta: path-level, idempotent writes

New helpers in **BS** (replace every whole-document write on `AM:task-*`):

| New | Implementation | Replaces |
| --- | --- | --- |
| `setTaskQueueMeta(accountId, taskQueueId, queueName, partial)` | `JSON.SET key '$.<queueName>' '{}' NX` then one `JSON.SET key '$.<queueName>.<field>' value` per field (pipeline/MULTI) | `updateTaskInRedis(… { [queue]: {...spread} })` in Q2:155, Q3:51, Q4:141, Q5:80, Q6:329, Q6:230, QS:791 |
| `setTaskRootMeta(accountId, taskQueueId, partial)` | `JSON.SET key '$.<field>'` per root field (`status`, `currentQueue`, `updatedAt`, `sellableCount`, `nextToken`, `channelId`) | root-level spreads (Q1:193 init keeps `JSON.SET $` with `isInit=true`; QS:1337 failure write) |
| `setBatchIndex(accountId, taskQueueId, queueName, nextIndex)` | `JSON.SET key '$.<queueName>.currentBatchIndex' <nextIndex>` + `$.updatedAt`; **explicit value**, not `NUMINCRBY`, so a duplicated call is harmless | `updateBatchIndexInRedis` |
| `fetchParentSKUsBatch` | reads `$.<queueName>` via `getJsonPartial`, writes only `$.<queueName>.status/batchSize` and `$.updatedAt` | its whole-document write (BS:4612-4622) |
| `requireTaskMeta(accountId, taskQueueId)` | strict read (§3.5); `null` → `UnrecoverableError(TASK_DATA_WAS_REMOVED)` | every `if (!redisTaskKeyData) throw new Error(...)` |

Queue names contain `-` (e.g. `sync-amazon-process-media`), so paths must use bracket notation: `$["sync-amazon-process-media"].currentBatchIndex`. Add `jsonPath(...segments)` in CACHE that quotes/escapes segments. `updateTaskInRedis` stays only for the `isInit` case (Q1) and is otherwise deprecated (throw in dev if called with `isInit=false` after migration, then remove).

`updateProductIdInRedis` (Q2) switches from `setKey(cpKey, wholeList)` to `setJsonArrayRange(cpKey, batchIndex*batchSize, batchCPProducts)` (per-item paths; the CP list is only read whole by Q1/end-of-import, never concurrently rewritten).

Q6 gets the same treatment (`initializeTaskStatusInRedis`, `fetchBatchFromRedis`, `handleOrderSyncFailure` → `setTaskQueueMeta`/`setBatchIndex`). Q6 keeps its own per-order tx and does not use `runCPBatch` (no SKU keys there).

### 3.3 Exclusive `syncedKeys` / `failedKeys` in one statement (TQS)

New `TQS.moveKeysBetweenRawDataInnerObjectSets(req, taskQueueId, innerObjectKey, addKey, removeKey, keys, transaction): Promise<{ moved: string[] }>`:

1. `SELECT raw_data->:inner->:removeKey FROM tbl_task_queues WHERE id = :id FOR UPDATE` (inside the caller's tx) → `moved = keys ∩ existingRemoveSet` (needed for D1 error cleanup and `importFailedCount` correction).
2. One `UPDATE`:

```sql
UPDATE tbl_task_queues
SET raw_data = jsonb_set(
	jsonb_set(
		jsonb_set(COALESCE(raw_data::jsonb, '{}'::jsonb), ARRAY[:inner], COALESCE(raw_data::jsonb->:inner, '{}'::jsonb), true),
		ARRAY[:inner, :addKey],
		(SELECT COALESCE(jsonb_agg(DISTINCT e), '[]'::jsonb)
		   FROM jsonb_array_elements(
		     CASE WHEN jsonb_typeof(raw_data::jsonb->:inner->:addKey) = 'array' THEN raw_data::jsonb->:inner->:addKey ELSE '[]'::jsonb END
		     || :keys::jsonb) e),
		true),
	ARRAY[:inner, :removeKey],
	(SELECT COALESCE(jsonb_agg(e), '[]'::jsonb)
	   FROM jsonb_array_elements(
	     CASE WHEN jsonb_typeof(raw_data::jsonb->:inner->:removeKey) = 'array' THEN raw_data::jsonb->:inner->:removeKey ELSE '[]'::jsonb END) e
	  WHERE NOT (:keys::jsonb ? (e #>> '{}'))),
	true)::json
WHERE id = :taskQueueId;
```

3. Socket: `TASK_QUEUE_UPDATE` via the afterCommit-aware trigger (§3.6). `captureSystemSnapshot()` as today.

`addKeysToRawDataInnerObjectSetWithSocketTrigger` stays for Shopify; `addSellableKeysToTaskQueue` (BS:4855) is replaced by the tracking tail in §3.1 (its own-transaction version is deleted; Shopify BS has its own copy).

`incrementRawDataRootKeyWithSocketTrigger` gains a `transaction` parameter and stops swallowing errors when a tx is passed (inside a tx a swallowed failure would leave the tx aborted anyway).

### 3.4 Product tracker (`productTracker`) without lost updates

Replace the WATCH/MULTI in `addTrackingProductsAtomic` with a server-side Lua script (`redis.defineCommand('mergeAmazonProductTracker', …)`), executed once per batch:

- ARGV: JSON array of `{ sku, productId, variantId, productName, hasErrors, metadata }`, `forceUpdate` flag.
- For each SKU: `JSON.GET key '$.productTracker["<sku>"]'` → merge with today's rules (BS:1649-1659: `hasErrors` sticky unless `forceUpdate`, metadata deep-merged one level) → `JSON.SET key '$.productTracker["<sku>"]'`; `JSON.SET key '$.productTracker' '{}' NX` first.
- Atomic per call, no retries needed, no interference with sibling-queue path writes.

If Lua is not wanted (**[ASK-2]**), fallback: keep WATCH/MULTI but watch and rewrite only `$.productTracker`, 10 attempts with jittered backoff (25–250 ms), and **throw** on exhaustion (caller wraps as best-effort and logs `TRACKER_UPDATE_LOST` with the SKUs).

`processAmazonSyncImportComplete` (QS:1001-1033) reads the tracker unchanged.

### 3.5 Redis error classification (CACHE, PROC, BS)

- New `core/cache/redis-errors.ts`: `class RedisUnavailableError extends Error`, `isRedisConnectionError(error)` (ioredis `ReplyError` is *not* a connection error; `MaxRetriesPerRequestError`, `ECONNREFUSED`, `ETIMEDOUT`, `Connection is closed`, `ensureRedisReady` timeout are).
- **CACHE** strict variants that **throw** `RedisUnavailableError` on connection failure and return `null` only when the key/path is absent: `getKeyStrict`, `getJsonPartialStrict`, `getJsonArrayRangeStrict`, `setKeyStrict`, `setJsonPartialStrict`, `setJsonArrayRangeStrict`, `deleteKeyStrict`. Existing lenient methods are untouched (other modules keep their behaviour, D3).
- **BS/Q1–Q6** switch every task-critical Redis access to the strict variants; `requireTaskMeta` / `requireCPList` / `requireChildList` wrap "null → `UnrecoverableError(GlobalEnums.channelProductErrors.TASK_DATA_WAS_REMOVED.message)`".
- **PROC** `process()` catch: `if (isDbConnectionError(error) || error instanceof RedisUnavailableError)` → same delayed-retry branch (`__dbRetryCount`, `DELAY_JOB_ON_DB_CONNECTION_ERROR`, `DB_CONNECTION_MAX_RETRIES`). Shared for all processors (D3) — a Redis outage should never consume BullMQ attempts for any queue.
- `handleAmazonSyncImportFailure` (QS:1248) already handles `UnrecoverableError` immediately (`shouldHandleFailedJobImmediately`); `scheduleJobBasedOnProgress` (BS:1370-1444) already restarts from Q1 when `isPreviousSyncDataAvailable` is false — both unchanged.

### 3.6 Sockets after commit (UTILS, TQS, CH, BS)

- **UTILS** `emitAfterCommit(transaction: Transaction | null | undefined, name: string, emit: () => Promise<void> | void)`: if `transaction?.afterCommit` exists → register (wrapped in try/catch → `consoleError('SOCKET_EMIT_AFTER_COMMIT_FAILED', { name })`); otherwise run immediately (mirrors `product.service.ts:698-731`). Note: Sequelize runs `afterCommit` hooks after the COMMIT returns; a rollback drops them — exactly the wanted semantics.
- **TQS**: `updateByIdWithSocketTrigger`, `updateRawDataInnerObjectWithSocketTrigger`, `incrementRawDataInnerObjectValueWithSocketTrigger`, `addKeysToRawDataInnerObjectSetWithSocketTrigger`, new `moveKeysBetweenRawDataInnerObjectSets`, `incrementRawDataRootKeyWithSocketTrigger`: when `transaction` is passed, the `socketUpdateTrigger` call goes through `emitAfterCommit`. Callers without a tx: unchanged.
- **CH**: `updateChannelWithSocketTrigger(…, transaction)` and `masterChannelSocketTrigger(req, transaction)` emit via `emitAfterCommit`. Callers without a tx: unchanged.
- **BS** `setProductChannelMarketplaceLoading` gains `transaction?: Transaction` (passed to `findAll`/`update`/`toggleProductInitialImport`); its `emitProductBulkPartialUpdate(SYNC_LOADING)` goes through `emitAfterCommit`. Same for `start/stopProductTranslationInProgress` (`transaction?`; `TRANSLATION_LOADING` socket deferred). `productSuccess` is called by Q2 inside `ctx.afterCommit('PRODUCT_CREATED', …)` registered **after** the loader emitter (ordered registration = ordered emission).
- `recalculateReadinessAfterProductMediaRecordsBestEffort` (BS:2305) passes the real `transaction` so `product.service` schedules it after commit.

### 3.7 Q2 idempotency and Redis/DB decoupling

- `processVariantPrimaryAttributes` no longer writes `AM:child-*`; it returns `skuToVariantIdMap` in `BatchWorkResult.variantIdBySku`. `onCommittedEssential` for Q2 = `updateProductIdInRedis` (per-item `setJsonArrayRangeStrict`) + `updateChildVariantIdsInRedis(variantIdBySku)` (path-level `JSON.SET $[?(@.sku=="…")].variantId` per SKU, or read-filter-write of the affected entries only — never the whole list).
- `newProductsToCreate` resolution: before `createBulkPrimaryProduct`, look up existing products of the account by `winnerSku` (SKU primary attribute, `channelId: null`, base language, `variantId: null`) — the same lookup Q1 uses in `createCPList`/`attachVariantIds` — and move hits to `existingProducts`. Makes a re-run after a lost `onCommittedEssential` idempotent instead of creating duplicates.
- `createMainImageAttachmentError` / `createMissingProductTypeError` receive `ctx.transaction` so they roll back with the batch.
- `subtractUncreatedSellableCount` stays in `work` (before the tracking tail; it uses `ctx.skuMap`, no Redis re-read).
- The `UnrecoverableError` "catalog imported but no products created" at queue end stays.

### 3.8 Processor-level lifecycle (`startQueue` / `finishQueue`)

New in **BS**:

```ts
startQueue({ req, taskQueueId, queueName })      // requireTaskMeta → setTaskQueueMeta(status: PROCESSING) → setTotalSellableCountForTaskQueue (unchanged)
finishQueue({ req, channel, taskQueueId, metaData, queueName, next: Array<{ type, data }> })
   // 1. completeTaskQueueJob (QS)  — completeTaskStep must THROW instead of returning { updated:false } [FIX F11]
   // 2. await Promise.all(next.map(n => TQS.updateRawDataInnerObjectWithSocketTrigger(…, n.type, { status: QUEUED })))   (independent rows-fields → parallel is fine)
   // 3. for (n of next) await QS.addJob({ type: n.type, accountId, userId, data: { req: sanitizedReq, channel, taskQueueId, metaData, ...n.data } })
```

- Every queue's `processAmazonProductImportQX` becomes: `startQueue` → loop `{ fetchParentSKUsBatch; if empty break; await runCPBatch(...) }` → queue-specific end (Q2: `setImportedProductsConciliation` + no-imported check) → `finishQueue`.
- `sanitizedReq` built once by `buildSanitizedReq(req)` in BS; Q1's `addJob` (Q1:474-485) uses it and stops passing `accessToken`.
- `completeTaskQueueJob` (QS:739): `updateTaskInRedis` → `setTaskRootMeta({status}) + setTaskQueueMeta(queueName, {status: SUCCESS})` (path-level, F1).
- `processAmazonSyncImportComplete` (QS:937): `await` the `stopProductsLoading` calls (they already catch internally). `handleAmazonSyncImportFailure`: `await stopProductsLoading` and `createErrorsForRetryProducts` (they catch internally; awaiting only makes ordering deterministic before `startDeferredBSTasks`).

### 3.9 Redis-unavailable inside cleanup/failure handlers

- Everything the handlers need (`skuMap`, imported/non-imported partitions, SKU list, product ids) is in `BatchContext`, captured in step 0. Handlers touch Redis only in step 8b/F4 (tracker, best effort) and step 9 (index, essential).
- If step 9 throws `RedisUnavailableError` after a *successful* batch: the job is delayed and retried; on retry step 1 skips the batch (keys in `syncedKeys`, CP list has productIds) and advances the index. After a *failed* batch: same, keys in `failedKeys` → step 1 does **not** skip (only `syncedKeys` skip), the batch re-runs; if it succeeds now, D1 move+cleanup applies. That is the intended "last outcome wins" semantics.

---

## 4. Per-queue mapping onto `runCPBatch`

| | Q2 `AMAZON_SYNC_CREATE_BASE_PRODUCT` | Q3 `AMAZON_SYNC_PROCESS_MEDIA` | Q4 `AMAZON_SYNC_VARIANT_SCHEMA_HANDLER` | Q5 `AMAZON_SYNC_TRANSLATION` |
| --- | --- | --- | --- | --- |
| `prepare` | — | collect image payload (`collectAmazonImages`, `processAmazonVariantImages`) and **download/upload to R2** (split `attachAmazonImages` into `uploadAmazonImages` → `{ digitalAssetPayload, uploadedObjects }` and `persistAmazonImages(tx)`) | `getProductTypeDataForMarketplaces` + `fetchMissingProductTypeSchemas` (own txs, Amazon API) | `startProductTranslationInProgress(validProductIds)` (no tx, best effort as today) |
| `work` (tx) | today's Q2:341-646 minus the child-Redis write; returns `{ productIds, newProductIds, variantIdBySku, productIdMetadata }` | `deletePreviousChannelMedia(isMainImage:false)` → `persistAmazonImages(tx)` → `clearSyncBaselines(MEDIA)`; returns `{ productIds: validProductIds }` | Q4:270-310 with a **throwing** commit (done by the lifecycle) ; returns `{ productIds: validParentProductIds }` | Q5:139-290 per-product localisation (no tx today — **[ASK-3]** keep non-transactional: `localizeAttributeValuesForProduct`/`suggestAmazonGroupAttributeValuesForProduct` own their transactions and call the LLM; wrapping them in one batch tx would hold locks across LLM calls). The lifecycle still opens the tx for the tracking tail only. |
| `onSuccessInTransaction` | `setProductChannelMarketplaceLoading({ productIds, isLoading:true, transaction })` (row updates; socket deferred) | — | `setProductChannelMarketplaceLoading({ productIds: validParentProductIds, isLoading:false, transaction })` (moved out of `finally`) | `stopProductTranslationInProgress(validProductIds, transaction)` |
| afterCommit order | task-queue → channel → master (if new products) → `SYNC_LOADING` → `PRODUCT_CREATED` (`productSuccess`, only if `newProductIds.length`) | task-queue → channel | task-queue → channel → `SYNC_LOADING`(off) | task-queue → channel → `TRANSLATION_LOADING`(off) |
| `onCommittedEssential` | `updateProductIdInRedis` + `updateChildVariantIdsInRedis` | — | — | — |
| `onCommittedBestEffort` | tracker (built-in) | tracker; `recalculateReadinessAfterProductMediaRecords` is already deferred via `product.service` afterCommit | tracker → `refreshAllSellableEntities` → `calculateImportedProductsReadiness` (awaited, wrapped) | tracker |
| `failure` | `{ PRODUCT_IMPORT_FROM_AMAZON_ERROR, trackNonImportedSkus: true }` | `{ AMAZON_MEDIA_PROCESSING_ERROR, false }` | `{ VARIANT_PROCESSING_ERROR, false }` | `{ PRODUCT_TRANSLATION_ERROR, false }` |
| `onFailureInTransaction` | — (loaders were never turned on for this batch) | — | `setProductChannelMarketplaceLoading(false, ftx)` (today's `finally` behaviour) | `stopProductTranslationInProgress(validProductIds, ftx)` **[ASK-1]** |
| `onFailureCleanup` | — | delete `uploadedObjects` from R2 (today done only when `attachAmazonImages` itself throws) | — | — |
| `channelSocket` | true (today) | **true** (today false — [ASK-4]) | true (today) | **true** (today false — [ASK-4]) |
| Skip check | all SKUs ∈ synced **and** all `cp.productId` set | imported SKUs ∈ synced | imported SKUs ∈ synced | imported SKUs ∈ synced |

Tracking metadata for the tracker (8b) is derived from `queueName`: `FETCH_PRODUCTS` + `CREATE_BASE_PRODUCT` + queues up to and including `queueName` are set to `FINISHED` (same values as today's literals in Q2:698-705, Q3:189-199, Q4:356-366, Q5:315-328; Q5 marks `VARIANT_SCHEMA_HANDLER` too, as today).

Q1 changes are limited to: strict Redis accessors, `buildSanitizedReq` in its `addJob`, drop the dead `AM:tracking-*` write, `updateTaskInRedis(…, isInit=true)` stays for the initial document. Q1's own failure handling (marketplace fetch failures → task FAILED, Q1:361-431) is already explicit and stays.

Q6 changes: path-level meta writes (§3.2), strict reads, `handleOrderSyncFailure` keeps its `Promise.allSettled` (independent cleanup steps, correct usage).

---

## 5. File-by-file change list

### 5.1 `core/cache/redis-errors.ts` (new)
`RedisUnavailableError`, `isRedisConnectionError(error)`.

### 5.2 `core/cache/cacheBase.service.ts`
- `private async callStrict<T>(operation, context, fn)`: `ensureRedisReady` + run; on error → `logRedisOperationFailure` then `throw new RedisUnavailableError(...)` if `isRedisConnectionError`, else rethrow original (e.g. RedisJSON path error = programming error).
- `getKeyStrict`, `getJsonPartialStrict`, `getJsonArrayRangeStrict`, `setKeyStrict`, `setJsonPartialStrict`, `setJsonArrayRangeStrict`, `deleteKeyStrict`, `jsonPath(...segments)`.
- Lua command registration `mergeAmazonProductTracker` (or generic `mergeJsonObjectEntries(key, path, entriesJson, forceFlag)`), defined once in the constructor via `redis.defineCommand`.

### 5.3 `common/lib/utils.ts`
- `emitAfterCommit(transaction, name, emit)`.
- `isTransientInfraError(error) = isDbConnectionError(error) || error instanceof RedisUnavailableError`.
- `tryCatchWrapper`: unchanged signature; add JSDoc stating the policy (§3.1). Optionally `runBestEffort(name, context, fn)` here instead of BS.

### 5.4 `modules/app/sync/queue/base-queue.processor.ts`
- `process()` catch: transient branch uses `isTransientInfraError`.

### 5.5 `modules/app/sync/queue/task-queue/task-queue.service.ts`
- `moveKeysBetweenRawDataInnerObjectSets(...)` (§3.3).
- `incrementRawDataRootKeyWithSocketTrigger(..., transaction?)`: pass tx to `rawQuery`; rethrow when tx provided.
- All `*WithSocketTrigger` methods: `emitAfterCommit` when a tx is passed.
- `completeTaskStep`: throw instead of `{ updated: false }`; `completeTaskQueueJob` callers (QS) already `await` it.

### 5.6 `modules/app/catalog/channel/channel.service.ts`
- `updateChannelWithSocketTrigger` / `masterChannelSocketTrigger`: `emitAfterCommit`.

### 5.7 `modules/app/sync/queue/queue.service.ts`
- `completeTaskQueueJob`: path-level Redis writes; remove the `if (!updated) return` (completeTaskStep throws now).
- `processAmazonSyncImportComplete`: `await` loader stops; use `getKeyStrict`? — no: end-of-import cleanup must stay lenient (it runs after the job completed; a Redis blip must not fail a completed job). Keep lenient reads here, add a log line when the tracker is missing.
- `handleAmazonSyncImportFailure`: `await stopProductsLoading`, `await createErrorsForRetryProducts`; Redis failure write via `setTaskRootMeta`/`setTaskQueueMeta`.

### 5.8 `amazon-product-bs.service.ts`
- New: `runCPBatch`, `isBatchAlreadySynced`, `startQueue`, `finishQueue`, `buildSanitizedReq`, `setTaskQueueMeta`, `setTaskRootMeta`, `setBatchIndex`, `requireTaskMeta`, `requireCPList`, `requireChildList`, `runBestEffort`, `updateChildVariantIdsInRedis`, `uploadAmazonImages` + `persistAmazonImages` (split of `attachAmazonImages`; keep `attachAmazonImages` as a thin wrapper for the other callers: Q2 main-image helpers, `productSearchByAsin` if it uses it).
- Modified: `fetchParentSKUsBatch` (path-level + strict), `getSellableEntitiesByCP` (strict child read), `addTrackingProductsAtomic` (Lua), `setProductChannelMarketplaceLoading(+transaction)`, `start/stopProductTranslationInProgress(+transaction)`, `createCategraErrors` (unchanged signature, already takes tx), `subtractUncreatedSellableCount` (accept `skuMap` instead of re-reading), `updateChannelSellableCount` (strict reads), `completeTaskWithNoProducts` (path-level), `removeDataFromRedis` (strict delete, log), `isPreviousSyncDataAvailable` (unchanged), `failTask` (unchanged).
- Removed: `addSellableKeysToTaskQueue`, `updateBatchIndexInRedis`; `updateTaskInRedis` restricted to `isInit`.

### 5.9 `amazon-product-bsq1…6.service.ts`
As per §4. Q2 additionally: SKU re-resolution (§3.7), `processVariantPrimaryAttributes` returns the map, `updateProductIdInRedis` per-item.

### 5.10 `core/types/sync/amazonSync.interface.ts`
`RunCPBatchOptions`, `BatchContext`, `BatchWorkResult`, `BatchFailureDescriptor`, `StartQueue`, `FinishQueue`; `ToggleLoaderforProductChannelMarketplaces.transaction?`, `SubtractUncreatedSellableCount.skuMap`.

### 5.11 `core/constants/global-enums.ts`
New `DEBUG_OPERATIONS`: `BATCH_SKIPPED_ALREADY_SYNCED`, `BATCH_TRACKING_FAILURE_PATH_FAILED`, `BATCH_POST_COMMIT_STEP_FAILED`, `TRACKER_UPDATE_LOST`, `SOCKET_EMIT_AFTER_COMMIT_FAILED`, `REDIS_TASK_META_MISSING`.

No `db-migrations` change: `raw_data` stays `json`, no new columns.

---

## 6. Before / after sequences (Q2 batch, the example from the task)

Before (Q2:648-707):
```
commit → updateProductIdInRedis(throws) → allSettled[SYNCED keys + channel socket + master socket ‖ loaders ON + SYNC_LOADING socket] → productSuccess → tracker
```
After:
```
work(tx) → loaders ON rows (tx) → move SYNCED (tx) → stale errors delete (tx) → importFailedCount −moved (tx) → channel counts (tx) → master data (tx)
→ COMMIT
→ afterCommit: TASK_QUEUE_UPDATE → CHANNEL_UPDATE(+list) → MASTER_CHANNEL → SYNC_LOADING → PRODUCT_CREATED
→ productIds/variantIds → Redis (essential, retried) → tracker (best effort) → index = batchIndex+1
```
Failure after:
```
rollback → failure tx: move FAILED (all SKUs) + importFailedCount +nonImported + CATEGRA errors + channel counts → COMMIT → afterCommit: TASK_QUEUE_UPDATE → CHANNEL_UPDATE
→ tracker(hasErrors) (best effort) → index = batchIndex+1
```

---

## 7. Edge cases to keep intact

| Case | Behaviour after the change |
| --- | --- |
| Retry-products import (`metaData.retryProducts`, `productVariantsMap`) | `getSellableEntitiesByCP` and `setProductChannelMarketplaceLoading` keep their variant-selection logic; `finishQueue` passes `metaData` through unchanged; `stopChannelProcessing`'s `retryProducts` guard unchanged. |
| Deferred tasks (`toBeStarted`, `startDeferredBSTasks`) | Triggered from `completeTaskQueueJob` as today; `removeDataFromRedis` uses strict deletes but logs and continues on `RedisUnavailableError` (it's a cleanup). |
| Parallel Q3/Q4/Q6 | Only path-level writes on `AM:task-*`; `tbl_task_queues` row lock held only during the tracking tail; `toBeFailed` logic in `handleAmazonSyncImportFailure` unchanged. |
| Worker restart mid-batch (stalled job re-queued) | Batch tx rolled back by Postgres; no keys written (tail is in the tx); index unchanged → batch re-runs → correct. If the crash was between commit and step 9: step 1 skips it. |
| Redis flushed mid-run | First strict read → `UnrecoverableError` → task FAILED immediately; manual retry → `scheduleJobBasedOnProgress` restarts from Q1 (existing path). |
| Redis briefly down | `RedisUnavailableError` → job delayed 20 s, up to `DB_CONNECTION_MAX_RETRIES`; batch outcome never recorded twice. |
| Commit failure | Real failure path (no more `safeCommitTransaction` in Q4) → FAILED keys + errors; DB state consistent. |
| Same SKU under two different parents in the CP list | Unchanged from today (keys are per SKU; last processed batch wins). Documented, not fixed. |

---

## 8. Testing plan

Unit (`*.spec.ts` next to the files, Jest):
- `TQS.moveKeysBetweenRawDataInnerObjectSets`: raw SQL against a pg test DB (or `pg-mem`): union + removal, empty sets, non-array existing values, `moved` result.
- `CacheBaseService` strict variants: connection error → `RedisUnavailableError`; missing key → `null`; RedisJSON path error → rethrown as-is.
- `BS.runCPBatch` with mocked sequelize transaction (`afterCommit` queue): asserts the exact call order of §3.1 for success, failure, transient error (no tracking, rethrow), skip; asserts the failure tx never throws out; asserts `onCommittedEssential` retry then `RedisUnavailableError`.
- `BS.setBatchIndex`/`setTaskQueueMeta`: path strings with bracket notation for hyphenated queue names.
- `addTrackingProductsAtomic` Lua: sticky `hasErrors`, `forceUpdate`, metadata merge.

Integration / manual (staging, api-worker):
1. Full import of a channel with ≥ 30 parents (batch size 3) — verify `syncedKeys ∩ failedKeys = ∅` for every queue at the end, `total`/`syncedKeys.length` consistent, channel card counts update after every Q3/Q5 batch.
2. Kill api-worker in the middle of Q3 and Q4 running in parallel; restart → no batch processed twice (log `BATCH_SKIPPED_ALREADY_SYNCED` at most once per batch), indexes monotonic.
3. Inject a failure in Q4 `linkAttributesData` for one batch, then retry the task → keys move from `failedKeys` to `syncedKeys`, CATEGRA errors for those SKUs removed, `importFailedCount` unchanged (Q4 doesn't count).
4. Same for Q2 (`createBulkPrimaryProduct` throws once) → `importFailedCount` incremented then decremented on the successful re-run; no duplicate products (SKU re-resolution).
5. `redis-cli DEBUG SLEEP 5` during Q5 → job delayed, resumes, no duplicated batch; `redis-cli FLUSHDB` during Q3 → task FAILED with `TASK_DATA_WAS_REMOVED`, manual retry restarts at Q1.
6. Socket order on the webapp product list: `SYNC_LOADING` (loaders on) arrives before `PRODUCT_CREATED` for every Q2 batch.

Lint/format: `pnpm lint && pnpm format` in `api/` (JSDoc on all new functions).

---

## 9. Open points — all answered 2026-09-19: **go with the proposed defaults** (ASK-1…ASK-6)

| # | Question | Decision |
| --- | --- | --- |
| ASK-1 | Q5: stop the translation loader for a **failed** batch inside the failure transaction (today it stays on until the end-of-import cleanup)? | Yes — behaviour change but consistent with Q4 |
| ASK-2 | Tracker merge via a Lua script (`redis.defineCommand`) vs. WATCH/MULTI with backoff | Lua |
| ASK-3 | Q5 work stays non-transactional (LLM calls inside `localizeAttributeValuesForProduct` own their transactions); only the tracking tail is transactional | Yes |
| ASK-4 | Emit the channel sellable-count socket after every Q3/Q5 batch too (today only Q2/Q4) | Yes — one consistent lifecycle; cost is one aggregate query per batch |
| ASK-5 | Q4 `safeCommitTransaction` → throwing commit means a commit failure now marks the batch FAILED instead of SYNCED. Confirm this is the wanted (correct) behaviour | Yes |
| ASK-6 | `completeTaskStep` throwing instead of `{ updated:false }`: a DB error at step completion now fails the job (BullMQ retries it) instead of silently leaving the task inconsistent | Yes |

---

## 10. Implementation order (each step compiles and can be deployed on its own)

1. §5.1–5.4: Redis error classification + strict cache API + `emitAfterCommit` + processor transient branch. (No behaviour change until callers switch.)
2. §5.5–5.6: TQS `moveKeys…`, afterCommit-aware socket triggers, CH triggers, `completeTaskStep` throw (ASK-6).
3. §5.8 BS: path-level task-meta helpers, strict `fetchParentSKUsBatch`/`getSellableEntitiesByCP`, Lua tracker; migrate `completeTaskQueueJob`, Q6 and Q1 to the new helpers. **This step alone removes the reported double-key blocker (F1).**
4. §5.8 BS: `runCPBatch`, `startQueue`, `finishQueue`; migrate Q3 (smallest) → verify → Q5 → Q4 → Q2 (with §3.7).
5. Remove `addSellableKeysToTaskQueue`, `updateBatchIndexInRedis`, dead `AM:tracking-*` write; lint/format; specs.

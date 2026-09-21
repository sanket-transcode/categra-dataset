Already done (plan §10 steps 1–3, part of 4)

- §5.1–5.4 (Redis error classification, strict cache API, emitAfterCommit, transient branch in BaseQueueProcessor)
- §5.5–5.6 (moveKeysBetweenRawDataInnerObjectSets, afterCommit-aware socket triggers on TQS/CH, completeTaskStep throws — ASK-6)
- §5.8 step 3: path-level task-meta helpers, strict fetchParentSKUsBatch/getSellableEntitiesByCP, Lua tracker merge, completeTaskQueueJob/Q1/Q6 migrated off whole-doc JSON.SET $
- Q4 safeCommitTransaction → throwing commit (ASK-5)
- QS processAmazonSyncImportComplete/handleAmazonSyncImportFailure already await stopProductsLoading/createErrorsForRetryProducts — F11 item already satisfied, nothing to do there
- runCPBatch engine itself + helpers (isBatchAlreadySynced, runBestEffort, runBatchFailurePath, updateChildVariantIdsInRedis, buildSanitizedReq) — compiles, but nothing calls it yet
- attachAmazonImages split into uploadAmazonImages/persistAmazonImages (groundwork for Q3's F6 fix)

Remaining

1. startQueue/finishQueue helpers (§3.8) — not implemented. Needs completeTaskQueueJob's actual BackwardSyncJobData shape and each queue's "queue next job" payload (QueueService.addJob).
2. Wire Q3 onto runCPBatch (§4 column, smallest queue — plan's own recommended starting point): prepare = collect images + uploadAmazonImages; work = deletePreviousChannelMedia → persistAmazonImages(tx) → clearSyncBaselines(MEDIA); onFailureCleanup = delete uploaded R2 objects; channelSocket: true (ASK-4). Fixes F6.
3. Wire Q5 onto runCPBatch: prepare = startProductTranslationInProgress; work = non-transactional localization (tx only for tracking tail, ASK-3); onSuccessInTransaction/onFailureInTransaction = stopProductTranslationInProgress with transaction (ASK-1 — stop loader on failure too); channelSocket: true (ASK-4).
4. Wire Q4 onto runCPBatch: prepare = getProductTypeDataForMarketplaces + fetchMissingProductTypeSchemas (own txs, unchanged); work = existing Q4 logic (throwing commit already in place); onFailureInTransaction = setProductChannelMarketplaceLoading(false, ftx); onCommittedBestEffort = refreshAllSellableEntities → calculateImportedProductsReadiness (awaited, wrapped instead of fire-and-forget).
5. Wire Q2 onto runCPBatch (largest, most invasive — do last): processVariantPrimaryAttributes stops writing AM:child-* mid-transaction, returns skuToVariantIdMap; onCommittedEssential = per-item updateProductIdInRedis + updateChildVariantIdsInRedis; SKU re-resolution idempotency (§3.7 — look up existing products by winnerSku before createBulkPrimaryProduct); thread ctx.transaction into createMainImageAttachmentError/createMissingProductTypeError; onSuccessInTransaction = setProductChannelMarketplaceLoading(true, tx); register SYNC_LOADING then PRODUCT_CREATED as ordered afterCommit callbacks.
6. Q1 minor cleanup (F11): its addJob still passes the full req + raw accessToken instead of buildSanitizedReq(req) — not yet switched.
7. Dead-code removal (plan §10 step 5): once Q2–Q5 no longer call updateBatchIndexInRedis directly, delete that deprecated wrapper. Note: addSellableKeysToTaskQueue itself will not be deleted per plan §5.8 — I deliberately kept it as runCPBatch's tracking-tail implementation (documented deviation in memory) rather than re-deriving it inline in runCPBatch.
8. §8 Testing plan — still explicitly out of scope per your "skip §8, no spec files" instruction; revisit when you want it.

Suggested execution order

1. Q3 (smallest surface, validates the engine end-to-end against a real queue, fixes F6)
2. Q5 (next-smallest, exercises the non-transactional-work path and ASK-1/ASK-3 semantics)
3. Q4 (exercises prepare-with-own-txs and onCommittedBestEffort sequencing)
4. Q2 (largest — do last since it needs the pattern proven on 3 simpler queues first, and carries the extra §3.7 idempotency work)
5. startQueue/finishQueue extraction — fold in once all four queues' outer loops are visibly identical, so the shared shape is obvious rather than guessed
6. Q1 sanitized-req cleanup + dead-code removal (updateBatchIndexInRedis) + final lint/format pass

Want me to start on Q3 now?

✻ Cogitated for 56s · done 9:45 AM

❯ Yes

7. Dead-code removal (plan §10 step 5): once Q2–Q5 no longer call updateBatchIndexInRedis directly, delete that deprecated wrapper. Note: addSellableKeysToTaskQueue itself will not be deleted per plan §5.8 — I deliberately kept it as runCPBatch's tracking-tail implementation (documented deviation in memory) rather than re-deriving it inline in runCPBatch.
8. §8 Testing plan — still explicitly out of scope per your "skip §8, no spec files" instruction; revisit when you want it.

Suggested execution order

1. Q3 (smallest surface, validates the engine end-to-end against a real queue, fixes F6)
2. Q5 (next-smallest, exercises the non-transactional-work path and ASK-1/ASK-3 semantics)
3. Q4 (exercises prepare-with-own-txs and onCommittedBestEffort sequencing)
4. Q2 (largest — do last since it needs the pattern proven on 3 simpler queues first, and carries the extra §3.7 idempotency work)
5. startQueue/finishQueue extraction — fold in once all four queues' outer loops are visibly identical, so the shared shape is obvious rather than guessed
6. Q1 sanitized-req cleanup + dead-code removal (updateBatchIndexInRedis) + final lint/format pass
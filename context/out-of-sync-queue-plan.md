# Plan — Queue-based out-of-sync baseline calculation (api + webapp)

Status: **plan only, nothing implemented yet.** Written 2026-09-16.

## 0. Decisions already taken (with the user)

| # | Decision | Choice |
|---|---|---|
| D1 | Queue count | **One queue** `sync-baseline-out-of-sync`, `baselineType` discriminator in the payload, one processor dispatching to the six existing services |
| D2 | Transaction safety | **`afterCommit` only** — when the caller passes a `transaction`, the job is enqueued in `transaction.afterCommit(...)`; no fixed delay; callers without a transaction enqueue immediately |
| D3 | Dedup / ordering | **None.** Every trigger = one job. No shared job ids, no per-product lock. Each service keeps its own transaction (as today) |
| D4 | awaitCompletion | **Not supported.** All call sites become fire-and-forget; UI relies on the existing socket events |
| D5 | `req` inside the job | Minimal `{ user: { id, accountId } }` (same as readiness). Verified: `BaseService` only reads `req.user.id` / `req.user.accountId`; `logError` tolerates a bare req; the six services + `SyncBaselineService` only read `req.user.id` / `req.user.accountId` |
| D6 | Failure policy | **1 attempt, log only.** `removeOnComplete: true`, `removeOnFail: 200`. Final failure is logged at error level in `BaseQueueProcessor.handleJobFailed` (same branch style as `EMAIL_QUEUE`) |
| D7 | Webapp | Audit only; patch only if a flow would read stale sync status. Audit result in §8: **no webapp change required** |
| D8 | Naming | Queue `GlobalEnums.QueueNames.SYNC_BASELINE_OUT_OF_SYNC = 'sync-baseline-out-of-sync'`; trigger in `sync/baselines/sync-baseline/trigger/`; processor `sync/queue/processors/sync-baseline.processor.ts` |
| D9 | Subscription capability | None (`getRequiredCapabilityForJob` default `null`) — same as readiness |
| D10 | Concurrency | `QUEUE_CONCURRENCY[...SYNC_BASELINE_OUT_OF_SYNC] = 10` (readiness uses 20; baseline jobs are heavier per job: multi-table upserts + socket emit) |
| D11 | `queue_config` DB row | Not added (would need `db-migrations`, out of scope). `getOrCreateQueue` falls back to `DEFAULT_JOB_OPTIONS`; readiness runs the same way today |

---

## 1. Current state (what the code does today)

Six services under `api/apps/api-main/src/modules/app/sync/baselines/` each expose one inline entry point that is called directly from mutation code paths, on the api-main request thread:

| Baseline type | Service | Entry point | Payload today |
|---|---|---|---|
| `ATTRIBUTE` | `AttributeSyncBaselineService` | `processAttributeOutOfSync(req, changedAttributes: Record<pagId, Partial<ProductAttribute>[]>, oldAttributes: Partial<ProductAttribute>[])` | plain objects (built by callers) |
| `VARIANT_ATTRIBUTE` | `VariantAttributeSyncBaselineService` | `processVariantAttributeOutOfSync(req, { changedRecords, oldRecords }: VariantAttributeSyncOutOfSyncRecord[])` | plain |
| `VARIANT_STRUCTURE` | `VariantStructureBaselineService` | `processVariantStructureOutOfSync(req, { records: VariantStructureOutOfSyncRecord[] })` | plain |
| `PRODUCT_MEDIA` | `ProductMediaSyncBaselineService` | `processProductMediaOutOfSync({ req, productId, variantId, channelId, wasInherited, oldMediaRecords: Partial<ProductMediaLinker>[] })` | `oldMediaRecords` come from `getMediaLinkersWithInheritanceStatus` → **Sequelize instances** |
| `MISC` | `MiscSyncBaselineService` | `processMiscSyncOutOfSync(req, { records: MiscSyncBaselineOutOfSyncRecord[] })` | plain |
| `PRICE` | `PriceSyncBaselineService` | `processPriceOutOfSync(req, body, productChannelMarketplace?: ProductChannelMarketplace, productChannel?: ProductChannel)` | `body` plain; **PCM / PC are Sequelize instances** |

All six: open their own `sequelize.transaction()`, upsert/delete baseline rows, flip `syncStatus` on PC/VC/PCM/VM, call `SyncBaselineService.restoreInSyncForClearedScopes` / `getScopeStatusChanges` / `emitSyncStatusScopes`, and emit `*_SYNC_BASELINES_UPDATED` via `SocketService.emitToAgainstAccountId` (which already relays through the Redis microservice when running inside api-worker, so worker-side emission works with no change).

Pattern to mirror: `ReadinessTriggerService` (`catalog/products/readiness/trigger/readiness-trigger.service.ts`) → `QueueService.addJob({ type, accountId, userId, data })` → `ReadinessRecalculationProcessor extends BaseQueueProcessor` → `ReadinessExecutorService.run(...)`.

### 1.1 Call-site audit (45 sites)

Legend: **TX-open** = called while the caller's transaction is still open (needs `transaction` passed → `afterCommit`); **after-commit** = called after `transaction.commit()` or with no transaction (enqueue immediately); **awaited** = caller currently `await`s the result.

| File | Line | Type | Today | Migration |
|---|---|---|---|---|
| `catalog/products/lifecycle/update-details.service.ts` | 448 | MISC | after-commit | immediate |
| same | 870 | ATTRIBUTE | after-commit | immediate |
| same | 1096 | ATTRIBUTE | after-commit | immediate |
| same | 1443 | MISC | **TX-open** (commit at 1472) | pass `transaction` |
| same | 1926 | ATTRIBUTE | **TX-open** (commit at 1937, may be `externalTransaction`) | pass `transaction` (afterCommit fires when the owning tx commits, external or not) |
| `catalog/products/listings/product-channel.service.ts` | 2890 | ATTRIBUTE | after-commit, **awaited**, in try/catch | immediate, drop `await` + try/catch (trigger never throws) |
| same | 3163 | ATTRIBUTE | after-commit (UpdateMarketplaceDetail commits at 4808), **awaited** | immediate, drop `await` + try/catch |
| same | 3748 | MISC | **TX-open** (`handleAmazonProductChannelDetails(..., transaction?)`, committed by caller at 2886) | pass `transaction` (may be undefined → immediate) |
| same | 5119 | MISC | after-commit | immediate |
| `catalog/products/shopifyProduct/shopify-product.service.ts` | 2407 | ATTRIBUTE | no tx | immediate |
| `catalog/products/product.service.ts` | 8233 | PRODUCT_MEDIA | no tx | immediate |
| same | 8359, 8538, 8760, 9244, 17174 | PRODUCT_MEDIA | after-commit | immediate |
| same | 12514, 12524 | PRODUCT_MEDIA | no tx | immediate |
| same | 10521 | VARIANT_STRUCTURE | after-commit | immediate |
| same | 20099, 20167, 20281, 20395 | VARIANT_STRUCTURE | no tx | immediate |
| same | 21521, 21711, 21993 | MISC | after-commit | immediate |
| `catalog/products/variants/product-variants-flow.service.ts` | 3262 | PRODUCT_MEDIA | no tx | immediate |
| same | 4339, 4830 | VARIANT_ATTRIBUTE | after-commit | immediate |
| same | 6461, 6984 | VARIANT_STRUCTURE | after-commit | immediate |
| `catalog/products/pricing/product-pricing.service.ts` | 6448 | PRICE | **TX-open** (`updateChannelPrices(..., transaction, ...)`), **awaited** | pass `transaction`, drop `await` |
| same | 7664 | PRICE | **TX-open** (commit at 7667), **awaited** | pass `transaction`, drop `await` |
| same | 8278 | PRICE | **TX-open** (commit at 8281), **awaited** | pass `transaction`, drop `await` |
| same | 9190 | PRICE | **TX-open** (`transaction` in scope, see 9155), **awaited** | pass `transaction`, drop `await` |
| `platforms/amazon/channel-config/product-type/amazon-channel-product-type.service.ts` | 2142 | VARIANT_STRUCTURE | after-commit (2096) | immediate |

Price sites additionally must stop passing ORM instances: pass `productChannelMarketplaceId` / `productChannelId` (numbers) and let the worker re-fetch.

---

## 2. Target architecture

```
mutation code (api-main)
   └─ SyncBaselineTriggerService.trigger({ req, source, transaction?, baselineType, payload })
        ├─ validate + serialise payload (plain JSON, ids only)
        ├─ if transaction → transaction.afterCommit(enqueue) else enqueue()
        └─ QueueService.addJob({ type: SYNC_BASELINE_OUT_OF_SYNC, accountId, userId, data })
                 │  (Redis)
api-worker
   └─ SyncBaselineOutOfSyncProcessor extends BaseQueueProcessor  (@Processor(SYNC_BASELINE_OUT_OF_SYNC))
        └─ SyncBaselineOutOfSyncExecutorService.run(payload)
             └─ switch (payload.baselineType) → the six existing process*OutOfSync methods (unchanged bodies)
                  └─ SocketService.emitToAgainstAccountId → Redis relay → api-main gateway → webapp
```

---

## 3. New files

### 3.1 `api/apps/api-main/src/modules/app/sync/baselines/sync-baseline/trigger/sync-baseline-out-of-sync.types.ts`

```ts
export type SyncBaselineOutOfSyncSource =
	| 'PRODUCT_PRIMARY_ATTRIBUTES_UPDATED'
	| 'PRODUCT_GROUP_ATTRIBUTES_UPDATED'
	| 'PRODUCT_ESSENTIALS_UPDATED'
	| 'SHOPIFY_ATTRIBUTE_MAPPING_UPDATED'
	| 'AMAZON_CHANNEL_ATTRIBUTES_UPDATED'
	| 'AMAZON_MARKETPLACE_ATTRIBUTES_UPDATED'
	| 'SHOPIFY_PRODUCT_ATTRIBUTES_UPDATED'
	| 'PRODUCT_CHANNEL_CONFIG_UPDATED'
	| 'PRODUCT_MEDIA_UPDATED'
	| 'VARIANT_MEDIA_UPDATED'
	| 'VARIANT_ATTRIBUTES_UPDATED'
	| 'VARIANT_STRUCTURE_UPDATED'
	| 'VARIANT_LISTING_STATUS_UPDATED'
	| 'AMAZON_ATTRIBUTE_PATH_UPDATED'
	| 'CONDITION_TYPE_UPDATED'
	| 'SHIPPING_TEMPLATE_UPDATED'
	| 'AMAZON_CATEGORY_UPDATED'
	| 'CHANNEL_PRICE_UPDATED'
	| 'AMAZON_PRICE_UPDATED'
	| 'SHOPIFY_PRICE_UPDATED'
	| 'BUSINESS_PRICE_UPDATED';

// One member per baseline type. `payload` is the exact argument shape of the target service,
// minus `req`, with ORM instances replaced by ids / plain objects.
export type SyncBaselineOutOfSyncRequest = { req: AuthenticatedRequest; source: SyncBaselineOutOfSyncSource; transaction?: Transaction } & (
	| { baselineType: 'ATTRIBUTE'; payload: { changedAttributes: Record<number, Partial<ProductAttribute>[]>; oldAttributes: Partial<ProductAttribute>[] } }
	| { baselineType: 'VARIANT_ATTRIBUTE'; payload: { changedRecords: VariantAttributeSyncOutOfSyncRecord[]; oldRecords: VariantAttributeSyncOutOfSyncRecord[] } }
	| { baselineType: 'VARIANT_STRUCTURE'; payload: { records: VariantStructureOutOfSyncRecord[] } }
	| { baselineType: 'PRODUCT_MEDIA'; payload: Omit<ProcessProductMediaOutOfSync, 'req'> }
	| { baselineType: 'MISC'; payload: { records: MiscSyncBaselineOutOfSyncRecord[] } }
	| { baselineType: 'PRICE'; payload: { body: any; productChannelMarketplaceId?: number | null; productChannelId?: number | null } }
);

export type SyncBaselineOutOfSyncType = SyncBaselineOutOfSyncRequest['baselineType'];

/** What travels inside a `sync-baseline-out-of-sync` job (`job.data.data`). */
export interface SyncBaselineOutOfSyncJobPayload {
	req: { user: { id: number | null; accountId: number } };
	source: SyncBaselineOutOfSyncSource;
	baselineType: SyncBaselineOutOfSyncType;
	payload: SyncBaselineOutOfSyncRequest['payload'];
}
```

`SyncBaselineOutOfSyncType` values are also added to `GlobalEnums` as `SyncBaselineTypes` (`ATTRIBUTE`, `VARIANT_ATTRIBUTE`, `VARIANT_STRUCTURE`, `PRODUCT_MEDIA`, `MISC`, `PRICE`) so the processor / executor `switch` uses enum members, matching how the rest of the codebase switches on `GlobalEnums.*`.

### 3.2 `.../sync-baseline/trigger/sync-baseline-out-of-sync.serializer.ts`

Pure functions, JSDoc'd, unit-tested:

- `toPlainRecord(value)` — if `value instanceof Model` → `value.get({ plain: true })`; arrays mapped; plain objects returned as-is. Applied to `oldAttributes`, `changedAttributes[*]`, `oldMediaRecords`.
- `normalizeSyncBaselinePayload(request)` — per `baselineType`:
  - `ATTRIBUTE`: reject when `!Object.keys(changedAttributes).length` (`'no_changed_attributes'`).
  - `VARIANT_ATTRIBUTE`: reject when `!changedRecords.length`.
  - `VARIANT_STRUCTURE` / `MISC`: reject when `!records.length`.
  - `PRODUCT_MEDIA`: coerce `productId`, `variantId`, `channelId` with `Number(...) || null`; reject when `!productId`.
  - `PRICE`: reject when `!body?.productId || !body?.channelId`; coerce ids to `Number | null`.
  - Returns `{ payload } | { rejection: string }` (same shape as `normalizeReadinessScope`).

These early-return rules replicate the guard clauses that already sit at the top of each `process*OutOfSync`, so an empty request never becomes a job.

### 3.3 `.../sync-baseline/trigger/sync-baseline-trigger.service.ts`

```ts
@Injectable()
export class SyncBaselineTriggerService {
	private readonly logger = new Logger(SyncBaselineTriggerService.name);

	constructor(@Inject(forwardRef(() => QueueService)) private readonly _queueService: QueueService) {}

	/**
	 * Request an out-of-sync baseline calculation. Never throws: an invalid payload is logged and
	 * skipped, a queue failure is logged and swallowed, so callers are never blocked.
	 * When `transaction` is given the job is enqueued only after that transaction commits
	 * (a rollback means no job).
	 * @returns {Promise<{ jobId: string | null }>} null when skipped or deferred to afterCommit.
	 */
	async trigger(request: SyncBaselineOutOfSyncRequest): Promise<{ jobId: string | null }>
}
```

Behaviour, in order:
1. `accountId = Number(request.req?.user?.accountId)`; if invalid → `logger.warn({ event: 'sync_baseline_trigger_skipped', reason: 'no_account', source, baselineType })`, return `{ jobId: null }`.
2. `normalizeSyncBaselinePayload(request)`; on rejection → warn `sync_baseline_trigger_skipped` with reason, return.
3. Build `SyncBaselineOutOfSyncJobPayload` with minimal `req`.
4. `enqueue = () => this._queueService.addJob({ type: GlobalEnums.QueueNames.SYNC_BASELINE_OUT_OF_SYNC, accountId: String(accountId), userId, data: payload }, { attempts: 1, removeOnComplete: true, removeOnFail: 200 })` wrapped in try/catch → `logger.error({ event: 'sync_baseline_trigger_enqueue_failed', ... })`.
5. If `request.transaction` → `request.transaction.afterCommit(() => void enqueue())`, log `sync_baseline_trigger_deferred`, return `{ jobId: null }`. Else `const job = await enqueue()`, log `sync_baseline_triggered` with `jobId`, return `{ jobId: String(job.id) }`.

No `QueueEvents`, no `OnApplicationShutdown` (D4).

### 3.4 `.../sync-baseline/trigger/sync-baseline-trigger.module.ts`

```ts
@Module({
	imports: [forwardRef(() => QueueModule)],
	providers: [SyncBaselineTriggerService],
	exports: [SyncBaselineTriggerService],
})
export class SyncBaselineTriggerModule {}
```

Kept separate from `SyncBaselineModule` (like `ReadinessTriggerModule` vs `ReadinessModule`) so the many mutation modules that already inject the six baseline services can swap to this light module without dragging the baseline module graph into more cycles.

### 3.5 `.../sync-baseline/execution/sync-baseline-out-of-sync-executor.service.ts`

```ts
@Injectable()
export class SyncBaselineOutOfSyncExecutorService {
	constructor(
		@Inject(forwardRef(() => AttributeSyncBaselineService)) ...,
		@Inject(forwardRef(() => VariantAttributeSyncBaselineService)) ...,
		@Inject(forwardRef(() => VariantStructureBaselineService)) ...,
		@Inject(forwardRef(() => ProductMediaSyncBaselineService)) ...,
		@Inject(forwardRef(() => MiscSyncBaselineService)) ...,
		@Inject(forwardRef(() => PriceSyncBaselineService)) ...,
		@Inject(forwardRef(() => ProductChannelMarketplaceService)) ...,
		@Inject(forwardRef(() => ProductChannelService)) ...,
	) {}

	/** Dispatch one job payload to the matching baseline service. Throws on unknown type. */
	async run(payload: SyncBaselineOutOfSyncJobPayload): Promise<void>
}
```

`run`:
- `const req = payload.req as unknown as AuthenticatedRequest;`
- `switch (payload.baselineType)`:
  - `ATTRIBUTE` → `processAttributeOutOfSync(req, p.changedAttributes, p.oldAttributes)`
  - `VARIANT_ATTRIBUTE` → `processVariantAttributeOutOfSync(req, p)`
  - `VARIANT_STRUCTURE` → `processVariantStructureOutOfSync(req, p)`
  - `PRODUCT_MEDIA` → `processProductMediaOutOfSync({ req, ...p })`
  - `MISC` → `processMiscSyncOutOfSync(req, p)`
  - `PRICE` → re-fetch `productChannelMarketplace = p.productChannelMarketplaceId ? await _productChannelMarketplaceService.findOne(req, { id }) : null`, same for `productChannel`; if an id was given but the row is gone → log + return (entity deleted between enqueue and run, nothing to mark). Then `processPriceOutOfSync(req, p.body, pcm, pc)`.
  - `default` → `throw new UnrecoverableError('Unknown baselineType')`.

Lives in `SyncBaselineModule` (`providers` + `exports`) — that module already `forwardRef`-imports all six baseline modules and `ProductChannelModule` / `ProductChannelMarketplaceModule`.

### 3.6 `api/apps/api-main/src/modules/app/sync/queue/processors/sync-baseline.processor.ts`

```ts
@Processor(GlobalEnums.QueueNames.SYNC_BASELINE_OUT_OF_SYNC, {
	concurrency: getQueueConcurrency(GlobalEnums.QueueNames.SYNC_BASELINE_OUT_OF_SYNC),
	...WORKER_OPTIONS,
})
export class SyncBaselineOutOfSyncProcessor extends BaseQueueProcessor {
	constructor(private readonly syncBaselineOutOfSyncExecutorService: SyncBaselineOutOfSyncExecutorService) { super(); }

	protected async doProcess(job: Job): Promise<void> {
		const payload: SyncBaselineOutOfSyncJobPayload = job.data?.data || {};
		if (!payload?.baselineType || !payload?.req?.user?.accountId) {
			this.logger.warn(`[${job.queueName}] Sync baseline job ${job.id} carries no baselineType/account; skipping.`);
			return;
		}
		await this.syncBaselineOutOfSyncExecutorService.run(payload);
	}
}
```

Identical shape to `readiness.processor.ts`.

### 3.7 Specs (colocated `*.spec.ts`)

- `sync-baseline-out-of-sync.serializer.spec.ts` — ORM instance → plain; each rejection rule; id coercion.
- `sync-baseline-trigger.service.spec.ts` — no account → skipped; rejection → skipped and `addJob` not called; immediate enqueue calls `addJob` with `type`, minimal `req`, `attempts: 1`; with `transaction` → `addJob` not called until the `afterCommit` callback fires; `addJob` throwing is swallowed.
- `sync-baseline-out-of-sync-executor.service.spec.ts` — each `baselineType` routes to the right service with the right arguments; PRICE re-fetches PCM/PC by id and skips when missing; unknown type throws `UnrecoverableError`.
- `sync-baseline.processor.spec.ts` — payload guard + delegation (mirrors whatever `readiness.processor` has; if there is no readiness spec, keep this minimal).

---

## 4. Edits to existing files

| File | Change |
|---|---|
| `core/constants/global-enums.ts` | `QueueNames.SYNC_BASELINE_OUT_OF_SYNC = 'sync-baseline-out-of-sync'`; new `static readonly SyncBaselineTypes = { ATTRIBUTE, VARIANT_ATTRIBUTE, VARIANT_STRUCTURE, PRODUCT_MEDIA, MISC, PRICE }` |
| `core/constants/queue-concurrency.ts` | `[GlobalEnums.QueueNames.SYNC_BASELINE_OUT_OF_SYNC]: 10` |
| `sync/queue/sync-consumers.module.ts` | add to `SYNC_QUEUES`, add `SyncBaselineOutOfSyncProcessor` to `PROCESSORS`, add `SyncBaselineModule` to `imports` (exports the executor) |
| `sync/queue/base-queue.processor.ts` | `handleJobFailed`: new branch `else if (queueName === GlobalEnums.QueueNames.SYNC_BASELINE_OUT_OF_SYNC) { this.logger.error(\`Sync baseline job ${job.id} failed permanently \| baselineType=${job.data?.data?.baselineType ?? 'n/a'} \| ${err?.message}\`, getErrorStack(err)); }` — placed before the `else` "Unknown job type" fallback, same style as the `EMAIL_QUEUE` branch. (Readiness currently falls into the "Unknown job type" warn; we do it properly for this queue.) |
| `sync/queue/sync-job-metrics.service.ts` | check `channelFromQueueName`; if it throws / mis-buckets unknown names, add the new name to whatever "internal" bucket readiness lands in (verify during implementation, no behaviour change expected) |
| `sync/baselines/sync-baseline/sync-baseline.module.ts` | add `SyncBaselineOutOfSyncExecutorService` to `providers` + `exports` |
| `sync/sync.module.ts` | add `SyncBaselineTriggerModule` to the aggregator imports/exports (wherever `ReadinessTriggerModule` is exposed for the readiness case — verify exact spot) |
| The 9 caller modules (`update-details`, `product-channel`, `shopify-product`, `product`, `product-variants-flow`, `product-pricing`, `amazon-channel-product-type` and their `*.module.ts`) | import `SyncBaselineTriggerModule`; constructor-inject `SyncBaselineTriggerService`; **remove** the injection of the concrete baseline service *only if* that service is no longer used for anything else in the file (e.g. `product-pricing.service.ts` still calls `_priceSyncBaselineService.deleteBaselineRecord` at 7654 → keep it) |
| 45 call sites (§1.1) | replace `this._xSyncBaselineService.processXOutOfSync(...)` with `void this._syncBaselineTriggerService.trigger({ req, source, transaction?, baselineType, payload })`; drop `await` and the surrounding `try/catch` where it only guarded this call |

The six `process*OutOfSync` method bodies are **not changed**; they stay public and become worker-only entry points (called by the executor).

### 4.1 Per-site payload mapping

- ATTRIBUTE: `{ baselineType: 'ATTRIBUTE', payload: { changedAttributes, oldAttributes } }`
- VARIANT_ATTRIBUTE: `{ payload: { changedRecords, oldRecords } }` (already the shape passed today)
- VARIANT_STRUCTURE / MISC: `{ payload: { records } }`
- PRODUCT_MEDIA: `{ payload: { productId, variantId, channelId, wasInherited, oldMediaRecords } }` — serializer flattens `oldMediaRecords` instances
- PRICE: `{ payload: { body: baselinePayload, productChannelMarketplaceId: pcm?.id ?? null, productChannelId: pc?.id ?? null } }`
  - 6448: `productChannelMarketplaceId: ch.isAmazon ? productChannelMarketplace?.id : null, productChannelId: !ch.isAmazon ? productChannelMarketplace?.id : null` — **note**: today line 6433 assigns `productChannel = productChannelMarketplace` for non-Amazon channels (the variable holds a PC for Shopify). Preserve that exact semantics by ids; do not "fix" it.

`source` per site is chosen from the union in §3.1 (one per mutation method; listed in the table's method names).

---

## 5. Sequencing (implementation order)

1. Enums + concurrency constant.
2. Types + serializer (+ spec).
3. Trigger service + module (+ spec).
4. Executor service in `SyncBaselineModule` (+ spec).
5. Processor + `SyncConsumersModule` + `BaseQueueProcessor` failure branch (+ spec).
6. `pnpm build:all` — confirm both apps compile with the queue wired but zero callers migrated (worker registers the processor, nothing enqueues).
7. Migrate call sites file by file, in this order (smallest blast radius first): `amazon-channel-product-type` → `shopify-product` → `update-details` → `product-channel` → `product-variants-flow` → `product.service` → `product-pricing`. After each file: `pnpm lint && pnpm format && pnpm build:main`.
8. Remove now-unused baseline-service injections / module imports from callers (only where truly unused; `product-pricing` keeps `PriceSyncBaselineService`).
9. `pnpm test` (api), `pnpm build:all`.
10. Manual verification (§7).

---

## 6. Failure / edge-case behaviour

| Case | Behaviour |
|---|---|
| Redis down when triggering | `addJob` throws → logged `sync_baseline_trigger_enqueue_failed`, request continues (same as readiness) |
| Caller's transaction rolls back | `afterCommit` never fires → no job (correct: nothing changed) |
| Job fails (DB error not connection-related) | 1 attempt → `handleJobFailed` logs; job kept in `failed` (up to 200) for inspection; **no retry** (D6) |
| DB connection error | `BaseQueueProcessor` already holds the job with `moveToDelayed` up to `DB_CONNECTION_MAX_RETRIES` — unchanged |
| Worker restart mid-job | `WORKER_OPTIONS.maxStalledCount` re-queues it; the services' own transaction guarantees atomicity |
| Two changes to the same product in quick succession | Two jobs, may run concurrently (D3). Same race exists today between two HTTP requests; each service compares against DB state inside its own transaction |
| Entity deleted between enqueue and run (PRICE) | executor logs + returns |
| Unknown `baselineType` | `UnrecoverableError` → immediate failure, logged |

---

## 7. Verification checklist

- Start `pnpm start:main:dev` + `pnpm start:worker:dev`; worker log shows `Creating new queue for: sync-baseline-out-of-sync` only if a `queue_config` row exists (it will not) — instead confirm the processor registered via a first job.
- Edit a product primary attribute on a published Amazon product → api-main log `sync_baseline_triggered`, worker log `Job … started | queue=sync-baseline-out-of-sync`, `tbl_attribute_sync_baselines` row appears, PCM `sync_status` flips to `OUT_OF_SYNC`, webapp details tab shows the baseline badge without reload (socket).
- Repeat once per type: variant attribute value (VARIANT_ATTRIBUTE), variant channel status toggle (VARIANT_STRUCTURE), upload media (PRODUCT_MEDIA), change condition type (MISC), change Amazon marketplace price with autoPublish off (PRICE).
- Price with autoPublish off inside `updatePricingForAmazonMarketplace`: confirm the job is enqueued **after** the HTTP handler commits (worker log timestamp > commit).
- Kill the worker, make a change, restart → job processed on boot.
- `pnpm lint`, `pnpm format`, `pnpm test`, `pnpm build:all` all clean.

---

## 8. Webapp audit result (D7)

Files inspected: `_components/socket/socket.ts`, `socket-client.tsx:190`, `products/[product]/_components/details/details-tab.tsx:1060-1235`, `channels/channel-details/nonMasterChannels/nonMasterChannels.tsx:430-480`, `hooks/useVariantBaselines.tsx`, `hooks/useVariantSyncStatusSocket.tsx`.

- Every one of the six `*_SYNC_BASELINES_UPDATED` events plus `SYNC_STATUS_SCOPES_UPDATED` already has a handler that **patches local state in place** (upsert/delete by identity, sync status by scope). The handlers do not depend on the HTTP response of the mutating call.
- The mutating endpoints (`updateProductDetails`, `updateProductChannelsData`, pricing updates, media upload, variant updates…) return `formatResponse(success, null)` — they never returned baseline data or sync status in the response body, so nothing in the webapp read baseline results from the response.
- Any post-save refetch (e.g. product channel list) that happens before the worker finishes will show the pre-change status for up to a few hundred ms and then be corrected by the socket patch — the same visual sequence that already happens today for the fire-and-forget sites (39 of 45).

**Conclusion: no webapp change required.** No file in `webapp/` is touched by this task.

---

## 9. Out of scope / not done

- No `db-migrations` change (D11).
- No dedup / coalescing / ordering (D3) — can be added later on top of the single queue via `jobId` if needed.
- No `awaitCompletion` (D4).
- No retries (D6).
- The six service bodies are untouched; refactoring their duplicated PC/VC/PCM/VM marking logic is a separate task.

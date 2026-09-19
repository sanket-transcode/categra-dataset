# Plan — Blocking scope lock for MEDIA out-of-sync baseline processing (api only)

Status: **plan only, nothing implemented yet.** Written 2026-09-17.

## 0. Problem

`updateImageSequenceScope` (`product.service.ts:9164`, HTTP path, api-main) reorders product media
and then calls `SyncBaselineTriggerService.trigger(...)` with `baselineType: MEDIA`, which enqueues a
`SYNC_BASELINE_OUT_OF_SYNC` job (see `categra-dataset/context/out-of-sync-queue-plan.md`, decision D3:
*"No dedup / per-product lock"*). That queue runs at **concurrency 20**
(`queue-concurrency.ts:38`), and `SyncBaselineOutOfSyncExecutorService.run()` dispatches `MEDIA` jobs to
`ProductMediaSyncBaselineService.processProductMediaOutOfSync(...)`.

Two jobs for the same `(accountId, productId, variantId, channelId)` scope can run on two different
worker slots concurrently (e.g. two rapid reorders, or a reorder plus another media mutation both
enqueuing a MEDIA job). Each reads/writes `tbl_product_media_linkers` + baseline rows for that scope
independently — a classic read-modify-write race that can leave sync-status/baseline rows
inconsistent with the last-committed media order.

This plan adds a **blocking, wait-till-free Redis lock keyed by scope**, applied narrowly to (a) the
`updateImageSequenceScope` HTTP handler and (b) the MEDIA branch of baseline job processing — nothing
else.

---

## 1. Decisions already taken (with the user)

| # | Decision | Choice |
|---|---|---|
| D1 | Wait mechanism | **Polling** with fixed interval (existing `acquireLock`/`releaseLock` SET-NX-EX primitive in `CacheBaseService`), not a Redis blocking-list/pub-sub scheme |
| D2 | Worker slot during wait | **Occupied.** A waiting MEDIA job holds its concurrency slot for the whole wait (matches "waits ... rather than returning"); acceptable since MEDIA is one of 6 baseline types sharing the queue's 20 slots |
| D3 | Function-level vs processor-level lock scope | **Distinct key namespaces.** The HTTP-path lock and the worker-path lock never contend with each other — each layer only serializes against itself. (User: *"same scope processing at function level shouldn't run [concurrently with itself] and similarly [for] the worker level, but the function and worker will not overlap anyway"*) |
| D4 | Scope key tenancy | **Include `accountId`.** Defensive even though `tbl_products.id` looks globally unique today |
| D5 | Scope key format | `{accountId}::{productId}::{variantId ?? 'null'}::{channelId ?? 'null'}` — literal string `'null'` for the optional segments (not JS `null`/`undefined`), so the key is stable and diffable in Redis |
| D6 | Where the processor-level lock lives | Inside `SyncBaselineOutOfSyncExecutorService.run()`'s `MEDIA` case (the "processor function" that actually does MEDIA work), **not** in the generic `SyncBaselineOutOfSyncProcessor.doProcess`, so the other 5 baseline types are untouched and the lock only wraps `processProductMediaOutOfSync(...)` |
| D7 | Lock TTL | **30s** safety-net TTL per acquisition (well under BullMQ's 120s job `lockDuration`, comfortably above expected DB-write + socket-emit time). The lock is always explicitly released in a `finally` on normal completion; TTL only matters if a process crashes mid-hold |
| D8 | Poll interval | **250ms base + 0–100ms jitter** between re-attempts, to avoid a thundering herd when several waiters unblock at once |
| D9 | Max wait | **None (unbounded polling)**, per the explicit instruction ("waits till the lock expires and stays there, rather than returning"). Redis TTL (D7) is the only ceiling — a crashed holder's lock self-expires in ≤30s, so no waiter can starve indefinitely |
| D10 | Failure handling | **Resolved: keep it uniform.** If Redis is down when trying to acquire, treat like the existing `acquireLock` — `CacheBaseService` already swallows and logs Redis errors and returns `null`; a `null` return from an *acquire attempt* (contention **or** Redis outage) just drives another poll iteration. No special-casing of Redis unavailability. |

---

## 2. New Redis lock primitive

`CacheBaseService.acquireLock`/`releaseLock` (`core/cache/cacheBase.service.ts:236-263`) already exist
but are **non-blocking** (single `SET NX EX` attempt, return `null` on contention — see the existing
call site `amazon-channel.service.ts:150`, which skips on contention). We add one new method next to
them, reusing the same `SET NX EX` + Lua-checked `DEL` release, instead of duplicating the primitive:

```ts
// core/cache/cacheBase.service.ts
/**
 * Acquire a lock, blocking (via polling) until it becomes available rather than
 * failing on first contention.
 * @param {string} key - lock key
 * @param {number} ttlSeconds - lock TTL (safety net if the holder crashes)
 * @param {number} pollIntervalMs - base delay between acquire attempts
 * @returns {Promise<string>} the lock token (always resolves; never gives up)
 */
async acquireLockBlocking(key: string, ttlSeconds = 30, pollIntervalMs = 250): Promise<string> {
	for (;;) {
		const token = await this.acquireLock(key, ttlSeconds);
		if (token) return token;
		const jitter = Math.floor(Math.random() * 100);
		await new Promise((resolve) => setTimeout(resolve, pollIntervalMs + jitter));
	}
}
```

`releaseLock` is reused as-is.

## 3. Scope key builder

New small pure helper (colocated with the MEDIA baseline code, imported by both call sites):

`api/apps/api-main/src/modules/app/sync/baselines/product-media-sync-baseline/media-scope-lock.util.ts`

```ts
/**
 * Build the Redis lock key for a media scope, namespaced per lock layer so the
 * HTTP-path lock and the worker-path lock never contend with each other.
 * @param {'http' | 'worker'} layer
 * @param {{ accountId: number; productId: number; variantId?: number | null; channelId?: number | null }} scope
 * @returns {string}
 */
export function buildMediaScopeLockKey(
	layer: 'http' | 'worker',
	scope: { accountId: number; productId: number; variantId?: number | null; channelId?: number | null },
): string {
	return `media-scope-lock:${layer}:${scope.accountId}::${scope.productId}::${scope.variantId ?? 'null'}::${scope.channelId ?? 'null'}`;
}
```

---

## 4. Call-site changes

### 4.1 `product.service.ts` — `updateImageSequenceScope` (function level, `layer: 'http'`)

Wrap the existing method body (currently starting with `const transaction = await this.sequelize.transaction()`
at line 9173) with acquire/release:

```ts
async updateImageSequenceScope(req, body): Promise<object> {
	const productId = Number(body.productId);
	const variantId = Number(body.variantId) || null;
	const channelId = Number(body.channelId) || null;
	const lockKey = buildMediaScopeLockKey('http', { accountId: req.user.accountId, productId, variantId, channelId });
	const lockToken = await this._cacheBaseService.acquireLockBlocking(lockKey);

	const transaction: Transaction = await this.sequelize.transaction();
	try {
		// ...unchanged body...
	} catch (error) {
		// ...unchanged...
	} finally {
		await this._cacheBaseService.releaseLock(lockKey, lockToken);
	}
}
```

Notes:
- `productId`/`variantId`/`channelId` parsing is hoisted above the transaction (it's pure `Number(...)`
  coercion of `body`, no DB access), so the lock key can be built before acquiring.
- `_cacheBaseService` needs to be injected into `ProductService` if not already present — verify during
  implementation (`CacheBaseService` is `@Global()` per `api/CLAUDE.md`, so it should be injectable
  without new module wiring).
- Release must happen in `finally` so a thrown error still frees the lock.

### 4.2 `sync-baseline-out-of-sync-executor.service.ts` — `MEDIA` case (processor level, `layer: 'worker'`)

```ts
case GlobalEnums.SYNC_BASELINE_GROUPS.MEDIA: {
	const { productId, variantId, channelId, wasInherited, oldMediaRecords } = payload.payload;
	const lockKey = buildMediaScopeLockKey('worker', { accountId: req.user.accountId, productId, variantId, channelId });
	const lockToken = await this._cacheBaseService.acquireLockBlocking(lockKey);
	try {
		await this._productMediaSyncBaselineService.processProductMediaOutOfSync({
			req, productId, variantId, channelId, wasInherited, oldMediaRecords,
		});
	} finally {
		await this._cacheBaseService.releaseLock(lockKey, lockToken);
	}
	return;
}
```

`SyncBaselineOutOfSyncExecutorService` needs `CacheBaseService` injected (it's `@Global()`, same as
above — no new module import expected, verify during implementation).

Only this `case` changes; the other five baseline-type branches are untouched, per D6.

---

## 5. Resolved: Redis unavailability during acquire

Confirmed with the user: keep it uniform (D10). No special-casing of Redis outages vs. contention.

---

## 6. Files touched (implementation phase, not yet done)

| File | Change |
|---|---|
| `core/cache/cacheBase.service.ts` | add `acquireLockBlocking` |
| `sync/baselines/product-media-sync-baseline/media-scope-lock.util.ts` | new — `buildMediaScopeLockKey` |
| `catalog/products/product.service.ts` | wrap `updateImageSequenceScope` body with acquire/release (`layer: 'http'`); inject `CacheBaseService` if missing |
| `sync/baselines/sync-baseline/execution/sync-baseline-out-of-sync-executor.service.ts` | wrap the `MEDIA` case with acquire/release (`layer: 'worker'`); inject `CacheBaseService` if missing |
| new spec files | `media-scope-lock.util.spec.ts` (key format incl. null segments); extend `cacheBase.service.spec.ts` if one exists (acquireLockBlocking retries then succeeds); extend the executor's spec for the MEDIA case (lock acquired before the service call, released after, even on throw) |

No entity/migration changes — this is pure Redis-key locking, no schema involved.

---

## 7. Edge cases

| Case | Behaviour |
|---|---|
| Two reorders on the same scope back-to-back (HTTP) | Second request blocks until the first's `finally` releases the lock, then proceeds against post-first-reorder state |
| Reorder (HTTP) racing a MEDIA job (worker) for the same scope | **Do not block each other** (D3/D6) — different key namespaces. Both can run "concurrently" by design; each is already transactionally consistent on its own tables, and the pre-existing (non-locked) race between HTTP mutation and its own triggered job already exists today for every baseline type, MEDIA included |
| Worker crashes mid-hold | Lock self-expires after 30s (D7); next waiter proceeds |
| Lock holder takes longer than 30s (slow DB) | Lock expires while still "logically" held → a waiter could acquire and run concurrently. Not fully closed by this plan; flag as a known limitation unless the user wants TTL-refresh (out of scope per "don't add too much complexity") |
| Other 5 baseline types (ATTRIBUTE, VARIANT_ATTRIBUTE, VARIANT_STRUCTURE, MISC, PRICE) | Untouched — no locking added |

---

## 8. Sequencing (once §5 is resolved)

1. `acquireLockBlocking` on `CacheBaseService` (+ spec).
2. `buildMediaScopeLockKey` util (+ spec).
3. Wrap executor's `MEDIA` case (+ spec update).
4. Wrap `updateImageSequenceScope` (+ spec update if one exists for this method).
5. `pnpm lint && pnpm format && pnpm test && pnpm build:all`.
6. Manual verification: fire two rapid reorder requests for the same scope, confirm the second's DB
   work starts only after the first's transaction + job trigger complete; confirm a MEDIA job for a
   scope currently locked by an HTTP request waits (log-based, via the poll retries) rather than
   erroring.

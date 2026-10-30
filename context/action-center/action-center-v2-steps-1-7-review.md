# Action Center V2 — review of steps 1–7

Scope: `action-center-v2-plan.html` §8 steps 1–7 (schema, CC detector refactor, detectors + evaluator, queue/trigger/sweep/seed, hook points, read API, mutation API + publish-start hook), reviewed in `api/` and `db-migrations/`. The webapp is not part of steps 1–7 and was not reviewed.

Method: static read of the code only. Nothing was run, no database was queried, and CC payload parity (step 2's check) was not executed. Where a finding depends on runtime behaviour I could not see, it says so.

Severity: **High** = data loss, crash or a wrong board in normal use. **Medium** = wrong in a realistic edge case, or a real plan deviation. **Low** = hardening, performance, or something to decide.

## Summary

| # | Sev | Flaw | Step |
|---|---|---|---|
| 1 | High | A failed signal query is read as "no issues", so the evaluator mass-clears tickets | 2/3 |
| 2 | High | The nightly sweep throws on `accountIds.map` (and staggers by the wrong list) | 4 |
| 3 | High | A failed or discarded publish leaves tickets stuck in Awaiting channel | 5/7 |
| 4 | High | The trigger drops pending work on job retry, on a same-bucket trigger and on empty product lists | 4 |
| 5 | Medium | The evaluator builds drafts outside the lock, so concurrent jobs can resurrect cleared tickets | 3 |
| 6 | Medium | "Still publishing" is computed twice with different rules | 3 |
| 7 | Medium | Amazon `MAIN_IMAGE` / `PRODUCT_TYPE_CHANGED` tickets have no marketplace, so the marketplace filter hides them | 2/6 |
| 8 | Medium | Marketplace filter shows every Amazon channel's channel-wide tickets | 6 |
| 9 | Medium | The publish hook moves tickets the publish does not fix (NOT_APPLICABLE, whole-product scope) | 7 |
| 10 | Medium | Mutations use stale reads: a primary action can undo a system move; assign writes events for cleared tickets | 7 |
| 11 | Medium | Awaiting channel flips back to In progress before the channel re-validates | 3/7 |
| 12 | Medium | CSV export: formula injection, truncated-looking success on error, no abort handling | 6 |
| 13 | Medium | `loadSignals` always runs every signal query, even for a Buy Box or channel-health trigger | 3/5 |
| 14 | Low | Smaller issues (see the list at the end) | — |

---

## 1. A failed signal query is read as "no issues", so the evaluator mass-clears tickets — High

**Where**
- `command-center-attention-signals.service.ts:127-143` — `safeRows` catches every error and returns `[]`.
- `command-center-attention-signals.service.ts:81-84` — `loadHealth().catch(...)` returns `{ channels: [], dataTrust: [] }`.
- `issue-ticket-evaluator.service.ts:112-160` — the evaluator takes those rows as the truth, builds drafts, and `applyDiff` deletes every stored ticket that has no draft.

**Why it is a flaw**
`safeRows` was written for the Command Center, where an empty card for one failed signal is acceptable. For tickets, "empty" means "the problem is gone". A timeout or deadlock on one query makes the evaluator clear that signal's tickets, write `CLEARED` events, and delete the rows with their status, assignee and snoozes.

**Example**
At 01:00 the sweep evaluates Channel 42. The listing-issues query hits a statement timeout, `safeRows` logs and returns `[]`, and the other queries succeed. The evaluator sees no listing issues, so every `AMAZON_LISTING_BLOCKER` ticket on Channel 42 is deleted. Priya was working 12 of them, and they were `IN_PROGRESS` and assigned to her. At 01:00 the next day the sweep recreates them as `REAPPEARED` with fresh `TODO` and unassigned (decision §5 #6), so her assignments and the history are gone.

The health path is worse. If `getHealthForAccount` throws, `connectionOpen` becomes false, the `CONNECTION` ticket is cleared, and the product tickets that should be frozen are evaluated as well.

**Direction**
The evaluator needs the failure, not the fallback. Give `loadSignals` a strict mode that rethrows, so BullMQ retries the job and the stored tickets are untouched. Keep `safeRows` as is for the Command Center only.

---

## 2. Nightly sweep throws on `accountIds.map` — High

**Where** `issue-ticket-sweep.service.ts:27,34-36`, called as `enqueueAll()` from `cron.service.ts:355`.

**Why it is a flaw**
`accountIds` is optional and the cron passes nothing, but line 34 calls `accountIds.map(...)`. That is a `TypeError` on every nightly run, which `sweepIssueTickets` logs and rethrows. Only the seed endpoint (which always passes `accountIds`) works. The sweep is the safety net for every missed trigger, so with it broken, any ticket whose trigger was lost (see #3 and #4) stays wrong indefinitely.

Even when `accountIds` is passed, the stagger is computed from the requested list. Accounts that have no live channel still take a slot, and `sweptAccountIds` (already computed on line 32) is the list that should be used. `delayMs` of `0` for the first account also matters for #4.

**Example**
Cron fires at 01:00 → `TypeError: Cannot read properties of undefined (reading 'map')` → zero channels queued → V2 is only ever updated by event triggers.

**Direction**
Build `delayByAccount` from `sweptAccountIds`.

---

## 3. A failed or discarded publish leaves tickets stuck in Awaiting channel — High

**Where** (hook runs first, the exit trigger only on the success path)
- `product-pricing.service.ts:3515-3541` — the hook runs before `updatePriceOnAmazon`; the trigger runs after. A throw goes to the `catch` at `3549` and returns an error, with no trigger.
- `amazon-product-fs.service.ts:3259-3266` (hook) and `3329-3347` (trigger only when the patch did not return an `Error`, and not at all on a throw).
- `shopify-product-fs.service.ts:566-605` — `if (!response.success) return …` and the `catch` both skip the trigger.
- `amazon-product-fsq3.service.ts:107-109` — `if (!redisFeedResultData) return;` exits without `triggerIssueTickets()`. When the Redis message keys have expired, `publishedProductIds` is empty, and `trigger.service.ts:45` (`if (request.productIds && !productIds) return`) silently drops the trigger too.
- `issue-ticket-evaluator.service.ts:420-425` — the only way out of `AWAITING_CHANNEL` needs `scope.sources.includes(fixingSource)`.

**Why it is a flaw**
The hook moves tickets to `AWAITING_CHANNEL` at publish start. The only system exit is an evaluation whose merged sources contain the matching publish source. Any path where the publish fails before reporting a result never produces that source. The nightly sweep cannot rescue it either: its source is `SWEEP`, which is not a fixing source.

**Example**
A user opens Pricing, fixes the price on "Linen Shirt" and clicks Publish. The Amazon connector throws a 429. The `MISSING_PRICE` ticket was already moved to Awaiting channel and says "Publishing…". No trigger fires, the sweep does not release it, and the card stays there until someone drags it back by hand.

**Direction**
Fire the exit trigger from a `finally` (or the error path) on every publish site. Also let the evaluator release `AWAITING_CHANNEL` tickets that have no in-flight publish regardless of source, for example after N minutes since `status_changed_at`. This also removes the dependence on `merged sources` in #4.

---

## 4. The trigger loses pending work in three cases — High

**Where** `issue-ticket-trigger.service.ts:48-82,101-115` and `issue-ticket-evaluation.processor.ts:27-37`.

1. **Retry loses the work.** `takePending` reads and deletes the Redis sets before the evaluation runs. Job options are `attempts: 3` (`constants.ts:517-522`). If `evaluateScope` throws (a DB blip, or #1 once fixed), attempt 2 starts, `takePending` returns nothing, `if (!pending.allProducts && !pending.productIds.length) return;` returns success, and the job is done. The failed products are never evaluated.
2. **Same-bucket trigger is ignored while the job is active.** The jobId contains `Math.floor(now / 10s)`. The job's delay is `delayMs ?? 10s`, so it normally runs in the next bucket and a trigger during the run gets a new id. With `delayMs: 0` (the first account of a staggered sweep, or any caller that passes 0) the job runs in the same bucket it was created in. A trigger that arrives while it is active builds the same jobId, BullMQ ignores the add, and the products are left in Redis until the 24h TTL expires.
3. **Empty product list is dropped.** `productIds: []` is truthy, `toSortedIds` returns null, and line 45 returns. Q3 sends `[...publishedProductIds]`, which can be `[]` (see #3).

**Example (1)**
BS import ends for 400 products and enqueues one job. The evaluator hits a deadlock on the advisory lock timeout and throws. Attempt 2 starts after 5s, finds no pending keys and succeeds. The 400 products are not evaluated until the next sweep (which is broken, see #2).

**Direction**
Move the pending set to a processing key (`RENAME`) and delete it only after success, or re-add it in a `catch`. Treat `[]` the same as an unscoped trigger or log it. Do not use a zero delay.

---

## 5. The evaluator builds drafts outside the lock — Medium

**Where** `issue-ticket-evaluator.service.ts:112-160`. Signals load at `112`; the advisory lock and the diff only start at `153-159`. Queue concurrency is 5 (`constants.ts:700-701`) across workers.

**Why it is a flaw**
The lock serialises the writes, but not the reads that decide them. Job A reads "price missing" at T0. Job B (the PRICE_PUBLISH evaluation) reads "price present" at T1, takes the lock and clears the ticket. Job A then takes the lock and applies its older drafts, so it recreates the ticket as `REAPPEARED`.

**Example**
Nightly sweep (A) starts reading Channel 42 at 01:00:00. A user publishes a price at 01:00:05 and job B clears the `MISSING_PRICE` ticket. Job A finishes loading and commits at 01:00:30. The ticket is back as a fresh `TODO` and stays until the next evaluation.

**Direction**
Take the advisory lock before loading signals, or keep one job per channel at a time (per-channel group/queue key).

---

## 6. "Still publishing" is computed twice with different rules — Medium

**Where** `issue-ticket-evaluator.service.ts:519-562` and `417-419` (the evaluator's `findPublishingKeys`) versus `issue-tickets.sql.ts:208-241` (`scopeFlagSql`, used by the board's fix state).

**Why it is a flaw**
The SQL fix state treats a `sync_in_progress` flag on a product-level row (PCM or PC) as "every item of this ticket is publishing". The evaluator instead builds `productId:variantId:marketplaceId` keys and matches an item only on the identical key:
- A parent-level flag has `variantId = null`, so it only matches items whose `variantId` is null (a standalone product). A ticket with variant items, e.g. `p:55:7`, never matches `p:0:7`.
- The product-channel rows (3rd `UNION ALL`) use marketplace `0`, so an Amazon item with marketplace 7 never matches them.
- Shopify: `pc.sync_in_progress` yields `p:0:0`, while Shopify variant items are `p:55:0`.

The board shows PUBLISHING while the evaluator decides nothing is publishing and moves the ticket out of Awaiting channel early.

**Direction**
Reuse the one SQL (`buildItemFixCte`) for this decision instead of a second implementation.

---

## 7. Amazon `MAIN_IMAGE` and `PRODUCT_TYPE_CHANGED` tickets have no marketplace — Medium

**Where** `attention.builder.ts:339-359` — both `addUnit` calls pass no `marketplaceId`, although `row.amazonMarketplaceId` exists on the key-signal row (the same function sets it for `AMAZON_NOT_SEARCHABLE` at `316`). Effect in `issue-tickets.sql.ts:96-102`.

**Why it is a flaw**
These tickets get `amazon_marketplace_id = NULL` and `product_id IS NOT NULL`. The marketplace filter keeps `t.amazon_marketplace_id = :marketplaceId` or *channel-wide* tickets (`product_id IS NULL`), so they match neither. The plan's own board mock-up shows a main-image ticket as "Amazon EU · FR".

**Example**
Command Center card "Main image missing" while the page shows marketplace FR → CTA opens `/action-center-v2?issueTypes=MAIN_IMAGE&marketplaceId=FR` → empty board, although the CC card counted listings. The step 14 check ("each CC card lands on the matching V2 filter") would fail for these two families. Counts per marketplace also differ from the CC, which restricts the same rows by marketplace.

`clearScope` has the same blind spot (`issue-ticket-evaluator.service.ts:181`): removing a variant from marketplace FR does not touch the `NULL`-marketplace ticket.

**Direction**
Decide per family. If the intended key is per marketplace, pass `marketplaceId` (and dedupe by marketplace). If it stays channel-wide per product, the marketplace filter and the plan wording need to say so. This one needs a product decision.

---

## 8. Marketplace filter shows every Amazon channel's channel-wide tickets — Medium

**Where** `issue-tickets.sql.ts:96-102`.

The clause `(t.product_id IS NULL AND t.amazon_marketplace_id IS NULL AND c.channel_type = :amazonType)` is not restricted to channels that own the marketplace. With no channel filter, an account with Amazon EU (DE) and Amazon NA (US) filtering marketplace = DE also gets NA's `CONNECTION` and `ACCOUNT_HEALTH` tickets. The activity feed does the same without even the channel-type check (`issue-ticket-insights.service.ts:118-122`), so a Shopify channel-level event shows under a marketplace filter.

**Direction**
Join `tbl_amazon_channel_marketplaces` on the channel and marketplace.

---

## 9. The publish hook moves tickets the publish does not fix — Medium

**Where** `issue-ticket-publish-hook.service.ts:59-67,88`, with `issue-tickets.sql.ts:340`.

- `fixState !== 'NOT_FIXED'` also counts `NOT_APPLICABLE`. The plan says "other than NOT_FIXED (READY_TO_PUBLISH *or* PUBLISHING)". `PRODUCT_TYPE_CHANGED` is always `NOT_APPLICABLE`, so any forward sync of that product moves it to Awaiting channel even when the product type is untouched. An FBA stock ticket (`NOT_APPLICABLE`) moves on any stock push for the product.
- The scope is product-wide. A ticket with 4 affected variants moves when a publish touches 1 of them; the other 3 are still unfixed.
- `marketplaceIds` empty means *all* marketplaces (`:67`). `amazon-product-fs.service.ts:3263` passes `null` when no marketplace is known, so a stock push for DE moves the FR ticket too.
- The hook runs two queries on every per-variant stock call (`3259`).

Most of these self-correct when the result trigger arrives, but see #3 and #11 for when it does not.

**Direction**
Match the plan: only `READY_TO_PUBLISH`/`PUBLISHING`, filter by the publishing variants, and require a marketplace for Amazon.

---

## 10. Mutations use stale reads — Medium

**Where** `issue-ticket-mutations.service.ts`.

1. `applyStatus` (`206-209`) guards only `old.status <> :status`, not the status the caller saw. `setStatus`/`runPrimaryAction` read the row at `159-173`, then update later. If the publish hook moves the ticket to `AWAITING_CHANNEL` in between, a primary-action click (which believed it was `TODO`) pulls it back to `IN_PROGRESS` and overrides the system move. Example: the user clicks "Review & publish", the hook fires on the publish, then the primary action lands.
2. `applyAssignee` (`243-257`) does an unguarded `UPDATE` and ignores the row count. If the evaluator cleared the ticket in between, the update affects 0 rows but an `ASSIGNEE_CHANGED` event is still written for a ticket that no longer exists, after its `CLEARED`.
3. The event's `fromValue` comes from the stale `ticket.assigneeUserId`, so two simultaneous reassignments both claim the same "from".
4. `runPrimaryAction` still runs for a disabled `PUBLISHING` action; the server never checks `primaryAction.disabled`.

**Direction**
Lock the row in the same statement (as `applyStatus` already does with `FOR UPDATE`), pass the expected old status, read the old assignee from the locked row, and skip the event when no row was updated.

---

## 11. Awaiting channel flips back before the channel has re-validated — Medium

**Where** `issue-ticket-evaluator.service.ts:415-439`.

The move out of Awaiting channel needs only "FS_RESULT was evaluated" and "no in-flight flag". For listing issues, the data the detector reads (`tbl_channel_product_errors`) is replaced by Amazon's later listing notification, not by the feed result. So ~10s after a successful feed the detector still sees the old error and returns the ticket to `IN_PROGRESS`. A few minutes later the notification clears it. The user sees Awaiting → In progress → gone, and the "Awaiting channel" lane loses its purpose for listing issues.

This is a design gap rather than a coding error; the plan (§4.4) says only "result evaluated → cleared or back to IN_PROGRESS". Decide whether listing tickets should wait for the notification (or a timeout) before moving back.

---

## 12. CSV export — Medium

**Where** `issue-tickets.service.ts:148-152`, `issue-tickets.controller.ts:76-88`.

1. **Formula injection.** `csvCell` only quotes `, " \r \n`. Product and variant names come from channel imports. A title like `=HYPERLINK("http://x","click")` or `@SUM(...)` runs when the CSV is opened in Excel. Prefix cells that start with `= + - @` (tab or CR too) with `'`.
2. **Errors look like success.** Headers go out before the first batch. If a query fails mid-stream, `finally { res.end() }` closes a 200 response with a partial file, and the exception then hits Nest's filter on an already-sent response.
3. **No abort/backpressure.** `res.write` ignores the return value and the loop continues after the client disconnects, so a 100k-item export keeps hitting the DB for nobody. With a Fix state filter every batch also recomputes `item_fix` for the whole filtered set.

---

## 13. `loadSignals` always runs every signal query — Medium (performance)

**Where** `command-center-attention-signals.service.ts:63-85`, `issue-ticket-evaluator.service.ts:108-123`.

`needsRows` is one boolean for all non-channel-level types. A `BUY_BOX_REFRESH` trigger (`buy-box-refresh.service.ts:147`: whole channel × one marketplace, Buy Box types only) still runs the inventory, pricing, listing and key-signal queries uncapped for the whole channel, then discards everything but Buy Box drafts. The same applies to a stock webhook (product-scoped, so cheap) and to channel-wide `AMAZON_NOTIFICATION`/`DATA_SYNC` paths. On a channel with tens of thousands of listings this is the dominant cost of the queue.

**Direction**
Run only the queries for the requested issue types.

---

## 14. Smaller issues

**Evaluator / queue**
- `issue-ticket-evaluator.service.ts:448` — if `SEED` is among the merged sources, *all* events are skipped, including real ones merged into the same job (e.g. a `FS_RESULT` during the seed window).
- `issue-ticket-evaluator.service.ts:274` — `eventSource = scope.sources[0]`, and `takePending` returns `SMEMBERS` in arbitrary order, so the audit source of a mixed job is random.
- `issue-ticket-evaluator.service.ts:410-413` — one `UPDATE` per kept ticket on every evaluation, even when nothing changed, all inside the transaction that holds the channel lock. A sweep of 5,000 tickets is 5,000 round-trips; update only changed tickets and bump `last_evaluated_at` in one statement.
- `issue-ticket-evaluator.service.ts:154` — `pg_advisory_xact_lock(int, int)` takes 32-bit ints but the ids are `BIGINT`. Fine today, a runtime error once ids pass 2^31.
- `issue-ticket-evaluator.service.ts:405` — `previousSubTypes.slice(0, 64)` can cut a sub type in the middle for the event text.
- The restore path of soft-deleted products (`product-deletion.service.ts`, `restoreSoftDeleteProducts`) has no trigger call (grep shows none), so a restored product only gets tickets at the next sweep.

**Read API**
- `issue-tickets.sql.ts:399-401` — offset paging over a live board: a ticket that moves lane or changes `affected_variant_count` between two page loads is skipped or shown twice. Keyset on the sort tuple would be stable.
- `issue-tickets.dto` `search` has no `MaxLength`, and the search is a leading-wildcard `ILIKE` over attributes (`issue-tickets.sql.ts:175,187,198`); assignee ids and snooze date have no bounds either (a snooze can be set to year 9999).
- `getProductTickets` goes through `buildTicketBase`, so the caller's *snoozed* tickets are hidden from the product drawer and its priority counts. Confirm that is wanted.
- `getProductTickets` caps at 200 and computes `priorityCounts` from those 200.
- `loadMembers` joins `tbl_user_channels × tbl_role_channels`, so a user with 20 + 20 channels returns 400 rows per call; every mutation and read calls it.

**Schema (step 1)**
- `uq_issue_ticket_items` uses `UNIQUE NULLS NOT DISTINCT` (migration `:271`), which needs PostgreSQL 15+. Confirm every environment is on 15 or newer, or the migration fails.
- No index leads with `channel_id` on `tbl_issue_tickets`, or with `user_id` on `tbl_issue_ticket_snoozes`; the cascade on channel/user delete scans the table.
- `tbl_issue_ticket_events` has no FK to product or channel (intended, so history survives), so events of a hard-deleted channel remain until the 180-day retention removes them.

**Ops**
- `crons.controller.ts:33` — `body.key !== process.env.CRON_KEY` is open to a request with no key when `CRON_KEY` is unset. Same pattern as the existing endpoint at `:20`, so not new, but the new endpoint can enqueue evaluation of every account.

---

## What I checked and found sound

- Ticket key: the DB-generated `ticket_key` matches `buildIssueTicketKey` / the parser (0 for null, same segment order).
- Unique `(account_id, ticket_key)` plus the advisory lock prevent duplicate tickets when two jobs create the same key.
- Product / variant delete: `clearScope` runs inside the delete transaction before the soft delete, as the plan requires, and writes `CLEARED` with `SCOPE_REMOVED`.
- Fix-state rules in SQL follow the plan's baseline semantics (baseline row = unpublished change).
- Access rules: assign-to-self needs view access only; assigning others needs `actionCenter.assign`; the target is checked for module and channel access.
- Publish hook and mutations never throw into callers (hook), and mutations return the updated card; a missing ticket returns `404 TICKET_CLEARED`.

## Decisions I need from you

1. **#7** — Should Amazon `MAIN_IMAGE` and `PRODUCT_TYPE_CHANGED` tickets be per marketplace (matching the plan mock-up and CC counts), or stay marketplace-less per product × channel?
2. **#11** — Should a listing-issue ticket wait for the Amazon notification (or a timeout) before leaving Awaiting channel?
3. **#14 product drawer** — Should snoozed tickets be hidden in the product drawer and its counts?

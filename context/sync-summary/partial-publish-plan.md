# Partial Publish — Implementation Plan (placeholder)

> **Status:** Not started — to be planned in a dedicated task.
> This is a stub so the work has a home; the detailed low-level plan will be filled in later.

## What this covers

The backend that actually **executes a selective sync** — accepting the per-change selection the
[Sync Change Review Dialog](./low-level-implementation-plan.md#f3-submit) sends and pushing only the
chosen changes to the external channel (Amazon / Shopify), instead of publishing the whole product.

This is the counterpart to the `mark-in-sync` route (stalled-status repair) already planned in
[low-level-implementation-plan.md](./low-level-implementation-plan.md). That plan intentionally scoped
its single backend deliverable to `mark-in-sync` only; **partial publish is deliberately deferred to
here.**

## Entry point

`POST /sync/product-syncing/publish-products-to-channel` — extended with a new selective array key
carrying `SyncSelectionPayload[]` (schema in [present-sync-changes.md §3](./present-sync-changes.md)).

## To be detailed later

- Parsing / validating the `SyncSelectionPayload[]` body.
- Backend **constraint expansion** (source of truth) per [present-sync-changes.md §2C](./present-sync-changes.md):
  pricing ↔ price-attributes, Amazon structure → wrapper siblings, Shopify structure → all variants,
  variant-attributes ↔ structure.
- Resolving each selector to concrete baseline rows and rebuilding full top-level objects for Amazon.
- Channel-wide media dedupe.
- Clearing the synced baselines on success + status transitions.
- Queue/BullMQ wiring and reuse of existing forward-sync flows.

## References

- [high-level-plan.md](./high-level-plan.md)
- [present-sync-changes.md](./present-sync-changes.md) — selection rules, constraints, payload format
- [sync-change-summary.md](./sync-change-summary.md) — change-summary response shape
- [low-level-implementation-plan.md](./low-level-implementation-plan.md) — `mark-in-sync` + frontend dialog

# Amazon Events — Tables & Services Analysis

Scope: `api/`, `webapp/`, `db-migrations/` · Date: 2026-10-07
Covers the 21 `tbl_amazon_event_*` tables and the services that read/write them.

> Paths below are relative to `api/apps/api-main/src/` unless stated otherwise.

---

## 0. TL;DR

**What the system is meant to do.** Amazon pushes notifications (orders changed, buy-box lost, account at risk, pricing problem…). The Amazon Events system stores every notification, works out which Categra customer/product it belongs to, classifies it into a business problem ("buy box lost", "pricing blocked", "account suspended"), keeps that problem open until Amazon says it is fixed, decides where to show it, and finally shows it in the Action Center / Command Center and sends an in-app notification.

**What it actually does today.** Only the first step (store the notification) and the subscription bookkeeping produce value. Everything after that is a **shadow pipeline** that runs on every webhook but produces almost nothing:

| Stage | Rows in DB | Why it stops here |
|---|---|---|
| Raw deliveries (every webhook) | 69,913 | — (this is the live inbox, used by the real handlers) |
| Raw messages (deduped) | 59,830 | dedupe result is computed but **never used to skip duplicates** (3,631 duplicates were processed again) |
| Scope resolution results | 11,304 | 11,347 are `ORDER_CHANGE`, whose only consumer (`order-refresh`) has its action **commented out** |
| Normalized observations | 27 | **all `SKIPPED`** (scope ambiguous/unresolved) |
| Occurrences, suppression, projection, AC materialization, notification outbox (+ all revision tables) | **0** | nothing ever reaches them; all consuming flags default to `false` and are not set anywhere |

**Recommendation (primary):** keep 3 tables, delete 18.

| Keep | Delete |
|---|---|
| `raw_deliveries` (absorb `raw_messages` dedupe into it), `destinations`, `subscriptions` | everything else — the shadow pipeline and its admin "shadow" screens |

If the product still wants Amazon-driven alerts (buy box / pricing / account health), §4 gives a collapsed **5-table** design instead of 21. Either way, ~18 of the 21 tables don't need to exist in their current form.

---

## 1. Assumptions & evidence

- **Live status (inferred from code; you were unsure):**
  - **Live:** raw ingestion (always runs in the webhook controller), subscriptions/destinations (auto-subscription defaults to **enabled**: `AMAZON_EVENTS_AUTO_SUBSCRIBE_ENABLED` default `true`), and the legacy handler that consumes `live_processing_status`.
  - **Effectively off:** Action Center materialization, Action Center rendering, Command Center section and in-app notifications. They are gated by `AMAZON_EVENTS_LIVE_ENABLED` + per-feature flags, all default `false`. None of the ~45 `AMAZON_EVENTS_*` flags is set in `api/.env`, `.env.example`, `ecosystem*.config.js` or `docs/`. The DB confirms it: 0 rows in every gated table.
  - **Runs regardless of flags:** scope resolution, the 3 normalizers, suppression and projection. They run synchronously inside every webhook request (see §3.1 finding F1).
- **DB evidence:** read-only queries (wrapped in `BEGIN READ ONLY`) against the DB configured in `api/.env` (Neon, `NODE_ENV=local`, 25 accounts / 52 channels). Data spans **2026-05-27 → 2026-10-07**. **This is not production.** Production volumes and the ambiguity rate may differ, but the structural findings (write-only tables, commented-out consumer, unmatched payload shapes) are code facts.
- **Admin routes:** all `admin/amazon-events/**` controllers are reachable. `AdminModule` is imported in `app.module.ts` and the controllers use absolute `admin/...` paths, guarded by `AdminJwtAuthGuard`. Their UI is not in this monorepo; `webapp/` only calls `command-center/amazon-events`.

---

## 2. How the pipeline flows today

```
Amazon (SQS / EventBridge)
  │  POST /amazon-notifications/sqs | /event-bridge
  ▼
AmazonNotificationController.persistLiveNotification            (amazon-notifications.controller.ts:74)
  └─ AmazonEventsRawService.ingestLiveNotification               (raw/amazon-events-raw.service.ts:156)
       ├─ INSERT raw_deliveries                                   ← always
       ├─ canonicalize → INSERT/UPDATE raw_messages + raw_processing_attempts (+ derivation_generations)
       └─ AmazonEventsScopeService.resolveCanonicalRawMessage    (scope/amazon-events-scope.service.ts:187)
            │  only for ACCOUNT_STATUS_CHANGED, ORDER_CHANGE, ANY_OFFER_CHANGED, PRICING_HEALTH
            ├─ INSERT scope_resolution_results (+ scope_identities, scope_aliases when RESOLVED+HIGH)
            ├─ ORDER_CHANGE        → AmazonEventsOrderRefreshService  → **no-op (commented out)**
            ├─ ACCOUNT_STATUS_CHANGED → AccountService  ┐
            ├─ PRICING_HEALTH      → PricePolicyService ├─ INSERT normalized_observations
            └─ ANY_OFFER_CHANGED   → BuyBoxService      ┘   → occurrences (+ revisions, sources)
                     → SuppressionService  → suppression_edges (+ revisions)
                     → ProjectionService   → projection_candidates (+ revisions)
                     → ACMaterializationService [flag-gated] → action_center_materializations (+ revisions)
                     → NotificationsService     [flag-gated] → notification_outbox (+ delivery_revisions) → tbl_notifications
  ▼ (only after all of the above returns)
AmazonNotificationsService.processSqsNotification / processEventBridgeNotification   ← the REAL business handling
  └─ AmazonEventsRawService.markLiveNotificationProcessingOutcome → UPDATE raw_deliveries.live_processing_*
```

So there are **two parallel processors for the same notification**:

| Notification | Legacy `AmazonNotificationsService` (real) | Shadow pipeline |
|---|---|---|
| `ORDER_CHANGE` | queues order sync (447 processed, 13,252 "channel mapping not found") | resolves scope, then does nothing |
| `ACCOUNT_STATUS_CHANGED` | updates `tbl_channels.connection_status` (`processAccountStatusChanged`, :3523) | tries to classify via regex → always skipped |
| `ANY_OFFER_CHANGED` | `unsupported_notification_type` | buy-box classifier → skipped (scope ambiguous) |
| `PRICING_HEALTH` | not handled | price-policy classifier. **0 deliveries ever received**, though 25 subscriptions say `SUBSCRIBED` |
| `LISTINGS_ITEM_*`, `FEED_/REPORT_PROCESSING_FINISHED`, `ITEM_PRODUCT_TYPE_CHANGE` | handled | not scope-resolvable; only stored raw |

---

## 3. Per-table analysis

Legend: **KEEP** · **KEEP + IMPROVE** · **MERGE → X** · **DELETE**.
"Shadow-only reader" means the only read is an admin `amazon-events-shadow-*` inspection screen.

### Layer A — Raw inbox

#### 3.1 `tbl_amazon_event_raw_deliveries` — **KEEP + IMPROVE**

**Purpose (plain):** the inbox. One row per HTTP call Amazon makes to us, saved *before* we do anything, so nothing gets lost and we can see what happened to each notification (processed / ignored / failed and why).

**Technical:**
- Writer: `AmazonEventsRawService.persistNormalizedRawBody` (raw service :207). `markLiveNotificationProcessingOutcome` (:175) updates `live_processing_status/reason/error/processed_at`.
- Readers: scope service, the 3 normalizers, order-refresh, admin shadow-raw, admin subscriptions ("receiving evidence").
- Columns: transport (`ingest_source`, `transport_kind`, `transport_delivery_id`, `transport_request_id`), provider identity (`provider_message_id`, `raw_notification_type`, `provider_published_at`), payload (`payload_json`, `payload_text`, `payload_hash`, `payload_byte_size`, `payload_storage_kind`, `headers_json`, `meta_json`), parse/dedupe (`parse_status`, `parse_error_summary`, `canonicalization_status`, `dedupe_status`, `semantic_identity_key`, `canonical_raw_message_id`), live outcome (`live_processing_*`), `ingested_at`.
- Data: 69,913 rows · **239 MB** (largest table) · ~3.5k rows/week · 23,899 rows older than 90 days.

**Problems:**
- **F1 – blocks the webhook.** The full shadow chain runs synchronously inside `persistLiveNotification` before the real handler starts and before Amazon gets its HTTP response.
- **F2 – no retention.** The admin cleanup explicitly *protects* this table (`assertProtectedTablesRemainUntouched`) and only runs manually.
- **F3 – payload stored twice.** `payload_json` holds `{ rawPayload, normalizedPayload, envelope }`; for SNS envelopes the body exists both as a string (`rawPayload.Message`) and parsed (`normalizedPayload`). `meta_json` duplicates `live_processing_*` (`markLiveNotificationProcessingOutcome` writes both).
- `ingest_source = INTERNAL_HTTP` (the shadow ingest endpoint) has 0 rows. That endpoint needs `AMAZON_EVENTS_SHADOW_INGEST_KEY`, which is not configured, so it always returns 503.

**Verdict:** keep. It is the only table here that the live handlers depend on. Absorb `raw_messages` dedupe (3.2), drop `meta_json` duplicates and `semantic_identity_key` (unused when `provider_message_id` exists, which is 100% of rows), store the payload once, add a retention job.

#### 3.2 `tbl_amazon_event_raw_messages` — **MERGE → `raw_deliveries`**

**Purpose (plain):** Amazon sometimes sends the same notification more than once. This table keeps one "unique message" per real event and counts how many copies arrived.

**Technical:**
- Writer: `canonicalizeDelivery` (raw service :368) creates a row or increments `delivery_count` per `canonical_dedupe_key` (`provider:<NotificationId>` in 100% of rows).
- Readers: scope service and normalizers (`raw_notification_type`, `extraction_summary_json`, timestamps), the Action Center evidence SQL join (`action-center.service.ts:3448`), AC sync evaluation (:1823), admin.
- Data: 59,830 rows, 90 MB. Every row has `dedupe_basis = PROVIDER_MESSAGE_ID`.

**Problems:** dedupe is computed but **not acted on**. The legacy handler still processes duplicates (3,631 `DUPLICATE` deliveries are `PROCESSED`). `extraction_summary_json` duplicates `raw_deliveries.meta_json` hints.

**Verdict:** merge. A unique partial index on `raw_deliveries(provider_message_id)` plus `first_delivery_id` / `is_duplicate` columns gives the same dedupe. Then use it in the controller to short-circuit duplicates. Joins that read `raw_notification_type` can read it from the delivery.

#### 3.3 `tbl_amazon_event_raw_processing_attempts` — **DELETE**

**Purpose (plain):** a log line per "processing step attempt" on a message.

**Technical:** written only by canonicalization (raw service :440, :511, :573). Each canonicalization writes one `SUCCESS` row: 69,908 rows ≈ 1:1 with deliveries, 24 MB. Readers: admin shadow-raw counts and admin cleanup. The only other stage value (`processing_stage`) is `CANONICALIZE`; no other stage writes here.

**Verdict:** delete. It is an audit of a step that cannot meaningfully fail, and failure text is already copied to `raw_deliveries.parse_error_summary`.

#### 3.4 `tbl_amazon_event_derivation_generations` — **DELETE**

**Purpose (plain):** a "version stamp" so the pipeline could be re-run with new rules and old/new results compared side by side.

**Technical:**
- Every stage calls its own `getOrCreateActiveGeneration` (7 near-identical copies, one per service). Rows carry `generation_id`, and the unique keys include it.
- Data: 4 rows. `LIVE_CANONICALIZATION` and `SCOPE_RESOLUTION` have been `ACTIVE` since day one. `PROVIDER_STATE_NORMALIZATION` rotated once (buy-box rule v2).
- Nothing replays or compares generations: `REPROCESS` type unused, `is_promoted_shadow_baseline` never read by logic.
- Readers: every stage service, admin shadow screens, Command Center (`command-center.service.ts:385`), cleanup.

**Verdict:** delete together with the shadow layers. If any derived layer survives (§4), replace the generation id with a plain `rule_version` string column on that table.

### Layer B — Scope (who does this event belong to?)

#### 3.5 `tbl_amazon_event_scope_resolution_results` — **DELETE** (or merge into the delivery, see §4)

**Purpose (plain):** for each notification, record our attempt to answer "which customer, channel, marketplace and product is this about?", and how sure we are.

**Technical:**
- Writer: `persistResolutionResult` (scope service :258); one row per (generation, raw message, scope kind).
- Readers: the 3 normalizers (`findByPk`), order-refresh, admin shadow-scope, cleanup, channel delete (`channel.service.ts:1372`), product delete (`product-deletion.service.ts:3011`).
- Columns: ~29, including 4 candidate JSON arrays, an evidence JSON and reason JSONs.
- Data: 11,304 rows (27 MB). 11,347 `ACCOUNT_SCOPE RESOLVED` (= `ORDER_CHANGE`), 24 `LISTING_SCOPE AMBIGUOUS` (`SELLER_MATCH_MULTIPLE_ACCOUNTS`: the same seller is connected to several Categra accounts), 2 unresolved, 1 account ambiguous, 1 account unresolved.

**Problems:**
- The 11k `ORDER_CHANGE` results feed `AmazonEventsOrderRefreshService.triggerScopeResult`, whose action is commented out (`amazon-events-order-refresh.service.ts:54`). That is pure write cost.
- The legacy handler already does its own, simpler routing via `tbl_amazon_event_subscriptions.provider_subscription_id`. So scope is resolved twice, with two different algorithms.

**Verdict:** delete. If alerts are kept (§4), store the resolved `account_id / channel_id / amazon_channel_marketplace_id / product_channel_marketplace_id / variant_marketplace_id` directly on the issue row and keep only a short `scope_status` + `scope_reason` on the delivery.

#### 3.6 `tbl_amazon_event_scope_identities` — **DELETE**

**Purpose (plain):** a stable ID for "this exact listing of this customer in this marketplace", so repeated events about the same thing group together.

**Technical:** created only when scope is `RESOLVED` + `HIGH` (`upsertScopeIdentity`, scope service :365), keyed by `scope_fingerprint` (e.g. `LISTING_SCOPE|account:1|channel:2|amazonChannelMarketplace:3|variantMarketplace:4`). Its columns are just the FKs already present in the fingerprint plus candidates. Readers: suppression, projection, AC materialization, Command Center, admin, channel/product deletion. Data: **1 row**.

**Verdict:** delete. It is a surrogate for a tuple of FKs that already exist in core tables (`tbl_product_channel_marketplaces`, `tbl_variant_marketplaces`). Group by those FKs directly.

#### 3.7 `tbl_amazon_event_scope_aliases` — **DELETE**

**Purpose (plain):** "other names" for a scope identity (seller ID, ASIN, SKU, internal IDs) so future events could be matched by any of them.

**Technical:** written by `syncScopeAliases` (scope service :433). It is **never used for lookup**: scope resolution always queries `AmazonChannel`, `ProductChannelMarketplace`, `VariantMarketplace`, `Wrapper`, `ProductCurrentOffers` directly. Readers: admin shadow-scope, cleanup. Data: 29 rows, all for the single identity.

**Verdict:** delete. It is write-only from the product's point of view.

### Layer C — Provider-state (what problem does this event describe?)

#### 3.8 `tbl_amazon_event_normalized_observations` — **DELETE** (or fold into the issue table, §4)

**Purpose (plain):** the result of reading one notification and labelling it, e.g. "this says the buy box was lost, severity error".

**Technical:**
- Writers: `normalizeScopeResult` in the account / price-policy / buy-box services; the code is triplicated (§5.3).
- Readers: suppression, projection, AC evidence SQL, AC sync evaluation, admin shadow-account/price-policy/buy-box.
- Data: 27 rows, **all `SKIPPED`**: 24 `LISTING_SCOPE_AMBIGUOUS`, 3 `SCOPE_NOT_RESOLVED`.

**Problems:**
- **F4 – Account classifier cannot match real payloads.** Real `ACCOUNT_STATUS_CHANGED` payload: `{"accountStatusChangeNotification":{"currentAccountStatus":"NORMAL","previousAccountStatus":"AT_RISK"}}`, with **no seller ID**. The classifier needs a seller ID for scope and then looks for words like "suspended", "under review" or "healthy" (`account.service.ts:74`), none of which appear. Result: it can never create an occurrence.
- Buy-box and price-policy classifiers are also free-text regex searches over the whole payload. The buy-box classifier matches phrases such as "buy box lost" rather than the structured `IsBuyBoxWinner` / `Offers[]` fields Amazon actually sends.
- `PRICING_HEALTH` has never been received in this DB.

**Verdict:** delete with the shadow pipeline. If alerts are kept, classification should read structured fields (`currentAccountStatus`, `IsBuyBoxWinner`, `Summary.BuyBoxEligibleOffers`…) and write directly onto the issue row.

#### 3.9 `tbl_amazon_event_occurrences` — **DELETE** (this *is* the table to keep if alerts are kept, §4)

**Purpose (plain):** an open "issue", e.g. "Buy box lost on listing X since Monday". It stays `ACTIVE` until a recovery event closes it (`COMPLETED`) or a worse issue replaces it (`SUPERSEDED`).

**Technical:**
- Writers: `openOccurrence` / `refreshOccurrence` / `supersedeWeakerOccurrences` / `complete*Occurrences` in the 3 normalizers.
- Readers: suppression, projection, AC evidence SQL (`action-center.service.ts:3444`), AC sync evaluation, Command Center, admin.
- Data: **0 rows**.

**Verdict:** delete now. It is the right core concept, but it has never held a row. If the feature is pursued, it becomes the single "issue" table (§4) and absorbs projection + AC materialization fields.

#### 3.10 `tbl_amazon_event_occurrence_revisions` — **DELETE**

**Purpose (plain):** history of every state change of an issue (opened, refreshed, closed…).

**Technical:** written 4× per normalizer; read only by admin shadow screens. Each row copies the occurrence + observation into `snapshot_json`. Data: 0 rows.

**Verdict:** delete. `opened_at / last_observed_at / closed_at + open/close observation` on the occurrence already cover what any consumer reads.

#### 3.11 `tbl_amazon_event_occurrence_sources` — **DELETE**

**Purpose (plain):** "which notifications contributed to this issue".

**Technical:** one row per revision: `(occurrence_id, revision_id, observation_id, raw_message_id)`. It is fully derivable (revision → observation → raw message). Read only by admin. Data: 0.

**Verdict:** delete. It is redundant with revisions.

### Layer D — Suppression & projection (should we show it, and where?)

#### 3.12 `tbl_amazon_event_suppression_edges` — **DELETE**

**Purpose (plain):** "don't nag about a smaller problem while a bigger one is active". For example, hide "buy box lost" while the whole account is suspended or the price is blocked.

**Technical:**
- Writer: `AmazonEventsSuppressionService.evaluateObservationContext`, with 3 rules (`ACCOUNT_CRITICAL_LISTING_PRECEDENCE`, `PRICE_POLICY_BLOCKED_BUY_BOX_PRECEDENCE`, `PRICE_POLICY_AT_RISK_BUY_BOX_AT_RISK_PRECEDENCE`).
- Readers: projection, admin.
- Data: 0.

**Verdict:** delete. Three precedence rules over at most a handful of open issues per listing can be evaluated at read time (or as a `suppressed_by_issue_id` column on the issue). A materialized edge graph is not needed.

#### 3.13 `tbl_amazon_event_suppression_edge_revisions` — **DELETE**

Write-only history of edges; read only by admin. 0 rows. Delete with 3.12.

#### 3.14 `tbl_amazon_event_projection_candidates` — **DELETE** (merge into the issue table, §4)

**Purpose (plain):** per issue, the decision on where to show it (Action Center, Command Center, both, admin only, nowhere) and what buttons/CTAs it gets.

**Technical:**
- Writer: `AmazonEventsProjectionService.evaluateObservationContext`.
- Readers: AC materialization, Command Center (`command-center.service.ts:399`), admin.
- 1:1 with an occurrence per generation: `projection_target`, `projection_status`, `suppressed_by_edge_id`, `cta_template_keys_json`, `completion_mode`, `dedupe_*`, `is_projectable`.
- Data: 0.

**Verdict:** delete. Every column is a deterministic function of (family, suppression, listing-protection context) and can be columns on the issue row or computed on read.

#### 3.15 `tbl_amazon_event_projection_candidate_revisions` — **DELETE**

Write-only history; read only by admin; 0 rows.

### Layer E — Surfacing (Action Center / Command Center / notifications)

#### 3.16 `tbl_amazon_event_action_center_materializations` — **DELETE** (merge into the issue table, §4)

**Purpose (plain):** the actual Action Center card created from an Amazon issue. It also records whether the card was hidden because the same problem already shows up from another source (duplicate blocker).

**Technical:**
- Writer: `AmazonEventsActionCenterMaterializationService.evaluateProjectionGenerationContext`, gated by `AMAZON_EVENTS_LIVE_ENABLED && …_ACTION_CENTER_ENABLED && …_MATERIALIZER_ENABLED`.
- Readers:
  - `action-center.service.ts:2110` (flag-gated: `…_RENDER_ENABLED`)
  - `action-center.service.ts:3436` evidence SQL, which is **not** flag-gated and runs on every listing-issue evidence request against empty tables
  - `action-center-sync-evaluation.service.ts:1818`
  - notifications service
  - channel/product deletion cleanups
- ~35 columns, mostly copied from the projection candidate.
- Data: 0.

**Verdict:** delete now. Remove the AC/AC-sync/Command Center read paths with it. If alerts are kept, this is where they surface, but as columns on the single issue table.

#### 3.17 `tbl_amazon_event_action_center_materialization_revisions` — **DELETE**

Write-only history except one read: the notifications service uses the latest `ACTIVE` revision id to build an `open_cycle_key` (`loadLatestActiveCycleRevisionId`). That key can be derived from the issue's `opened_at`. 0 rows.

#### 3.18 `tbl_amazon_event_notification_outbox` — **DELETE**

**Purpose (plain):** queue of in-app notifications to send for an issue, one per recipient, with dedupe/cooldown so users aren't spammed.

**Technical:**
- Writer/consumer: `AmazonEventsNotificationsService`, gated by `…_NOTIFICATIONS_ENABLED && …_ORCHESTRATOR_ENABLED`.
- **It is not a real outbox.** Rows are delivered synchronously in the same call (`processPendingDeliveries` → `deliverOutboxRow`, notifications service :578) into the existing `NotificationsService.createNotification`, which already stores `dedupeKey` in `customValues`.
- Readers: itself, cleanup, deletion paths.
- Data: 0.

**Verdict:** delete. Call `NotificationsService` directly with the dedupe key; cooldown can be a lookup on `tbl_notifications`.

#### 3.19 `tbl_amazon_event_notification_delivery_revisions` — **DELETE**

Strictly write-only: one `create` in the notifications service, **no reader anywhere**, 0 rows.

### Layer F — Subscription control plane (live)

#### 3.20 `tbl_amazon_event_destinations` — **KEEP**

**Purpose (plain):** the endpoints we have registered with Amazon to receive notifications (our SQS queue / EventBridge bus per region).

**Technical:**
- Writers: `AmazonEventsAutoSubscriptionService.findOrCreateDestinationRow` and the admin subscriptions service (duplicated logic, §5.6).
- Readers: the same two services.
- Data: 4 rows: `SQS`/`EVENTBRIDGE` × `na`/`eu`; the two `eu` ones are `ERROR`.

**Verdict:** keep. It is small, live and correct in purpose.

#### 3.21 `tbl_amazon_event_subscriptions` — **KEEP + IMPROVE**

**Purpose (plain):** per connected Amazon channel and notification type, whether we are subscribed at Amazon, with which Amazon subscription ID, and the last error.

**Technical:**
- Writers: auto-subscription service, called on channel connect/authorize (`amazon-channel.service.ts:545`, `:722`), channel update (`channel.service.ts:502`), marketplace change (`amazon-channel-marketplace.service.ts:346`), plus the admin subscriptions service.
- Readers: **the live legacy handler routes notifications through it** (`resolveOrderChangeSubscriptionsForNotification`, `amazon-notifications.service.ts:165`; listing subscriptions :2594; account status :3572).
- Data: 140 rows:

  | Type | Subscribed | Error |
  |---|---|---|
  | `ORDER_CHANGE` | 27 | 2 |
  | `ACCOUNT_STATUS_CHANGED` | 29 | — |
  | `PRICING_HEALTH` | 25 | 4 |
  | `FULFILLMENT_ORDER_STATUS` | 12 | 12 |
  | `ANY_OFFER_CHANGED` | — | 29 (all `ERROR`) |

**Problems:** a second, older table `tbl_amazon_channel_subscriptions` (not one of the 21; 116 rows: `FEED_/REPORT_PROCESSING_FINISHED`, `LISTINGS_ITEM_ISSUES/STATUS_CHANGE`) holds the same kind of data for other notification types. The legacy handler reads **both** (`amazon-notifications.service.ts:2547` and `:2594`).

**Verdict:** keep, and merge `tbl_amazon_channel_subscriptions` into it so all subscription state lives in one table and one service.

---

## 4. Target shape

### 4.1 Primary — remove the shadow pipeline (3 tables)

| Table | Change |
|---|---|
| `tbl_amazon_event_raw_deliveries` | absorb dedupe from `raw_messages`; payload stored once; drop `meta_json` duplicates, `semantic_identity_key`, `canonicalization_status`; add retention (e.g. 90 days, keep `FAILED`) |
| `tbl_amazon_event_destinations` | unchanged |
| `tbl_amazon_event_subscriptions` | absorb `tbl_amazon_channel_subscriptions` |

Deleted (18): `raw_messages`, `raw_processing_attempts`, `derivation_generations`, `scope_resolution_results`, `scope_identities`, `scope_aliases`, `normalized_observations`, `occurrences`, `occurrence_revisions`, `occurrence_sources`, `suppression_edges`, `suppression_edge_revisions`, `projection_candidates`, `projection_candidate_revisions`, `action_center_materializations`, `action_center_materialization_revisions`, `notification_outbox`, `notification_delivery_revisions`.

**Risk:**
- Loses the (empty) Command Center "Amazon events" section and the (empty) AC materialized rows. Both are flag-off and contain no data today. `webapp/src/app/(index)/(menu-layout)/command-center/main.tsx` already hides the section when `items.length === 0`.
- `ACCOUNT_STATUS_CHANGED` handling continues via the legacy handler (fix bug F6 there).

### 4.2 Alternative — keep Amazon-driven alerts, collapsed (5 tables)

Use this only if product wants buy-box / pricing / account-health issues in Action Center.

| Table | Replaces |
|---|---|
| `raw_deliveries` (+ `scope_status`, `scope_reason`, `classification_status`, `classification_reason`) | raw_messages, raw_processing_attempts, derivation_generations, scope_resolution_results, normalized_observations |
| `amazon_event_issues` (account/channel/marketplace/PCM/VM FKs, family, severity, lifecycle state, opened/last_observed/closed, open/close delivery ids, `suppressed_by_issue_id`, projection target + CTA keys, AC visibility/duplicate-blocker fields, `rule_version`) | scope_identities, scope_aliases, occurrences, occurrence_revisions, occurrence_sources, suppression_edges(+rev), projection_candidates(+rev), action_center_materializations(+rev) |
| notifications go straight to `tbl_notifications` with `dedupeKey` | notification_outbox, notification_delivery_revisions |
| `destinations`, `subscriptions` | unchanged (+ merge legacy channel subscriptions) |

Preconditions before this is worth building:
1. Classifiers must read structured payload fields (F4).
2. Seller→account ambiguity must be resolved via `subscriptions.provider_subscription_id` (as the legacy handler does) instead of seller-ID matching.
3. Processing must move off the webhook request into a BullMQ queue (F1).

---

## 5. Services — what they actually do, and what to merge / improve / remove

### 5.1 `AmazonNotificationController` + `AmazonNotificationsService` (3,936 lines) — **KEEP, slim down**

- **Actually does:** authenticates the webhook (`x-custom-auth-amazon`), then dispatches by type:

  | Notification type | Handler behaviour |
  |---|---|
  | `ORDER_CHANGE` / `FULFILLMENT_ORDER_STATUS` | queue order sync |
  | `LISTINGS_ITEM_ISSUES_CHANGE` / `STATUS_CHANGE` | persist listing issues / update listing status & ASIN |
  | `ITEM_PRODUCT_TYPE_CHANGE` | enqueue product-type change + platform notification |
  | `FEED_PROCESSING_FINISHED` | match feed job |
  | `REPORT_PROCESSING_FINISHED` | match report job; **49,366 of 49,367 are `unhandled_report_type`** |
  | `ACCOUNT_STATUS_CHANGED` | update channel connection status |

  Writes the outcome back to `raw_deliveries`.
- **Improve:**
  - Short-circuit duplicates using the dedupe result (3,631 duplicates re-processed).
  - Respond 200 to Amazon before heavy work (or enqueue).
  - `REPORT_PROCESSING_FINISHED` is 71% of all traffic and almost never useful. Unsubscribe or filter to the report types we request.
  - Split the 3.9k-line file per notification family.
  - Remove `console.log('authToken: ', authToken)` (`:465`), which logs the shared secret on every EventBridge call.

### 5.2 `AmazonEventsRawService` (1,457 lines) — **KEEP, shrink to inbox only**

- **Actually does:**
  1. Unwraps SNS envelopes.
  2. Extracts type / message ID / timestamps and walks the payload for seller/marketplace/ASIN/SKU/order hints.
  3. Stores the delivery, upserts the deduped message, writes a processing-attempt row and gets-or-creates a generation.
  4. Then **synchronously** triggers scope resolution.

  It also serves `POST internal/amazon-events/shadow/raw/ingest` (dead: secret not configured, 0 `INTERNAL_HTTP` rows).
- **Remove:**
  - the shadow ingest endpoint + controller
  - generation and processing-attempt logic
  - the call into scope
  - the hint extraction (only scope uses it)
  - the three-way dedupe basis (only `PROVIDER_MESSAGE_ID` ever occurs)

  What remains is ~200 lines: normalize envelope → insert delivery (dedupe by `provider_message_id`) → mark outcome.

### 5.3 Account / Price-policy / Buy-box normalizers (1,034 + 1,321 + 1,278 lines) — **REMOVE** (or rewrite as one, §4.2)

- **Actually do:**
  1. Load the scope result, raw message and delivery.
  2. Run a gauntlet of skip checks; in practice every event is skipped at the scope checks.
  3. Regex-scan payload text for keywords.
  4. Run an open/refresh/supersede/complete lifecycle on occurrences, writing revision + source rows for each transition.
- **Duplication:** the three services share the same skeleton nearly line for line (`normalizeScopeResult`, `buildObservationPlan`, `applyOccurrenceLifecycle`, `supersedeWeakerOccurrences`, `refreshOccurrence`, `openOccurrence`, `createOccurrenceSource`, `buildRevisionSnapshot`, `findActiveOccurrence`, `getOrCreateActiveGeneration`, `canReuseNormalizationGeneration`, `buildNormalizationSelectionBounds`, `isActiveSeriesUniqueViolation`). Price-policy and buy-box also share the whole payload-segment scanner (`extractRelevantPayloadSegments`, `collectRelevant*`, `buildStructuredIndicator`, `isRelevantPath`, …). Only the signal-definition tables and the precedence rules differ.
- **Dead branch:** `price-policy.service.ts:622-635`: both branches call `openOccurrence`.
- **If kept:** one `AmazonEventIssueService` with a per-family strategy object (`classify(payload) → {family, severity} | recovery | null`, `precedence`).

### 5.4 `AmazonEventsScopeService` (2,161 lines) + `AmazonEventsOrderRefreshService` (200 lines) — **REMOVE**

- **Scope service actually does:**
  1. Matches seller IDs to `tbl_amazon_channels`.
  2. Matches the marketplace to `tbl_amazon_channel_marketplaces`.
  3. For listing events, matches ASIN to PCM/VM, then falls back to `Wrapper` and a scan of the latest 250 `ProductCurrentOffers` rows by SKU.
  4. Persists result + identity + aliases.
  5. Fans out to normalizers → suppression → projection → AC → notifications, each wrapped in `try/catch console.error`, so failures are invisible.
- **Order-refresh:** builds a context, then does nothing (`:54`, call commented out).
- **Why remove:** the legacy handler already resolves ownership more reliably through `subscriptions.provider_subscription_id`. That key is present in every notification's metadata and doesn't break when one seller is connected to several accounts, which is the cause of all 24 listing ambiguities here.

### 5.5 Suppression (1,300) / Projection (1,702) / AC Materialization (1,550) / Notifications (786 + 148 prefs) — **REMOVE** (or collapse into the issue service)

- **Suppression:** 3 precedence rules, materialized as edges with revisions.
- **Projection:** decides target/CTA/completion per occurrence. Also queries listing-protection context (wrappers, Amazon channel marketplaces) to downgrade targets.
- **AC Materialization:** turns candidates into AC rows. Hides them when a legacy AC issue already exists for the same product/channel/marketplace (`loadVisibleLegacyIssues`), then triggers `ActionCenterSyncEvaluationService`.
- **Notifications:** resolves recipients (`AmazonEventsNotificationPreferencesService` → `UserNotificationSettings` family defaults), writes an outbox row, delivers immediately, writes a revision.
- **Each re-implements** `getOrCreateActiveGeneration`, `readBooleanFlag`, `readNumericAllowlist`, `sanitizeDetail`, `uniqueNumericIds`, `normalizeTextValue`. The same flag-reader exists 5 times: AC materialization, AC service, Command Center, notifications, auto-subscription.
- **Consumers to clean up with them:**
  - `action-center.service.ts`: `loadAmazonEventMaterializedRows` (:2110), `loadAmazonEventListingIssueEvidence` (:3350, ungated SQL), `getAmazonEventActionCenterReadConfig` (:5197)
  - `action-center-sync-evaluation.service.ts`: `evaluateAmazonEventMaterialization` (:354) + helpers
  - `command-center.service.ts`: `getAmazonEventsSummary` (:171), `amazon-events.builder.ts`
  - webapp `command-center/main.tsx` `AmazonEventsSection` + `lib/api.ts` `GET_COMMAND_CENTER_AMAZON_EVENTS`
  - deletion SQL in `channel.service.ts:1368-1380` and `product-deletion.service.ts` (:2918, :3011-3042, :4168-4178, :4393, :4683-4699)

### 5.6 `AmazonEventsAutoSubscriptionService` (2,470) + admin `AmazonEventsSubscriptionsService` (2,488) — **KEEP, merge**

- **Actually do:** create/verify Amazon notification destinations and subscriptions per connected channel (grantless token for destinations, seller token for subscriptions). Both handle marketplace-scoped `ORDER_CHANGE` filters, sibling-subscription reuse across channels of the same seller, and remote discovery.
  - Auto-subscription runs on channel authorize/update.
  - The admin service powers manual admin actions (setup / subscribe / recheck / disable) and an overview with "receiving evidence", computed by scanning `raw_deliveries`.
- **Duplication:** 24 private helpers have identical names and near-identical bodies in both files: `resolveDestinationConfig`, `findOrCreateDestinationRow`, `getSellerAccessToken`, `resolveGrantlessRegionForChannel`, `resolveSellerRegionForChannel`, `inferGrantlessRegionFromMarketplaces`, `validateRemoteSubscriptionShape`, `requiresMarketplaceDirectives`, `requiresOrderChangeDirective`, `resolveProviderPayloadVersion`, `extractRemote*`, `throwIfAmazonRequestFailed`, `isAmazonNotFoundError`, `normalizeOptional*`, …
- **Merge:** the admin service should become a thin presentation layer over `AmazonEventsAutoSubscriptionService`. The admin controller already calls `auto-subscription.reconcile` for `/reconcile`. Also fold in `AmazonChannelSubscriptionService` (legacy `tbl_amazon_channel_subscriptions`), and drop subscribing to `ANY_OFFER_CHANGED` / `PRICING_HEALTH` if §4.1 is chosen. These two only feed the shadow pipeline; `ANY_OFFER_CHANGED` is 29/29 `ERROR` anyway.

### 5.7 Admin modules — **REMOVE the 7 shadow modules; keep catalog + subscriptions; replace cleanup**

| Module | Actually does | Verdict |
|---|---|---|
| `amazon-events` (`overview`, `catalog`, `types/:type`) | static catalog of notification types from `amazon-events.catalog.ts` (571 lines) | keep (useful reference), trim to supported types |
| `amazon-events/subscriptions` | admin control plane (5.6) | keep, slim |
| `amazon-events/cleanup` (`POST phase-1/run`) | manual batch delete of attempts (60d), scope results (120d), inactive aliases (180d), terminal outbox (180d); explicitly protects raw tables | replace with a scheduled retention for `raw_deliveries` (the only big table left) |
| `shadow-raw`, `shadow-scope`, `shadow-account`, `shadow-price-policy`, `shadow-buy-box`, `shadow-suppression`, `shadow-projection` (~4.8k lines) | read-only inspection screens over the shadow tables | remove with the tables. Keep a small "deliveries" list (from shadow-raw) if ops needs to browse the inbox |

### 5.8 Constants (`core/constants/amazon-events/*`, ~1.5k lines)

Remove everything for the removed layers:

| File | Removed for |
|---|---|
| `amazonEventsScope` | scope |
| `amazonEventsAccount`, `amazonEventsPricePolicy`, `amazonEventsBuyBox`, `amazonEventsProviderState` | normalizers |
| `amazonEventsSuppression` | suppression |
| `amazonEventsProjection` | projection |
| `amazonEventsActionCenterMaterialization` | AC materialization |
| `amazonEventsNotifications` | notifications |
| `amazon-events-cleanup` | cleanup |
| most of `amazonEventsRaw` (generation types/statuses, processing stages, canonicalization statuses, dedupe basis) | raw shadow logic |

Keep `amazonEventsAutoSubscription`, `amazon-events-subscriptions.constants`, `amazonEventsSubscriptionCompatibility`, `amazon-events.catalog`, and the live-processing statuses.

---

## 6. Findings list (bugs & risks)

| # | Severity | Finding | Where |
|---|---|---|---|
| F1 | High | Full shadow chain (scope → 3 normalizers → suppression → projection → AC → notifications, many DB round-trips) runs synchronously inside the Amazon webhook request, before the real handler and before the HTTP response | `amazon-notifications.controller.ts:31,54,74` → `raw.service.ts:265` |
| F2 | High | `raw_deliveries` (239 MB, +~3.5k rows/week) has no retention; cleanup protects it and is manual-only | `amazon-events-cleanup.service.ts:636` |
| F3 | Medium | Dedupe is computed but ignored; 3,631 duplicate deliveries were fully re-processed | raw service + legacy handler |
| F4 | High (for shadow) | Account classifier cannot match real payloads: no seller ID (scope always unresolved) and regex vocab ≠ `NORMAL/AT_RISK/DEACTIVATED` | `account.service.ts:74`, scope service :518 |
| F5 | Medium | `ORDER_CHANGE` scope results (11k rows) feed an order-refresh whose only action is commented out | `amazon-events-order-refresh.service.ts:54` |
| F6 | Medium | Legacy account-status parser reads `accountStatus`/`status` keys, but Amazon sends `currentAccountStatus`. It falls back to regex over the whole JSON, which also contains `previousAccountStatus`. A `DEACTIVATED → NORMAL` recovery matches `deactivated` first and marks the channel `DEACTIVATED` | `amazon-notifications.service.ts:3642-3670` |
| F7 | Low | AC listing-issue evidence SQL joins 5 empty amazon-event tables on every request, without the flag gate used elsewhere | `action-center.service.ts:3350-3480` |
| F8 | Low (security) | Shared webhook secret logged in plain text on every EventBridge call | `amazon-notifications.service.ts:465-467` |
| F9 | Low | `PRICING_HEALTH`: 25 channels "SUBSCRIBED", 0 deliveries ever. `ANY_OFFER_CHANGED`: 29/29 subscriptions `ERROR` | `tbl_amazon_event_subscriptions` |
| F10 | Low | `REPORT_PROCESSING_FINISHED` is 71% of inbox volume and 99.99% `unhandled_report_type` | legacy handler :2264 |
| F11 | Low | Dead branch in price-policy lifecycle (both branches call `openOccurrence`) | `price-policy.service.ts:622-635` |
| F12 | Low | `ensureIngestSecret` endpoint unreachable in practice (`AMAZON_EVENTS_SHADOW_INGEST_KEY` unset → 503) | `raw.service.ts:1187` |

---

## 7. Appendix — DB snapshot (read-only, 2026-10-07)

| Table | Rows | Size |
|---|---|---|
| raw_deliveries | 69,913 | 239 MB |
| raw_messages | 59,830 | 90 MB |
| scope_resolution_results | 11,304 | 27 MB |
| raw_processing_attempts | 69,908 | 24 MB |
| subscriptions | 140 | 168 kB |
| scope_aliases | 29 | 168 kB |
| normalized_observations | 27 | 160 kB |
| derivation_generations | 4 | 144 kB |
| scope_identities | 1 | 128 kB |
| destinations | 4 | 120 kB |
| projection_candidates, suppression_edges, occurrences, action_center_materializations, notification_outbox, and all 5 revision tables + occurrence_sources | 0 | 24–88 kB each |

Inbox by type:

| Type | Deliveries | Share |
|---|---|---|
| `REPORT_PROCESSING_FINISHED` | 49,367 | 71% |
| `ORDER_CHANGE` | 13,702 | 20% |
| `LISTINGS_ITEM_ISSUES_CHANGE` | 5,956 | — |
| `FEED_PROCESSING_FINISHED` | 443 | — |
| `LISTINGS_ITEM_STATUS_CHANGE` | 390 | — |
| `ANY_OFFER_CHANGED` | 26 | — |
| `FBA_INVENTORY_AVAILABILITY_CHANGES` | 15 | — |
| `ACCOUNT_STATUS_CHANGED` | 5 | — |
| `APPLICATION_OAUTH_*` | 6 | — |
| untyped | 3 | — |

Migrations that created these tables (all `db-migrations/src/database/migrations/`): `20260422153000` raw layer, `20260422190000` parse-status hardening, `20260423103000` scope, `20260423130000` account/occurrence, `20260423170000` + `20260423193000` enum extensions, `20260423213000` + `20260423220000` suppression (the second only extends the generation-type enum despite the identical name), `20260423233000` suppression revisions, `20260424003000` + `20260424013000` projection, `20260424113000` AC materialization, `20260424153000` notification outbox, `20260424213000` subscriptions control plane, `20260424230000` cleanup indexes.

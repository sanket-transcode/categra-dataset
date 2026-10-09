# Amazon Events — Code Improvement Plan

> Status: **plan only, nothing executed.** Written 2026-10-08, after the stalled-table removal (`latest-task-backup.md`) landed.
> Repos: `api`, `webapp`, `db-migrations`, and `admin`. The user brought the admin repo into scope.
> Baseline: the six removal migrations (`20261008035156` … `20261008035218`) are already applied on `categra_dev`. Only `tbl_amazon_event_raw_deliveries` and `tbl_amazon_event_subscriptions` remain.

---

## 0. TL;DR

The removal left three loosely related folders and a 3.8k-line god-service:

- `amazon-notifications/`: webhook + every handler.
- `amazon-events/raw/`: the inbox.
- `notifications/amazon-events-auto-subscription/`: the control plane.

The admin side has a second, near-copy of the control plane. Notification types are listed in 7 places, and the Amazon envelope is parsed in 3 places.

**Target:** one `AmazonEventsModule` master module, composed of focused sub-modules. This mirrors `ReadinessModule`. The design patterns come from `LlmModule`:

- a **notification-type registry**: a single table that drives subscribing, routing, queueing and handling;
- a **handler factory**: type → handler strategy;
- a **dispatcher (orchestrator)**: dedupe → resolve handler → run → record outcome.

**Every** Amazon notification is acknowledged after the raw row is stored, then handled on BullMQ. Adding a future notification type means one registry entry plus one handler class.

Constants: the `core/constants/amazon-events/` folder (5 files) is **deleted**. Its live content is spread across the three shared files:

- value maps go to `GlobalEnums` (`global-enums.ts`);
- data tables, the registry and types go to `constants.ts`;
- pure functions go to `common/lib/utils.ts`.

Duplicates collapse and dead entries are dropped (§3.6).

Data changes:

- `raw_deliveries` loses 13 dead columns, stores its payload once, gains an indexed `provider_subscription_id`, and gets a 90-day retention cron.
- `subscriptions` gains `ITEM_PRODUCT_TYPE_CHANGE`.

---

## 1. Decisions (from the clarification round)

| # | Topic | Decision |
|---|---|---|
| D1 | Processing model | All webhook notifications are processed through queues, now and in future. The webhook only persists and enqueues. Concurrency, limiter and retry are tuned per queue. New types must plug in without touching the plumbing. |
| D2 | Duplicates | Skip deliveries whose `dedupe_status = DUPLICATE`. Mark them `IGNORED` / `duplicate_delivery`; no handler runs. |
| D3 | `REPORT_PROCESSING_FINISHED` | Store the raw row. Fast-ignore every report type we don't request (today only `GET_XML_BROWSE_TREE_DATA` is used) before any DB lookup. |
| D4 | `ITEM_PRODUCT_TYPE_CHANGE` | Add it to auto-subscription (EventBridge) and keep the handler. |
| D5 | Admin contract | The admin repo is in scope, so admin API URLs and shapes may change. Drop the "shadow" naming. |
| D6 | Behaviour fixes | Four fixes are in: account-status parser, no deleting a shared remote subscription on disable, one SKU matcher, periodic reconcile cron. |
| D7 | Raw table | 90-day retention. Drop the 13 no-value columns (§6). Add `provider_subscription_id`. Store the payload once. A migration rewrites existing rows; code reads one shape only. |
| D8 | Constants location | Delete every file in `core/constants/amazon-events/`. Each item moves to `global-enums.ts`, `constants.ts` or `common/lib/utils.ts`, whichever fits; mapping in §3.6. New Amazon-events constants created by this refactor follow the same rule: no new constants files. |

---

## 2. Current state (post-removal)

### 2.1 Code inventory

| Area | File(s) | Lines | Role |
|---|---|---|---|
| Webhook + handlers | `platforms/amazon/amazon-notifications/amazon-notifications.service.ts` | 3,801 | Auth, dispatch (`if/else` per type, twice: SQS and EventBridge), 7 handlers, order routing, logging, payload accessors, user notifications |
| Webhook controller | `…/amazon-notifications.controller.ts` | 166 | `POST /amazon-notifications/sqs` and `/event-bridge`. Persists raw, then runs handlers **inside the request** |
| Inbox | `platforms/amazon/amazon-events/raw/amazon-events-raw.service.ts` + controller | 871 + 26 | Envelope unwrap, hint extraction, dedupe, outcome marking. Also the dead `internal/amazon-events/shadow/raw/ingest` endpoint (secret never configured) |
| Control plane | `platforms/amazon/notifications/amazon-events-auto-subscription/…service.ts` | 1,446 | Reconcile / provision rows and remote subscriptions |
| Admin control plane | `admin/amazon-events/amazon-events-subscriptions.service.ts` | 1,609 | Overview, rows, receiving evidence, plus its **own** subscribe / recheck / disable implementation |
| Admin inbox viewer | `admin/amazon-events-shadow-raw/*` | 440 | Deliveries list / detail |
| Admin catalog | `admin/amazon-events/amazon-events.service.ts` + `core/constants/amazon-events/amazon-events.catalog.ts` | 183 + 571 | Static reference catalog |
| Constants | `core/constants/amazon-events/*` (5 files) | ~860 | Types, statuses, flags, payload versions, destinations |
| Order worker | `sync/queue/processors/order-notification.processor.ts` | 52 | Shared Amazon + Shopify order-notification queue |

### 2.2 Ambiguity and redundancy

**Notification-type knowledge lives in 7 places:**

- `AmazonNotificationsService.supportedNotifications`
- `AMAZON_EVENTS_AUTO_SUBSCRIPTION_SUPPORTED_NOTIFICATION_TYPES`
- `AMAZON_EVENT_ADMIN_SUPPORTED_NOTIFICATION_TYPES` (an alias)
- `AMAZON_EVENTS_SQS_DESTINATION_NOTIFICATION_TYPES`
- `AMAZON_EVENT_PAYLOAD_VERSION_BY_NOTIFICATION_TYPE`
- `AMAZON_EVENTS_AUTO_SUBSCRIPTION_FAMILY_FLAGS`
- the catalog's `ACTIVE_LEGACY_PROTECTED_TYPES`, plus the admin UI's `AMAZON_RECONCILE_NOTIFICATION_TYPES`

They already disagree: `ITEM_PRODUCT_TYPE_CHANGE` has a handler, a flag and a payload version, but is not subscribable.

**The envelope is parsed in 3 places:**

- the raw service (`extractRawNotificationType`, `extractProviderMessageId`, …);
- the notification service (`getAmazonNotificationType` / `Payload` / `Metadata` / `SafeContext`);
- order queue context (`buildOrderNotificationQueueContext`).

On top of that:

- `processOrderChangeNotification` re-extracts the same fields with its own alias chains.
- `subscriptionId` extraction is copy-pasted 5×.

**The two dispatchers duplicate each other.** `processSqsNotification` and `processEventBridgeNotification` each have their own `if/else` ladder, auth check and outcome marking. Only the type set differs, and that difference is accidental: LISTINGS types are accepted on both.

**The listing-issue handler duplicates itself** (~600 lines). Its variant branch and parent branch repeat the same 12 steps:

- token → `getProduct` → fallback persist → persist state → product-type gate → `processAmazonIssues`

The listing-status handler repeats the same target resolution again, with a different SKU matcher (exact only, against percent-decoded for issues). It also runs in parallel, where issues run sequentially.

**Feed matching is three copy-pasted blocks** (forward-sync / product-delete / close-listing). They differ only in Redis key, status field, job id and queue.

**Order routing is resolved twice.** It happens at enqueue in the webhook and again in the worker (`processOrderChangeNotification`). The worker also sets `request.user.id` from `subscription.account.userCompanies`, which the candidate never loads, so the value is always `undefined`.

**About 15 helpers are duplicated** between auto-subscription and the admin subscriptions service:

- `resolveGrantlessRegionForChannel`, `resolveSellerRegionForChannel`, `inferGrantlessRegionFromMarketplaces`, `getSellerAccessToken`
- `validateRemoteSubscriptionShape`, `requiresOrderChangeDirective`, `extractRemoteEventFilterType`, `resolveProviderPayloadVersion`
- `throwIfAmazonRequestFailed`, `isAmazonNotFoundError`, `extractErrorCode`, `extractSafeErrorDetail`
- `normalizeOptional*` ×3, `toNullableNumber`, `isGrantlessRegion`

Admin `subscribe` / `recheck` / `disable` re-implement what the provisioner does, but without sibling reuse or remote discovery.

**Constants are duplicated:**

- the grantless region list and scope exist twice (`AMAZON_EVENTS_AUTO_SUBSCRIPTION_GRANTLESS_*` and `AMAZON_EVENT_ADMIN_GRANTLESS_*`);
- `AMAZON_ORDER_CHANGE_SUBSCRIBED_STATUS` / `AMAZON_ROUTABLE_SUBSCRIPTION_STATUSES` are local to the handler file.

**Config is read directly from `process.env` in 4 files:**

- auth key, 2 destination ids, 8 family flags, 2 allowlists, region fallback, 3 application ids, shadow ingest key.
- Nothing is validated at boot. A missing destination id surfaces only as row `ERROR`s.

**Logging is scattered:**

- `console.log` / `warn` / `error` are mixed with `_productAttributeService.logError(...)`, which borrows an unrelated service just to reach `BaseService.logError`.
- The skip-code catalogue (`getAmazonNotificationSkipDefaults`) lives in the handler.
- `console.log('authToken: ', authToken)` prints the shared secret on every EventBridge call.

**Dead code:**

- the shadow ingest endpoint plus `ensureIngestSecret`;
- `AmazonConnectorService.createGrantlessDestination` / `getGrantlessDestination`, which no longer have callers;
- the admin `POST /subscriptions/setup`, which the admin UI never calls;
- `AMAZON_EVENT_CANONICALIZATION_STATUSES`, `AMAZON_EVENT_PAYLOAD_STORAGE_KINDS`, `AMAZON_EVENT_RAW_INGEST_*`;
- the `'AMAZON_EVENT'` Action Center evidence source type, which nothing produces any more;
- the admin home page still links to 6 removed shadow pages (`/amazon-events/scope-resolution`, `…-shadow`), left over from the removal task.

### 2.3 What the dev data says (read-only, `categra_dev`, 2026-10-08)

**Size and age.** 193,947 deliveries, **820 MB**, oldest from 2026-05-13, at 20–63k rows/month.

**Duplicates.** 128,013 rows (66%) are `DUPLICATE`.

- `LISTINGS_ITEM_ISSUES_CHANGE`: **113,600 of 116,913 are duplicates**, with a median gap to the first copy of 9.6 h (p90 21 h). Up to 10+ copies per message.
- **Cause:** EventBridge retries because the handler runs synchronously and calls SP-API (`getProduct`) inside the request. Every retry repeats the SP-API call and the issue rewrite. This is the strongest argument for D1 + D2.

**Envelope.** `kind = DIRECT` on 100% of rows, so `payload_json` always holds `rawPayload` and `normalizedPayload` as identical copies, plus `envelope`.

**`meta_json` is 203 MB, larger than the payloads (166 MB).** Its contents are `eventHints`, `orderChangeHints` and `identityHints` (41 / 36 / 31 MB), plus copies of 8 real columns.

**Constant or empty columns:**

- `transport_delivery_id`, `transport_request_id`, `payload_text`: always null.
- `payload_storage_kind`: always `INLINE_JSON`.
- `ingest_source`: 1:1 with `transport_kind`.
- `parse_status`: `PARTIAL` on 99.99% of rows, because of the warning `raw_notification_type_supplied_by_meta`. `parse_error_summary` also collects unrelated processing errors ("Validation error", "Connection terminated").
- `headers_json`: only ever `content-type`.

**Processing outcomes:**

- `ORDER_CHANGE`: 17,638 of 17,699 are `UNMATCHED`. The shared SQS destination delivers notifications for sellers subscribed by other environments.
- `REPORT_PROCESSING_FINISHED`: 57,068 `IGNORED`, as unhandled report types.
- 837 rows are stuck at `RECEIVED`: the process died mid-request.

**Subscriptions.** 1,222 rows:

- ACCOUNT_STATUS_CHANGED: 41 `ERROR`
- ORDER_CHANGE: 40 `ERROR`
- FULFILLMENT_ORDER_STATUS: 125 `ERROR`
- ITEM_PRODUCT_TYPE_CHANGE: 1 `SUBSCRIBED`, which receives deliveries but is never re-provisioned

---

## 3. Target architecture

### 3.1 Module tree

The readiness pattern: one master module, sub-modules grouped by responsibility, each with a header comment saying what it owns.

```
api-main/src/modules/app/platforms/amazon/events/
├── amazon-events.module.ts                    ← master: imports + re-exports the sub-modules below
│                                                (the registry, enums, types and pure helpers live in
│                                                 global-enums.ts / constants.ts / common/lib/utils.ts, §3.6)
├── config/
│   ├── amazon-events-config.module.ts
│   └── amazon-events-config.service.ts        ← every env read; validated on bootstrap
├── envelope/
│   └── amazon-notification-envelope.parser.ts ← the ONE parser → AmazonNotificationEnvelope
├── inbox/                                     ← raw_deliveries owner
│   ├── amazon-events-inbox.module.ts
│   ├── amazon-events-inbox.service.ts         ← persist, dedupe, mark outcome
│   └── amazon-events-inbox-retention.service.ts
├── intake/                                    ← HTTP edge
│   ├── amazon-events-intake.module.ts
│   ├── amazon-notifications.controller.ts     ← same URLs (/amazon-notifications/sqs | /event-bridge)
│   └── amazon-notification-intake.service.ts  ← auth → persist → short-circuit → enqueue → 200
├── dispatch/                                  ← orchestrator + factory (LLM pattern)
│   ├── amazon-events-dispatch.module.ts
│   ├── amazon-notification-dispatcher.service.ts
│   ├── amazon-notification-handler.factory.ts
│   └── amazon-notification-handler.interface.ts
├── routing/
│   └── amazon-subscription-route.resolver.ts  ← providerSubscriptionId → routed channels (+ marketplace / order-sync gates)
├── handlers/                                  ← one strategy per family; sub-modules only where a family needs extra deps
│   ├── amazon-events-handlers.module.ts
│   ├── order/order-change.handler.ts                  (ORDER_CHANGE + FULFILLMENT_ORDER_STATUS)
│   ├── order/new-order-notifier.service.ts
│   ├── listing/listing-target.resolver.ts             ← SKU matcher + PC/PCM/VM resolution (shared)
│   ├── listing/listing-issues.handler.ts
│   ├── listing/listing-status.handler.ts
│   ├── listing/listing-state.refresher.ts             ← token → getProduct → persist state (shared)
│   ├── product-type/product-type-change.handler.ts
│   ├── product-type/product-type-change.notifier.ts
│   ├── feed/feed-processing-finished.handler.ts       ← table of FeedJobMatchers
│   ├── report/report-processing-finished.handler.ts   ← table of ReportTypeHandlers
│   └── account/account-status-changed.handler.ts
├── logging/
│   └── amazon-notification-log.service.ts     ← skip/failure code catalogue + error-log persistence
└── subscriptions/                             ← control plane
    ├── amazon-events-subscriptions.module.ts
    ├── amazon-subscription.client.ts          ← SP-API calls + error helpers (no business logic)
    ├── amazon-notification-region.resolver.ts
    ├── amazon-subscription-provisioner.service.ts  ← per-row: subscribe / verify / discover / reuse sibling / disable
    └── amazon-subscription-reconcile.service.ts    ← bulk reconcile, provisionForAuthorizedChannel, cron entry
```

```
api-main/src/modules/admin/amazon-events/      ← presentation only; no SP-API logic
├── amazon-events-admin.module.ts               ← master (imports app AmazonEventsSubscriptionsModule + InboxModule)
├── catalog/      amazon-events-catalog.{controller,service}.ts        (was amazon-events.*)
├── subscriptions/amazon-events-subscriptions.{controller,service}.ts  (rows, overview, receiving evidence; actions delegate to provisioner)
├── deliveries/   amazon-events-deliveries.{controller,service}.ts     (was amazon-events-shadow-raw/*)
└── amazon-events-admin-error.ts                (was amazonEventsErrorResponse.ts)
```

The queue processor stays where every processor lives, in `sync/queue/processors/amazon-notification.processors.ts`. It is a thin class that calls `AmazonNotificationDispatcherService.dispatch(job.data)`.

### 3.2 Notification registry (single source of truth)

The registry lives in **`constants.ts`** as `AMAZON_NOTIFICATION_DEFINITIONS`, together with the `AmazonNotificationDefinition` interface. The type names come from `GlobalEnums.AmazonNotificationTypes`, and lookups go through utils (§3.6).

```ts
// constants.ts
export interface AmazonNotificationDefinition {
	type: AmazonNotificationType;              // GlobalEnums.AmazonNotificationTypes.ORDER_CHANGE, …
	destination: 'SQS' | 'EVENTBRIDGE';        // replaces AMAZON_EVENTS_SQS_DESTINATION_NOTIFICATION_TYPES
	payloadVersion: string;                    // replaces AMAZON_EVENT_PAYLOAD_VERSION_BY_NOTIFICATION_TYPE
	autoSubscribe: boolean;                    // replaces *_SUPPORTED_NOTIFICATION_TYPES (both copies)
	familyFlag: string | null;                 // env flag name; replaces AMAZON_EVENTS_AUTO_SUBSCRIPTION_FAMILY_FLAGS
	marketplaceFilter: boolean;                // ORDER_CHANGE processingDirective (replaces requiresOrderChangeDirective ×2)
	queue: AmazonNotificationQueueLane;        // which queue lane handles it (§4)
	routing: 'CHANNEL' | 'CHANNEL_MARKETPLACE' | 'NONE'; // what the route resolver must prove
}
```

| Type | Destination | Auto-subscribe | Lane | Routing |
|---|---|---|---|---|
| `ORDER_CHANGE` | SQS | ✔ | `orders` | CHANNEL_MARKETPLACE + order sync on |
| `FULFILLMENT_ORDER_STATUS` | EVENTBRIDGE | ✔ | `orders` | CHANNEL_MARKETPLACE + order sync on |
| `LISTINGS_ITEM_ISSUES_CHANGE` | EVENTBRIDGE | ✔ | `listings` | CHANNEL_MARKETPLACE |
| `LISTINGS_ITEM_STATUS_CHANGE` | EVENTBRIDGE | ✔ | `listings` | CHANNEL_MARKETPLACE |
| `ITEM_PRODUCT_TYPE_CHANGE` | EVENTBRIDGE | ✔ **(new, D4)** | `listings` | CHANNEL_MARKETPLACE |
| `FEED_PROCESSING_FINISHED` | SQS | ✔ | `control` | NONE (matched by feed id) |
| `REPORT_PROCESSING_FINISHED` | SQS | ✔ | `control` | CHANNEL (only for handled report types) |
| `ACCOUNT_STATUS_CHANGED` | SQS | ✔ | `control` | CHANNEL |

**Everything derives from this table:**

- the auto-subscription type list;
- the admin "supported" set and the admin UI reconcile list, which the API returns in the overview instead of a hard-coded TS array;
- destination selection and payload versions;
- the handler factory's coverage check;
- the catalog's "handled" flag.

The catalog keeps its descriptive docs for unsupported types, but stops keeping its own legacy-protected set.

**Boot-time assertion.** Every registry type with `autoSubscribe` must have a handler registered in the factory. Every handler must declare registry types only.

### 3.3 Flow

```
Amazon ──POST /amazon-notifications/{sqs|event-bridge}──▶ AmazonNotificationsController
   └─ AmazonNotificationIntakeService.accept(transport, body, headers)
        1. auth (x-custom-auth-amazon vs config)          → 401-style FAILED row, no enqueue
        2. envelope = EnvelopeParser.parse(body)          (type, notificationId, subscriptionId, publishTime, payload)
        3. delivery = InboxService.record(transport, envelope)   (dedupe → first_delivery_id)
        4. short-circuit (no job):
             • DUPLICATE                → IGNORED  duplicate_delivery          (D2)
             • type not in registry     → IGNORED  unsupported_notification_type
             • REPORT with unrequested reportType → IGNORED report_type_not_requested (D3)
             • parse failure / no type  → IGNORED  notification_type_missing
        5. enqueue {deliveryId} on registry[type].queue, jobId = amazon-notification:<deliveryId>
           → delivery status QUEUED
        6. 200 {accepted:true}   (only persistence/enqueue failure → 503 so Amazon retries)

Worker ── AmazonNotificationProcessor(lane) ──▶ DispatcherService.dispatch(deliveryId)
        1. load delivery, re-parse envelope from payload_json (job carries only the id)
        2. handler = HandlerFactory.resolve(type)
        3. routes = RouteResolver.resolve(envelope, registry[type].routing)   (skipped when routing = NONE)
             • no route → UNMATCHED <reason>
        4. outcome = handler.handle({ envelope, routes, delivery })
        5. InboxService.markOutcome(deliveryId, outcome)       (PROCESSED / IGNORED / UNMATCHED / FAILED)
        6. retryable error → rethrow (BullMQ retry); final attempt → FAILED with error
```

**Handler contract.** It mirrors `ILlmProvider`:

```ts
export interface AmazonNotificationHandler {
	readonly notificationTypes: readonly AmazonNotificationType[];
	handle(context: AmazonNotificationContext): Promise<AmazonNotificationOutcome>;
}
// outcome = { status: PROCESSED|IGNORED|UNMATCHED, reason: string, detail?: Record<string, unknown> }
// handlers RETURN business skips as outcomes and THROW only for retryable/unexpected failures.
```

Today handlers swallow every error and the dispatcher stamps `PROCESSED` anyway. Under the contract above, a failed listing write becomes `FAILED` and is retried, instead of being silently "processed".

### 3.4 Component details

**Config — `AmazonEventsConfigService`.** Modelled on `LlmConfigService`.

- Holds every `AMAZON_*` env read for this domain:
  - `AMAZON_NOTIFICATION_AUTH_KEY`
  - `AMAZON_SQS_DESTINATION_ID` / `AMAZON_EVENT_BRIDGE_DESTINATION_ID`
  - `AMAZON_EVENTS_AUTO_SUBSCRIBE_ENABLED` plus the family flags (names from the registry)
  - the account/channel allowlists
  - `AMAZON_EVENTS_ADMIN_SP_API_REGION` (renamed `AMAZON_EVENTS_DEFAULT_SP_API_REGION`; the old name is still read for one release)
  - the three application ids
  - `AMAZON_EVENTS_RAW_RETENTION_DAYS` (default 90)
  - per-lane concurrency overrides
- `onApplicationBootstrap` warns once if auto-subscribe is enabled and a destination id or the auth key is missing.
- Replaces `readBooleanFlag` / `readNumericAllowlist`, `resolveAmazonEventDestination` and the direct `process.env` reads.
- `AMAZON_EVENTS_SHADOW_INGEST_KEY` is deleted.

**Envelope — `AmazonNotificationEnvelopeParser`.** A pure function, unit-testable.

- Handles the shapes seen today: EventBridge `detail.*`, SQS direct `{notificationType, payload, notificationMetadata}`, and SNS-wrapped `Message` (kept for safety even though 0 rows use it).
- Returns `{ notificationType, notificationId, subscriptionId, applicationId, publishTime, payload, transport }`, plus typed accessors per family:
  - `orderChange()` → `{sellerId, amazonOrderId, marketplaceId, changeType, signature}`
  - `listing()` → `{sellerId, marketplaceId, sku, asin, status, issues, productType, previousProductType, itemName}`
  - `feed()`, `report()`, `accountStatus()`
- Replaces the 3 parsers and every alias chain. The PascalCase/camelCase alias lists live here once.
- `safeLogContext()` replaces `getAmazonNotificationSafeContext`.

**Inbox — `AmazonEventsInboxService`.** About 200 lines.

- `record()`: computes `payload_byte_size`, sets `provider_subscription_id`, and dedupes by `provider_message_id`. `firstDeliveryId` behaves as today.
- `markOutcome()`: writes only the `processing_*` columns. No more `meta_json` copies.
- Hint extraction (`extractIdentityHints`, `extractOrderChangeHints`, `extractEventHints`, `walkPayload`) and the semantic key are deleted.
- The raw controller and `ingest()` are deleted.

**Retention — `AmazonEventsInboxRetentionService.prune()`.** Called from a new `CronService` job, `@Cron('20 4 * * *')`, written in the same style as `pruneAmazonProductDataHistory`: local-env guard and `isForceRun`.

- Deletes rows with `ingested_at < now() - retentionDays` in batches of 5,000.
- `first_delivery_id` is `ON DELETE SET NULL`, so pruning first copies is safe.

**Route resolver — `AmazonSubscriptionRouteResolver`.** One query builder replaces 4:

- `resolveOrderChangeSubscriptionsForNotification`
- `findListingNotificationSubscriptions`
- the account-status lookup
- the report lookup

Input: `{subscriptionId, notificationType, sellerId?, marketplaceId?, requireOrderSync?}`.

Output: `AmazonNotificationRoute[]`, each `{subscriptionRowId, accountId, channelId, amazonChannel, amazonChannelMarketplace?, amazonMarketplace?, region}`. The result is deduped per (amazonChannel, marketplace), which absorbs `dedupeListingNotificationSubscriptions`.

The status gate is `AMAZON_ROUTABLE_SUBSCRIPTION_STATUSES` (`SUBSCRIBED`, `PENDING_VERIFICATION`). It moves to the constants file and is used by **every** type. Today ORDER_CHANGE alone requires `SUBSCRIBED`.

**Handlers.**

- **Order** (`ORDER_CHANGE`, `FULFILLMENT_ORDER_STATUS`):
  - Body moved from `processOrderChangeNotification`.
  - Routing happens once, in the worker. For multiple routes the handler processes each, exactly as the webhook fan-out did.
  - The request user is taken from `UserAccount` (the account's first accepted admin), fixing the `undefined` user id.
  - `notifyNewOrder` moves to `NewOrderNotifierService`.
  - The Amazon branch leaves `ProcessOrderNotificationProcessor`, which stays Shopify-only.
- **Listing issues / status:**
  - `ListingTargetResolver.resolve(route, sku)` → `{productId, productChannel, pcm, variantId?, variantMarketplace?}`. One SKU matcher (exact, then percent-decoded) per D6; variant and parent are one code path with `variantId` nullable.
  - `ListingStateRefresher` does the shared token → `getProduct` → `persistAmazonListingStateFromListingData`, with the event-issues fallback when the token or SP-API fails.
  - Issues handler: resolve → ASIN update/conflict → refresh → product-type gate → `processAmazonIssues`. This replaces ~600 duplicated lines with ~150.
  - Status handler: resolve → ASIN update → refresh → catalog metadata update. It keeps its catalog `getAmazonProducts` + `AmazonProductData.metaData` step.
- **Product type:**
  - The handler finds affected PCMs by ASIN across the routes, then enqueues `PRODUCT_TYPE_CHANGE` (existing queue).
  - The notifier keeps today's in-app notification. `resolveProductSkuAndName` moves with it.
  - The wrong `handler: 'processListingItemStatusChange'` log labels are fixed.
- **Feed.** `FeedJobMatcher[]`, three entries for forward-sync, product-delete and close-listing: `{redisKey(feedId), pendingStatus, statusPath, jobId(feedId), queue}`. A single loop replaces the 3 copies; behaviour is identical.
- **Report.** `ReportTypeHandler` map `{GET_XML_BROWSE_TREE_DATA: categoryTreeReport}`.
  - The intake reads the same map to fast-ignore (D3). Requesting a new report type later means adding one entry.
- **Account status (D6 fix).** The parser reads `accountStatusChangeNotification.currentAccountStatus` (alias-tolerant) and maps:
  - `NORMAL` → CONNECTED
  - `AT_RISK` → CONNECTED (log warning)
  - `DEACTIVATED` → DEACTIVATED

  The whole-payload regex fallback is removed, so `previousAccountStatus` can no longer win. Unknown values → `IGNORED unknown_account_status`.

**Logging — `AmazonNotificationLogService`.**

- `skip(code, envelope, extra)` / `failure(code, envelope, error, extra)` hold the code→defaults catalogue now in `getAmazonNotificationSkipDefaults`.
- Persists through `BaseService.logError` with `className: 'AmazonNotificationsService'` kept. Admin `errors.service.ts` keys its source mapping on that class name; keeping it avoids an admin errors regression. The mapping is also switched to read `sourceLabel` / `endpoint` from the context, which already carries them.
- Removes the `_productAttributeService.logError` borrowing and the secret-logging `console.log`.

**Subscriptions (control plane):**

- `AmazonSubscriptionClient`: create / get / getGrantlessById / deleteGrantlessById wrappers, `throwIfFailed`, `isNotFound`, `extractErrorCode`, `extractSafeErrorDetail`, `extractProviderSubscriptionId`, `extractEventFilterType`.
- `AmazonNotificationRegionResolver`: `grantlessRegion(channel)`, `sellerRegion(channel)` (marketplace country map + default region config).
- `AmazonSubscriptionProvisionerService`, per row:
  - `ensureSubscribed(row, channel)`: verify local → reuse sibling → discover remote → create.
  - `verify(row, channel)`: the admin recheck.
  - `disable(row, channel)`: the admin disable. **D6 fix:** when another row with the same `provider_subscription_id` is still routable, only the local row is unlinked; otherwise the remote is deleted.
  - Admin `subscribe` now goes through `ensureSubscribed`, gaining sibling reuse and discovery.
- `AmazonSubscriptionReconcileService`: today's `reconcile` / `provisionForAuthorizedChannel` / `ensureLocalRows` / summary.
  - **D6:** `CronService` gets `@Cron('15 */6 * * *') reconcileAmazonEventSubscriptions()`, with the local-env guard. It calls `reconcile({ onlyUnhealthy: true })`, which still runs `ensureLocalRows`, so new registry types get rows. Remote work is limited to rows not `SUBSCRIBED`, processed sequentially, so `PENDING_VERIFICATION` and `ERROR` rows are verified or repaired without waiting for a channel edit.
  - The callers (`amazon-channel.service`, `channel.service`, `amazon-channel-marketplace.service`) keep the same method names on the new service.

**Admin presentation:**

- The subscriptions service keeps rows, overview, receiving evidence, filters and action-state presentation (~700 lines). All SP-API and row-mutation logic delegates to the provisioner.
- **Receiving evidence** becomes `MAX(ingested_at)` per `provider_subscription_id` (indexed), where `dedupe_status <> 'DUPLICATE'`. It is an exact match to the row, replacing the seller/marketplace hint guessing (`loadReceivingResolutionChannels`, `normalizeIdentityHintValues`). For sellers shared across channels, evidence follows the subscription row itself, so the "ambiguous evidence" state disappears.
- `resolvePresentationMode` / `legacyProtected` are derived from the registry.
- `POST /subscriptions/setup` is removed (unused by admin).

### 3.5 Shared helpers

`normalizeOptionalString` / `Upper` / `Lower` and `toNullableNumber` exist 3× today, across the inbox, auto-subscription and admin. They move once into `common/lib/utils.ts`, reusing any existing equivalent there. No module-local util file is created (D8).

### 3.6 Constants relocation (D8)

`core/constants/amazon-events/` is deleted. Its five files are:

- `amazonEventsRaw.constants.ts`
- `amazonEventsSubscriptionCompatibility.ts`
- `amazonEventsAutoSubscription.constants.ts`
- `amazon-events-subscriptions.constants.ts`
- `amazon-events.catalog.ts`

Their 5 `export *` lines in `core/index.ts` are removed. `global-enums.ts`, `constants.ts` and `common/lib/utils.ts` are already exported through `src/core` / `src/common`, so consumers keep importing from the same barrels.

**Placement rule:**

| Target | Gets |
|---|---|
| `global-enums.ts` | Plain value maps (`static readonly X = { KEY: 'VALUE' }` on `GlobalEnums`), in the style of `ChannelConnectionStatus` / `QueueNames` |
| `constants.ts` | Data tables, the registry, derived lists, numeric limits, env-name maps, job options, and the TS `type` / `interface` declarations that describe them (as it already does with `ProductAttributeTierKey`) |
| `common/lib/utils.ts` | Pure functions: lookups, predicates and builders that read the registry or catalog **at call time** |

**Circular-import guard.** `utils.ts` imports from `src/core`, so `constants.ts` must never call a utils function at module-evaluation time. That's why the catalog is stored as plain seed data in `constants.ts`; derived fields are built lazily in utils (see the catalog rows below).

#### `amazonEventsRaw.constants.ts`

| Export | Destination |
|---|---|
| `AMAZON_EVENT_TRANSPORT_KINDS` | `GlobalEnums.AmazonEventTransportKinds = { SQS, EVENTBRIDGE }`. `DIRECT_HTTP` and `UNKNOWN` are dropped: they were only reachable through the deleted shadow ingest |
| `AMAZON_EVENT_DEDUPE_STATUSES` | `GlobalEnums.AmazonEventDedupeStatuses` (`UNIQUE`, `DUPLICATE`, `UNDETERMINED`) |
| `AMAZON_EVENT_LIVE_PROCESSING_STATUSES` | `GlobalEnums.AmazonEventProcessingStatuses`, adding `QUEUED` (M4 renames the columns to `processing_*`) |
| `AMAZON_EVENT_RAW_MAX_REQUEST_BYTES` | `constants.ts` → `AMAZON_NOTIFICATION_MAX_PAYLOAD_BYTES` (intake guard) |
| `AMAZON_EVENT_INGEST_SOURCES`, `AMAZON_EVENT_PARSE_STATUSES`, `AMAZON_EVENT_CANONICALIZATION_STATUSES`, `AMAZON_EVENT_PAYLOAD_STORAGE_KINDS` | **Deleted**: their columns are dropped in M5 |
| `AMAZON_EVENT_RAW_INGEST_SHARED_SECRET_HEADER`, `AMAZON_EVENT_RAW_INGEST_SECRET_ENV` | **Deleted**, with the shadow ingest endpoint |
| `AMAZON_EVENT_ALLOWED_META_HEADER_KEYS`, `AMAZON_EVENT_ALLOWED_META_KEYS` | **Deleted**: `headers_json` / `meta_json` are dropped |
| The 7 `type AmazonEvent…` aliases (no consumers) | **Deleted**. Any status type still needed is derived next to its use in `constants.ts`: `type AmazonEventProcessingStatus = (typeof GlobalEnums.AmazonEventProcessingStatuses)[keyof …]` |

#### `amazonEventsSubscriptionCompatibility.ts`

| Export | Destination |
|---|---|
| `AMAZON_EVENTS_SQS_DESTINATION_NOTIFICATION_TYPES` | Absorbed into the registry's `destination` field |
| `AMAZON_EVENT_PAYLOAD_VERSION_BY_NOTIFICATION_TYPE` (private) | Absorbed into the registry's `payloadVersion` field |
| `AMAZON_EVENT_DESTINATION_TYPES` | `GlobalEnums.AmazonEventDestinationTypes` (`SQS`, `EVENTBRIDGE`) |
| `AMAZON_EVENT_DESTINATION_ENV_BY_TYPE` | `constants.ts`, unchanged name. Read only by `AmazonEventsConfigService` |
| `AmazonEventDestinationType`, `AmazonEventDestinationConfig` | `constants.ts` types |
| `normalizeNotificationType` (private) | `utils.ts` → `normalizeAmazonNotificationType(value)` |
| `getAmazonEventCompatiblePayloadVersion` | `utils.ts` → `getAmazonNotificationPayloadVersion(type, currentValue?)`. Also absorbs the two private `resolveProviderPayloadVersion` copies |
| `getAmazonEventRequiredDestinationType` | `utils.ts` → `getAmazonNotificationDestinationType(type)` |
| `requiresAmazonEventSqsDestination` (no consumers) | **Deleted** |
| `resolveAmazonEventDestination` | Split: the type lookup is the utils function above; the env read becomes `AmazonEventsConfigService.getDestination(type)`, which throws the same "not configured" error |

#### `amazonEventsAutoSubscription.constants.ts`

| Export | Destination |
|---|---|
| `AMAZON_EVENTS_AUTO_SUBSCRIPTION_SUPPORTED_NOTIFICATION_TYPES` | `constants.ts` → `AMAZON_AUTO_SUBSCRIBE_NOTIFICATION_TYPES`, **derived** from the registry (`autoSubscribe: true`). Also replaces the admin alias |
| `AMAZON_EVENTS_AUTO_SUBSCRIPTION_GRANTLESS_REGIONS` | `GlobalEnums.AmazonGrantlessRegions = { NA: 'na', EU: 'eu', FE: 'fe' }`. One copy, which absorbs `AMAZON_EVENT_ADMIN_GRANTLESS_REGIONS`; `type AmazonGrantlessRegion` goes in `constants.ts` |
| `AMAZON_EVENTS_AUTO_SUBSCRIPTION_GRANTLESS_SCOPE` | `constants.ts` → `AMAZON_NOTIFICATIONS_GRANTLESS_SCOPE`. One copy, which absorbs `AMAZON_EVENT_ADMIN_GRANTLESS_SCOPE` |
| `AMAZON_EVENTS_AUTO_SUBSCRIPTION_SHARED_DESTINATION_SCOPE` | **Deleted**: a destination-table leftover that is only written into `configuration_snapshot_json.destinationScope` and never read |
| `AMAZON_EVENTS_AUTO_SUBSCRIPTION_SUBSCRIPTION_STATUSES` | `GlobalEnums.AmazonEventSubscriptionStatuses` (`NOT_SUBSCRIBED`, `SUBSCRIBED`, `PENDING_VERIFICATION`, `ERROR`). One copy, which absorbs `AMAZON_EVENT_ADMIN_SUBSCRIPTION_STATUSES` |
| `AMAZON_EVENTS_AUTO_SUBSCRIPTION_FAMILY_FLAGS` | Absorbed into the registry's `familyFlag` field |
| `AMAZON_EVENTS_AUTO_SUBSCRIPTION_MARKETPLACE_COUNTRY_REGION_MAP` | `constants.ts` → `AMAZON_MARKETPLACE_COUNTRY_GRANTLESS_REGION_MAP` |
| `isAmazonEventsAutoSubscriptionNotificationTypeSupported` | `utils.ts` → `isAmazonAutoSubscribeNotificationType(type)`. One function, which absorbs `isAmazonEventAdminNotificationTypeSupported` |

#### `amazon-events-subscriptions.constants.ts`

| Export | Destination |
|---|---|
| `AMAZON_EVENT_ADMIN_DESTINATION_STATUSES` | `GlobalEnums.AmazonEventDestinationStatuses` (`MISSING`, `CONFIGURED`) |
| `AMAZON_EVENT_ADMIN_SUBSCRIPTION_STATUSES` | Duplicate → `GlobalEnums.AmazonEventSubscriptionStatuses` |
| `AMAZON_EVENT_ADMIN_RECEIVING_STATUSES` | `GlobalEnums.AmazonEventReceivingStatuses` (`RECEIVING`, `NOT_RECEIVING`, `UNKNOWN`) |
| `AMAZON_EVENT_ADMIN_SUPPORTED_NOTIFICATION_TYPES` / `…_SET` | Duplicate → `AMAZON_AUTO_SUBSCRIBE_NOTIFICATION_TYPES` (the Set is replaced by the utils predicate) |
| `AMAZON_EVENT_ADMIN_GRANTLESS_REGIONS`, `AMAZON_EVENT_ADMIN_GRANTLESS_SCOPE` | Duplicates → see above |
| `AMAZON_EVENT_ADMIN_RECEIVING_LOOKBACK_DAYS` | `constants.ts` → `AMAZON_EVENT_RECEIVING_LOOKBACK_DAYS` |
| `AMAZON_EVENT_ADMIN_STATUS_FILTER_VALUES` | `constants.ts` → `AMAZON_EVENT_SUBSCRIPTION_STATUS_FILTER_VALUES`, built from the three `GlobalEnums` maps |
| `isAmazonEventAdminNotificationTypeSupported` | Duplicate → `isAmazonAutoSubscribeNotificationType` |

#### `amazon-events.catalog.ts`

| Export / internal | Destination |
|---|---|
| `AmazonEventNoiseLevel`, `AmazonEventSupportLevel`, `AmazonEventObservedRuntime`, `AmazonEventInitialCatalogStatus`, `AmazonEventCatalogEntry`, `AmazonEventCatalogSeed` | `constants.ts` types |
| `COMMON_NOTIFICATION_DOCS` | `constants.ts` → `AMAZON_NOTIFICATION_COMMON_DOC_URLS` |
| The 26 catalog entries (`buildCatalogEntry({...})` calls) | `constants.ts` → `AMAZON_EVENT_CATALOG_SEEDS: AmazonEventCatalogSeed[]`, plain data with no builder call |
| `ACTIVE_LEGACY_PROTECTED_TYPES` / `AMAZON_EVENT_LEGACY_PROTECTED_TYPES` | **Deleted.** "Handled by Categra" is derived from registry membership. The admin catalog field is renamed `handledTypes` (admin is in scope, D5) |
| `OBSERVED_RUNTIME_ONLY_TYPES` / `AMAZON_EVENT_OBSERVED_RUNTIME_ONLY_TYPES` | `constants.ts` → `AMAZON_EVENT_OBSERVED_RUNTIME_ONLY_TYPES` (`ORDER_STATUS_CHANGE`) |
| `buildCatalogEntry` | `utils.ts` → `buildAmazonEventCatalogEntry(seed)`. It computes support, observed-runtime and initial status from the registry plus the observed list |
| `AMAZON_EVENT_CATALOG`, `AMAZON_EVENT_CATALOG_LOOKUP` | `utils.ts` → `getAmazonEventCatalog()` / `getAmazonEventCatalogEntry(type)`. They build on first call and memoise in a module-level variable, so nothing is evaluated during `constants.ts` load |
| `AMAZON_EVENT_CATALOG_VERSION` | `constants.ts`, unchanged |

#### New items introduced by this plan

| Item | Destination |
|---|---|
| `GlobalEnums.AmazonNotificationTypes` (the 8 handled types) | `global-enums.ts`. `type AmazonNotificationType` goes in `constants.ts` |
| The 3 queue lanes (`AMAZON_NOTIFICATION_ORDERS` / `_LISTINGS` / `_CONTROL`) | `GlobalEnums.QueueNames` |
| `AMAZON_NOTIFICATION_DEFINITIONS` + `AmazonNotificationDefinition` | `constants.ts` |
| `getAmazonNotificationDefinition(type)` / `isAmazonHandledNotificationType(type)` | `utils.ts` |
| `AMAZON_ROUTABLE_SUBSCRIPTION_STATUSES` (now a local const in the notification service) | `constants.ts` |
| `AMAZON_HANDLED_REPORT_TYPES` (`['GET_XML_BROWSE_TREE_DATA']`) | `constants.ts`. Read by the intake fast-ignore (D3) and the report handler |
| Feed-matcher table (Redis key prefix, pending status, job-id prefix, queue name; ×3) | `constants.ts` → `AMAZON_FEED_NOTIFICATION_MATCHERS` |
| Skip/failure log-code catalogue (today `getAmazonNotificationSkipDefaults`) | `constants.ts` → `AMAZON_NOTIFICATION_LOG_CODES` |
| Payload alias lists used by the envelope parser (`['SellerId','sellerId','SellerID','sellerID']`, …) | `constants.ts` → `AMAZON_NOTIFICATION_FIELD_ALIASES` |
| Job options for the 3 lanes | `constants.ts`, next to `PROCESS_ORDER_NOTIFICATION`, plus the `queueConfigs` / `SYNC_QUEUE_NAMES` entries (`QUEUE_CONCURRENCY` stays in `queue-concurrency.ts`, where it lives today) |
| Normalize helpers (§3.5) | `utils.ts` |

**Consumers to repoint** (grep for every old name at implementation time):

- api: the entity `amazonEventRawDelivery.entity.ts`, the inbox / intake / dispatcher / handlers / subscriptions services, the admin catalog / subscriptions / deliveries services.
- admin: none, since it doesn't import API constants.
- webapp: none.

---

## 4. Queue design (D1)

Lanes are separate BullMQ queues, so slow SP-API work never blocks cheap control events, and each lane gets its own concurrency and limiter. A new type picks a lane in the registry; no new queue is needed unless its cost profile is new.

| Lane / queue name (`GlobalEnums.QueueNames`) | Types | Concurrency | Limiter | Attempts / backoff | Why |
|---|---|---|---|---|---|
| `AMAZON_NOTIFICATION_ORDERS` = `amazon-notification-orders` | ORDER_CHANGE, FULFILLMENT_ORDER_STATUS | 3 | — | 3, exponential 10 s | Same as today's `PROCESS_ORDER_NOTIFICATION` (3). Order fetches use the orders API, whose throttling `AmazonOrderService` already handles |
| `AMAZON_NOTIFICATION_LISTINGS` = `amazon-notification-listings` | LISTINGS_ITEM_ISSUES_CHANGE, LISTINGS_ITEM_STATUS_CHANGE, ITEM_PRODUCT_TYPE_CHANGE | 4 | `{ max: 4, duration: 1000 }` | 4, exponential 15 s | Each job calls `getListingsItem` (SP-API 5 rps / burst 10 per app+seller); stays under it with headroom. After D2 the real load is ~3.3k issues/5 months, not 117k |
| `AMAZON_NOTIFICATION_CONTROL` = `amazon-notification-control` | FEED_/REPORT_PROCESSING_FINISHED, ACCOUNT_STATUS_CHANGED | 5 | — | 3, exponential 5 s | Cheap: Redis key lookup + job promote, or one row update. Report fetch for the browse tree is rare (47 in 5 months) |

All three lanes share these options:

- `jobId: amazon-notification:<deliveryId>`: idempotent enqueue. The existing hash-based order job id is dropped, because dedupe now happens at the inbox.
- `removeOnComplete: { age: 3600, count: 1000 }` and `removeOnFail: { age: 7 * 86400 }`.
- `WORKER_OPTIONS` as everywhere else.

Wiring follows existing conventions:

- Concurrency goes in `QUEUE_CONCURRENCY` (`queue-concurrency.ts`), overridable through the config service. `JobsOptions` constants and the `QUEUE_CONFIG` entries go in `constants.ts`.
- The three lanes are added to `SYNC_QUEUE_NAMES` and registered in `SyncConsumersModule`. One `AmazonNotificationProcessorBase` holds three `@Processor` subclasses that differ only in queue name.
- The job payload is `{ deliveryId }` only. The worker reloads the payload, so job size stays tiny and nothing request-shaped crosses the queue. Today `rawDeliveryContext` and the whole body are serialised.
- `PROCESS_ORDER_NOTIFICATION` remains for Shopify only, with its Amazon `case` removed.

**Rollout safety.** In-flight `PROCESS_ORDER_NOTIFICATION` jobs with Amazon bodies at deploy time are drained by keeping the Amazon branch for one release, delegating to the new order handler with a body → envelope adapter. It is deleted in the follow-up release; this is called out in §9.

---

## 5. Behaviour changes users/ops will notice

| Change | Effect |
|---|---|
| Webhook acks after persist + enqueue | Amazon/EventBridge stop redelivering. Duplicate volume should collapse (the 97% `LISTINGS_ITEM_ISSUES_CHANGE` duplicates). Handling becomes asynchronous: seconds of delay instead of in-request. |
| Duplicates skipped | No repeated SP-API calls or issue rewrites for redeliveries. |
| Handler failures are `FAILED` + retried | Previously swallowed and marked `PROCESSED`. Admin delivery stats will show real failures. |
| Unrequested report types | `IGNORED report_type_not_requested` without DB work (reason renamed from `report_processing_finished_unhandled_report_type`). |
| `ITEM_PRODUCT_TYPE_CHANGE` auto-subscribed | All connected channels start receiving product-type-change platform notifications (today only 1 channel is subscribed). |
| Account-status fix | A recovery event no longer flips the channel to `DEACTIVATED`. |
| Disable safety | Disabling one channel's row keeps the remote subscription alive for sibling channels. |
| Same SKU matcher for status + issues | Status events for percent-encoded SKUs now map (previously skipped). |
| 6-hourly reconcile cron | `PENDING_VERIFICATION` / `ERROR` / `NOT_SUBSCRIBED` rows, plus rows missing for new types, are verified and repaired automatically. `SUBSCRIBED` rows are **not** touched by the cron. Today's reconcile does one grantless `getSubscriptionById` per row, and running that over all ~1.2k rows every 6 h would be heavy on a rate-limited API. Manual and authorize-time reconcile keep the full check. |
| Raw table | 90-day history only. The admin delivery drawer loses headers, meta, parse status and ingest-source fields. |
| `RECEIVED` → `QUEUED` | New processing status; rows stuck before enqueue are now distinguishable from rows waiting for a worker. |

---

## 6. Database changes (`db-migrations`)

Create every migration with `npm run migrate:create -- <name>`, one file per kind of operation, and **do not execute**. Order:

| # | Migration (name) | Kind | What |
|---|---|---|---|
| M1 | `add-provider-subscription-id-to-amazon-event-raw-deliveries` | add column | `provider_subscription_id VARCHAR(128) NULL` + index `(provider_subscription_id, ingested_at)` |
| M2 | `backfill-amazon-event-raw-delivery-provider-subscription-id` | data | `UPDATE … SET provider_subscription_id = COALESCE(payload_json #>> '{normalizedPayload,notificationMetadata,subscriptionId}', payload_json #>> '{normalizedPayload,detail,NotificationMetadata,SubscriptionId}', …)`, in batches by id range. 176k of 194k rows on dev have one |
| M3 | `collapse-amazon-event-raw-delivery-payload` | data | Batched `UPDATE … SET payload_json = payload_json->'normalizedPayload' WHERE payload_json ? 'normalizedPayload'`. `down()` rebuilds `{rawPayload, normalizedPayload, envelope:{kind:'DIRECT'}}` (lossless, since every row is `DIRECT`) |
| M4 | `rename-amazon-event-raw-delivery-live-processing-columns` | rename | `live_processing_status/reason/error`, `live_processed_at` → `processing_status/reason/error`, `processed_at`; rename the matching index. The status column is `VARCHAR(32)`, so the new `QUEUED` value needs no schema change |
| M5 | `remove-stale-columns-from-amazon-event-raw-deliveries` | drop columns | Drop `ingest_source`, `transport_delivery_id`, `transport_request_id`, `payload_hash`, `payload_storage_kind`, `payload_text`, `headers_json`, `meta_json`, `parse_status`, `parse_error_summary`, `canonicalization_status`, `semantic_identity_key`, `created_at`. Also drop their indexes (`…canonicalization_ingested_idx`, `…ingest_source_ingested_idx`, `…parse_ingested_idx`, `…payload_hash_idx`, `…transport_identity_idx`) and any enum types created for them. `down()` re-adds them (nullable, with the original defaults) and the indexes. It restores structure only, not data, like the earlier stale-column migrations |

**Notes:**

- `subscriptions` needs no schema change.
- No migration is needed for `ITEM_PRODUCT_TYPE_CHANGE`; reconcile creates its rows.
- Run order on deploy: migrations first, then the API. The new code reads only the new shape (no dual-shape fallback, per D7). Running the old code against the migrated table would break, so this deploy can't be rolled out gradually.
- Update `productDeletionTables.txt` and the channel-deletion SQL in `channel.service.ts` only if they reference dropped columns (verify with grep at implementation time; today they reference the table by `amazon_channel_id` only).

---

## 7. Changes per repo

### 7.1 `api`

1. **Constants (D8).** Delete `core/constants/amazon-events/` (all 5 files) and its 5 barrel lines in `core/index.ts`. Every item moves into `global-enums.ts`, `constants.ts` or `common/lib/utils.ts`, exactly per the §3.6 mapping; duplicates collapse and dead entries are dropped.
2. **New `platforms/amazon/events/` tree** (§3.1). Move code out of:
   - `amazon-notifications/` (deleted once empty);
   - `amazon-events/raw/` (deleted);
   - `notifications/amazon-events-auto-subscription/` (deleted).

   `PlatformsModule` imports `AmazonEventsModule` in place of the 3 old modules. `ChannelModule`, `AmazonChannelMarketplaceModule`, admin and the worker import the specific sub-module they need (`AmazonEventsSubscriptionsModule`, `AmazonEventsDispatchModule`), not the master. This avoids the `forwardRef` web the notification module has today (19 forward refs).
3. **Entity `amazonEventRawDelivery`**: matches M1–M5.
4. **Queues**: enums, `QUEUE_CONCURRENCY`, `QUEUE_CONFIG`, `SYNC_QUEUE_NAMES`, the processor file, the `SyncConsumersModule` registration, and the Amazon branch removed from `order-notification.processor.ts` (after the drain release).
5. **`CronService`**: `pruneAmazonEventDeliveries` (daily) and `reconcileAmazonEventSubscriptions` (6-hourly).
6. **Admin backend**: restructure per §3.1.

   | Old route | New route |
   |---|---|
   | `admin/amazon-events/shadow/raw/{overview,deliveries,deliveries/:id}` | `admin/amazon-events/deliveries/{overview,/,/:id}` |
   | `POST admin/amazon-events/subscriptions/setup` | removed |

   The deliveries detail drops headers, meta and parse fields, and adds `providerSubscriptionId` and `processingStatus`.
7. **Transport**: delete `createGrantlessDestination` / `getGrantlessDestination` from `amazon.service.ts`.
8. **Action Center**: remove the dead `'AMAZON_EVENT'` evidence source type from:
   - `action-center-sync-context.type.ts`
   - `actionCenterSyncContext.ts`
   - `action-center-sync-evaluation.service.ts`
   - `action-center-remediation-policy.service.ts`
   - `action-center-listing-issue-read-model.service.ts`

   Grep first to confirm there are still zero producers.
9. **`admin/errors/errors.service.ts`**: keep the `AmazonNotificationsService` mapping; prefer `endpoint` from context over the hard-coded `/event-bridge`.
10. **Docs**: update `api/CLAUDE.md` (pipeline line) and `.env.example`:
    - remove `AMAZON_EVENTS_SHADOW_INGEST_KEY` if present;
    - add `AMAZON_EVENTS_RAW_RETENTION_DAYS` and `AMAZON_EVENTS_DEFAULT_SP_API_REGION`;
    - document the family flags.

### 7.2 `webapp`

- Remove `'AMAZON_EVENT'` from `ActionCenterEvidenceSourceType` and its filter list (`lib/action-center-remediation-navigation.ts:100,858`).
- Remove it from the `ProductActionCenterIssueDrawer.tsx:11564` help-text branch.

No other webapp surface touches Amazon events.

### 7.3 `admin` (`categra/admin`)

- `RawIngest.tsx`, `amazonEventsRaw.types.ts` and `AmazonEventsRawDeliveryDrawer.tsx` change as follows:
  - point at the new `/amazon-events/deliveries/*` endpoints;
  - drop the removed fields (ingest source, transport delivery id, parse status/summary, headers, meta);
  - rename `live*` to `processing*`;
  - show `providerSubscriptionId` and the `QUEUED` status;
  - rename "Shadow" types.
- `Subscriptions.tsx`:
  - take the reconcile type list from the overview response, not the hard-coded `AMAZON_RECONCILE_NOTIFICATION_TYPES`;
  - the receiving-evidence labels lose the ambiguous state;
  - `ITEM_PRODUCT_TYPE_CHANGE` appears as a configurable row.
- `index.tsx`: remove the 6 dead links to deleted shadow pages (leftover from the removal task).
- Route `/amazon-events/raw-ingest` → `/amazon-events/deliveries` in `App.tsx` and `AmazonEventsNav.tsx`, with a redirect from the old path.

---

## 8. Implementation phases

Each phase compiles and passes lint and format (api `tsc` for main + worker, webapp `tsc`, admin `tsc` baseline ≤ 146) before the next one starts.

| Phase | Content | Risk |
|---|---|---|
| P1 | Constants relocation (§3.6): delete `core/constants/amazon-events/`, fill `global-enums.ts` / `constants.ts` / `utils.ts`, repoint every consumer. Plus the registry, `AmazonEventsConfigService` and the envelope parser. Old services switch to read from them (no behaviour change) | Low |
| P2 | Subscriptions sub-module: client, region resolver, provisioner, reconcile. Auto-subscription and admin actions delegate to them. D6 disable fix. `ITEM_PRODUCT_TYPE_CHANGE` enabled. Reconcile cron | Medium: touches channel authorize flow |
| P3 | Inbox sub-module + migrations M1–M5 + retention cron + entity update + admin deliveries API/UI | Medium: schema |
| P4 | Route resolver + handlers extracted one family at a time (control → orders → listings), each behind the dispatcher. The webhook still calls the dispatcher synchronously in this phase, so it's directly comparable with today | Medium |
| P5 | Queues: lanes, processors, intake short-circuits (D2, D3), webhook switches to enqueue-and-ack. Order processor drain shim | High: changes the live ack contract |
| P6 | Delete the old folders/modules, dead transport methods, `'AMAZON_EVENT'` evidence type (api + webapp), admin dead links, docs | Low |

**P1 status (done 2026-10-08).**

- `core/constants/amazon-events/` is deleted.
- The content lives in `global-enums.ts` / `constants.ts` / `common/lib/utils.ts`.
- `AmazonEventsConfigService` + `AmazonEventsConfigModule` are in `platforms/amazon/events/config/`.
- `AmazonNotificationEnvelopeParser` is in `platforms/amazon/events/envelope/`.
- The notifications, raw, auto-subscription and admin services read from them.
- Checks:
  - api `tsc` (main + worker) is clean, and oxfmt / oxlint pass;
  - admin `tsc` has 140 errors, under the 146 baseline, none in AmazonEvents;
  - parser output matched the stored type, message id and publish time on 214 dev deliveries.

Deviations from the mapping, by design:

- **Kept until M5 (P3).** The column values for `ingest_source`, `parse_status`, `canonicalization_status` and `payload_storage_kind` stay as `GlobalEnums.AmazonEvent{IngestSources,ParseStatuses,CanonicalizationStatuses,PayloadStorageKinds}`. The meta/header allowlists stay as `AMAZON_EVENT_RAW_ALLOWED_{META,HEADER}_KEYS`. Live code still writes those columns.
- **Done early.** The shadow ingest endpoint (controller, `ingest()`, secret check) is already deleted. Its secret was never configured, so the endpoint always returned 503.
- **Registry entry.** `ITEM_PRODUCT_TYPE_CHANGE` has `autoSubscribe: false` until P2 flips it (D4). `FULFILLMENT_ORDER_STATUS` has `payloadVersion: null`, which keeps today's `'v1'` fallback.
- **Admin catalog change.** The catalog now derives `handled` from the registry, so ACCOUNT_STATUS_CHANGED, ORDER_CHANGE and FULFILLMENT_ORDER_STATUS show support `full` (they were `none`). The renames `legacyProtected` → `handled`, `activeLegacyProtectedTypes` → `handledTypes` and `'legacy_protected'` → `'handled'` are applied in the admin types too.
- **Deferred.** `AMAZON_FEED_NOTIFICATION_MATCHERS`, the lane job options and the lane `QueueNames` are not added yet; they land with their consumers in P4 / P5. `AMAZON_NOTIFICATION_LOG_CODES`, `AMAZON_HANDLED_REPORT_TYPES`, `AMAZON_ROUTABLE_SUBSCRIPTION_STATUSES` and `AMAZON_NOTIFICATION_FIELD_ALIASES` are added and used.
- **Small additions:**
  - the SKU alias list gains `sellerSku` for the listing handlers (the product-type handler already had it);
  - the EventBridge handler no longer logs the shared auth secret;
  - `AMAZON_EVENTS_DEFAULT_SP_API_REGION` is added to `.env.example`, and the old name is still read as a fallback.

**Follow-up (2026-10-08, subscriptions table).** Migration `20261008103049` drops `configuration_snapshot_json` (only read by a sibling pre-check that `validateRemoteShape` already covers) and `last_disabled_at` (never read, empty on dev), and adds the partial index `(provider_subscription_id, notification_type) WHERE provider_subscription_id IS NOT NULL` for routing. Not run.

**Follow-up (2026-10-08).** The env-configured default grantless region is removed (no `AMAZON_EVENTS_DEFAULT_SP_API_REGION` / `AMAZON_EVENTS_ADMIN_SP_API_REGION`); the region comes from the channel region or its marketplace coverage only. Raw retention is the constant `AMAZON_EVENT_RAW_RETENTION_DAYS` (90), no env override. Types live in `core/types/*.type.ts` (Amazon events: `amazon-events.type.ts`).

**P2–P6 status (done 2026-10-08, not deployed; migrations M1–M5 written but not run).**

What exists now:

- api: `platforms/amazon/events/` with `config/`, `envelope/`, `inbox/`, `intake/`, `dispatch/`, `routing/`, `handlers/{order,listing,product-type,feed,report,account}`, `logging/`, `subscriptions/`, and the master `AmazonEventsModule` (mounted by `PlatformsModule`).
- Processors: `sync/queue/processors/amazon-notification.processors.ts` (3 lanes), registered in `SyncConsumersModule`.
- Cron jobs: `pruneAmazonEventDeliveries` (`20 4 * * *`) and `reconcileAmazonEventSubscriptions` (`15 */6 * * *`, `onlyUnhealthy`).
- Admin backend: `admin/amazon-events/{catalog,subscriptions,deliveries}` + `AmazonEventsAdminModule`. `POST /subscriptions/setup` is removed. The deliveries routes live at `admin/amazon-events/deliveries/{overview,/,/:id}`.
- Deleted: `amazon-notifications/`, `amazon-events/raw/`, `notifications/amazon-events-auto-subscription/`, `admin/amazon-events-shadow-raw/`, the two dead transport destination methods, and `'AMAZON_EVENT'` (api + webapp).
- db-migrations: `20261008070653` … `20261008070710` (M1–M5).
- admin repo: `RawIngest.tsx` → `Deliveries.tsx`, the drawer and types renamed. `/amazon-events/raw-ingest` redirects to `/amazon-events/deliveries`. The reconcile type list comes from the overview `supportedNotificationTypes`. The six dead shadow links are removed from `index.tsx`.
- Checks:
  - api `tsc` (main + worker) is clean, and oxfmt / oxlint pass;
  - admin `tsc` has 140 errors, under the 146 baseline;
  - webapp `tsc` is clean;
  - every injected provider is exported by an imported module or provided by a global one;
  - a read-only replay of 240 dev deliveries through the new parser and route resolver showed no query errors and routing parity with the old queries. Listing events with no route had no local subscription row in the old query either; the old code still stamped them `PROCESSED`.

Deviations, by design:

- **Handlers call the route resolver themselves.** The dispatcher does not resolve routes, because each family extracts different keys and gates (order: order sync + enabled marketplace; listing / product type: connected channel; account / report: no channel gate). The registry `routing` field decides whether a marketplace is required.
- **Job id.** It is `amazon-notification-<deliveryId>`: BullMQ custom ids must not contain `:`.
- **Lane concurrency override.** Overrides go through the existing DB queue config (`AppConfiguration`, applied by `BaseQueueProcessor`), not a new env var.
- **Listing skip reporting.** Listing handlers no longer write the per-notification summary error log. Each skip is logged individually and the delivery row records `PROCESSED` / `IGNORED <first skip reason>` / `UNMATCHED`.
- **ITEM_PRODUCT_TYPE_CHANGE without an ASIN** is `IGNORED missing_required_mapping_fields`. Before, it matched every listing with a NULL ASIN in the routed marketplaces.
- **Auth failures** answer `401` and record `FAILED authorization_failed`. Before, they got `200 "Authorization failed"`.
- **SKU aliases.** `currentProductType` and `currentAccountStatus` aliases were added to `AMAZON_NOTIFICATION_FIELD_ALIASES`.

**Verification per phase:**

- `tsc` / lint / format in all touched repos.
- A local replay script feeds stored `payload_json` samples (one per type, from dev) through `intake.accept` → dispatcher, and diffs outcomes against the stored `live_processing_*` values. It is written to the scratchpad and not committed.
- Migrations are dry-read only. Nothing is executed unless the user asks.

---

## 9. Risks & checks before deploy

1. **The ack contract changes (P5).** Once we return 200 after enqueue, Amazon never redelivers. Redis/BullMQ must be healthy: an enqueue failure still returns 503, so Amazon retries in that case.
2. **Old-shape `PROCESS_ORDER_NOTIFICATION` jobs** are drained via the shim for one release, then the shim is deleted.
3. **The M3 payload rewrite** touches ~194k rows on dev, more on prod. It is batched by id range. Run it in a low-traffic window, and run `VACUUM (ANALYZE)` afterwards (outside the migration) to reclaim space.
4. **`ITEM_PRODUCT_TYPE_CHANGE` rollout** creates subscriptions for every connected channel on the first cron or reconcile. User-visible notifications start then.
5. **The reconcile cron** must respect the existing allowlists and `AMAZON_EVENTS_AUTO_SUBSCRIBE_ENABLED`, and skip local env, like other crons. It limits remote calls to unhealthy rows: grantless notification calls are tightly rate-limited by Amazon.
6. **ORDER_CHANGE `UNMATCHED` volume** (99.7% on dev) comes from the shared destination delivering other environments' sellers. This is not a bug and stays as is. The `foreign_environment_event` ownership hint keeps classifying it, and the intake could fast-ignore by `applicationId` in future (not in scope).

## 10. Out of scope

- Re-introducing Amazon-driven alerts, pricing events or `ANY_OFFER_CHANGED`.
- Changing the SQS/EventBridge AWS infrastructure or the webhook URLs.
- Per-environment destinations.
- Moving Shopify webhooks onto the same registry. The handler and dispatcher interfaces are platform-neutral enough to allow it later.

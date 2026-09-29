# Command Center (v1): how it works today

> **Scope:** `webapp/` and `api/` only. This document explains what the page does as the code stands (read on 2026-09-29). No code was changed.
> **Entry file:** `webapp/src/app/(index)/(menu-layout)/command-center/page.tsx`
> **Depth rules used (agreed before writing):**
> - Action Center snapshot: a summary of each step, not a line-by-line walkthrough.
> - Upstream pipelines (readiness, Amazon events, orders…): a short lifecycle for each.
> - Buttons: the route, the params passed, and what the user gets there.
>
> **Legend:** 🗄️ = database table · 🧠 = in-memory / Redis cache · 🌐 = external or non-DB source · ⚠️ = something odd seen while reading the code (not verified at runtime)

---

## Contents

1. [What the Command Center is](#1-what-the-command-center-is)
2. [Page map at a glance](#2-page-map-at-a-glance)
3. [How a user gets to the page](#3-how-a-user-gets-to-the-page)
4. [Access control (webapp + API)](#4-access-control-webapp--api)
5. [Page lifecycle (boot → render)](#5-page-lifecycle-boot--render)
6. [Shared plumbing: endpoints, request context, caching, section states](#6-shared-plumbing)
7. [Top chrome: banners and the summary strip](#7-top-chrome-banners-and-the-summary-strip)
8. [Setup mode (brand-new account)](#8-setup-mode-brand-new-account)
9. [Dashboard mode: the operational snapshot (feeds NOW / NEXT / AT-RISK / STOCK)](#9-the-operational-snapshot)
10. [NOW](#10-now)
11. [NEXT](#11-next)
12. [AT-RISK PRODUCTS](#12-at-risk-products)
13. [CATALOG STATUS (Readiness + Publishing)](#13-catalog-status)
14. [AMAZON EVENTS](#14-amazon-events)
15. [STOCK](#15-stock)
16. [CAPACITY](#16-capacity)
17. [TREND](#17-trend)
18. [Upstream pipelines that produce the data](#18-upstream-pipelines)
19. [Reference: tables per section](#19-reference-tables-per-section)
20. [Reference: caches and freshness](#20-reference-caches-and-freshness)
21. [Reference: every click target](#21-reference-every-click-target)
22. [Observations seen while reading the code](#22-observations)
23. [File index](#23-file-index)

---

## 1. What the Command Center is

The Command Center is the **default landing page after login** (`/command-center`). It is a **read-only summary dashboard**. It answers four questions for the signed-in user's account, limited to the channels that user is allowed to see:

| Question | Section(s) |
|---|---|
| *What needs attention right now?* | **NOW**, **AT-RISK PRODUCTS**, **AMAZON EVENTS**, **STOCK** |
| *What should I do next?* | **NEXT** |
| *How healthy is my catalog?* | **CATALOG STATUS** (Readiness + Publishing) |
| *How is the business trending, and how much of my plan am I using?* | **TREND**, **CAPACITY** |

The page itself changes no data. Every interactive element **navigates** to another page (Action Center, Products, Product detail, Channel Management, Billing, Create Product), where the user takes the action.

Four of the sections (NOW, NEXT, AT-RISK, STOCK) are **views over the Action Center's "Sales Operations" snapshot**. The other sections have their own data sources.

---

## 2. Page map at a glance

```
/command-center
│
├─ SubscriptionResourcesAlert        (only when a scheduled downgrade would exceed limits)
├─ RestrictedModeBanner              (only when the account is over its limits after a downgrade)
├─ Summary strip                     "SUMMARY VIEW · <scope label>      🕒 Last updated <x ago> [REFRESHING]"
│
├─ (A) While the capacity request loads ───────► Setup loading skeleton
├─ (B) 0 products AND 0 channels ─────────────► SETUP MODE
│        ├─ "Start your Command Center" hero  [New product] [Connect Amazon/Shopify]
│        ├─ Setup checklist (3 static steps)
│        └─ PLAN CAPACITY (+ trial counters / [Upgrade plan] on trial)
│
└─ (C) Otherwise ─────────────────────────────► DASHBOARD MODE
         ├─ Row 1:  NOW  |  NEXT
         ├─ AT-RISK PRODUCTS
         ├─ CATALOG STATUS  [Scope ▼] [Marketplace ▼]
         │     ├─ Readiness  (Ready / Almost ready / Needs attention / Blocked)
         │     └─ Publishing (Published / Out of sync / Not published / Inactive)
         ├─ AMAZON EVENTS   (rendered only when at least one item exists)
         ├─ Row:  STOCK  |  CAPACITY
         └─ TREND  [7d 30d 90d 6m 12m]
```

**Five API calls feed the page.** All are `POST`, and all fire in parallel when the page mounts.

| # | Webapp hook | API route | Feeds |
|---|---|---|---|
| 1 | `operationalSection` | `POST /command-center/operational-summary` | NOW, NEXT, AT-RISK, STOCK |
| 2 | `catalogStatusSection` | `POST /command-center/catalog-status` | CATALOG STATUS; also the scope label in the summary strip |
| 3 | `capacitySection` | `POST /command-center/capacity-summary` | CAPACITY / PLAN CAPACITY; also the **setup vs dashboard** decision |
| 4 | `trendSection` | `POST /command-center/trend-summary` | TREND |
| 5 | `amazonEventsSection` | `POST /command-center/amazon-events` | AMAZON EVENTS |

A sixth endpoint, `POST /command-center/summary`, bundles #1 to #4 into one payload. It is defined in `webapp/src/lib/api.ts` (`GET_COMMAND_CENTER_SUMMARY`) but **the webapp never calls it**.

---

## 3. How a user gets to the page

| Entry point | Where | Behaviour |
|---|---|---|
| **Default authenticated route** | `webapp/src/lib/action-center.ts` → `DEFAULT_AUTHENTICATED_ROUTE = '/command-center'` | Post-login and home redirects resolve here. |
| After **paid** registration | `app/(auth)/register/_components/payment.tsx` → `PAID_REGISTRATION_SUCCESS_ROUTE` | Redirects to `/command-center`. |
| After **trial** registration | `app/(auth)/register/_components/registration-flow.tsx` → `TRIAL_REGISTRATION_SUCCESS_ROUTE` | Redirects to `/command-center`. |
| Public site "Access Categra" button | `app/(home)/_components/navbar-user-info.tsx` | Links to `/command-center`. |
| Action Center fallback | `actionCenter/main.tsx` (`onOpenFallback`) | Pushes `/command-center`. |
| Sidebar | `(menu-layout)/_components/nav-container.tsx` | There is a "Command Center" entry (`icon: 'chart'`, permissions = Action Center access), but it is declared with `visible: false`, so it is **hidden** from the menu. |
| Onboarding guide | `components/onboarding-guide/guideRegistry.ts` → step `dashboard_overview` | An "explain" step anchored on `/command-center`. |
| Browser tab title | `page.tsx` → `metadata.title = translate('Command_Center_Nav_Title')` | Shows the page name in the tab. |

**URL query parameters the page reads** (see §5.3):

| Param | Values | Effect |
|---|---|---|
| `period` | `7d` · `30d` (default) · `90d` · `6m` · `12m` | TREND window only |
| `channelId` | numeric channel id | Narrows the operational, catalog-default-scope, trend and Amazon-event requests |
| `marketplaceId` | numeric Amazon marketplace id | Same as above, one level narrower |

There is **no control on the page** that sets `channelId` or `marketplaceId`. They take effect only when they are already in the URL. The period selector is the only control that writes to the URL.

---

## 4. Access control (webapp + API)

### 4.1 Webapp gate: `CommandCenterPermissionGate`

File: `command-center/_components/command-center-permission-gate.tsx`

1. Reads `authStatus` and `user` from `useAuthStore`, and `packageResources` from `useGlobalStore`.
2. Waits until auth is `ready` and the user is resolved (has an id, is owner/admin, or has permissions). Until then it renders **nothing**.
3. Checks **`hasActionCenterAccessPermission(user)`** (`lib/action-center-permissions.ts`). The user passes if **any** of these is true:
   - `isOwner`
   - `isAdministrator`
   - the permissions list contains `actionCenter.*` or `actionCenter.view`
4. **No access:** `router.replace(getAuthorizedRoute(findPath(user.permissions)))`, which sends the user to the first route their permissions allow (via `useNavigationViaPermission`).
5. **Inactive subscription** (`getSubscriptionAccessUiState(packageResources).isInactiveAccount`): renders `<InactiveSubscriptionFeatureBlockedState feature="action-center" />` instead of the page. An account is "inactive" when `canAccess === false`, when the status or account status is `EXPIRED` or `INACTIVE`, when it is effectively cancelled, or when it is suspended. The exception is when the account is only in *restricted over-limit* mode.
6. Otherwise it renders `<CommandCenterMain />`. `page.tsx` loads that component lazily with `next/dynamic` and shows a spinner in the meantime.

### 4.2 API gate (on every endpoint)

File: `api/apps/api-main/src/modules/app/system/operations/orchestration/command-center/command-center.controller.ts`

| Layer | What it enforces |
|---|---|
| `@UseGuards(JwtAuthGuard)` on the controller | A valid Bearer JWT; populates `req.user` (`accountId`, `id`, permissions, owner/admin flags). |
| `@RequiresSubscriptionCapabilities('ACTION_CENTER_VIEW')` on each route | `SubscriptionCapabilityGuard` → `SubscriptionCapabilityService.assertRequestCapability`. `ACTION_CENTER_VIEW` is a **read** capability: it is allowed in restricted/over-limit mode and on an expired trial. It is **denied** for inactive accounts, and it **fails closed** if the subscription can't be resolved. A denial returns **403** with `data.code = 'SUBSCRIPTION_CAPABILITY_DENIED'`. |
| Channel access scope (inside the service) | `ActionCenterService.getChannelAccessScope(req)` → `ChannelService.getUserWiseChannels(req)`. If the user has `isMasterChannelAccess` or an administrator role, then `allowAllChannels = true` and `canAccessMaster = true`. Otherwise the user is limited to the ids in `tbl_user_channels` and has **no master access**. This scope filters every query and is part of every cache key. |

**Tables used by the access checks:** 🗄️ `tbl_users` (`is_master_channel_access`), `tbl_user_channels`, `tbl_user_accounts`, `tbl_roles` (`is_administrator_role`), `tbl_account_subscriptions` / `tbl_packages` (capability resolution). 🧠 Redis key `user_wise_channels:{accountId}:{userId}` caches the channel scope.

---

## 5. Page lifecycle (boot → render)

File: `command-center/main.tsx` → `CommandCenterMain`

### 5.1 On mount

1. **Navbar:** `setTitle('Command Center')` and `setBreadcrumbs([{ title, route: '/command-center' }])`. Both are cleared on unmount.
2. **Package snapshot refresh:** `useRefreshPackageResourcesSnapshot()` calls `POST package-subscription-manager/fetch-resources` (with retry) and stores the result in `useGlobalStore.packageResources`. The banners, the inactive/restricted checks, the plan name and the trial counters all read from this.
3. **URL parsing:** `period = normalizePeriod(searchParams.get('period'))` (defaults to `30d`), plus `channelId` and `marketplaceId`.
4. **Request bodies** (memoised):
   - `scopedBody = { channelId?, marketplaceId? }`, used by operational, catalog-status and amazon-events.
   - `trendBody = { period, channelId?, marketplaceId? }`, used by trend.
   - `emptyBody = {}`, used by capacity.
5. **Five `useCommandCenterSection` hooks start**, each firing its `POST` in a microtask (§6.4). All five fire in **setup mode too**, even though setup mode renders only capacity.

### 5.2 Choosing what to render

```
capacitySection.loading && !capacitySection.data     → <CommandCenterSetupLoadingState/>   (skeleton)
capacity loaded, no error, not restricted,
  products.used <= 0 AND channels.used <= 0          → SETUP MODE (§8)
otherwise                                            → DASHBOARD MODE (§9–§17)
```

The capacity request alone decides the mode. If it **fails**, the page falls through to dashboard mode.

### 5.3 When the URL changes

| Change | Which requests re-run |
|---|---|
| `period` (period selector) | Trend only. `router.replace('/command-center?…', { scroll: false })` → `trendBody` changes → the trend hook reloads with a reset (skeleton). |
| `channelId` / `marketplaceId` | Operational, catalog-status, trend and amazon-events. Capacity uses an empty body, so it doesn't re-run. |

There is **no polling or auto-refresh**. Data refreshes only on a page load, a URL change, or a section **Retry**.

---

## 6. Shared plumbing

### 6.1 API module wiring

- **Module:** `command-center.module.ts`. It imports Auth, PackageSubscriptionManager, Order, Product, ProductPricing, Channel, ActionCenter and Account modules, and provides `CommandCenterService` and `CommandCenterCatalogStatusService`.
- **Routing:** the controller prefix is `command-center`, and the module is registered through the system module (`system.module.ts`).
- **DTO** (`dto/command-center.dto.ts`): `{ channelId?: string; marketplaceId?: string; period?: '7d'|'30d'|'90d'|'6m'|'12m' }`, validated by class-validator.

### 6.2 Request context (`CommandCenterService.resolveContext`)

Every endpoint starts with these two lookups, run in parallel:

1. `actionCenterService.getChannelAccessScope(req)` → `{ allowAllChannels, allowedChannelIds, canAccessMaster }` (§4.2).
2. `accountConfigurationService.findOne({ accountId }, ['baseCurrency'])` → 🗄️ `tbl_account_configurations.base_currency`, falling back to `USD`.

Then it normalises the inputs:
- `channelId` and `marketplaceId` become positive integers, or `null`.
- `period` falls back to `30d` if missing or invalid.

### 6.3 Caching

| Cache | Where | TTL | Key |
|---|---|---|---|
| 🧠 Command Center section cache (Redis, `CacheBaseService`) | `summary`, `operational`, `catalog-status`, `capacity`, `trend` | **90 s** | `command-center:<section>:phase-7-channel-blockers:account:<id>:user:<id>:master:<0/1>:scope:<ALL \| sorted ids \| NONE>:channel:<id \| all>:marketplace:<id \| all>:period:<p>:currency:<CUR>` |
| **Amazon events** | none | none | Computed on every call. |
| 🧠 Action Center base snapshot (in-process `Map`) | inside `ActionCenterService` | **15 s**, plus de-duplication of identical in-flight requests | account, user, mode, search, channel, marketplace, product, health, issueType, includeSourceDataIssues, smartReview, channel scope |
| 🧠 User channel scope (Redis) | `ChannelService.getUserWiseChannels` | `REDIS_TTL` | `user_wise_channels:{accountId}:{userId}` |

The period is part of every Redis key, **including the sections that don't use it**. So a period change creates new cache entries for all of them. The webapp only re-requests trend, though, so only the trend entry is actually created.

### 6.4 Section hook: `useCommandCenterSection` (webapp)

Each section's state is `{ data, loading, refreshing, error, restricted, retry }`.

1. `loadSection({ resetData })` bumps a request id, which makes any older in-flight response get ignored when it arrives.
2. It calls `genericService.post(endpoint, body, undefined, { suppressGlobalRestrictionUI: true })`, so a 403 won't trigger the global restriction modal.
3. It normalises the payload: the payload itself if it has `meta`, otherwise `response.data`.
4. On a **403 `SUBSCRIPTION_CAPABILITY_DENIED`**: `restricted = true` and the error becomes the capability message ("… not available on your plan").
5. On **any other error**: `error = <server message or "Failed to load <section>">`.
6. `retry()` reloads **without** clearing the data, so the "Refreshing" badge shows while the old data stays on screen.

**What a section renders in each state:**

| State | Render |
|---|---|
| `loading` | `LoadingSection` skeleton; its shape differs per section |
| no data, with an error | `SectionFallback`: the error title, a hint, and a **Retry** button (the button is hidden when `restricted`) |
| data present | the section UI |

### 6.5 API error envelope

`CommandCenterController.executeRequest` wraps every handler. It logs the error, keeps the original HTTP status for an `HttpException` (500 otherwise), and returns `{ status: 'error', data: null, message }`.

---

## 7. Top chrome: banners and the summary strip

### 7.1 `SubscriptionResourcesAlert` (surface `command-center`)

- **Source:** `packageResources.exceedResources` (from the package snapshot).
- **Shows when:** a **scheduled downgrade** would push SKUs, channels or storage over the new plan's limits. It lists each resource that would exceed the limit. It uses the Command Center wording of the component (`surface === 'command-center'`).

### 7.2 `RestrictedModeBanner area="command-center"`

- **Shows when:** `getRestrictedModeState(packageResources).isRestrictedDowngradeOverLimit`. That is: exceeded resources exist, the payment has not failed, and the account is explicitly over-limit-restricted, `SUSPENDED` or `CANCELLED`.
- **Message:** the Command Center "stays visible / read-only" copy (`Command_Center_Stays_Visible_Cleanup_Message` / `Command_Center_Read_Only_Message`).

### 7.3 Summary strip

| Element | Logic |
|---|---|
| **"SUMMARY VIEW"** badge | Static. |
| Scope label | While capacity is still deciding the mode: "Preparing setup view". Otherwise `getCommandCenterUtilityScopeLabel`: <br>• no `channelId` in the URL → **"Account-wide view"** <br>• `channelId` present but not among the catalog scopes → "Filtered scope" <br>• `channelId` only → "Filtered to <channel name>" <br>• `channelId` + `marketplaceId` → "Filtered to <channel> / <marketplace>" <br>The channel and marketplace names come from the **catalog-status** response. |
| 🕒 "Last updated …" | The **oldest** `meta.generatedAt` across operational, catalog, capacity and trend (plus amazon-events when that section is shown). Formatted as "Just now", "N min ago", "N h ago", or a short date after 24 h. A section served from cache keeps its original `generatedAt`, so it can be up to 90 s old. |
| "REFRESHING" pill | Shown while any section is refreshing, which only happens after a Retry. |

---

## 8. Setup mode (brand-new account)

**Condition:** the capacity response loaded, with no error and no restriction, and `capacity.products.used <= 0` **and** `capacity.channels.used <= 0`.

### 8.1 Hero: `CommandCenterSetupEmptyState`

| Element | Action |
|---|---|
| Badge "Command Center setup", title "Start your Command Center", subtitle | Static copy. |
| **[New product]** (primary) | `router.push(getCreateProductRouteWithReturnTo(pathname, searchParams))` → **`/products/create?returnTo=/command-center?<current query>`** (keeps `uniqId` if present). Opens the product creation form; after saving, the user is returned to the Command Center. |
| **[Connect Amazon/Shopify]** (secondary) | `resetChannelSetupState()`, then `changeChannel(null)`, then `setCreateChannelDialogOpen(true)` in `useCMSharedStore`, then `router.push('/channelManagement')`. Channel Management opens with the **"create channel" dialog already open** at the provider-selection step. |
| Setup checklist (3 rows) | Static, not clickable: 01 "Add catalog data", 02 "Connect a sales channel", 03 "Start seeing operational insights". |

### 8.2 PLAN CAPACITY: `CommandCenterSetupCapacitySection`

- **Title:** "PLAN CAPACITY".
- **Subtitle:** "Plan limits for <plan>" when a plan name is known, otherwise "Current subscription limits". The plan name is `packageResources.planName`, then `package.displayName`, then `package.name`.
- **Header right:**
  - On a **trial**: an **[Upgrade plan]** button → `router.push('/settings/billing?tab=subscription#billing-manage-subscription')`, which opens Billing on the Subscription tab, scrolled to "Manage subscription".
  - Otherwise, when there is a plan name: a plan-name pill (not clickable).
- **Body:** the same `CapacitySection` cards as the dashboard (§16).
- **Trial extras** (only when `getTrialUiState(packageResources).isTrial`):
  - "AI generations" counter: `trial.aiGenerations.{consumed, limit, remaining}`
  - "Product sync actions" counter: `trial.syncActions.{…}`
  - "Trial window" card: days remaining from `trial.endsAt`, or "Expired"
- **Sources:** 🗄️ `tbl_trial_usage_summaries` (via `PackageSubscriptionManagerService.buildTrialPackageResourcesPayload`), `tbl_account_subscriptions`, `tbl_packages`.

---

## 9. The operational snapshot

*This is the shared data pipeline behind NOW, NEXT, AT-RISK and STOCK.*

**Endpoint:** `POST /command-center/operational-summary` → `CommandCenterService.getOperationalSummaryForContext`

### 9.1 Steps (API)

1. **Redis cache check** (`command-center:operational:…`, 90 s). On a hit, return the cached payload.
2. **Call the Action Center:**
   `actionCenterService.getSalesOperationsSnapshot(req, { includeSourceDataIssues: true, channelAccess, channelId, marketplaceId })`
   - The mode is always **`SALES_OPERATIONS`**.
   - `includeAmazonEventMaterializations` is left at its default (**false**), so Action Center rows materialised from Amazon events are **not** included.
3. **Merge in channel-blocker rows:** `snapshot.channelIssueRows` are converted by `mapChannelIssueRowsToOperationalRows` into product-shaped rows with `productId = 0`, `health = 'BLOCKED'` (unless another value is given), `channelError = true`, and `workflowStatus = 'OPEN'`.
4. **Normalise and de-duplicate:** `normalizeOperationalWorkflowRows([...workflowRows, ...channelRows])` (`operational-rows.ts`):
   - It keeps only the **visible workflow statuses**: `OPEN`, `IN_PROGRESS`, `PENDING_VERIFICATION`.
   - It de-duplicates by **logical identity**:
     - `workflow:<workflowIdentityKey>` when the key exists; otherwise
     - `product:<productId>:<issueType>:<channel>:<marketplace>:<language>`; or
     - `row:<id>:…` for channel rows.
   - On a collision, rows are **merged**:
     - The most recently updated row wins, then the higher `priorityScore`, then the higher status rank.
     - The affected entity ids are unioned, and `health` takes the worst of the two.
     - The boolean issue flags are OR-ed together, and the counts take the maximum.
5. **Build the four sections from the same row list:**
   - `now`: `buildNowSection(rows, scope)`, see §10
   - `next`: `buildNextSection(rows, scope)`, see §11
   - `atRiskProducts`: `buildAtRiskProductsSection(rows)`, see §12
   - `stock`: `buildStockSection(rows, scope)`, see §15
6. Stamp `meta = { generatedAt: now, period }`, cache it for 90 s, and return it.

### 9.2 Inside `getSalesOperationsSnapshot` (Action Center: step-level summary)

File: `api/…/system/operations/action-center/action-center.service.ts`. It calls `getOverviewSnapshot`, which calls `getCachedOverviewBaseSnapshot` (15 s in-memory cache), which calls `buildOverviewBaseSnapshot`.

| Step | What happens | Tables / sources |
|---|---|---|
| 1. Channel scope | Reuses the scope from §4.2. Master access is required to include source-data issues. | 🗄️ `tbl_users`, `tbl_user_channels`, `tbl_roles` · 🧠 Redis |
| 2. Workflow storage check | `isWorkflowOverlayStorageAvailable()` checks whether the overlay table exists. | 🗄️ `tbl_action_center_issue_workflows` |
| 3. Parallel context loads | • `getAccountState`: product count, connected sales channel count <br>• `getFilterMetadata`: the channel list <br>• `getPrimaryMediaMinimum`: the minimum media count from readiness config <br>• `resolveActionCenterRuntimeSettings`: the account's issue priority ordering and classifications <br>• `resolveActionCenterCollaborationState`: whether assignments are readable | 🗄️ `tbl_products`, `tbl_product_channels`, `tbl_channels`, `tbl_account_configurations`, `tbl_readiness_configurations`, `tbl_attributes`, `tbl_action_engine_settings` |
| 4. **Channel blocker rows** (`getChannelIssueRows`) | Only when there is **no `marketplaceId`** and the request isn't for one product. Selects active, non-deleted Amazon/Shopify channels in scope where: <br>• `connection_status = CONNECTION_LOST` → **`CHANNEL_CONNECTION_ERROR`** <br>• `connection_status = NOT_CONNECTED` → **`CHANNEL_NOT_CONNECTED`** <br>• `CONNECTED` + `attribute_mapping_status = INCOMPLETE` → **`CHANNEL_SETUP_INCOMPLETE`** <br>Each becomes one `BLOCKED` row per channel, with the quick action "open connection settings". | 🗄️ `tbl_channels` |
| 5. Early exit | If the account has 0 products, the snapshot returns only the channel blocker rows. | — |
| 6. **Sales scope rows** (`getSalesScopeRows`) | Two raw SQL queries run in parallel, one for **Amazon** and one for **Shopify**. They produce one row per product × channel (× marketplace for Amazon) with these fields: <br>• price: `missingPriceCount` <br>• stock: `missingStockCount`, `lowStockCount` <br>• sync: `syncStatus`, `hasSyncErrors`, `isOutOfSync` <br>• listing: `hasAmazonError`/`Warning`, `hasChannelWarning`, `listingStatus`, `searchableStatus`, buyable/discoverable flags <br>• catalog: readiness, media count, variant count, and 7-day/30-day sales impact. <br>Details in §9.3. | see §9.3 |
| 7. **Decorate** (`decorateScopeItem`) | Converts the raw flags into issue booleans, then `issueTypes`, then `health`, `topIssue`, `quickAction`, `priorityScore` (§9.4). In `SALES_OPERATIONS` mode, rows with only source-data issues are dropped unless `includeSourceDataIssues` is on. | — |
| 8. Expand + boost | `expandDecoratedScopeItemRows` expands a row into per-issue rows where applicable. `applyBusinessImpactBoost` raises the priority of rows with recent sales. | — |
| 9. Sort + identity | `sortScopeItems`, then `attachWorkflowIdentityToRows`, which assigns a durable `workflowIdentityKey` per ticket scope. `decorateScopeItemWithRuntimeRouting` adds `routing` (the display issue label, the primary action, the target area). | 🗄️ `tbl_action_engine_settings` (via runtime settings) |
| 10. **Workflow overlay** (`applyWorkflowProjection`) | Overlays the shared workflow status (`OPEN` / `IN_PROGRESS` / `PENDING_VERIFICATION` / `CLOSED` / `SNOOZED` …) and the user's personal snooze, both saved from Action Center actions. | 🗄️ `tbl_action_center_issue_workflows`, `tbl_action_center_lifecycle_events` |
| 11. **Assignment overlay** (`applyAssignmentProjectionToRows`) | Adds the owner and assignee details. | 🗄️ `tbl_action_center_issue_assignments`, `tbl_users`, `tbl_attachments` (avatars) |
| 12. Return | `getOverviewSnapshot` also builds the summary, breakdowns, priority actions, recent activity and (when flagged) the compound projection. **The Command Center uses only `workflowRows` and `channelIssueRows`.** | — |

### 9.3 What the sales SQL reads (step 6 above)

Shared CTEs (both queries):

| CTE | Purpose | Tables |
|---|---|---|
| `base_language` | The account's base language | 🗄️ `tbl_account_configurations` |
| `filtered_products` | Non-deleted, non-archived products in the channel scope (+ search) | 🗄️ `tbl_products`, `tbl_product_channels`, `tbl_product_channel_marketplaces`, `tbl_product_variants`, `tbl_variant_marketplaces`, `tbl_product_attributes`, `tbl_product_attribute_groups`, `tbl_attributes` |
| `product_attributes` | Name, description and category values | 🗄️ `tbl_product_attributes`, `tbl_product_attribute_groups`, `tbl_attributes` |
| `variant_counts` | Number of variants | 🗄️ `tbl_product_variants` |
| `media_counts` / `product_main_image` | Media count and the thumbnail | 🗄️ `tbl_product_media_linkers`, `tbl_digital_assets` |
| `product_sales_impact` | Orders, units and revenue over 7 d and 30 d per product × channel × marketplace (for business-impact boosting) | 🗄️ `tbl_orders`, `tbl_order_items` |

**Amazon query** (`buildAmazonSalesQuery`):

**Scope**
- Active product channels on **active, connected Amazon channels**, joined to active, non-deleted `tbl_product_channel_marketplaces`.
- Those are joined to active `tbl_amazon_channel_marketplaces` and `tbl_amazon_marketplaces`.
- The `marketplaceId` filter applies to `tbl_amazon_marketplaces.id`.

**Stock: resolved per product (no variants) or per active variant**
- The listing is **FBA**, or is mapped to an Amazon fulfilment-centre warehouse → use `tbl_amazon_fba_inventories.fulfillable_quantity`.
- Otherwise, it has mapped non-FC warehouses (`tbl_channel_product_warehouses` → `tbl_warehouse`) → use `SUM(tbl_warehouse_products.total_quantity)`.
- Otherwise → use `amazon_stock` on the PCM or VM row.

**Low-stock threshold**
- `tbl_product_variant_channels.low_stock_threshold`, falling back to `tbl_product_channels.low_stock_threshold`.
- It is used **only for warehouse-mapped (non-FBA) stock**. FBA listings and listings with no warehouse mapping never count as "low".

**Out of stock / low stock**
- Out of stock: `resolved_stock <= 0`.
- Low stock: `0 < resolved_stock <= threshold`.
- Both are counted per variant, giving `missing_stock_count` and `low_stock_count`.

**Price**
- `tbl_product_variant_prices.pricing_details`. The **sale price, if above 0**, is preferred; otherwise the **offer price**.
- Sources are tried in this order:
  1. Channel + marketplace, not inherited.
  2. Master price for that marketplace.
  3. Master price for no marketplace.
- `missing_price_count` counts items whose resolved price is ≤ 0 or null.

**Listing health**
- `external_status` JSON (`BUYABLE`, `DISCOVERABLE`)
- `listing_errors`
- `sync_status` (`ERROR`, `OUT_OF_SYNC`)
- `has_error`
- Product- and variant-level errors from `tbl_channel_product_errors`

**Currency**
- `tbl_currencies` is matched to the marketplace currency.

**Shopify query** (`buildShopifySalesQuery`):
- The same pattern over Shopify channels: `tbl_product_channels`, `tbl_product_variant_channels`, `tbl_shopify_channels`, warehouse mapping for stock, `tbl_product_variant_prices` for price, `tbl_channel_product_errors`, and `tbl_readiness_values`.

### 9.4 Issue → health rules (`decorateScopeItem`, `buildIssueTypes`, `getHealth`)

| Flag set when… | Issue type | Health contribution |
|---|---|---|
| `missingPriceCount > 0` | `MISSING_PRICE` | **BLOCKED** |
| `missingStockCount > 0` | `OUT_OF_STOCK` | **BLOCKED** |
| not out of stock and `lowStockCount > 0` | `LOW_STOCK` | AT_RISK |
| sync status `ERROR`, `has_sync_errors`, or a product-scoped channel error | `SYNC_FAILED` | **BLOCKED** |
| out of sync (for Amazon, only when the listing has an explicit status) | `OUT_OF_SYNC` | AT_RISK |
| Amazon listing errors | `AMAZON_ERROR` | **BLOCKED** |
| Amazon warnings | `AMAZON_WARNING` | AT_RISK |
| channel warning | `CHANNEL_WARNING` | AT_RISK |
| `NOT_SEARCHABLE`, or explicit status without `DISCOVERABLE` | `AMAZON_NOT_SEARCHABLE` | AT_RISK |
| missing price + not buyable + no error or sync failure | `AMAZON_MISSING_OFFER` | **BLOCKED** |
| `listingStatus = STATUS_PENDING` | `LISTING_STATUS_PENDING` | NEEDS_ATTENTION |
| readiness < 75 (only with `includeSourceDataIssues`, which needs master access) | `LOW_READINESS`, `MISSING_MEDIA`, `MISSING_CONTENT`, `MISSING_ATTRIBUTES`, `SOURCE_DATA` | NEEDS_ATTENTION |
| channel blocker rows (§9.2 step 4) | `CHANNEL_NOT_CONNECTED` / `CHANNEL_CONNECTION_ERROR` / `CHANNEL_SETUP_INCOMPLETE` | **BLOCKED** |

**`topIssueType`** is the first issue in the account's priority ordering. The code default is:
`CHANNEL_NOT_CONNECTED → CHANNEL_CONNECTION_ERROR → CHANNEL_SETUP_INCOMPLETE → CHANNEL_ERROR → AMAZON_ERROR → SYNC_FAILED → MISSING_PRICE → OUT_OF_STOCK → SHOPIFY_UNPUBLISHED → SHOPIFY_INACTIVE → AMAZON_MISSING_OFFER → PRICE_POLICY_BLOCKED → BUY_BOX_EXCLUDED → BUY_BOX_LOST → LOW_STOCK → OUT_OF_SYNC → LISTING_STATUS_PENDING → AMAZON_NOT_SEARCHABLE → AMAZON_WARNING → CHANNEL_WARNING → MISSING_MEDIA → MISSING_CONTENT → MISSING_ATTRIBUTES → LOW_READINESS → SOURCE_DATA`.
An account can override it in `tbl_action_engine_settings`.

> **Important for every section below:** NOW, NEXT, AT-RISK and STOCK read **only `topIssueType`**. A row that is both out of stock and missing a price is counted **only under its top issue** (`MISSING_PRICE`, by default).

### 9.5 Operational grouping (shared by NOW and NEXT)

`operational-grouping.ts` → `buildOperationalGroupingMetadata` (in `orchestration/operational-grouping/operationalGrouping.ts`) builds these fields for each row:

- **`capability`** is resolved from the target area or issue:
  - Pricing: missing price, missing offer, buy box
  - Inventory: out of stock, low stock
  - Media
  - Content: readiness, content, attributes
  - Sync: sync failed, out of sync
  - Channel Health: channel configuration issues
  - Publishing: listing and provider errors
  - Compliance
- **`fixSource`**: `<channel name | "Master Catalog"> -> Stock / Pricing / Media / Listing / Sync / Channel settings / Details`
- **`issueSource`**: `readiness` · `provider rejection` · `sync divergence` · `operational alert` · `pending publish`
- **`severity`**: `Critical` (BLOCKED or AT_RISK) · `Warning` · `Info`
- **`operationalScope`** (the label shown in the UI): `<channel> / <marketplace> / <language>`, or `Master Catalog …`
- **`canonicalGroupKey`**: `channel=…|marketplace=…|language=…|capability=…|issue=…|fix=…|scope=product|variant|field=…|providerCode=…`. It is stable and independent of row ids. The same key is used by Action Center, so it drives the drill-down there.
- **`affectedEntities`**: products, variants, marketplaces, languages, providerRows, syncRows

---

## 10. NOW

**Title / subtitle:** "NOW" · "Current operational state"
**Builder:** `now.builder.ts` → `buildNowSection`

### 10.1 Objective
Give a snapshot of the work queue: how many issues are urgent, being worked on, or waiting, and what the top five issue groups are.

### 10.2 UI anatomy
1. **Three counter cards.** They are not clickable.

| Card | Value | Rule (over the de-duplicated visible rows) |
|---|---|---|
| **Urgent** | `urgentCount` | `workflowStatus = OPEN` **and** health ∈ {BLOCKED, AT_RISK} |
| **In progress** | `inProgressCount` | `workflowStatus = IN_PROGRESS` |
| **Waiting** | `blockedCount` | `workflowStatus ∈ {WAITING_BLOCKED, PENDING_VERIFICATION}`. Rows are filtered to visible statuses first, so in practice this is `PENDING_VERIFICATION` only. ⚠️ |

The counts are **logical issue rows** (product × issue × channel × marketplace × language, or one row per channel blocker), **not distinct products**.

2. **Top signals list.** Up to 5 clickable rows. Each row shows:
   - the issue label (for example, "Missing price")
   - a severity pill (High / Medium / Low)
   - a subline: the operational scope label, or "Open in Action Center."
   - the count on the right, and a ↗ icon
3. **Empty state:** "No immediate issues to review" / "Nothing urgent needs review in this scope right now."

### 10.3 How the signals are built
1. Take the visible rows whose `topIssueType` is one of: `MISSING_PRICE`, `OUT_OF_STOCK`, `LOW_STOCK`, `SYNC_FAILED`, `AMAZON_ERROR`, `CHANNEL_ERROR`, `CHANNEL_NOT_CONNECTED`, `CHANNEL_CONNECTION_ERROR`, `CHANNEL_SETUP_INCOMPLETE`, `SHOPIFY_UNPUBLISHED`, `SHOPIFY_INACTIVE`.
2. Map health to severity: BLOCKED or AT_RISK → `high`; NEEDS_ATTENTION → `medium`; anything else → `low`.
3. **Group by `canonicalGroupKey`.** For each group:
   - `count` = the number of rows
   - `affectedVariants` = the sum across rows
   - `severity` = the highest in the group
   - `label` = the first row's `topIssueLabel`, else a built-in label, else the humanised type
4. Sort by severity (descending), then count (descending), then label. Keep the **top 5**.
5. Build each group's CTA:
   - `type: 'OPEN_ACTION_CENTER'`
   - `filters = { mode: 'SALES_OPERATIONS', issueType, channelId, marketplaceId, languageCode, …drilldown }`, where `channelId` and `marketplaceId` come from the request scope if given, otherwise from the group.
   - The drilldown fields are `canonicalGroupKey`, `operationalScope`, `fixSource`, `capability`, `severity`, `issueSource`, `affectedProducts`, `affectedVariants`, `affectedMarketplaces`, `affectedLanguages`, `providerRows`, `syncRows`.
6. **Webapp de-duplication:** `getCommandCenterTopSignalRenderItems` (`_lib/render-keys.ts`) collapses signals that share a render key and keeps the higher severity, then the higher count.
   - The key is built from, in order of preference: `canonicalGroupKey`, then `workflowIdentityKey`, then an action/issue/entity/channel/marketplace/language key.

### 10.4 Click → where
**Row click** → `openActionCenter({ ...cta.filters, ...drilldown })` → `router.push('/actionCenter?<params>')`. Params that are `null`, `''` or `false` are dropped.

On the **Action Center** (`actionCenter/main.tsx` → `parseActionCenterUrlFilters` + `parseOperationalDrilldownContext`):
- It reads `channelId`, `marketplaceId`, `issueType`, `health`, `workflowStatus` and `issueGroupKey`/`category` into its compact filters. It **does not read `mode`**; the Action Center opens in Sales Operations by default.
- When `canonicalGroupKey` is present, it **restricts the list to the matching operational group** (compound parents and children that share the key).
- **Result:** the user lands on the Action Center pre-filtered to exactly that issue group, where they can open, assign, snooze or resolve tickets.

### 10.5 Data sources
Everything in §9. 🗄️ The main tables are `tbl_product_channels`, `tbl_product_channel_marketplaces`, `tbl_variant_marketplaces`, `tbl_product_variant_prices`, `tbl_amazon_fba_inventories`, `tbl_warehouse_products`, `tbl_channel_product_warehouses`, `tbl_channel_product_errors`, `tbl_channels`, `tbl_action_center_issue_workflows` and `tbl_action_center_issue_assignments`.

---

## 11. NEXT

**Title / subtitle:** "NEXT" · "Recommended next actions"
**Builder:** `next.builder.ts` → `buildNextSection`

### 11.1 Objective
Turn the open issue groups into up to **five fixed, templated recommendations**, ranked by impact. These are canned text templates per issue type; no AI is involved.

### 11.2 UI anatomy
Each card shows:
- a "NEXT STEP" badge and an impact pill (High / Medium / Low)
- the title, and the operational scope label (when present)
- the description
- a **CTA button** on the right. Its label comes from the CTA type: `OPEN_CHANNEL` → "Open channel", `OPEN_PRODUCTS` → "Open products", otherwise → **"Review"**. In practice every recommendation is `OPEN_ACTION_CENTER`, so every button reads **"Review"**.

**Empty state:** "No next-step recommendations right now".

### 11.3 Recommendation catalogue (`RECOMMENDATION_CONFIG`)

| Issue type | id | Title | Impact |
|---|---|---|---|
| `MISSING_PRICE` | add-missing-prices | Add sellable prices to affected listings | high |
| `OUT_OF_STOCK` | restock-products | Restock unavailable products | high |
| `LOW_STOCK` | replenish-low-stock | Replenish low stock before it turns into outages | medium |
| `SYNC_FAILED` | fix-sync-issues | Review highest-impact sync failures | high |
| `AMAZON_ERROR` | resolve-amazon-issues | Resolve highest-impact publication blockers | high |
| `CHANNEL_ERROR` | review-channel-issues | Review channel publication blockers | medium |
| `CHANNEL_NOT_CONNECTED` | connect-sales-channel | Reconnect sales channel · *single channel: `Connect <Provider> channel "<name>"`* | high |
| `CHANNEL_CONNECTION_ERROR` | fix-channel-connection | Fix failed channel connection · *single channel: `Reconnect <Provider> channel "<name>"`* | high |
| `CHANNEL_SETUP_INCOMPLETE` | complete-channel-setup | Complete channel setup · *single channel: `Complete setup for <Provider> channel "<name>"`* | high |
| `SHOPIFY_UNPUBLISHED` | review-unpublished-listings | Publish affected listings | high |
| `SHOPIFY_INACTIVE` | review-inactive-listings | Reactivate inactive listings | high |

Each description is generated from the count. For example: "3 affected listings still need pricing before they can sell cleanly in this scope."

### 11.4 How recommendations are built
1. Take only rows with **`workflowStatus = OPEN`** and a `topIssueType` that appears in the catalogue.
2. **Group by `canonicalGroupKey`.** For each group:
   - `count` = the sum of `affectedItemCount` (at least 1 per row)
   - `maxPriorityScore` and `affectedVariants` are tracked across the group
3. Sort by impact, then `maxPriorityScore`, then count, then key. Keep the **top 5**.
4. Build the title. It is the template title, except when `count === 1` and the group is a channel blocker; then it uses the channel name and provider.
5. Build the CTA: `OPEN_ACTION_CENTER` with `{ mode: 'SALES_OPERATIONS', issueType, workflowStatus: 'OPEN', channelId, marketplaceId, languageCode, …drilldown }`.
6. **Webapp:** `getCommandCenterRecommendationRenderItems` de-duplicates by render key and keeps the higher impact, then the higher number of affected products.

### 11.5 Click → where
**[Review]** → `handleRecommendationNavigation`. Because the type is `OPEN_ACTION_CENTER`, it calls `openActionCenter(payload + drilldown)` → **`/actionCenter?mode=SALES_OPERATIONS&issueType=…&workflowStatus=OPEN&…&canonicalGroupKey=…`**. The Action Center opens filtered to that issue group, **Open** tickets only.

The handler also supports two other CTA types, which are never produced by NEXT today:
- `OPEN_PRODUCTS` → the Products flow (§12.4)
- `OPEN_CHANNEL` → `/channelManagement?<payload>`

### 11.6 Data sources
Same as NOW (§9 / §10.5).

---

## 12. AT-RISK PRODUCTS

**Title / subtitle:** "AT-RISK PRODUCTS" · "Products needing the most attention right now"
**Builder:** `at-risk.builder.ts` → `buildAtRiskProductsSection`

### 12.1 Objective
List the **five products** with the worst current issue, each with a **direct link to the fix screen** for that product.

### 12.2 UI anatomy
A two-column grid of clickable rows. Each row shows:
- the product name (or the SKU, or "Product <id>"), and a severity pill
- the issue label: `routing.displayIssueLabel`, else `topIssueLabel`, else the humanised type
- the context: `<channel> / <marketplace>`, or "Open the fastest fix path."
- a ↗ icon

**Empty state:** "No at-risk products right now".

### 12.3 Selection rules
1. Skip rows with no `productId`, which excludes the channel blocker rows (`productId = 0`).
2. Skip rows with `health = HEALTHY`.
3. Keep only these `topIssueType` values: `OUT_OF_STOCK`, `MISSING_PRICE`, `SYNC_FAILED`, `AMAZON_ERROR`, `CHANNEL_ERROR`, `OUT_OF_SYNC`, `SHOPIFY_UNPUBLISHED`, `SHOPIFY_INACTIVE`, `LOW_STOCK`.
4. Keep **one row per product**, the best one by `compareRows`. The rows are compared on:
   1. workflow priority: OPEN 4 > PENDING_VERIFICATION 3 > IN_PROGRESS 2 > WAITING_BLOCKED/TODO 1
   2. severity: BLOCKED 3 > AT_RISK 2 > NEEDS_ATTENTION 1
   3. issue priority: OUT_OF_STOCK 9 > MISSING_PRICE 8 > SYNC_FAILED 7 > AMAZON_ERROR 6 > CHANNEL_ERROR 5 > OUT_OF_SYNC 4 > SHOPIFY_UNPUBLISHED 3 > SHOPIFY_INACTIVE 2 > LOW_STOCK 1
   4. `priorityScore`
   5. the most recently updated
   6. the name
5. Sort all products with the same comparison and keep the **top 5**.
6. Map severity: BLOCKED or AT_RISK → high; NEEDS_ATTENTION → medium.

### 12.4 Click → where (the `OPEN_PRODUCTS` flow)
The CTA payload comes from the row's `routing.primaryAction`, falling back to its `quickAction`:

| Action type | Destination | Params |
|---|---|---|
| *(no action)* | `/products/<id>?q=details` | `activateChannelId = <channelId or 'all'>` |
| `OPEN_CHANNEL_TAB` | `/products/<id>?q=channels` | `channelId`, `marketplaceId`, `variantId`, `variantSKU`; activate the channel |
| `FIX_IN_MASTER` | `/products/<id>?q=details` | `variantId`, `languageCode`; activate **Master**; `tempData.search` |
| `EDIT_PRICE` | `/products/<id>?q=pricing` | `variantId`, `variantSKU`; `tempData { search, channelId, marketplaceID }` |
| `UPDATE_STOCK` / `ADD_STOCK_IN_WAREHOUSE` | `/products/<id>?q=stock` | same as above |
| `OPEN_ERROR_DETAILS` and any other type | `/products/<id>?q=syncError` | plus `ErrorChannelId` and `marketplaceId` (for `OPEN_ERROR_DETAILS`); `tempData` |

**Webapp `openProducts(payload)`:**
1. **`applyProductsChannelContext`**: when `activateChannelId` is set, it sets the global active channel tab.
   - `'all'`: the Master tab, via `setActiveChannelTabId`, `setActiveChannel` and `setChannelData`.
   - A channel id: that channel's tab, but **only if** it is already in `useGlobalStore.tabChannelsData.channels`. That list is filled by pages that load the channel bar (products, orders, categories, readiness). If the user opened the Command Center directly, the list may be empty, and the channel tab isn't switched.
2. It builds the query from every other key, **excluding** `activateChannelId`, `productId` and `tempData`.
3. With a `productId`: `/products/<id>?<query>`. When `tempData` is present, it is appended with `buildUrlWithTempData` (encoded temporary navigation data).
4. Without a `productId`: `/products?<query>`.

**On the product detail page** (`products/[product]/page.tsx`):
- `q` selects the tab: `overview` · `details` · `variants` · `stock` · `pricing` · `readiness` · `channels` · `syncError`. If the user lacks permission for that tab, they are redirected to the default permitted one.
- `ErrorChannelId`, `variantId`, `variantSKU`, `marketplaceId`/`marketplaceID` and `languageCode` pre-select the context inside the tab.
- **Result:** the user lands on the exact fix screen, such as Pricing, Stock, Channel listing or Sync errors, for that product, channel and marketplace.

### 12.5 Data sources
Same as NOW (§9). Row routing also depends on 🗄️ `tbl_action_engine_settings` (runtime routing).

---

## 13. CATALOG STATUS

**Title / subtitle:** "CATALOG STATUS" · "Content readiness and publishing overview"
**Endpoint:** `POST /command-center/catalog-status` → `getCatalogStatusForContext`
**Files:** `catalog-status.service.ts` (data), `catalog-status.builder.ts` (shape)

### 13.1 Objective
Show, for a chosen **scope** (Master / All channels / one channel) and optionally a **marketplace**:
- **Readiness**: how many sellable units are content-complete.
- **Publishing**: where the listings stand on the external channel.

### 13.2 UI anatomy
- **Selectors:** a **Scope** dropdown and a **Marketplace** dropdown. The Marketplace dropdown appears only when the scope has marketplace options.
  - These selections are **local component state**. They don't change the URL and don't trigger an API call; all scope and marketplace views arrive in one payload.
- **Left card: Readiness**
  - A "Tracked records: N" chip and "Readiness for <scope>".
  - Four clickable metric cards: **Ready** (green), **Almost ready**, **Needs attention** (amber), **Blocked** (red).
- **Right card: Publishing**
  - A "Tracked listings: N" chip and "Publishing for <scope>".
  - Four clickable metric rows: **Published / live**, **Out of sync**, **Not published**, **Inactive / hidden**.
  - For the **Master** scope, the card shows a quiet message instead: "Publishing status is limited to channel scopes".
- **Fallback:** "Catalog unavailable" when there is no scope at all.

### 13.3 How the data is gathered (API)

1. **Cache check** (`command-center:catalog-status:…`, 90 s).
2. Run two loads in parallel:
   - **a. `catalogStatusService.getCommandCenterCatalogStatusData(req, scope)`.** If it throws, the error is logged and the empty `{readinessRows: [], publishRows: []}` is used, so the section still renders.
     1. **`getFullProductRecords`** (loaded once, shared by readiness and publishing):
        - It selects non-deleted, non-archived products that have an **active** `tbl_product_channels` row on a **CONNECTED Amazon or Shopify** channel in scope.
        - It includes the channel's `tbl_shopify_language_mappings` and `tbl_shopify_channels.default_currency`.
        - It includes the active `tbl_product_channel_marketplaces` (with the active `tbl_amazon_channel_marketplaces` + `tbl_currencies`) and their active `tbl_variant_marketplaces`.
        - It includes the variants with their active `tbl_product_variant_channels`.
     2. **Readiness rows** (`buildCommandCenterReadinessRows`), described in 13.4.
     3. **Publish rows** (`buildCommandCenterPublishRows`), described in 13.5.
   - **b. `channelService.getRoleBasedChannelsForSetup(req, undefined, scope)`**: all **active, non-deleted** channels the user can access (any type), plus the Amazon marketplace-language map. 🗄️ `tbl_channels`, `tbl_amazon_channels`, `tbl_amazon_channel_marketplaces`.
3. **`buildCatalogStatusSection`** builds the scope options and views (13.6).
4. Cache for 90 s and return.

### 13.4 Readiness counting

**Readiness values:** 🗄️ `tbl_readiness_values`, one row per product × variant × channel × language, with `readiness_value` from 0 to 100. The value is floored, and a missing value counts as **0**.

**Unit counted:** a product with no variants counts as **1 unit**. A product with variants counts **each variant** (for channel scopes, only the variants active in that channel).

**Buckets** (the same everywhere):

| Bucket | Readiness | Products-page filter used by the CTA |
|---|---|---|
| **Ready** | = 100 | `readinessValue=100` |
| **Almost ready** | 75–99 | `readinessValue=75-99` |
| **Needs attention** | 50–74 | `readinessValue=50-74` |
| **Blocked** | < 50 | `readinessValue=0-49` |

| Scope | How a unit's score is picked |
|---|---|
| **Master** | • Summary: the **average** of the unit's master readiness (`channel_id IS NULL`) across all its language rows. <br>• Per language: the score for each account language plus the base language. <br>• The master row is included **only if** the user has all-channel access and `getMasterSellableCount() > 0`. <br>🗄️ `tbl_account_configurations` (base language), `tbl_account_languages`, `tbl_products`, `tbl_product_variants` |
| **Amazon channel** | • Summary: the average of the unit's channel readiness across the languages of the **marketplaces it is listed on**. <br>• Per marketplace: the score for that marketplace's language. <br>• A unit on an Amazon channel with **no active marketplace listing** is counted under the language `unassigned` as **Blocked** (score 0). |
| **Shopify channel** | • Summary: the average across the channel's **Shopify language mappings**. <br>• Per language: the score for each mapped language. |

### 13.5 Publishing counting

**Prices:** 🗄️ `tbl_product_variant_prices` rows with `channel_id IS NOT NULL` (channel prices). An `offerPrice` that is not inherited is used first; the code then tries the default (master) price. ⚠️ Master prices are excluded by the query filter, so that fallback never finds anything (§22).

For each channel and each unit:

| Platform | Iterated over | Status source |
|---|---|---|
| Amazon | each active PCM (marketplace) × (product without variants, or each active variant) | PCM or VM `sync_status`, `external_status`, `listing_errors`, `amazon_stock` + price |
| Shopify | **each mapped language** × unit, so a unit counts once per language | product channel / variant channel `sync_status`, `external_status`, `external_stock` + price |

**Classification of each listing:**
1. `sync_status = NOT_PUBLISHED` → **Not published**.
2. Otherwise:
   - If `sync_status = OUT_OF_SYNC`, also add 1 to **Out of sync**. This is an overlay; the listing is still classified below.
   - Then work out the main status (`getFullStatusConfig` / `getShopifyFullStatusConfig`):
     - active marker present (Amazon `BUYABLE`, Shopify `ACTIVE`) → `ACTIVE`
     - not active + error `CATALOG_ITEM_REMOVED` or `BRAND_ABUSE_DETECTED` → `INACTIVE`
     - not active + no price → `MISSING_OFFER`
     - not active + no stock → `OUT_OF_STOCK`
     - not active otherwise → `INACTIVE`

**Metric roll-up:**

| Metric | Formula | Products-page filter used by the CTA |
|---|---|---|
| **Published / live** | ACTIVE + MISSING_OFFER + OUT_OF_STOCK | `listingStatus=ACTIVE,MISSING_OFFER,OUT_OF_STOCK` |
| **Out of sync** | the OUT_OF_SYNC overlay count | `syncStatus=out_of_sync` |
| **Not published** | NOT_PUBLISHED | `syncStatus=not_published` |
| **Inactive / hidden** | INACTIVE | `listingStatus=INACTIVE` |
| **Tracked listings** (total) | Published + Not published + Inactive (Out of sync is **not** added) | — |

### 13.6 Scopes, marketplaces and defaults (`catalog-status.builder.ts`)

| Scope id | Shown when | Readiness | Publishing | CTA base payload |
|---|---|---|---|---|
| `master` | the user has master access | the master row | **null** (the quiet message) | `activateChannelId: 'all'` |
| `all_channels` | at least one accessible channel | the **sum** of all channel rows (a unit in 2 channels is counted twice) | the sum of all publish rows | `activateChannelId: 'all'`, `channelFilter: '<id,id,…>'` |
| `channel:<id>` | one per accessible channel | that channel's row | that channel's row | `activateChannelId: '<id>'` |

- **Marketplace options** exist only for channel scopes.
  - They are built from each `languageData` entry that has a `marketplaceId` (in practice, Amazon).
  - The label is the last segment of the language code in upper case: `de_DE` → **DE**, and `unassigned` → "Unassigned". The options are sorted by label.
  - A marketplace view adds `marketPlace=<amazonMarketplaceId>` to every CTA.
- **Default scope:** the channel scope matching the URL's `channelId`, then Master, then All channels, then the first scope.
- **Default marketplace:** the URL's `marketplaceId`, only when it belongs to the default channel; otherwise "All marketplaces".
- **The scope selection only picks the default view.** The readiness and publish rows are **not** filtered by the request's `channelId` or `marketplaceId`; every accessible channel's view comes back in one payload.
- The webapp keeps the user's selection across refetches as long as that scope or marketplace still exists.

### 13.7 Click → where
Every metric → `OPEN_PRODUCTS` → `openProducts(payload)`. There is no `productId`, so the destination is **`/products?<params>`**:

1. `activateChannelId` switches the global channel tab: Master for `'all'`, or the channel (§12.4 caveat applies).
2. The Products list (`products/_components/list/List.tsx`) reads:
   - `readinessValue` → the readiness range filter
   - `marketPlace` → the marketplace filter
   - `listingStatus` → the listing status filter (comma list)
   - `syncStatus` → the sync status filter
   - `channelFilter` → a channel id list (for "All channels" under the Master tab)
3. **Result:** the product grid is pre-filtered to the exact set that was counted, for example "Blocked readiness on Amazon DE".

### 13.8 Data sources
🗄️ `tbl_products`, `tbl_product_variants`, `tbl_product_channels`, `tbl_product_variant_channels`, `tbl_product_channel_marketplaces`, `tbl_variant_marketplaces`, `tbl_channels`, `tbl_amazon_channels`, `tbl_amazon_channel_marketplaces`, `tbl_shopify_channels`, `tbl_shopify_language_mappings`, `tbl_currencies`, `tbl_readiness_values`, `tbl_product_variant_prices`, `tbl_account_configurations`, `tbl_account_languages`.

---

## 14. AMAZON EVENTS

**Title / subtitle:** "AMAZON EVENTS" · "Amazon signals surfaced from the live projection cohort"
**Endpoint:** `POST /command-center/amazon-events` → `getAmazonEventsSummaryForContext` (**not cached**)
**Builder:** `amazon-events.builder.ts`

### 14.1 Objective
Surface up to five **live Amazon provider signals** (account health, Buy Box, price policy). These come from the Amazon notification pipeline, not from Categra's own catalog state.

### 14.2 Visibility
The whole section is rendered **only if `items.length > 0`**. When it is hidden, its `generatedAt` and refreshing state are ignored in the summary strip. A failed request also just hides the section; no error is shown.

### 14.3 Gates (API), in order
Any gate that fails returns `amazonEvents: null`.

1. **Env feature flags**, all required: `AMAZON_EVENTS_LIVE_ENABLED`, `AMAZON_EVENTS_COMMAND_CENTER_ENABLED`, `AMAZON_EVENTS_COMMAND_CENTER_SECTION_ENABLED`. Each accepts `1`, `true` or `yes`; the default is off.
2. **Account allowlist:** `AMAZON_EVENTS_COMMAND_CENTER_ACCOUNT_ALLOWLIST` (comma-separated ids). Empty means all accounts.
3. **Family flags** (each defaults to **on**):
   - `…_ACCOUNT_CRITICAL_RESTRICTION_ENABLED`
   - `…_ACCOUNT_DEGRADED_ENABLED`
   - `…_PRICE_POLICY_AT_RISK_ENABLED`
   - `…_BUY_BOX_LOST_ENABLED`
   - `…_BUY_BOX_EXCLUDED_ENABLED`
4. **Accessible Amazon channels:** active, non-deleted `AMAZON` channels, filtered by the URL's `channelId` or the user's channel scope, and then by `AMAZON_EVENTS_COMMAND_CENTER_CHANNEL_ALLOWLIST`. 🗄️ `tbl_channels`
5. **Active projection generation:** the latest `tbl_amazon_event_derivation_generations` row with `generation_type = PROJECTION_EVALUATION` and `generation_status = ACTIVE`.

### 14.4 Data gathering steps
1. **Candidates** 🗄️ `tbl_amazon_event_projection_candidates`, filtered to:
   - this generation and this account
   - `family` in the enabled families
   - `projection_target ∈ {COMMAND_CENTER, BOTH}`
   - `projection_status = PROJECTABLE`, `is_projectable = true`, `suppressed_by_edge_id IS NULL`
   - ordered by `updatedAt` descending
2. **Joins**, run in parallel:
   - 🗄️ `tbl_amazon_event_occurrences` (`current_severity`, `last_observed_at`)
   - 🗄️ `tbl_amazon_event_scope_identities` (channel, Amazon channel, marketplace, product, listing, ASIN, seller SKU)
3. **Filter** to the accessible channels and, when given, to the URL's `marketplaceId` (compared to `scopeIdentity.amazonMarketplaceId`).
4. **Marketplace labels:** 🗄️ `tbl_amazon_channel_marketplaces` → `tbl_amazon_marketplaces`, formatted as "`Country (CC)`".
5. **Listing protection:** read from the candidate's `projection_reason_detail_json.listingProtection`, falling back to `evidence_summary_json.listingProtection`. This yields `isListingProtected`, `contextStatus` and `wasProjectionBoosted`.
6. **Remove duplicates of Action Center Buy Box work.** For a `BUY_BOX_LOST` or `BUY_BOX_EXCLUDED` row on a protected listing with a product and channel:
   - The service takes another Action Center snapshot, which hits the 15 s cache.
   - It drops the row when an open Action Center row with `topIssueType ∈ {BUY_BOX_LOST, BUY_BOX_EXCLUDED, BUY_BOX_UNAVAILABLE, BUY_BOX_AT_RISK}` exists for the same product × channel × marketplace. ⚠️ See §22.
7. **Presentation** (`buildAmazonEventsSection`):

| Family | Rank | Title | Informational? | CTA label | Preferred CTA template(s) | Extra rule |
|---|---|---|---|---|---|---|
| `ACCOUNT_CRITICAL_RESTRICTION` | 1 | Critical Amazon account restriction | no | Review connection | `OPEN_ACCOUNT_CONNECTION_SETTINGS` | — |
| `BUY_BOX_LOST` | 2 | Protected listing lost the Buy Box | no | Review pricing | `OPEN_PRODUCT_PRICING_PAGE` → `OPEN_CHANNEL_LISTING_PAGE` | only **protected** listings |
| `BUY_BOX_EXCLUDED` | 2 | Protected listing excluded from Buy Box | no | Review listing | `OPEN_CHANNEL_LISTING_PAGE` → `OPEN_PRODUCT_PRICING_PAGE` | only **protected** listings |
| `ACCOUNT_DEGRADED` | 3 | Amazon account health degraded | no | Review connection | `OPEN_ACCOUNT_CONNECTION_SETTINGS` | — |
| `PRICE_POLICY_AT_RISK` | 4 | Price policy risk detected | **yes** | Review pricing | `OPEN_PRODUCT_PRICING_PAGE` | — |

   - The **template** is the first preferred key that appears in the candidate's `cta_template_keys_json`. If none matches, the item is dropped. Pricing and listing templates also need a `productId`.
   - **Severity** comes from the occurrence severity: `ERROR` → high; `WARNING` → medium (low for price policy); `INFO` → low. When the severity is missing: high for a critical restriction, medium otherwise.
   - Items are sorted by rank, then `lastObservedAt` (descending), then candidate id (descending). **Only 5 are kept**; `totalCount` is the full count.

### 14.5 UI anatomy
- A header chip, "N signals" (the number of items shown), and a phase-one notice (`Command_Center_Amazon_Events_Phase_One_Notice`).
- Each item row shows:
  - the title, a severity pill, a **"Protected listing"** pill (when protected) and an **"Informational"** pill (price policy)
  - the description
  - a context line: `<channel> | <Country (CC)> | Product #<id> | ASIN <asin>`
  - the CTA label pill and a ↗ icon

### 14.6 Click → where (whole row)

| Template | Navigation | Destination |
|---|---|---|
| `OPEN_ACCOUNT_CONNECTION_SETTINGS` | `OPEN_CHANNEL { channelId }` | **`/channelManagement?channelId=<id>`**. The channel list finds that channel and **opens its edit/config dialog** (Amazon or Shopify), where the user reviews the connection. `reauthorize=true` would jump to the re-auth step, but it is not sent from here. |
| `OPEN_PRODUCT_PRICING_PAGE` | `OPEN_PRODUCTS { productId, q: 'pricing', activateChannelId, tempData {channelId, marketplaceID} }` | **`/products/<id>?q=pricing`** with the channel tab activated: the product's **Pricing** tab, scoped to the channel and marketplace. |
| `OPEN_CHANNEL_LISTING_PAGE` | `OPEN_PRODUCTS { productId, q: 'channels', activateChannelId, channelId, marketplaceId, tempData }` | **`/products/<id>?q=channels&channelId=…&marketplaceId=…`**: the product's **Channels** tab on that listing. |

### 14.7 Data sources
🗄️ `tbl_channels`, `tbl_amazon_event_derivation_generations`, `tbl_amazon_event_projection_candidates`, `tbl_amazon_event_occurrences`, `tbl_amazon_event_scope_identities`, `tbl_amazon_channel_marketplaces`, `tbl_amazon_marketplaces` (plus the Action Center tables from §9, for the de-duplication step).
🌐 **Environment variables:** the feature flags and allowlists in §14.3.

---

## 15. STOCK

**Title / subtitle:** "STOCK" · "Inventory risk in the selected scope"
**Builder:** `stock.builder.ts` → `buildStockSection` (from the same operational rows as §9)

### 15.1 Objective
Give two headline inventory numbers, each linked to the matching Action Center queue.

### 15.2 UI and rules

| Row | Count | Tone |
|---|---|---|
| **Out of stock** | visible rows with `topIssueType = OUT_OF_STOCK` | red |
| **Low stock** | visible rows with `topIssueType = LOW_STOCK` | amber |

- These are **row counts** (product × channel × marketplace), not unit counts.
- A row counts only when stock is its **top** issue. For example, an out-of-stock row whose top issue is `MISSING_PRICE` is not counted (§9.4).
- The stock and threshold resolution (FBA, then warehouse, then channel stock) is in §9.3.

### 15.3 Click → where
Each row → `OPEN_ACTION_CENTER { mode: 'SALES_OPERATIONS', issueType: 'OUT_OF_STOCK' | 'LOW_STOCK', channelId?, marketplaceId? }` → **`/actionCenter?issueType=OUT_OF_STOCK…`**.

The Action Center opens filtered to that issue type, with **no** `canonicalGroupKey`, so across all groups. From there, the row quick actions lead to the product's Stock tab.

### 15.4 Data sources
§9 tables, especially 🗄️ `tbl_amazon_fba_inventories`, `tbl_warehouse`, `tbl_warehouse_products`, `tbl_channel_product_warehouses`, `tbl_product_channels.low_stock_threshold`, `tbl_product_variant_channels.low_stock_threshold`, and `amazon_stock` on `tbl_product_channel_marketplaces` / `tbl_variant_marketplaces`.

---

## 16. CAPACITY

**Title / subtitle:** "CAPACITY" · "Current package and resource usage" (in setup mode: "PLAN CAPACITY", §8.2)
**Endpoint:** `POST /command-center/capacity-summary` (empty body) → `getCapacitySummaryForContext` → `PackageSubscriptionManagerService.getPackageResourcesSnapshot(req)` → `buildCapacitySection`

### 16.1 Objective
Show usage against the plan limits for **Channels**, **Products (SKUs)** and **Storage**.

### 16.2 How the data is gathered (API)
1. **Current subscription:** `getRuntimeCurrentSubscription` → 🗄️ `tbl_account_subscriptions` (with the package limits from 🗄️ `tbl_packages` and the `consumed_package_resources` JSON).
   - If there is no subscription, it returns null limits and zero usage.
2. **Channels used is counted live:**
   - trial: `countTrialSalesChannels`
   - paid: `countPackageChannels`
   - 🗄️ `tbl_channels`. This overrides `consumedChannels`.
3. A pending plan change is loaded to work out exceeded resources and the compliance state. This isn't shown in the capacity cards.
4. **Metrics:**

| Metric | Limit | Used |
|---|---|---|
| Products | `totalAllowedSKUs` (`-1` = unlimited) | `consumedPackageResources.consumedSKUs` |
| Channels | `totalAllowedChannels` | the live channel count |
| Storage | the package storage (for example "10 GB"), parsed to bytes | `consumedPackageResources.consumedStorage`, parsed to bytes |

5. **`buildCapacitySection`:**
   - `limit: null` means the limit is unknown. `-1` means unlimited.
   - Storage is converted to **MB or GB**: the package unit if set, otherwise GB when the limit (or usage) is ≥ 1 GiB. It is rounded to 2 decimals for GB and 0 for MB.
6. Cache for 90 s.

### 16.3 Webapp rendering (`CapacitySection`)

**Channels and Products:** `buildCapacityItem`

| Limit | Shown as |
|---|---|
| `-1` | "Unlimited" / "Unlimited plan" |
| `null` | "Loading…" / "Loading limits" |
| `0` | "Limit reached" when used > 0, else "0 used" |
| positive | "N% used" (`formatCapacityUsagePercent`, which gives finer decimals below 10%) |

**Storage:** `resolveCommandCenterStorageDisplay` (`_lib/storage-display.ts`) prefers the **live account storage summary** from the global store:

| Value | Source, in order of preference |
|---|---|
| Used label | `summaryData.storage.used`, else the API value |
| Limit label | `summaryData.storage.total` (unless it is 0 or empty), else `packageResources.storage`, else the API limit (`-1` → Unlimited) |
| Percent | `formattedPercentageUsed`, else `usedPercentage`, else computed |

🌐 The storage summary comes from `GET accounts/get-account-usage-summary` (`useAccountSummary`) and is kept live by the **socket event `STORAGE_INFORMATION`**, which the sidebar listens to.

**Warning badge:** usage ≥ 100% → **"At limit"**; above 80% → **"Above 80%"**. When the badge shows, the progress bar turns orange.

**Clickable elements:** none in dashboard mode. In setup mode there is an [Upgrade plan] button on trials (§8.2).

### 16.4 Data sources
🗄️ `tbl_account_subscriptions`, `tbl_packages`, `tbl_channels`, `tbl_trial_usage_summaries` (trial counters in setup mode).
🌐 Account usage summary API + socket `STORAGE_INFORMATION`; `package-subscription-manager/fetch-resources` (global `packageResources`).

---

## 17. TREND

**Title / subtitle:** "TREND" · "Shipped orders and revenue for the selected period"
**Endpoint:** `POST /command-center/trend-summary` → `getTrendSummaryForContext` → `OrderService.getCommandCenterTrendData` → `buildTrendSection`

### 17.1 Objective
Compare **revenue from shipped orders** and **order count** for the selected period against the previous period of the same length, and plot the series.

### 17.2 Controls
**Period selector** (the section header): `7d · 30d · 90d · 6m · 12m`. A click writes `?period=` with `router.replace` (no scroll), which refetches **trend only**.

### 17.3 Windows and series granularity (`utils/sellerTrendMetrics.ts`)

All windows use **UTC** boundaries and "now" as the as-of date.

| Period | Current window | Previous window | Series bucket |
|---|---|---|---|
| `7d` | the last 168 hours (hour-aligned) | the 168 hours before that | **hour** (168 points) |
| `30d` | today and the 29 days before | the 30 days before that | **day** |
| `90d` | today and the 89 days before | the 90 days before that | **day** |
| `6m` | from the Monday 25 weeks ago to today | the 26 weeks before | **week** (Monday start) |
| `12m` | from the 1st of the month 11 months ago to today | the 12 months before | **month** |

### 17.4 Eligibility and formulas
- **Eligible order item:**
  - `GREATEST(quantity_shipped, 0) > 0`, and
  - the order status is not `CANCELED` or `UNFULFILLABLE` (null status is allowed).
- **Date basis:** `tbl_orders.purchase_date`. There is no normalised ship date.
- **Orders** = `COUNT(DISTINCT order_id)` over eligible items.
- **Revenue** = `Σ quantity_shipped × item_price_amount`.
  - Shipping, tax, discounts, refunds and returns are **not** netted.
- **Scope filters:**
  - The channel is limited to the user's accessible channels, or to the URL's `channelId`.
  - `marketplaceId` filters `tbl_orders.amazon_channel_marketplace_id`. ⚠️ This is a different id space from the other sections (§22).
- **Currency conversion** to the account **base currency**:
  - The item currency is `item_price_currency_code`, falling back to `order_total_currency_code`.
  - Revenue is grouped by window or bucket, by conversion date (**hour** for `7d`, otherwise **day**), and by currency.
  - Each group is converted with the **nearest historical rate snapshot** from 🗄️ `tbl_currency_exchange_rates`. The snapshots loaded are those in the range plus the nearest before and after, and rates are cached per currency and date.
  - A group that can't be converted is **skipped** and reported.
- **Revenue status:**
  - `available`: every group converted
  - `partial`: some groups skipped (lists `skippedCurrencyCodes` and `skippedRevenueRowCount`)
  - `unavailable`: no group converted, but some existed
  - When unavailable, all revenue values are sent as 0.
- **Delta:** `(current − previous) / previous × 100`, to 2 decimals. It is `null` when previous = 0 and current > 0, and `0` when both are 0.

### 17.5 UI anatomy
- **Revenue from shipped orders** panel:
  - The value, formatted as currency with no decimals, or "Revenue unavailable".
  - A delta pill:
    - "New" when previous = 0 and current > 0 (positive)
    - "Flat" when both are 0
    - "+N%" in green, or "−N%" in red
  - A "Up/Down from <previous>" line, or "Flat vs previous period".
  - When `partial`: a **PARTIAL** badge and the helper "Revenue partially converted to <CUR>", followed by "Missing rates: <codes>" or "<n> revenue rows excluded".
- **Orders** panel: the value, the delta pill and the comparison line.
- **Chart** (`_components/command-center-trend-chart.tsx`, a Recharts `LineChart`, lazy-loaded without SSR):
  - It plots **revenue**, or **orders** when revenue is unavailable.
  - The x-axis labels depend on the period: hour for 7d, month for 12m, day otherwise.
  - The tooltip shows revenue (with "partial" when applicable) and orders.
  - A caption reads "Revenue in <CUR>" or the exchange-rate-unavailable note.
- **No data** (0 orders in both windows and in every point): "No trend data" message instead of the chart.
- **Clickable elements:** only the period buttons.

### 17.6 Data sources
🗄️ `tbl_orders` (`purchase_date`, `order_status`, `channel_id`, `amazon_channel_marketplace_id`, `order_total_currency_code`), `tbl_order_items` (`quantity_shipped`, `item_price_amount`, `item_price_currency_code`), `tbl_currency_exchange_rates`, `tbl_account_configurations` (`base_currency`).
🌐 The rates themselves come from **exchangerate-api.com** (§18.5).

---

## 18. Upstream pipelines

These run **outside** the Command Center request and produce the rows it reads. Each is a lifecycle summary.

### 18.1 Readiness values → `tbl_readiness_values` (CATALOG STATUS; LOW_READINESS in the snapshot)
1. Any product, attribute, media, price, stock or channel mutation calls `ReadinessTriggerService` (`catalog/products/readiness/trigger/`).
2. The trigger merges the requested scopes and enqueues a **BullMQ `READINESS_RECALCULATION`** job (`QueueService`).
3. `readiness.processor.ts` (sync consumers) runs the job with `ReadinessExecutorService`:
   - It plans the scopes: master, then each channel, then each language or marketplace, then the parent and each variant.
   - It evaluates the readiness rules (`engine/*`: attribute, media, price, stock, bullet points…).
4. It writes one `tbl_readiness_values` row per product × variant × channel × language, with `readiness_value` from 0 to 100.
5. The webapp is notified over a socket. The Command Center simply re-reads the values on its next load, up to 90 s later because of the cache.

Related docs: `categra-dataset/context/readiness/*`.

### 18.2 Listing, sync and stock state (NOW, NEXT, AT-RISK, STOCK, CATALOG STATUS)
- **Forward sync** (Categra → channel) sets `sync_status` (`NOT_PUBLISHED`, `OUT_OF_SYNC`, `ERROR`…), `has_sync_errors` and `is_data_updated`, and writes failures to `tbl_channel_product_errors`. The fields live on `tbl_product_channels`, `tbl_product_channel_marketplaces`, `tbl_variant_marketplaces` and `tbl_product_variant_channels`.
- **Backward sync and Amazon notifications** (channel → Categra) update `external_status` (`BUYABLE`/`DISCOVERABLE`, or Shopify `ACTIVE`), `listing_errors`, `asin`, `amazon_stock` / `external_stock`, and FBA quantities in `tbl_amazon_fba_inventories`.
- **Warehouse module** maintains `tbl_warehouse`, `tbl_warehouse_products.total_quantity`, and the channel-to-warehouse mappings in `tbl_channel_product_warehouses`.
- **Pricing module** writes `tbl_product_variant_prices.pricing_details` (`offerPrice`, `salePrice`, `isInherited`).
- **Channel connection flows** set `tbl_channels.connection_status` and `attribute_mapping_status` (the channel blockers).

### 18.3 Action Center workflow state (the workflow columns in NOW and NEXT)
When users act in the Action Center (start, move to pending verification, close, snooze, assign), the changes are written to:
- 🗄️ `tbl_action_center_issue_workflows`
- 🗄️ `tbl_action_center_issue_assignments`
- 🗄️ `tbl_action_center_lifecycle_events`

Rows the snapshot re-detects on each read are overlaid with this state. A row with no saved state is **OPEN**.

### 18.4 Orders → `tbl_orders` / `tbl_order_items` (TREND; business-impact boost)
- **Amazon:** order processing (`commerce/orders/order/amazon/amazon-order.service.ts`, the queue processor `amazon-order-q1.processor.ts`), triggered by sync and order notifications. Rows are created or updated with purchase date, status, `quantity_shipped` and item price and currency.
- **Shopify:** backward sync `sync/productSyncing/shopify/backwardSync/shopify-order-bsq5.service.ts`.

### 18.5 Exchange rates → `tbl_currency_exchange_rates` (TREND)
`CronService.currencyExchangeRates`:
- It runs on `@Cron('02 00 */2 * *')`, which is 00:02 UTC every second day.
- For each currency it calls 🌐 `https://v6.exchangerate-api.com/v6/<key>/latest/<code>` and stores a snapshot row.
- The trend picks the nearest snapshot for each purchase hour or day.

### 18.6 Plan usage (CAPACITY)
- `tbl_account_subscriptions.consumed_package_resources` (`consumedSKUs`, `consumedStorage`, `consumedChannels`) is maintained by the billing and usage services (`customer-subscriptions.service.ts` and related) on subscription changes and usage events.
- Channels used is recounted live on each read (§16.2).
- Trial counters are kept in `tbl_trial_usage_summaries` (`trial-usage.service.ts`).
- The live storage numbers come from the account usage summary and the `STORAGE_INFORMATION` socket pushes.

### 18.7 Amazon events → projection candidates (AMAZON EVENTS)
1. 🌐 Amazon SP-API notifications arrive over **SQS** or **EventBridge** at `POST /amazon-notifications/sqs` and `POST /amazon-notifications/event-bridge` (`amazon-notifications.controller.ts`).
2. **Raw inbox** (`amazon-events/raw`): `ingestLiveNotification` saves the delivery, then canonicalises and de-duplicates the message. 🗄️ `tbl_amazon_event_raw_deliveries`, `tbl_amazon_event_raw_messages`, `tbl_amazon_event_raw_processing_attempts`
3. **Scope resolution** (`amazon-events/scope`): maps seller, marketplace and SKU/ASIN to Categra's channel, product and listing. 🗄️ `tbl_amazon_event_scope_identities`, `tbl_amazon_event_scope_aliases`, `tbl_amazon_event_scope_resolution_results`
4. **Provider-state normalisation** (the `account`, `buy-box` and `price-policy` services, in a `PROVIDER_STATE_NORMALIZATION` generation): creates normalised observations and **occurrences**, each with a family and `current_severity`. 🗄️ `tbl_amazon_event_normalized_observations`, `tbl_amazon_event_occurrences` (+ `_revisions`, `_sources`)
5. **Suppression** (`amazon-events/suppression`): builds edges that suppress redundant signals. 🗄️ `tbl_amazon_event_suppression_edges` (+ `_revisions`)
6. **Projection** (`amazon-events/projection`, in a `PROJECTION_EVALUATION` generation marked **ACTIVE**):
   - It decides for each occurrence whether it is projectable, and its target (Command Center, Action Center or both).
   - It records the CTA templates and the listing-protection evidence.
   - 🗄️ `tbl_amazon_event_projection_candidates` (+ `_revisions`), `tbl_amazon_event_derivation_generations`
7. The Command Center reads the active generation's candidates (§14).
8. Separately, Action Center materialisations (`tbl_amazon_event_action_center_materializations`) and notifications (`tbl_amazon_event_notification_outbox`) use the same candidates. The Command Center operational snapshot **excludes** those materialisations.

### 18.8 Channel access cache
`ChannelService.getUserWiseChannels` caches `{ allowAllChannels, allowedChannels }` in Redis per user. A change to a user's channel assignments takes effect once that entry is invalidated or expires.

---

## 19. Reference: tables per section

| Section | Tables (read) | Non-DB sources |
|---|---|---|
| Access (all) | `tbl_users`, `tbl_user_channels`, `tbl_user_accounts`, `tbl_roles`, `tbl_account_subscriptions`, `tbl_packages`, `tbl_account_configurations` | JWT; Redis (channel scope); `fetch-resources` in the webapp store |
| Summary strip | *(derived from the section responses)* | — |
| Banners | *(package snapshot)*: `tbl_account_subscriptions`, `tbl_packages` | `fetch-resources` API |
| NOW · NEXT · AT-RISK · STOCK | `tbl_products`, `tbl_product_variants`, `tbl_product_attributes`, `tbl_product_attribute_groups`, `tbl_attributes`, `tbl_product_media_linkers`, `tbl_digital_assets`, `tbl_product_channels`, `tbl_product_variant_channels`, `tbl_product_channel_marketplaces`, `tbl_variant_marketplaces`, `tbl_amazon_channel_marketplaces`, `tbl_amazon_marketplaces`, `tbl_shopify_channels`, `tbl_channels`, `tbl_currencies`, `tbl_product_variant_prices`, `tbl_amazon_fba_inventories`, `tbl_warehouse`, `tbl_warehouse_products`, `tbl_channel_product_warehouses`, `tbl_channel_product_errors`, `tbl_readiness_values`, `tbl_readiness_configurations`, `tbl_orders`, `tbl_order_items`, `tbl_account_configurations`, `tbl_action_engine_settings`, `tbl_action_center_issue_workflows`, `tbl_action_center_lifecycle_events`, `tbl_action_center_issue_assignments`, `tbl_users`, `tbl_attachments` | Redis (90 s); in-memory Action Center cache (15 s) |
| CATALOG STATUS | `tbl_products`, `tbl_product_variants`, `tbl_product_channels`, `tbl_product_variant_channels`, `tbl_product_channel_marketplaces`, `tbl_variant_marketplaces`, `tbl_channels`, `tbl_amazon_channels`, `tbl_amazon_channel_marketplaces`, `tbl_shopify_channels`, `tbl_shopify_language_mappings`, `tbl_currencies`, `tbl_readiness_values`, `tbl_product_variant_prices`, `tbl_account_configurations`, `tbl_account_languages` | Redis (90 s) |
| AMAZON EVENTS | `tbl_channels`, `tbl_amazon_event_derivation_generations`, `tbl_amazon_event_projection_candidates`, `tbl_amazon_event_occurrences`, `tbl_amazon_event_scope_identities`, `tbl_amazon_channel_marketplaces`, `tbl_amazon_marketplaces`, and the Action Center tables (de-duplication) | Env flags and allowlists; upstream SP-API notifications (SQS/EventBridge) |
| CAPACITY / PLAN CAPACITY | `tbl_account_subscriptions`, `tbl_packages`, `tbl_channels`, `tbl_trial_usage_summaries` | Account usage summary API; socket `STORAGE_INFORMATION`; `fetch-resources` |
| TREND | `tbl_orders`, `tbl_order_items`, `tbl_currency_exchange_rates`, `tbl_account_configurations` | exchangerate-api.com (through the cron); Redis (90 s) |

---

## 20. Reference: caches and freshness

| Layer | TTL | Effect on what the user sees |
|---|---|---|
| Redis Command Center section cache | 90 s | Numbers can be up to 90 s old. "Last updated" shows the oldest section's `generatedAt`. |
| Action Center base snapshot (in-process) | 15 s | Shared by the operational section and the Amazon-events de-duplication. The cache is per API instance. |
| Redis user channel scope | `REDIS_TTL` | Channel permission changes appear after invalidation or expiry. |
| Readiness values | async (BullMQ) | Scores trail edits by the queue latency. |
| Exchange rates | every 2 days | Revenue conversion uses the nearest snapshot. |
| Amazon events | not cached | Always reflects the current ACTIVE projection generation. |
| Webapp | no polling | Refreshes only on a reload, a URL change or a Retry. |

---

## 21. Reference: every click target

| # | Element | Section | Navigation | Params / state |
|---|---|---|---|---|
| 1 | **[New product]** | Setup | `/products/create` | `returnTo=/command-center?…`, `uniqId?` |
| 2 | **[Connect Amazon/Shopify]** | Setup | `/channelManagement` | store: reset setup, `changeChannel(null)`, `setCreateChannelDialogOpen(true)` |
| 3 | **[Upgrade plan]** | Plan capacity (trial) | `/settings/billing?tab=subscription#billing-manage-subscription` | — |
| 4 | Top signal row | NOW | `/actionCenter` | `mode`, `issueType`, `channelId?`, `marketplaceId?`, `languageCode?`, `canonicalGroupKey` + drilldown fields |
| 5 | **[Review]** | NEXT | `/actionCenter` | same as #4, plus `workflowStatus=OPEN` |
| 6 | Product row | AT-RISK | `/products/<id>?q=details\|channels\|pricing\|stock\|syncError` | `variantId?`, `variantSKU?`, `channelId?`, `marketplaceId?`, `ErrorChannelId?`, `languageCode?`, `tempData`; store: active channel tab |
| 7 | Scope / Marketplace selects | CATALOG STATUS | *(no navigation; local state)* | — |
| 8 | Ready / Almost ready / Needs attention / Blocked | CATALOG STATUS | `/products` | `readinessValue=100\|75-99\|50-74\|0-49`, `channelFilter?`, `marketPlace?`; store: active channel tab |
| 9 | Published / Out of sync / Not published / Inactive | CATALOG STATUS | `/products` | `listingStatus=ACTIVE,MISSING_OFFER,OUT_OF_STOCK\|INACTIVE` or `syncStatus=out_of_sync\|not_published`, `channelFilter?`, `marketPlace?`; store: active channel tab |
| 10 | Event row (connection) | AMAZON EVENTS | `/channelManagement?channelId=<id>` | opens that channel's config dialog |
| 11 | Event row (pricing) | AMAZON EVENTS | `/products/<id>?q=pricing` | `activateChannelId`, `tempData {channelId, marketplaceID}` |
| 12 | Event row (listing) | AMAZON EVENTS | `/products/<id>?q=channels` | `channelId`, `marketplaceId`, `tempData` |
| 13 | Out of stock / Low stock | STOCK | `/actionCenter` | `mode`, `issueType=OUT_OF_STOCK\|LOW_STOCK`, `channelId?`, `marketplaceId?` |
| 14 | Period buttons | TREND | `/command-center?period=…` (replace) | refetches trend only |
| 15 | **Retry** | any section fallback | *(re-POST the section)* | hidden when the section is plan-restricted |

---

## 22. Observations

These are behaviours seen while reading the code for this document. None of them was verified at runtime, and none was changed.

1. **The "Waiting" counter only counts `PENDING_VERIFICATION`.** `buildNowSection` counts `WAITING_BLOCKED` too, but `normalizeOperationalWorkflowRows` has already removed that status.
2. **Only `topIssueType` is counted.** NOW, NEXT, AT-RISK and STOCK read `topIssueType` alone. With the default priority order, a product that is out of stock and also missing a price appears only as `MISSING_PRICE`, so STOCK can under-count.
3. **Some issue types never fire.** No code path was found in the snapshot's product rows that sets `shopifyUnpublished`, `shopifyInactive` or `channelError` to true; the Shopify branch explicitly sets `channelError = false`. The signal and recommendation entries for `SHOPIFY_UNPUBLISHED`, `SHOPIFY_INACTIVE` and `CHANNEL_ERROR` therefore look unreachable today.
4. **The Amazon-events de-duplication looks like a no-op.** It only removes an event when an Action Center row has `topIssueType` BUY_BOX_*. Those rows come only from Amazon-event materialisations, which `getSalesOperationsSnapshot` excludes by default (`includeAmazonEventMaterializations: false`).
5. **The trend's `marketplaceId` uses a different id space.** The trend compares it with `tbl_orders.amazon_channel_marketplace_id`. The operational SQL and the Amazon-events filter compare it with the Amazon **marketplace** id (`tbl_amazon_marketplaces.id` / `amazonMarketplaceId`).
6. **Shopify publishing columns aren't loaded.** `getFullProductRecords` loads only `id` and `channelId` from `tbl_product_channels` and `tbl_product_variant_channels`. As a result, `syncStatus`, `externalStatus` and `externalStock` are `undefined` for Shopify, which means:
   - Not published and Out of sync always count 0 for Shopify.
   - No Shopify listing can be `ACTIVE`, so each one lands in Missing offer, Out of stock or Inactive.
7. **The master price fallback never matches.** `buildCommandCenterPublishRows` loads prices with `channelId IS NOT NULL`, so the fallback to the default (master) price never finds a row.
8. **"All channels" readiness double-counts.** It sums the channel rows, so a SKU listed on 2 channels is counted twice.
9. **The Catalog status scope comes from the URL only.** The request's `channelId` and `marketplaceId` only choose the **default** view; there is no in-page control that writes them to the URL.
10. **The channel tab switch can silently fail.** The `activateChannelId` handoff to Products depends on `useGlobalStore.tabChannelsData` already holding the channel. On a cold load straight into the Command Center, the tab may not switch.
11. **The `summary` endpoint is unused** by the webapp, and **the sidebar entry is hidden** (`visible: false`).
12. **Extra cache entries.** The period is part of the Redis key for every section, so each period creates separate cache entries even for sections that ignore it.

---

## 23. File index

**Webapp** (`webapp/src/`)
- `app/(index)/(menu-layout)/command-center/page.tsx`: route, metadata, gate + lazy main
- `app/(index)/(menu-layout)/command-center/main.tsx`: whole page UI, section hooks, navigation handlers
- `app/(index)/(menu-layout)/command-center/_components/command-center-permission-gate.tsx`
- `app/(index)/(menu-layout)/command-center/_components/command-center-trend-chart.tsx`
- `app/(index)/(menu-layout)/command-center/_lib/types.ts`: response types
- `app/(index)/(menu-layout)/command-center/_lib/render-keys.ts`: signal and recommendation de-duplication
- `app/(index)/(menu-layout)/command-center/_lib/storage-display.ts`: storage label and percent resolution
- `lib/api.ts` (`GET_COMMAND_CENTER_*`), `lib/action-center-permissions.ts`, `lib/subscription-restricted-mode.ts`, `lib/trial-ui.ts`, `lib/create-product-return.ts`, `lib/hooks/use-refresh-package-resources-snapshot.ts`, `lib/action-center-remediation-navigation.ts` (drilldown parsing)
- Destinations: `app/(index)/(menu-layout)/actionCenter/main.tsx`, `products/_components/list/List.tsx`, `products/[product]/page.tsx`, `channelManagement/_components/list/List.tsx`

**API** (`api/apps/api-main/src/`)
- `modules/app/system/operations/orchestration/command-center/`
  - `command-center.controller.ts`, `command-center.module.ts`, `dto/command-center.dto.ts`
  - `command-center.service.ts`: context, cache, section orchestration, Amazon events
  - `operational-rows.ts`, `operational-grouping.ts`, `now.builder.ts`, `next.builder.ts`, `at-risk.builder.ts`, `stock.builder.ts`
  - `catalog-status.service.ts`, `catalog-status.builder.ts`
  - `capacity.builder.ts`, `trend.builder.ts`, `amazon-events.builder.ts`, `commandCenter.types.ts`
- `modules/app/system/operations/orchestration/operational-grouping/operationalGrouping.ts`
- `modules/app/system/operations/action-center/action-center.service.ts` (`getSalesOperationsSnapshot` and the base snapshot pipeline)
- `modules/app/commerce/orders/order/order.service.ts` (`getCommandCenterTrendData`) + `utils/sellerTrendMetrics.ts`
- `modules/app/commerce/billing/package-subscription-manager/package-subscription-manager.service.ts` (`getPackageResourcesSnapshot`)
- `modules/app/commerce/billing/subscription-capabilities/*` (`ACTION_CENTER_VIEW` guard)
- `modules/app/catalog/channel/channel.service.ts` (`getUserWiseChannels`, `getRoleBasedChannelsForSetup`)
- `modules/app/platforms/amazon/amazon-events/*`, `modules/app/platforms/amazon/amazon-notifications/amazon-notifications.controller.ts`
- `modules/app/catalog/products/readiness/trigger|execution|engine/*`, `modules/app/sync/queue/processors/readiness.processor.ts`
- `modules/app/system/cron/cron.service.ts` (exchange rates)

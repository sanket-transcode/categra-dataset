# Action Center (v1): how it works today

> **Scope:** `webapp/` and `api/` only. This document explains what the page does as the code stands (read on 2026-09-29). No code was changed.
> **Entry file:** `webapp/src/app/(index)/(menu-layout)/actionCenter/page.tsx`
> **Depth rules used (agreed before writing):**
> - Two render paths exist. The **Kanban board** (the default today) is explained in full. The **Compound workspace** (behind an env flag) is summarised.
> - Upstream pipelines (readiness, sync errors, stock, prices, orders, Amazon events): every step, but summarised. Not line by line.
> - Also in scope: the ways into `/actionCenter`, the `/actionCenter/issue/[rowId]` page, and the product-page Action Center drawer.
> - Files that nothing imports are listed in an appendix only.
>
> **Legend:** 🗄️ = database table · 🧠 = in-memory / Redis cache · 🌐 = external or non-DB source · 💾 = browser storage · ⚠️ = something odd seen while reading the code (not verified at runtime)

---

## Contents

1. [What the Action Center is](#1-what-the-action-center-is)
2. [Page map at a glance](#2-page-map-at-a-glance)
3. [How a user gets to the page](#3-how-a-user-gets-to-the-page)
4. [Access control (webapp + API)](#4-access-control-webapp--api)
5. [Core concepts](#5-core-concepts)
6. [Page lifecycle (boot → render)](#6-page-lifecycle-boot--render)
7. [API: endpoints and the shared snapshot](#7-api-endpoints-and-the-shared-snapshot)
8. [Top chrome: header strip, toolbar, banners](#8-top-chrome-header-strip-toolbar-banners)
9. [Filters panel](#9-filters-panel)
10. [Ownership filter strip](#10-ownership-filter-strip)
11. [Channel blockers section](#11-channel-blockers-section)
12. [The Kanban board (lanes and cards)](#12-the-kanban-board-lanes-and-cards)
13. [The insights rail (right column)](#13-the-insights-rail-right-column)
14. [Running a card's action (execution flow)](#14-running-a-cards-action-execution-flow)
15. [Workflow changes: move, mark solved, snooze (API)](#15-workflow-changes-move-mark-solved-snooze-api)
16. [Ownership: assign, unassign, bulk assign](#16-ownership-assign-unassign-bulk-assign)
17. [Dialogs: waiting reason, snoozed items, review variants](#17-dialogs-waiting-reason-snoozed-items-review-variants)
18. [Empty, loading and error states](#18-empty-loading-and-error-states)
19. [Issue detail page (`/actionCenter/issue/[rowId]`)](#19-issue-detail-page-actioncenterissuerowid)
20. [Compound workspace (flag-gated, summary)](#20-compound-workspace-flag-gated-summary)
21. [Product-page Action Center drawer](#21-product-page-action-center-drawer)
22. [Upstream pipelines that produce the data](#22-upstream-pipelines-that-produce-the-data)
23. [Reference: tables per section](#23-reference-tables-per-section)
24. [Reference: non-DB sources, caches, flags, browser storage](#24-reference-non-db-sources-caches-flags-browser-storage)
25. [Reference: every click target](#25-reference-every-click-target)
26. [Observations seen while reading the code](#26-observations)
27. [Appendix: files nothing imports](#27-appendix-files-nothing-imports)
28. [File index](#28-file-index)

---

## 1. What the Action Center is

The Action Center (`/actionCenter`) is the **operational work queue**. The Command Center tells you *that* something is wrong. The Action Center lists each problem as a **ticket** (a "card") and lets the team work through it:

| Question | Where on the page |
|---|---|
| *What is broken in my selling scope right now?* | Kanban lane **To do**, plus the **Channel blockers** section |
| *What is someone already handling, and why is it waiting?* | Lane **In progress / waiting** (with a waiting reason) |
| *What was fixed but still needs the marketplace to confirm?* | Lane **Pending verification** |
| *Where do I go to fix it?* | Card button → the exact product tab or channel settings |
| *Who owns it?* | Card footer, ownership menu, "Mine / Unassigned / Teammate" filters |
| *How fast are we clearing work?* | Right rail: **Resolution velocity**, **Recently cleared**, **Recent activity** |

Key facts:

- The page itself **does not fix data**. Its buttons **navigate** to the place where the fix happens (Pricing tab, Stock tab, Channel tab, Details tab, Sync errors tab, Channel Management).
- It **does change workflow state**: moving cards between lanes, marking solved, snoozing, and assigning owners. These are stored in 🗄️ `tbl_action_center_issue_workflows`, 🗄️ `tbl_action_center_issue_assignments` and 🗄️ `tbl_action_center_lifecycle_events`.
- The tickets are **computed on every read** from live catalog/channel data (SQL over products, listings, stock, prices, errors). There is no ticket table: a ticket exists while the problem exists. The workflow tables only remember *what the team did* with it.
- The live page always runs in **Sales Operations** mode (`SALES_OPERATIONS`). A second mode, **Catalog Preparation**, exists in the API and is used by the product-page drawer (§21), but the page never switches to it.

---

## 2. Page map at a glance

```
/actionCenter  (page.tsx → ActionCenterPermissionGate → ActionCenterMain in main.tsx)
│
├─ Sticky header strip
│    ├─ "Workflow view" pill + one-line utility copy
│    └─ ActionCenterViewToolbar
│         ├─ Saved views menu (system + your own)            §8.2
│         ├─ Refresh scope                                    §8.2
│         ├─ Ownership switch: All / Mine / Unassigned        §8.2 (team accounts only)
│         ├─ Teammate owner dropdown                          §8.2 (team accounts only)
│         ├─ Filters (count badge)  → opens Filters panel     §9
│         ├─ Active category chip (×)                         §8.2
│         ├─ Snoozed (count)       → Snoozed dialog           §17.2
│         └─ Export to CSV                                    §8.2
│
├─ RestrictedModeBanner (area = action-center)               §8.3
├─ Filters panel (when open)                                 §9
├─ Ownership filter strip (when an ownership filter is on)   §10
├─ Channel blockers section (channel-level problems)         §11
│
└─ Body (one of):
     ├─ Loading skeletons / setup empty state / error state  §18
     └─ ActionCenterKanbanBoard                               §12
          ├─ Lane: To do                  (OPEN)
          ├─ Lane: In progress / waiting  (IN_PROGRESS)
          ├─ Lane: Pending verification   (PENDING_VERIFICATION)
          └─ Insights rail                                    §13
               ├─ Category breakdown (ring + issue types list)
               ├─ Resolution velocity (7 days)
               ├─ Recently cleared (5)
               └─ Recent activity (5, "See all" dialog)

Overlays: waiting-reason dialog §17.1 · snoozed dialog §17.2 · review-variants modal §17.3 · assign-teammate dialog §16.3

/actionCenter/issue/[rowId]?issueType=…&languageCode=…   → Issue detail page §19
```

When env `ACTION_CENTER_COMPOUND_PROJECTION_ENABLED` is on, the body is replaced by the **Compound workspace** (§20). It is off in `api/.env` today.

---

## 3. How a user gets to the page

| From | Code | What is passed |
|---|---|---|
| **Sidebar** item "Action Center" | `webapp/src/app/(index)/(menu-layout)/_components/nav-container.tsx` (route `/actionCenter`, `permissions: ACTION_CENTER_ACCESS_PERMISSIONS`, `optional: true`) | Nothing. Shown when the user has `actionCenter.*` or `actionCenter.view` (any one is enough), or is owner/admin. |
| **Command Center: NOW row** | `command-center/main.tsx` → `openActionCenter()` | `/actionCenter?mode=…&issueType=…&channelId=…&marketplaceId=…&languageCode=…&canonicalGroupKey=…` plus drill-down fields. Empty values are dropped. |
| **Command Center: NEXT "Review"** | same | Same as NOW, plus `workflowStatus=OPEN`. |
| **Command Center: STOCK rows** | same | `issueType=OUT_OF_STOCK` or `LOW_STOCK`, optional `channelId`, `marketplaceId`. |
| **Issue detail page** "Back to Action Center" / breadcrumb | `action-center-issue-detail-page.tsx` | `/actionCenter` (plus any drill-down params that were on the URL). Also writes a "return target" so the board scrolls back to the same card (§6.6). |
| **Coming back from a product page** | Browser back, or any link | A "return target" in 💾 `sessionStorage` is consumed and the card is re-focused (§6.6). |
| **Direct URL** | — | Any of the URL params in §6.2. |

The Command Center's own explainer (`categra-dataset/context/command-center/command-center-v1-explained.md`, §10–§15) describes how those params are built.

The page also changes some global chrome:
- `TrialExpiredBanner` hides itself on `/actionCenter` and `/actionCenter/*`.
- `BillingHeaderStatus` shows its payment/billing alerts in the header on this page.
- The sidebar uses a different "desktop expanded" breakpoint on `/actionCenter*` (`sidebar.tsx`).

---

## 4. Access control (webapp + API)

There are four layers. All four must pass.

### 4.1 Webapp gate: `ActionCenterPermissionGate`

File: `actionCenter/_components/action-center-permission-gate.tsx`. Wraps both `/actionCenter` and `/actionCenter/issue/[rowId]`.

1. Waits until `authStatus === 'ready'` and the user object is resolved. Renders nothing before that.
2. `hasActionCenterAccessPermission(user)` (`webapp/src/lib/action-center-permissions.ts`): true for owner or administrator, or if the permissions list contains `actionCenter.*` or `actionCenter.view`.
3. No access → `router.replace()` to the first page the user *can* open (`useNavigationViaPermission().findPath`).
4. Inactive subscription (`getSubscriptionAccessUiState(packageResources).isInactiveAccount`) → renders `InactiveSubscriptionFeatureBlockedState feature="action-center"` instead of the page and sets the navbar title/breadcrumb.
5. Otherwise renders the page.

### 4.2 Webapp per-button permissions

`main.tsx` removes buttons the user could not follow (`sanitizeQuickActionForPermissions`). A removed action falls back to the next route option, or to "Review issue".

| Action type | Hidden unless the user has |
|---|---|
| `EDIT_PRICE` | `products.pricing` or `products.*` |
| `UPDATE_STOCK`, `ADD_STOCK_IN_WAREHOUSE` | `stock.list` or `stock.*` |
| `OPEN_ACCOUNT_CONNECTION_SETTINGS` | `channels.list` or `channels.*` |
| `OPEN_CHANNEL_TAB` | `products.channels` or `products.*` |
| `OPEN_ERROR_DETAILS` | `products.*` |
| `FIX_IN_MASTER` | master (all-channel) access **and** `products.details` or `products.*` |
| Fallback "open product overview" | `products.overview` or `products.*` |

### 4.3 API gates (every endpoint)

Controller: `api/apps/api-main/src/modules/app/system/operations/action-center/action-center.controller.ts`, mounted at `/action-center`.

1. **`JwtAuthGuard`**: validates the token and also checks that the route is in the user's role routes. The route lists are in `api/.../common/security/permissions.data.ts`:
   - `actionCenter.view` → overview, workspace-shell, workflow-lane, insights, recently-cleared, resolution-velocity, row-variants, **workflow**, remediation groups (GET + validate), lifecycle-candidate(s), listing-issue-detail, sync-correlation (and the Command Center endpoints).
   - `actionCenter.assign` → **assignment**, and remediation **variant-operational-values** (the save).
2. **`@RequiresSubscriptionCapabilities`**:
   - `ACTION_CENTER_VIEW` on every read.
   - `ACTION_CENTER_MUTATE` on `workflow`, `assignment`, and the remediation save. This capability is denied in restricted modes (trial expired, "phase 2A" hard-deny, inactive account). `updateWorkflow`/`updateAssignment` also re-assert it inside the service.
   - A denial returns 403 with `data.code = 'SUBSCRIPTION_CAPABILITY_DENIED'`. The webapp swallows this code silently (no toast) and shows empty data.
3. **Channel access scope** (`ActionCenterService.getChannelAccessScope` → `ChannelService.getUserWiseChannels`, 🧠 cached per user): `{ allowAllChannels, allowedChannelIds, canAccessMaster }`. Every SQL query adds `AND c.id IN (:allowedChannels)` (or `1 = 0` when the list is empty). Only master-access users can use Catalog Preparation mode or see source-data issues.
4. **Ownership rules** (`resolveActionCenterCollaborationState`), see §16.1.

---

## 5. Core concepts

### 5.1 Modes

| Mode | Used by | What a row is |
|---|---|---|
| `SALES_OPERATIONS` | The page (always), issue detail page, drawer | One product **in one selling scope**: Shopify channel, or Amazon channel + marketplace. |
| `CATALOG_PREPARATION` | Drawer only; master-access users only | One product with no channel (source-data problems only). |

`autoMode` is always `SALES_OPERATIONS`. A non-master user asking for Catalog Preparation silently gets Sales Operations.

### 5.2 A row (ticket)

A row (`ScopeItem` in the API, `ActionCenterRow` in the webapp) is one **(product × scope × issue type)**. In Sales Operations each "separate" issue type gets its **own** row (see §7.4 step 7), so one product in one marketplace can produce several cards (e.g. *Missing price* and *Out of stock*).

Row id: `productId:channelId|catalog:marketplaceId|base`, e.g. `812:4:1`. The same id is shared by the rows of different issue types, so the webapp keys rows by `id + topIssueType + languageCode`.

Durable ticket identity (`buildActionCenterDurableTicketIdentity` in `actionCenter.lifecycle.ts`): account, mode, scope level (CHANNEL / PRODUCT / VARIANT), lifecycle family/subtype, product, channel, marketplace, variant, language, joined with `|`. Used to link workflow state and history.

### 5.3 Issue types

Defined in `ISSUE_PRIORITY` (`action-center.service.ts` ~L1332). Health comes from `BLOCKED_ISSUE_TYPES` / `AT_RISK_ISSUE_TYPES`. Priority group comes from `actionCenter.priority-ordering.ts`. The verification policy comes from `actionCenter.lifecycle.ts`.

| Issue type | Label (default) | Health | Priority group | Default action | Verification policy |
|---|---|---|---|---|---|
| `CHANNEL_NOT_CONNECTED` | Not connected | Blocked | Critical system blockers | Open connection settings | Manual |
| `CHANNEL_CONNECTION_ERROR` | Connection error | Blocked | Critical | Open connection settings | Manual |
| `CHANNEL_SETUP_INCOMPLETE` | Mapping incomplete | Blocked | Critical | Open connection settings | Manual |
| `CHANNEL_ERROR` | Connection issues | Blocked | Critical | Open error details | Sync refresh (6 h) |
| `AMAZON_ERROR` | Listing issues | Blocked | Critical | Open error details | Provider refresh (12 h) |
| `SYNC_FAILED` | Sync failed | Blocked | Critical | Open error details | Sync refresh (6 h) |
| `MISSING_PRICE` | Missing price | Blocked (quick win) | Pricing issues | Edit price | Local runtime |
| `OUT_OF_STOCK` | Out of stock | Blocked (quick win) | Out of stock | Update stock | Local runtime |
| `SHOPIFY_UNPUBLISHED` | Unpublished listing | Blocked | Listing blockers | Open channel tab | Provider refresh |
| `SHOPIFY_INACTIVE` | Inactive listing | Blocked | Listing blockers | Open channel tab | Provider refresh |
| `AMAZON_MISSING_OFFER` | Missing offer | Blocked | Listing blockers | Edit price | Provider refresh |
| `PRICE_POLICY_BLOCKED` | Price policy blocked | Blocked | Pricing issues | Open channel tab | Provider refresh |
| `BUY_BOX_EXCLUDED` | Buy Box excluded | Blocked | Pricing issues | Open channel tab | Manual |
| `BUY_BOX_LOST` | Buy Box lost | Blocked | Pricing issues | Edit price | Manual |
| `LOW_STOCK` | Low stock | At risk (quick win) | Low stock | Update stock | Local runtime |
| `OUT_OF_SYNC` | Out of sync | At risk | Listing warnings | Open channel tab | Sync refresh |
| `LISTING_STATUS_PENDING` | Listing status pending | Needs attention | Listing warnings | Open channel tab | Provider refresh |
| `AMAZON_NOT_SEARCHABLE` | Not searchable | At risk | Listing warnings | Open channel tab | Provider refresh |
| `AMAZON_WARNING` | Listing issues | At risk | Listing warnings | Open error details | Manual |
| `CHANNEL_WARNING` | Sync issues | At risk | Listing warnings | Open error details | Manual |
| `MISSING_MEDIA` | Missing media | Needs attention | Readiness / content quality | Fix in master | Local runtime |
| `MISSING_CONTENT` | Missing content | Needs attention | Readiness / content | Fix in master | Local runtime |
| `MISSING_ATTRIBUTES` | Missing attributes | Needs attention | Readiness / content | Fix in master | Local runtime |
| `LOW_READINESS` | Low readiness | Needs attention | Readiness / content | Fix in master | Local runtime |
| `SOURCE_DATA` | Source-data advisory | Needs attention | Informational advisory | Fix in master | Local runtime |

- The three `BUY_BOX_*` / `PRICE_POLICY_BLOCKED` rows come **only** from Amazon events (§7.6).
- Source-data types (last five) appear only when `includeSourceDataIssues` is true (master access). The live page does **not** request them, so in practice the board shows channel issues only.
- Labels can be overridden per target area by `getActionCenterOperationalIssueLabel` (`presentation/actionCenterPresentation.ts`) and by the Action Engine issue catalog.

### 5.4 Health

`BLOCKED` > `AT_RISK` > `NEEDS_ATTENTION` > `HEALTHY`. A row's health is the worst health of its issue types. Cards show it as a coloured left border and a severity chip.

### 5.5 Workflow statuses

Two layers are stored in the same table 🗄️ `tbl_action_center_issue_workflows`:

| Layer | `scope_user_id` | Statuses | Who sees it |
|---|---|---|---|
| **Shared** (team) | `0` | `TODO`, `WAITING_BLOCKED`, `PENDING_VERIFICATION`, `CLEARED` | Everyone |
| **Personal snooze** | the user's id | `SNOOZED` + `snooze_until` | Only that user |

The webapp shows **legacy names**:

| Shared status | Lane / display status |
|---|---|
| `TODO` (or no row at all) | `OPEN` → lane **To do** |
| `WAITING_BLOCKED` | `IN_PROGRESS` → lane **In progress / waiting** |
| `PENDING_VERIFICATION` | `PENDING_VERIFICATION` → lane **Pending verification** |
| `CLEARED` | `CLOSED` → not on the board (goes to "Recently cleared") |
| personal `SNOOZED` | not on the board (goes to the Snoozed dialog) |

State can be kept **per variant**. For a row with variants, each affected variant can have its own status. The row then carries counts (`workflow.openCount / inProgressCount / pendingVerificationCount / closedCount / snoozedCount / totalCount`), and its displayed status is the first non-zero of: open → in progress → pending → closed → snoozed (`getPrimaryWorkflowStatusForRow`).

### 5.6 Routing (where a button goes)

Every row gets a `quickAction` (from the issue table in §5.3) and a `routing` object:

1. Classify the top issue into a **target area** (e.g. PRICE, STOCK, LISTING) and a **responsible team**, using the Action Engine settings (`runtimeSettings.routing.legacyIssueClassifications`).
2. Look up the target area's **CTA mapping**: primary, secondary and fallback **CTA templates**.
3. Build an action from each template if the row has the context the template needs (ACCOUNT / CHANNEL / MARKETPLACE / PRODUCT / VARIANT / LANGUAGE). Otherwise that option is marked unavailable.
4. Remove duplicates. The first available option is the **primary action**. The old `quickAction` is added as a **LEGACY_FALLBACK** option if it is different.
5. If a configured option is unavailable: `fallbackNote` = "One configured route is not available in this scope yet…".

The webapp re-runs the same resolution on the client (`_lib/cta-routing.ts`) and then applies §4.2 permission filtering.

### 5.7 Runtime settings (Action Engine)

`ActionEngineSettingsResolver.resolveSettingsOverview(accountId)` merges **platform defaults**, 🗄️ `tbl_action_engine_settings` rows with `scopeType = 'GLOBAL'`, and rows with `scopeType = 'ACCOUNT'`. They are edited from the admin backend (`modules/admin/action-engine`). If resolving fails, code defaults are used. The payload gives:

- workflow thresholds (waiting overdue days, pending stale hours, snooze-expiring-soon hours, to-do light threshold)
- **reason catalogs** for *waiting*, *pending* and *snooze*
- **snooze presets**
- buy-box competitiveness settings (Compound only)
- **priority ordering**: pinned group `CRITICAL_SYSTEM_BLOCKERS`, then the sortable groups (default order: listing blockers, out of stock, pricing, listing warnings, low stock, readiness/content, advisory)
- **business impact** ranking boost (metric, look-back, strength, groups)
- visibility rail flags (e.g. `snoozedAccessMode = 'RAIL_ACCESS'` shows the Snoozed button)
- routing catalogs (target areas, teams, CTA templates, classifications, target-area → CTA mappings)

---

## 6. Page lifecycle (boot → render)

### 6.1 Mount order

1. `page.tsx` sets metadata title "Action Center" and renders `ActionCenterPermissionGate` → `ActionCenterMain`.
2. `ActionCenterMain` sets the navbar title and breadcrumb to "Action Center".
3. It parses **URL filters** (§6.2) and the **operational drill-down** (`parseOperationalDrilldownContext`: `canonicalGroupKey` / `opGroupKey`, `opChannelId`, `opMarketplaceId`, `opProductId`, `opVariantId`, `opVariantSKU`, `languageCode`).
4. It **restores UI state** from 💾 storage (§6.5), keyed by the user's profile link:
   - active saved view → apply its snapshot, otherwise
   - last context (filters, ownership filter, layout) → apply, otherwise
   - defaults (no filters, ownership "All").
   It also consumes any **return target** (§6.6). Then `initialUiStateReady = true`.
5. If the URL carries filters, they **win** over restored state (`mergeActionCenterFiltersWithUrlSeed`) and the active saved view is cleared.
6. `loadOverview()` runs (§6.3).

### 6.2 URL params the page reads

| Param | Effect |
|---|---|
| `channelId`, `marketplaceId` | Pre-set those filters |
| `issueGroupKey` or `category` | Pre-set the issue **group** filter (clears `issueType`) |
| `issueType` | Pre-set issue type (only if no group key); the group key is derived from it |
| `health` | `BLOCKED` / `AT_RISK` / `NEEDS_ATTENTION` / `HEALTHY` |
| `workflowStatus` | `OPEN` / `IN_PROGRESS` / `PENDING_VERIFICATION` |
| `canonicalGroupKey` (+ `op*`) | Operational drill-down: only rows whose operational grouping matches are shown; passed on to product pages when a card is opened |

`mode` from the Command Center link is ignored by the board (it always loads `SALES_OPERATIONS`).

### 6.3 `loadOverview()`: the loading sequence

```
POST /action-center/workspace-shell  { mode: SALES_OPERATIONS }
   │   (light: account state, permissions, collaboration, filter metadata,
   │    runtime settings, channel blocker rows, empty-state hint, feature flags)
   │
   ├─ setup empty state? (no products / no connected channels and no blocker rows)
   │     → show setup empty state, stop                                   §18
   │
   ├─ featureFlags.compoundProjection && !forceLegacyWorkspace ?
   │     → POST /action-center/overview → Compound workspace              §20
   │
   └─ otherwise (default):
         progressiveBoardEnabled = true
         in parallel:
           POST /action-center/workflow-lane { status: OPEN }
           POST /action-center/workflow-lane { status: IN_PROGRESS }
           POST /action-center/workflow-lane { status: PENDING_VERIFICATION }
         each lane fills independently (own loading / error / retry)
```

After the shell arrives, three side loaders fire (effects on `overview` + applied filters):

| Loader | Endpoint | Body | Re-runs when |
|---|---|---|---|
| `loadInsightsPanel` | `POST /action-center/insights` | mode, channelId, marketplaceId, issueType, includeSourceDataIssues | filters change |
| `loadRecentlyCleared` | `POST /action-center/recently-cleared` | same + `assigneeUserId` (from ownership filter) + `limit: 5` | filters or ownership filter change |
| `loadResolutionVelocity` | `POST /action-center/resolution-velocity` | mode, channelId, marketplaceId, assigneeUserId | channel/market or ownership filter change |

Each loader uses a request-id ref, so a late response from an older request is dropped. All requests are "passive" (`ACTION_CENTER_PASSIVE_REQUEST_OPTIONS`). A 403 capability denial clears the section silently.

**Filters are applied in the browser.** The lane requests carry no filters. Channel, market, issue type, health, workflow and category filters all run on the loaded rows (§9).

### 6.4 Refresh triggers

| Trigger | What happens |
|---|---|
| **Refresh scope** button | `loadOverview({ silent: true })` + toast "Action Center scope refreshed using persisted data". |
| Window **focus** or tab becomes **visible** | Silent reload, at most once every 1.5 s. |
| The current user's **profile link** changes | Silent reload. |
| After any move, snooze, mark solved or assignment | Silent reload ("reconcile"). |
| Lane error **Retry** | Reloads that one lane. |
| Rail card **Retry** | Reloads that rail card's endpoint. |

The API keeps a 🧠 15-second in-process snapshot cache (§7.3), so rapid reloads are cheap. Any successful workflow, assignment or remediation save **clears that cache**.

### 6.5 What is remembered in the browser

| Key | Storage | Holds |
|---|---|---|
| `categra:action-center-saved-workflow-views:v1` | 💾 localStorage | Per user (profile-link scope key): your saved views + which view is active |
| `categra:action-center-last-context:v1` | 💾 localStorage | Per user: last filters, ownership filter, selected owner, layout |
| `categra:action-center-return-target:v1` | 💾 sessionStorage | Card to re-focus on return: rowId, issueType, languageCode, surface (`BOARD` / `REVIEW_ISSUE` / `ISSUE_DETAIL`) |

### 6.6 Return target (coming back to the same card)

Before any navigation away (running an action, opening the issue page), the page writes a return target. On the next load, once lanes have finished loading:

- `ISSUE_DETAIL` → `router.replace()` back to that issue page.
- `REVIEW_ISSUE` and the row has variants → reopens the review-variants modal (§17.3).
- `BOARD` → scrolls the card (`[data-action-center-row-key]`) into the centre and focuses it.
- If the row no longer exists but it is in "Recently cleared", the review modal opens on a synthetic row built from the history item.

### 6.7 Body modes

`bodyMode` decides what is drawn:

| Mode | When | Rendered |
|---|---|---|
| `bootstrap-loading` | UI state not restored yet | Header skeleton + setup loading state |
| `setup-decision-pending` | No overview yet, still loading | same |
| `setup-empty` | No products, or no connected channels | Setup empty state (§18) |
| `workflow-loading` | Any lane still loading | Toolbar + 3-lane skeleton |
| `workflow-empty` | Various empty cases | Toolbar + an empty-state panel (§18) |
| `workflow-ready` | Rows to show | Toolbar + board |
| `error` | Load failed | `ActionCenterUnavailableState` |

---

## 7. API: endpoints and the shared snapshot

### 7.1 Module wiring

- `ActionCenterModule` (`system/operations/action-center/action-center.module.ts`) provides `ActionCenterService`, `ActionCenterSyncCorrelationService`, `ActionCenterSyncEvaluationService`. It imports Channel, TaskQueue, BuyBox, ActionEngineSettings, AccountSubscriptions, SubscriptionCapability and Readiness modules.
- `ActionCenterService` is also used by the Command Center (`getSalesOperationsSnapshot`) and by the Amazon events scope/materialization services.

### 7.2 Endpoint list

All are under `/action-center`. All reads are `POST` (except one `GET`).

| Method + path | Service method | Returns | Used by |
|---|---|---|---|
| `POST overview` | `getOverview` → `getOverviewSnapshot` | Full payload: summary, breakdowns, priority actions, channel blockers, rows for a status, all workflow rows, recent activity, compound projection, empty state | Compound path, issue detail page, drawer |
| `POST workspace-shell` | `getWorkspaceShell` | Same metadata but **no rows** | Page boot |
| `POST workflow-lane` | `getWorkflowLane` | `{ status, rows }` for one lane | Kanban lanes |
| `POST insights` | `getInsights` | `{ recentActivity }` (uses up to 96 retained cleared items) | Rail: Recent activity |
| `POST recently-cleared` | `getRecentlyCleared` | Cleared tickets in the last 7 days (limit 1–100, default 50) | Rail: Recently cleared |
| `POST resolution-velocity` | `getResolutionVelocity` | Daily clears for 7 days + previous 7 days + trend | Rail: Resolution velocity |
| `POST row-variants` | `getRowVariants` | Affected variants for one row, with routing | Review issue, move/assign target resolution |
| `POST listing-issue-detail` | `getListingIssueDetail` | Normalised issue detail (summary, fix surface, affected sub-items, technical details) | Navigation (to pick the right tab), drawer |
| `GET remediation/groups/:key?productId=` | `getRemediationGroup` | Stock/price remediation group (variants + current values) | Drawer |
| `POST remediation/groups/:key/validate` | `validateRemediationGroup` | Validation result for draft values | Drawer |
| `PATCH remediation/groups/:key/variant-operational-values` | `saveRemediationVariantOperationalValues` | Saves stock/price values, re-runs readiness | Drawer |
| `POST sync-correlation` | `SyncCorrelation.getSyncCorrelationFor{Issue,Subitem}` | Sync tasks linked to an issue | Drawer |
| `POST lifecycle-candidate` / `lifecycle-candidates` | `SyncEvaluation.getLifecycleCandidate*` | Whether the latest sync suggests the issue is fixed | Drawer |
| `POST workflow` | `updateWorkflow` | `{ updated, action }` | Move, mark solved, snooze, unsnooze |
| `POST assignment` | `updateAssignment` | `{ updated, action }` | Assign / unassign |

Response envelope: `{ status: 'success' | 'error', data, message }` (via `GlobalResponses.formatResponse`). On error the controller returns the `HttpException` status (or 500) with `data: null`.

### 7.3 Caching

- 🧠 `progressiveSnapshotCache` (in-process `Map`, TTL **15 s**) holds the *base snapshot*. It is keyed by account, user, mode, search, channel, marketplace, product, health, issueType, includeSourceDataIssues, smartReview and the channel-access scope. An in-flight map stops duplicate builds, so the three lane requests share one build.
- The cache is **cleared** after any successful workflow, assignment or remediation save.
- Table-existence checks (`to_regclass`) for the workflow, assignment and lifecycle tables are cached for 15 s. If a table is missing, workflow features turn off gracefully (only the To do lane is offered).

### 7.4 Building the base snapshot (`buildOverviewBaseSnapshot`)

This is the core. Every read endpoint (overview, shell, lane, insights, detail) goes through it.

**Step 1: resolve request context.** Account, user, search, channel, marketplace, product, health, issueType, smartReview. Mode (non-master forced to Sales Operations). `includeSourceDataIssues` (master only; defaults to true only for Catalog Preparation). Compound flag (env + Sales mode).

**Step 2: load context in parallel.**

| Call | Reads | Produces |
|---|---|---|
| `getAccountState` | 🗄️ `tbl_products`, `tbl_product_channels`, `tbl_channels`, `tbl_account_configurations` | products count, sales channels count, **connected** sales channels count (Amazon/Shopify only), base language (default `en_US`) |
| `getFilterMetadata` | 🗄️ `tbl_channels`, `tbl_amazon_channels`, `tbl_amazon_channel_marketplaces`, `tbl_amazon_marketplaces` | channel list (active Amazon/Shopify), marketplace list (connected Amazon, active marketplaces) |
| `getPrimaryMediaMinimum` | 🗄️ `tbl_readiness_configurations`, `tbl_attributes` | Min images from the account's master "Primary media" readiness rule |
| `resolveActionCenterRuntimeSettings` | 🗄️ `tbl_action_engine_settings` | §5.7 |
| `resolveActionCenterCollaborationState` | subscription (🌐 `AccountSubscriptionsService`), 🗄️ `tbl_user_accounts`, `tbl_users`, `tbl_roles` | §16.1 |

If ownership is readable, `getActionCenterAssigneeCandidates` is also loaded (🗄️ `tbl_user_accounts`, `tbl_users`, `tbl_roles`, `tbl_attachments`, `tbl_user_channels`, `tbl_role_channels`).

**Step 3: filter metadata** gets modes, health states, the issue-type list (§5.3, relabelled), and workflow statuses (only "Open" if the workflow table is missing).

**Step 4: channel blocker rows** (Sales mode, no product filter): see §11.1.

**Step 5: stop early if the account has 0 products.**

**Step 6: raw scope rows.**
- Sales Operations → `getSalesScopeRows`: one Shopify SQL and one Amazon SQL in parallel (§7.5), results concatenated.
- Catalog Preparation → `getCatalogScopeRows` (`buildCatalogQuery`): product name/SKU/description/category/identity, readiness (master, base language), media count, variant count, thumbnail.

**Step 7: decorate each raw row** (`decorateScopeItem`):
- `readinessValue < 75` → `lowReadiness`.
- `missingMedia` = low readiness **and** media count < primary-media minimum.
- `missingContent` = low readiness **and** name, description or category missing.
- `missingAttributes` = low readiness and neither of the above.
- **Shopify:** `missingPrice` (any priced item missing), `outOfStock` (any stock ≤ 0), `lowStock` (not out of stock and any stock ≤ threshold), `syncFailed` (sync status ERROR, `has_sync_errors`, or an unresolved product error), `outOfSync`, `channelWarning`.
- **Amazon:** the same plus `amazonError` (listing errors / marketplace error rows), `amazonWarning`, `amazonNotSearchable` (not `DISCOVERABLE`), `amazonMissingOffer` (missing price **and** not `BUYABLE` **and** no Amazon error and not sync-failed), and `outOfSync` only once Amazon has reported a status.
- Build `issueTypes` (§5.3 order). Drop the row if it has no operational issue and source-data issues are off.
- Compute health, top issue (lowest issue rank within the priority group order), root-cause label (e.g. "Amazon - Germany", channel name, or "Fix in source data"), quick action, quick-win flag, **affected item count** (missing-price count, missing-stock count, low-stock count, or active variant count), and a **priority score**.

**Step 8: split into one row per issue** (`expandDecoratedScopeItemRows`, Sales only). Each of the 14 "separate" sales issue types (§5.3 rows `MISSING_PRICE` … `AMAZON_MISSING_OFFER`) becomes its own row with its own health, count, ASIN list and score. Leftover (source-data) types are bundled into one extra row.

**Step 9: business-impact boost** (`applyBusinessImpactBoost`), if enabled in settings. Rows in the configured priority groups (never critical connection blockers) with sales > 0 get a boost by **percentile** of the chosen metric (orders, revenue or units; 7 d, 30 d, or 7 d + 30 d):

| Percentile | ≥ 95 % | ≥ 80 % | ≥ 50 % | > 0 |
|---|---|---|---|---|
| LOW | +14 | +10 | +6 | +3 |
| MEDIUM | +30 | +20 | +10 | +5 |
| HIGH | +40 | +28 | +16 | +8 |

**Priority score** (`getPriorityScore`):
`(10 − groupIndex) × 1000` + `500` if a critical connection blocker + `healthRank × 120` (Blocked 4, At risk 3, Needs attention 2, Healthy 1) + `140` if quick win + `80` if operational + `10` if source-data + `min(affected, 25) × 8` + `(30 − rank within group)` (+ business boost).

**Step 10: sort** (`sortScopeItems`): quick wins first only when `smartReview` is true (the page never sends it) → priority score ↓ → affected count ↓ → updated time ↓ → readiness ↑ → product name.

**Step 11: filter** by requested issueType (matches the row's top issue) and health. Attach durable identity keys and runtime routing (§5.6).

**Step 12: workflow projection** (`applyWorkflowProjection`), if the workflow table exists:
1. Delete this user's **expired personal snoozes** (`snooze_until <= NOW()`).
2. Load all overlays for the account + mode (shared rows + this user's snoozes).
3. **Reconcile** each shared overlay against current truth:
   - Overlay's issue no longer produces a row (or the variant is no longer affected): Pending → **Cleared**; Waiting → run the "likely fix" rules (§15.3); Cleared stays cleared. Personal/snooze overlays for vanished issues are **deleted**.
   - Row still present but its evidence (`updatedAt`) is newer than the overlay, or a Pending overlay is **stale** (older than 6 h for sync issues or 12 h for provider issues): Pending → **To do** (issue still there) or **Cleared**; Cleared + issue back → **To do** (reopened).
   - Every change is written in one transaction and logged as a lifecycle event (§15.4).
4. For rows with variants and variant-level overlays, load affected variants (`resolveRowVariantItemsInternal`) and count statuses per variant.
5. Set each row's `workflow` counts, `workflowStatus`, `workflowUpdatedAt`, `workflowActor` (who made the last shared change; 🗄️ `tbl_users`, `tbl_attachments`), waiting reason code/note, and the personal snooze fields.

**Step 13: assignment projection** (`applyAssignmentProjectionToRows`): reads 🗄️ `tbl_action_center_issue_assignments` for the rows and sets `ownerState` (UNASSIGNED / ASSIGNED / MIXED when variants have different owners), `owner`, `ownerAssignedBy`, `ownerAssignedAt`, `ownerWasReassigned`.

**Step 14: Amazon event rows** (Sales mode, when enabled): §7.6. They are appended and the lists re-sorted.

**Step 15: per endpoint.**
- `workflow-lane`: `filterProjectedRowsByStatus`: OPEN → `openCount > 0`, IN_PROGRESS → `inProgressCount > 0`, PENDING → `pendingVerificationCount > 0`. A row with mixed variant statuses can therefore appear in more than one lane.
- `overview`: adds `summary` (counts per health, quick wins, …), `breakdowns` (per health / issue type / channel), `priorityActions` (top 10 by score), `recentActivity`, `emptyState` (`NO_PRODUCTS` / `NO_CHANNELS` / `NO_ISSUES` / `NO_RESULTS`). Channel blocker rows are included only for the OPEN/TODO status.

### 7.5 What the sales SQL reads

Both queries share CTEs: base language, **filtered products** (account, not deleted/archived, accessible through an allowed channel, optional search on SKU / product name / product ASIN / variant ASIN), product attributes, category, media counts, main image, variant counts, and sales impact.

| Part | Shopify (`buildShopifySalesQuery`) | Amazon (`buildAmazonSalesQuery`) |
|---|---|---|
| **Scope** | `tbl_product_channels` on a CONNECTED Shopify channel, `is_active`, has `external_id`, `external_status = ACTIVE` | `tbl_product_channels` × `tbl_product_channel_marketplaces` (status true, not deleted) × active `tbl_amazon_channel_marketplaces` × `tbl_amazon_marketplaces`, on a CONNECTED Amazon channel |
| **Variants in scope** | `tbl_product_variant_channels` (active, has external id) | `tbl_variant_marketplaces` + `tbl_product_variant_channels` |
| **Stock** | Mapped non-FBA warehouses (`tbl_channel_product_warehouses` → `tbl_warehouse_products.total_quantity`), otherwise `external_stock` | FBA listing or FBA warehouse → `tbl_amazon_fba_inventories.fulfillable_quantity`; else mapped warehouses; else `amazon_stock` |
| **Low-stock threshold** | `low_stock_threshold`, **only** when warehouses are mapped | same; never for FBA |
| **Price** | `tbl_product_variant_prices`: sale price if > 0, else offer price, channel-specific (not inherited) first, then master; currency = Shopify default currency (`tbl_shopify_channels`, `tbl_currencies`) | same, per Amazon marketplace currency (`tbl_currencies`) |
| **Errors** | `tbl_channel_product_errors` (not resolved, not ignored) per product+channel → error / warning | Same, split into marketplace-level and channel-level, and per variant |
| **Listing status** | fixed `PUBLISHED` / `NOT_APPLICABLE` | `LISTING_ERROR` / `BUYABLE` / `NOT_BUYABLE` / `STATUS_PENDING` from `listing_errors` and `external_status` JSON |
| **Searchable** | n/a | `SEARCHABLE` / `NOT_SEARCHABLE` / `STATUS_PENDING` from `DISCOVERABLE` |
| **Readiness** | `tbl_readiness_values` channel value, else master value | marketplace value, else channel base, else master |
| **ASIN lists** | — | Per-issue ASIN lists (missing price, stock, …) for card display |
| **Sales impact** | `tbl_orders` × `tbl_order_items`: orders, units, revenue for 7 d and 30 d | same |

### 7.6 Amazon event rows (`loadAmazonEventMaterializedRows`)

Only when **all** of these are true (all default **off**): env `AMAZON_EVENTS_LIVE_ENABLED`, `AMAZON_EVENTS_ACTION_CENTER_ENABLED`, `AMAZON_EVENTS_ACTION_CENTER_RENDER_ENABLED`. Optional account and channel allowlists apply, and per-family flags (default on).

1. Build the set of (product, channel, marketplace) scopes present in the base rows.
2. Read 🗄️ `tbl_amazon_event_action_center_materializations`: active, enabled family (`PRICE_POLICY_BLOCKED`, `BUY_BOX_LOST`, `BUY_BOX_EXCLUDED`), status `ACTIVE_MATERIALIZED` or `SOURCE_PENDING_VERIFICATION`, matching scope.
3. Turn each into a row (copying product and scope data from a base row). Health = Blocked. Routing comes from the stored CTA template keys.
4. Apply workflow and assignment projection.
5. **Duplicate guard:** skip a Buy Box row if the same scope already shows a Buy Box duplicate/blocker issue type, and skip a price-policy row if a price-policy blocker is already visible. Order: price policy → excluded → lost.

### 7.7 Error envelope and messages

- Unknown errors → 500 with a fixed fallback text per endpoint ("Failed to load Action Center lane.").
- Missing tables → 503 with "…unavailable until the latest database migration is applied."
- Remediation errors also log the upstream payload and return `data: { upstreamStatus, error: 'upstream_error' }`.

---

## 8. Top chrome: header strip, toolbar, banners

### 8.1 Header strip

A sticky rounded bar. Left: "Workflow view" pill + one line of copy (`ACTION_CENTER_UTILITY_STRIP_COPY`). Right: the toolbar.

### 8.2 `ActionCenterViewToolbar`

File: `_components/action-center-view-toolbar.tsx`.

| Control | Visible when | Click → |
|---|---|---|
| **Saved views** menu (`action-center-saved-views-menu.tsx`) | Always | Lists **Default workflow views** (system) and **Your saved views**. Selecting one applies its snapshot (filters + ownership + layout) and toasts "{label} applied". "Save current view" opens a name dialog (placeholder "My Amazon blockers") → a new user view in 💾 localStorage. "Update current saved view" / "Delete current saved view" only for your own views. |
| **Refresh scope** | Always | Silent reload + toast (§6.4). |
| **Ownership switch**: All / Mine / Unassigned | Team accounts where ownership is readable (§16.1) | Client-side ownership filter; also passed as `assigneeUserId` to Recently cleared and Resolution velocity. |
| **Teammate owner dropdown** | Same | "All owners" or a teammate → shows only that teammate's cards. |
| **Filters** (count badge) | Always | Opens/closes the Filters panel (§9). |
| **Category chip** with × | A category is focused (from the rail or the URL) | × clears the category filter. |
| **Snoozed** (count) | `runtimeSettings.visibilityRail.snoozedAccessMode === 'RAIL_ACCESS'` | Opens the Snoozed dialog (§17.2). |
| **Export to CSV** | At least one visible card | Downloads `action-center-workflow.csv` of the **visible** cards: Product, SKU, Identifier, Channel, Market, Issue, Health, Workflow, Listing status, Searchable, Suggested action, Updated. Built in the browser. |

**System saved views** (`buildSystemSavedWorkflowViews`):

| View | Snapshot | Shown when |
|---|---|---|
| All work | no filters | always |
| My work | ownership = Mine | ownership readable |
| Unassigned | ownership = Unassigned | ownership readable |
| In progress / waiting | workflowStatus = IN_PROGRESS | always |
| Pending verification | workflowStatus = PENDING_VERIFICATION | always |
| Blocked issues | health = BLOCKED | always |
| Pricing issues | issueType = MISSING_PRICE | that issue type is in filter metadata |

If the current state stops matching the active view's snapshot, the view is unmarked (it stays saved).

### 8.3 Banners

- **`RestrictedModeBanner area="action-center"`** (`components/restricted-mode/restricted-mode-banner.tsx`): shown in restricted subscription states. The copy says the Action Center stays visible but is read-only, or cleanup-only. Mutations fail with the capability denial (§4.3).
- **Header billing alerts** (`BillingHeaderStatus`): payment-failed and other subscription alerts.

---

## 9. Filters panel

File: `_components/action-center-filters-panel.tsx`. Opens under the header.

| Field | Options come from |
|---|---|
| **Issue filter** (read-only chip + "Clear issue filter") | Only when a category is focused |
| **Health** | Blocked / At risk / Needs attention / Healthy |
| **Channel** | `filterMetadata.channels` (active, connected-or-not Amazon/Shopify you can access) |
| **Market** | `filterMetadata.marketplaces` for the chosen channel **plus** any marketplace that appears on loaded rows |
| **Issue type** | Issue **groups** built from the visible rows' labels, sorted by count |
| **Workflow** | To do / In progress / waiting / Pending verification |

Buttons: **Clear all** (resets the draft only) and **Apply filters** (applies the draft, closes the panel, clears the active saved view, saves the last context).

How filters apply (all client-side, `redesign.ts`):
- Cards: `filterWorkflowRows` (channel, market, issue type, health, workflow) → drill-down filter (§6.2) → ownership filter (§10) → category filter (issue group).
- Channel blockers: `filterChannelIssueRows`. **Any market filter hides all blockers**, and a workflow filter other than To do hides them.
- Snoozed dialog: `filterSnoozedRows` (same filters, no workflow).
- After each shell load, filter values that no longer exist (channel gone, market gone, unknown issue type) are dropped automatically.

---

## 10. Ownership filter strip

Shown when ownership is readable and a filter other than "All" is on. Text examples: "Showing only workflow items currently assigned to you", "…unassigned…", "…owned by {teammate}". If nothing matches it also says so. **Clear ownership filter** resets to All.

Filter logic (`filterRowsByOwnership` in `_lib/ownership-helpers.ts`): Mine = the owner is you. Unassigned = no owner. Teammate = the owner is that user.

---

## 11. Channel blockers section

File: `_components/action-center-channel-blockers-section.tsx`. Title "Channel blockers" + count. Shown above the board whenever there are matching blocker rows.

### 11.1 How the rows are built (API `getChannelIssueRows`)

1. Skipped when a marketplace is requested or the status is not OPEN/TODO.
2. SQL on 🗄️ `tbl_channels` (account, not deleted, active, Amazon/Shopify, allowed) where connection is `NOT_CONNECTED`, `CONNECTION_LOST`, or `CONNECTED` with `attribute_mapping_status = INCOMPLETE`.
3. Map to `CHANNEL_NOT_CONNECTED` / `CHANNEL_CONNECTION_ERROR` / `CHANNEL_SETUP_INCOMPLETE`. Health Blocked. Id `channel:{channelId}:{issueType}`.
4. Filter by channel, issue type, health, and search (channel name/type/label). Sort by score, then name.

These rows are **not workflow-managed**: always "Open", no drag, no owner.

### 11.2 UI and click

Each row shows the channel icon/name, the current state ("Not connected", "Connection error", "Mapping incomplete"), a Blocked badge, and the button **Open connection settings** → `/channelManagement?channelId={id}`. The button is hidden without `channels.list`/`channels.*` (§4.2).

---

## 12. The Kanban board (lanes and cards)

File: `_components/action-center-kanban-board.tsx` (`ActionCenterKanbanBoard`). Uses `react-dnd` (HTML5 backend).

### 12.1 Layout

- **≥ 1680 px**: three equal lanes + a 320 px rail on the right. The board height is fitted to the window (minimum 540 px), and each lane scrolls on its own.
- **< 1680 px**: lanes are 336 px wide and scroll sideways. The rail goes below.

### 12.2 Lanes

| Lane | Status | Note under the title | Colour |
|---|---|---|---|
| **To do** | `OPEN` | "Needs action now" | pink |
| **In progress / waiting** | `IN_PROGRESS` | "Already being handled" | amber |
| **Pending verification** | `PENDING_VERIFICATION` | "Waiting for sync check" | slate |

Lane header: title, card count, "Loading" pill while loading, and **Assign visible lane** (only if you can assign and there are eligible teammates, §16.3).

Lane body:
- Loading with no cards → skeleton. Error with no cards → error message + **Retry** (reloads that lane).
- Cards are rendered **24 at a time**. The footer shows "Showing X of Y", with **Load more (24)** and **Show all**.
- No cards → lane empty state.

**Card order in a lane** (`sortRowsForBoardStatus`):
- To do: cards never moved (no `workflowUpdatedAt`) first, by priority score ↓. Then cards moved back to To do, oldest move first.
- Other lanes: oldest last change first, then priority score ↓.

### 12.3 Card anatomy (`KanbanCardSurface`)

```
┌▌───────────────────────────────────────────────── [⋮] ┐   ▌ left border = health colour
│▌ [thumbnail]  [ISSUE BADGE] [3 variants]              │
│▌              (i) Product name (2 lines max)          │   (i) = info popover
│▌              Channel · 🇩🇪 DE                          │
│▌ ───────────────────────────────────────────────────  │
│▌ (avatar) Owner name • 2h ago      [Primary CTA] [▾]  │
└───────────────────────────────────────────────────────┘
```

| Part | Content / source |
|---|---|
| Thumbnail | Main product image (`256x256_` from 🗄️ `tbl_digital_assets` via `tbl_product_media_linkers`), else a placeholder |
| Issue badge | Display label of the top issue (§5.3), coloured by category |
| Variant count chip | "N variants" when the drill-down count is > 1 |
| Info popover (i) | SKU, ASIN (or a list for several ASINs, with copy buttons), owner, ownership activity, last updated (absolute + relative) |
| Product name | `productName`, clamped to 2 lines |
| Context line | Channel name (or "Shared scope"), market flag + code |
| Footer left | Team account: avatar + owner name (or "Unassigned" / "Mixed owners") + relative time. Clicking opens the **ownership menu** (Assign to me / Assign to teammate / Unassign). Solo account: generic icon + "Unassigned" / "Mixed owners" / "Team activity". |
| Primary CTA | Label of the routing primary action ("Edit price", "Update stock", "Open error details", …) |
| ▾ **More routes** | The other available routing options + **Review issue** (only for rows with variants) |
| ⋮ **Workflow menu** | §12.4 |

### 12.4 Card interactions

| Interaction | Result |
|---|---|
| **Click the card** (or Enter/Space) | Row has variants → opens the **issue detail page** `/actionCenter/issue/{rowId}?issueType=…&languageCode=…` (§19). Otherwise → runs the primary action (§14). |
| **Primary CTA** | Runs the primary action (§14). |
| **More routes → option** | Runs that route's action (§14). |
| **More routes → Review issue** | Issue detail page (§19). |
| **Drag and drop** to another lane | Allowed only to lanes in `getAllowedWorkflowStatusesForRow` (Pending verification only if the row has a channel). Calls `handleMoveRow` (§15.1). While hovering, a "Drop here" slot marks the position (visual only; order is not saved). |
| **⋮ → primary action** | Same as the CTA. |
| **⋮ → Move to {lane}** | Move to **In progress** opens the **waiting-reason dialog** (§17.1). Other lanes move directly (§15.1). |
| **⋮ → Mark solved** / "Mark all affected as solved" | Only in To do → `MARK_SOLVED` (§15.3). |
| **⋮ → Snooze for me** | To do or Pending only. Presets from runtime settings (fallback 1, 2, 3, 7 days) or **Pick date** (calendar, not in the past) → `SNOOZE` (§15.5). |
| **⋮ → Unsnooze** | Only when snoozed → `UNSNOOZE`. |
| **⋮ → Ownership** | Assign to me / Assign to teammate (dialog, §16.3) / Unassign. |

Busy cards (a request in flight) are disabled.

---

## 13. The insights rail (right column)

File: `_components/action-center-insights-panel.tsx`. Order from top: Category breakdown → Resolution velocity → Recently cleared → Recent activity.

### 13.1 Category breakdown

- **Data:** the board rows after the filters and ownership filter, **before** the category filter (`breakdownRows = ownershipFilteredRows`). Computed in the browser.
- **Ring chart**: each row's top issue is mapped to a category (`getActionCenterIssueCategory`: Pricing, Stock, Sync/"Verification", Listing, Media, Content) and shown as percentages.
- **Issue types list**: groups by issue label (`getActionCenterIssueGroup`), with count, % of visible cards, and a bar.
- **Click an issue type** → `handleSelectBreakdownIssueType`: toggles the **category filter** (`issueGroupKey`) on the board. Clicking again clears it. The toolbar chip appears. This is saved in the last context.

### 13.2 Resolution velocity

- **API** (`buildResolutionVelocity`): counts 🗄️ `tbl_action_center_lifecycle_events` with `to_status = 'CLEARED'`, grouped by UTC day. Covers 7 days up to today, plus the 7 days before. Filtered by allowed channels, channel, marketplace, and assignee (when an ownership filter is on and ownership is readable; else 403).
- **UI**: "Cleared today N", a small daily bar chart, and a trend vs the previous period (delta and %; 100 % when the previous period is 0 and the current is > 0).
- If both periods are 0: "No clears yet". Error → **Retry velocity**.

### 13.3 Recently cleared

- **API** (`buildRecentlyClearedItems`): for each durable ticket, takes the **latest** lifecycle event. Keeps tickets whose latest event is `CLEARED` within the last **7 days**. Joined with product name/image/SKU, variant label, channel and marketplace names. Filters: mode, issue type, source-data exclusion, channel, marketplace, assignee. The page asks for **5**.
- **UI**: up to 5 entries: issue label, product, SKU, scope, "from {previous status}", "Cleared by {actor}", time. Client-side also filters by category and ownership.
- **Click** → opens the review-variants modal (§17.3) for the live row if it still exists, otherwise for a synthetic row built from the history item.

### 13.4 Recent activity

- **API** (`buildRecentActivity`, via `/insights`): lifecycle events from the last 7 days for the visible rows (and up to 96 recently cleared tickets), newest first, **max 96**. Each event has a status and an actor.
- **UI**: 5 newest. Headlines: "{actor} moved {item} to In progress", "…to Pending", "cleared …", "returned … to To do". **See all** opens a dialog with up to 24.
- **Click** → review-variants modal for the live row, else for the matching recently cleared item, else toast "This item is no longer available in the current review view".

---

## 14. Running a card's action (execution flow)

Entry points: card click (no variants), primary CTA, More routes option, variant row button, and the issue detail page buttons. All call `executeRowEntryPoint` (`main.tsx`).

1. **Lock** (`executionLockRef`) so double clicks do nothing.
2. **Auto-move to In progress:** if the card is not already In progress and the action type is one of `EDIT_PRICE`, `UPDATE_STOCK`, `ADD_STOCK_IN_WAREHOUSE`, `OPEN_ERROR_DETAILS`, `OPEN_CHANNEL_TAB`, `FIX_IN_MASTER` (`execution-gate.ts`), then `POST /action-center/workflow { action: MOVE, status: IN_PROGRESS, targets, assignToSelfIfUnassigned? }`. Toast "Moved to In progress / waiting". If that fails, stop.
3. **Otherwise, auto-assign:** if ownership is readable and the card is unassigned → `ASSIGN_TO_SELF` in the background.
4. **Pick the exact variant:** if the row has variants, exactly 1 affected item, and no explicit action, load `row-variants`. If exactly one variant is currently affected, use **that variant's** action.
5. **Load issue detail** for `FIX_IN_MASTER`, `OPEN_CHANNEL_TAB` and `OPEN_ERROR_DETAILS`: `POST /action-center/listing-issue-detail` (no technical details). Its `fixSurface` can send you to a **different tab** than the action type suggests (e.g. an "Open channel tab" whose real fix is on the Details tab).
6. **Build the remediation context** (`buildActionCenterRemediationContextFromInputs`) and the **operational drill-down** (from the row's `operationalGrouping` canonical key).
7. **Guard:** stock/price actions on a row with **several** affected variants need a canonical group key. Without it: toast "stock/price remediation context unavailable", stop.
8. **Resolve the href** (`getActionCenterQuickActionHref`, table in §14.1). No href → fall back to `/products/{id}?q=overview` if allowed, else toast "You do not have permission to open this destination".
9. **Save the return target** (§6.6) and `router.push(href + ac*/op* params)`.

### 14.1 Where each action type goes

| Action type | Destination |
|---|---|
| `OPEN_ACCOUNT_CONNECTION_SETTINGS` | `/channelManagement?channelId={id}` |
| `EDIT_PRICE` | `/products/{id}?q=pricing&variantId=&variantSKU=` + temp data (`channelId`, `marketplaceID`, `search` when variant-scoped) |
| `UPDATE_STOCK` | `/products/{id}?q=stock&…` (same pattern) |
| `ADD_STOCK_IN_WAREHOUSE` | `/products/{id}?q=stock&…` + temp data `openAddStock=1`, `warehouseId` |
| `OPEN_ERROR_DETAILS` | `/products/{id}?q=syncError&ErrorChannelId=&marketplaceId=&variantId=&variantSKU=` + temp `search` |
| `OPEN_CHANNEL_TAB` | `/products/{id}?q=channels&channelId=&marketplaceId=&variantId=&variantSKU=&search=` |
| `FIX_IN_MASTER` | `/products/{id}?q=details&variantId=&languageCode=&search=` |
| Any, when the issue detail's `fixSurface` differs | Stock → `stock`, Pricing → `pricing`, Details/Media → `details`, Variants → `variants`, Sync errors → `syncError`, Channel → `channels` |

"Temp data" is written by `buildUrlWithTempData` (`products/_components/navigationTemp`) and read by the product page.

Extra params added to the product URL:
- `ac*` (remediation context): `acSource=action-center`, `acIssueKey`, `acIssueCorrelationKey`, `acRowId`, `acProductId`, `acChannelId`, `acMarketplaceId`, `acIssueType`, `acIssueDomain`, `acFixSurface`, `acVariantIds`, `acSkus`, `acAsins`, `acFields`, `acMediaSlots`, `acAffectedCount`, `acIssueTitle`, `acEvidenceSources`, `acEventPurposes`, `acFocus`, `acHighlight`, `acSession` (lists capped at 20 values).
- `op*` (operational drill-down): `opGroupKey`, `opChannelId`, `opMarketplaceId`, `opProductId`, `opVariantId`, `opVariantSKU`.

The product page uses these to highlight the fields and open the Action Center drawer (§21).

---

## 15. Workflow changes: move, mark solved, snooze (API)

### 15.1 Webapp side: moving a card (`handleMoveRow`)

1. Check the target lane is allowed. Otherwise toast "This item does not require pending verification".
2. **Optimistic update:** the card jumps to the new lane immediately and is marked busy.
3. **Resolve targets** (`resolveWorkflowTargetsForRow`): no variants → one row-level target. With variants → load `row-variants` and make one target per **currently affected** variant. If fewer targets than `affectedItemCount` are found → no targets → toast "This item cannot be moved right now", revert.
4. `POST /action-center/workflow { mode, action: 'MOVE', status, targets, reasonCode?, note?, assignToSelfIfUnassigned? }`. `assignToSelfIfUnassigned` is sent when moving to In progress and the card is unassigned (team accounts).
5. `updated = 0` → revert, reload, toast ("cannot move…" / "cannot move to pending verification").
6. Success → toast ("Moved to In progress / waiting", "Moved to pending verification", "Moved to To do"), then a silent reload to reconcile.

### 15.2 API side: `updateWorkflow`

Body: `mode`, `action` (`MOVE` | `MARK_SOLVED` | `SNOOZE` | `UNSNOOZE`), `targets[]` (rowId, productId, issueType, variantId, channelId, marketplaceId, languageCode), `status`, `snoozeUntil`, `etaAt`, `reasonCode`, `note`, `referenceCode`, `assignToSelfIfUnassigned`.

1. Re-check the `ACTION_CENTER_MUTATE` capability. Load runtime settings and channel access.
2. Normalise and de-duplicate targets. None → `{ updated: 0 }`.
3. Validate:
   - `SNOOZE` needs `snoozeUntil`.
   - `MOVE` needs a status.
   - Status `WAITING_BLOCKED` (raw) needs a `reasonCode` from the **waiting** catalog.
   - Pending needs a reason from the **pending** catalog if one is given. Snooze likewise from the **snooze** catalog.
4. Workflow table missing → 503.
5. Delete this user's expired snoozes.
6. For each target:
   - **Can apply?** Sales mode: the channel must be accessible and active. Catalog mode: master access.
   - **Still present?** The issue (or the variant) must still be produced by the current data. Otherwise it is skipped.
   - Then per action (§15.3–15.5), each inside a DB transaction.
7. If anything was updated → clear the snapshot cache.

### 15.3 `MOVE` and `MARK_SOLVED`

**MOVE to a lane** (TODO / WAITING_BLOCKED / PENDING_VERIFICATION):
- Pending is only allowed in Sales mode, with a channel, and not for stock issues unless a reason is given (`canUsePendingVerificationStep`).
- Delete the user's personal overlay for the target.
- If `assignToSelfIfUnassigned` and ownership is readable and there is no current owner → insert an assignment to self.
- `transitionSharedWorkflowOverlay`: upsert the shared row (`scope_user_id = 0`) with the status, reason code/note (waiting/pending only), ETA and reference (waiting only), and `created_by`. Remove duplicates. If the status really changed, write a lifecycle event.

**MARK_SOLVED** ("likely fix", `resolveWorkflowLikelyFixResolution`) uses the issue's verification policy (§5.3):

| Policy | Issue still present? | Result |
|---|---|---|
| Local runtime (price, stock, content) | No | **Cleared** |
| Local runtime | Yes | Back to **To do** |
| Provider refresh / Sync refresh (listing, sync) | No | **Cleared** |
| Provider / Sync refresh | Yes, and pending allowed | **Pending verification** |
| Provider / Sync refresh | Yes, pending not allowed | Back to **To do** |
| Manual (Buy Box, warnings, channel connection) | — | **Pending verification** if allowed, else **Cleared** (if the type is ticketable) |

When the result is Cleared, the ticket's **assignments are deleted**.

The webapp toast says "Marked solved and moved to pending verification" whenever the row has a channel. That is a client guess; the real result shows after the reload.

### 15.4 Lifecycle events and later reconciliation

Every shared status change inserts a row into 🗄️ `tbl_action_center_lifecycle_events`: account, mode, durable identity, lifecycle family/subtype, issue type, row id, product/variant/channel/marketplace/language, `from_status`, `to_status`, actor, assignee, `occurred_at`. These rows feed the rail (§13) and the "Resolved" variants (§19).

On every later read, the projection (§7.4 step 12) keeps statuses honest:
- **Pending verification** → **Cleared** when the issue disappears. → **To do** when it is still there after the stale window (6 h sync, 12 h provider) or when newer evidence arrives.
- **Cleared** → **To do** (reopened) if the same issue comes back with newer evidence.
- **Waiting** whose issue vanished → likely-fix rules.
- Reconciliation changes use the original overlay's creator as the actor.

### 15.5 `SNOOZE` / `UNSNOOZE`

- **Snooze** is **personal**: an overlay with `scope_user_id = you`, status `SNOOZED`, `snooze_until`, optional reason/note. Your card leaves your lanes (its snoozed count equals its total). Teammates still see it.
- Allowed from To do or Pending (webapp rule).
- **Unsnooze** deletes your overlay for that target. Snoozes also end by themselves: expired ones are deleted on the next read or write.
- Webapp toasts: "Snoozed", "Returned from snooze".

---

## 16. Ownership: assign, unassign, bulk assign

### 16.1 When ownership exists (`resolveActionCenterCollaborationState`)

| Mode | Condition | Ownership |
|---|---|---|
| `SOLO_PACKAGE` | Subscription package has no `allowTeamManagement` | Off |
| `TEAMS_SOLO` | Team package, but only 1 active user | Off |
| `COLLABORATIVE` | Team package, > 1 active user (invitation accepted, not suspended or deleted) | **Readable** if 🗄️ `tbl_action_center_issue_assignments` exists |

- `assignmentReadable` → you can see owners, filter by owner, **Assign to me**, and unassign yourself.
- `assignmentAvailable` = readable **and** (owner, admin, or `actionCenter.*` / `actionCenter.assign`) → you can also assign others, unassign others, and bulk assign. The permission is re-read fresh from 🗄️ `tbl_user_accounts` / `tbl_roles` (a suspended or uninvited user gets none).
- **Assignee candidates:** active users of the account who have master access (owner, admin role, user/role master flag) or at least one channel (`tbl_user_channels` / `tbl_role_channels`).

### 16.2 API `updateAssignment`

Body: `mode`, `action` (`ASSIGN` | `ASSIGN_TO_SELF` | `UNASSIGN`), `assigneeUserId`, `targets[]`.

1. Capability check. Ownership must be readable; otherwise 503 with the reason ("unavailable for the current package" / "…once another teammate joins" / "…migration").
2. `ASSIGN`, or `UNASSIGN` of someone else, needs assign permission, else 403.
3. Keep targets whose channel you can access.
4. `ASSIGN`: the assignee must be a candidate who can access **every** target (channel or master), else 403 "The selected teammate does not have access to the requested scope".
5. Per target: `UNASSIGN` → delete the assignment(s). Otherwise, if the issue is still present for the assignee → upsert the assignment (`assignee_user_id`, `assigned_by`).
6. Clear the snapshot cache.

### 16.3 Webapp flows

| Action | Where | Request |
|---|---|---|
| Assign to me | Ownership menu, workflow menu | `ASSIGN_TO_SELF`, toast "Assigned to you" |
| Unassign | Same | `UNASSIGN` (+ `assigneeUserId` if unassigning a single other owner), toast "Card unassigned" |
| Assign to teammate | Same → **dialog** | Dialog lists **eligible** teammates (computed from the card's targets: `filterAssignmentCandidatesForTargets`). Shows the card context. **Assign teammate** → `ASSIGN`, toast "Assigned to {name}" |
| Assign visible lane | Lane header → **dialog** | Same dialog for all visible cards of that lane → `ASSIGN` with all targets, toast "Assigned visible lane to {name}" |

Targets are resolved as in §15.1 step 3. The update is optimistic (owner shown at once), then reconciled with a silent reload.

---

## 17. Dialogs: waiting reason, snoozed items, review variants

### 17.1 Waiting-reason dialog (`ActionCenterPreActionDialog`)

Opened by **⋮ → Move to In progress / waiting** (board), by "Move to waiting" in the review modal, and by the issue detail page.

- Shows the selected item (issue label, product image, name, SKU).
- **Reason** (required): options from `runtimeSettings.reasonCatalogs.waiting`, sorted.
- **Extra detail** (optional note).
- **Move to waiting** → the dialog closes at once and, in the background, `MOVE` with `status: WAITING_BLOCKED`, `reasonCode`, `note`, and auto-assign-to-self if unassigned. Toast "Moved to waiting". On failure the dialog reopens with your draft and the error.
- **Cancel** closes it.

The reason and note show on the card later (`workflowReasonCode` / `workflowReasonNote`).

### 17.2 Snoozed dialog (`ActionCenterSnoozedDialog`)

Opened from the toolbar **Snoozed** button, or from the "No workflow rows" empty state. It lists **your** snoozed cards (`workflow.snoozedCount > 0` and `personalSnoozeUntil` set, current filters applied). Each shows product, issue, scope, owner, when snoozed, "returns {time}" / until date, reason and note. **Resume now** → `UNSNOOZE`.

### 17.3 Review-variants modal (`ActionCenterVariantsModal`)

Opened from **Recently cleared** / **Recent activity** clicks and by the return target (`REVIEW_ISSUE`). (A normal card click goes to the issue detail page instead, §19.)

- Loads `POST /action-center/row-variants` (forced reload).
- Shows "What needs attention", the issue source, target area, "Best handled by" (responsible team), the recommended next step, counts (affected / active variants / ASINs), and the variant list (Active / Resolved).
- Buttons: the primary fix route (**Open product** etc.), **More routes**, **Move to waiting** (opens §17.1). Variant rows have their own action buttons → §14 with that variant's action.

---

## 18. Empty, loading and error states

| State | Trigger | Title | Buttons → |
|---|---|---|---|
| **No products** (setup) | 0 products | "Build your catalog foundation", with preview cards (pricing gaps, stock blockers, channel issues) and a 3-step checklist | **Create product** → `/products`; **Import from Amazon or Shopify** → `/channelManagement` |
| **No channels** (setup) | Products exist, 0 connected sales channels, no blocker rows | "No sales channels are connected yet" | **Connect a sales channel** → `/channelManagement`; **Go to channels** → `/channelManagement` |
| **No results** | Rows exist but the filters match none | "No workflow items match this view" | **Clear filters**; **Adjust filters** (opens panel); **Refresh scope** |
| **No workflow rows** | Filters match, but every item is cleared or snoozed | "No workflow items need attention right now" | **Open snoozed** (if any) or **Review filters**; **Refresh scope** |
| **No issues** (clean) | Nothing actionable | "Nothing needs attention right now" | **Refresh scope**; **Review filters** |
| **Ownership no match** | Ownership filter hides everything | "No workflow items are assigned to you" (etc.) | **Show all work**; **Adjust filters**; **Refresh scope** |
| **Category no match** | Category focus hides everything | "No visible cards match {category}" | **Show all categories** |
| **Lane loading** | Lanes in flight | 3-lane skeleton + rail skeleton | — |
| **Unavailable** (`ActionCenterUnavailableState`) | Shell load failed | "Action Center is taking a moment to reconnect", with reassurance copy (no catalog data changed; persisted data only) | **Try again** → `loadOverview()`; **Open Command Center** → `/command-center` |

Empty states are skipped while the history rail still has content; the board is shown instead.

---

## 19. Issue detail page (`/actionCenter/issue/[rowId]`)

Files: `actionCenter/issue/[rowId]/page.tsx` → `ActionCenterPermissionGate` → `ActionCenterIssueDetailPage` (`_components/action-center-issue-detail-page.tsx`). Metadata title from the `Review_Issue` translation.

### 19.1 Data loading

1. Read `rowId` (path), `issueType` and `languageCode` (query), and any drill-down params.
2. `POST /action-center/overview { mode: SALES_OPERATIONS }` (the **full** overview, not lanes).
3. Find the matching row in `workflowRows` (same id, same issue type, same language). Apply routing and permissions as on the board.
4. `POST /action-center/row-variants` for that row (forced).
   - API: `resolveRowVariantItemsInternal` picks the signal query by issue type and channel type: source-data → catalog variant signals (🗄️ `tbl_product_variants`, `tbl_readiness_values`, `tbl_product_attributes`, `tbl_product_media_linkers`); Shopify → Shopify variant signals; Amazon (with marketplace) → Amazon variant signals (price, stock, errors, listing status per variant).
   - Then `mergeRetainedVariantLifecycleItems` adds variants **cleared in the last 7 days** (from 🗄️ `tbl_action_center_lifecycle_events`) as "Resolved".
   - Each variant gets its own routing.
5. Navbar title = product name. Breadcrumb: Action Center → this issue.

### 19.2 UI

- **Header card**: back arrow (**Back to Action Center**), product thumbnail, product name, issue label, market flag, SKU and ASIN identifiers, variant/scope label ("N variants" or "Product-level scope"), workflow status.
- **Buttons**: primary fix action; **Move to waiting** (dialog §17.1); **More routes** (other routes); **Refresh scope** (reload + toast "Issue detail refreshed").
- **Affected variants panel** (`ActionCenterAffectedVariantsPanel`): filter chips **All / Active / Resolved** with counts. Each variant shows a thumbnail, name, SKU/ASIN, "Updated / Resolved {time}", its primary action, and **More routes**. Loading, error (**Retry**), stale-data warning, "no affected variants" and "no variants in this view" (**Show all variants**) states.
- **Not found** (row no longer in scope): "Issue no longer in current Action Center scope", with **Back to Action Center** and **Refresh scope**.
- **Load error**: "Issue detail unavailable" + **Back to Action Center**.

### 19.3 Actions here

Same engine as the board (§14–§16): primary/variant actions (auto-move to In progress, navigate), move to waiting, assign/unassign. Back and navigation write a return target (surface `ISSUE_DETAIL` or `BOARD`), so the board can re-focus the card (§6.6).

---

## 20. Compound workspace (flag-gated, summary)

Enabled only when env `ACTION_CENTER_COMPOUND_PROJECTION_ENABLED` is `1/true/yes` (API) **and** mode is Sales Operations **and** the user has not picked the "Standard" layout in a saved view.

**What changes:** the page calls `POST /action-center/overview` once and draws `ActionCenterCompoundWorkspace` (board `action-center-compound-board.tsx` + drawer `action-center-compound-drawer.tsx`) instead of the lanes and rail.

**How the projection is built** (`buildCompoundProjection`):
1. Load external evidence (Shopify/Amazon errors with 🗄️ `tbl_channel_product_errors` + `tbl_channel_error_definitions`) and Buy Box evidence (🗄️ `tbl_product_current_offers` via `BuyBoxService`).
2. Create **child targets** from evidence (specific families such as missing/invalid required attributes, invalid media/content, listing rejected, compliance restriction) and from the normal scope rows (price, stock, sync…). The 26 families are listed in `COMPOUND_ISSUE_FAMILIES` (`actionCenter.compound.ts`).
3. Apply workflow and assignment state, stock routing (preferred warehouse), and operational grouping.
4. Group children into **parents** (per product/scope; competitive-pricing parents grouped separately). Compute parent health, status, owner summary and summary counts.

**UI:** three lanes **TODO / WAITING_BLOCKED / PENDING_VERIFICATION** of parent cards. A drawer shows the parent checklist (visible targets, unresolved count, Buy Box competitor details: gap, Prime/FBA edge, data freshness). The same actions exist at parent level: move, clear, snooze/unsnooze, assign to me, assign to user, unassign, bulk-assign a lane, and run a child target (navigation as in §14.1). The toolbar hides export, snoozed access and the ownership switch in this mode.

---

## 21. Product-page Action Center drawer

Files: `products/[product]/_components/action-center/` (`ProductActionCenterIssueDrawerLauncher.tsx`, `ProductActionCenterIssueDrawer.tsx` (~16.8k lines), `…EdgeTrigger.tsx`, `ProductActionContextActiveDestination.tsx`, `productActionContextLocalReadiness.ts`). Mounted on `/products/[product]` (`page.tsx` → `ProductActionCenterIssueDrawerLauncher`).

### 21.1 Opening

- An **edge trigger** on the product page shows three counters: **actions**, **signals**, **sync**. Hovering preloads the drawer code. Clicking opens it.
- It **auto-opens** when the URL has an active remediation context (`acActive*`) or an operational drill-down (`op*` / `canonicalGroupKey`), i.e. when you arrived from an Action Center card. `ac*` params alone only preload it.

### 21.2 Data it loads

| Purpose | Request |
|---|---|
| All issues of this product | `POST /action-center/overview` **twice**: `{ mode: SALES_OPERATIONS, productId, includeSourceDataIssues: true }` and `{ mode: CATALOG_PREPARATION, … }` → merged into "product issue contexts", sorted |
| Issue detail | `POST /action-center/listing-issue-detail` with `includeTechnicalDetails: true` |
| Stock/price bulk fix | `GET /action-center/remediation/groups/{canonicalGroupKey}?productId=` |
| Validate draft values | `POST …/{key}/validate` |
| Save values | `PATCH …/{key}/variant-operational-values` |
| Provider validation rows | `POST` product sync errors (`GET_PRODUCT_SYNC_ERRORS`) |
| Master readiness | `GET_PRODUCT_READINESS_VALUES` + `GET_PRODUCT_READINESS_VIEW` |
| Sync history for the issue | `POST /action-center/sync-correlation` |
| "Did the last sync fix it?" | `POST action-center/lifecycle-candidates` (batch) |
| Sync the affected scope | `POST sync/product-syncing/sync-product-to-channel` with an `actionCenterSyncContext`, then opens the task manager |

### 21.3 UI (high level)

Scope heading and counter chips; an **active remediation banner** (which card you came from, progress through targets, skip/next); issue context cards with **Next best action**; **Affected items** (sub-item cards per variant/SKU/ASIN); a **read-only / editable operational remediation panel** for stock and price groups; **Supporting details** and **Technical details** (collapsed); **Sync** / **Sync affected** / **Auto sync** with states "Sync required", "Syncing…", "Submitted, awaiting provider". 💾 sessionStorage keeps skipped targets and progress (`categra:product-action-context:*`).

### 21.4 Remediation save (API `saveRemediationVariantOperationalValues`)

1. Parse the canonical group key (`channel=…|marketplace=…|language=…|capability=…|issue=…|scope=variant`). Only `inventory/out-of-stock` and `pricing/missing-price` at variant scope are supported.
2. Resolve the affected variants and their current values. Validate the drafts (numbers, stale-value check via `valueVersion` / `evidenceUpdatedAt`).
3. Save the valid ones:
   - **Stock**: Amazon → `tbl_variant_marketplaces.amazon_stock`; Shopify → `tbl_product_variant_channels.external_stock` (+ `external_stock_details.total`). Sync status becomes `OUT_OF_SYNC` unless `NOT_PUBLISHED`.
   - **Price**: update or insert `tbl_product_variant_prices` (price + `pricing_details.offerPrice`) in the scope's currency.
4. Trigger **readiness recalculation** (`ReadinessTriggerService`, rule type stock/price) for the saved variants.
5. Reload targets and return the refreshed context plus a save summary (saved / failed / stale / skipped / validation errors / remaining). The publish message depends on `auto_publish_price` (`tbl_product_channels` / `tbl_product_channel_marketplaces`).
6. Log an audit JSON line and clear the snapshot cache.

---

## 22. Upstream pipelines that produce the data

The Action Center only **reads** these. A short lifecycle for each:

| Signal | Written by | Read as |
|---|---|---|
| **Channel connection & mapping** | Channel connect / OAuth, token refresh, attribute mapping flows | `tbl_channels.connection_status`, `attribute_mapping_status` → channel blockers |
| **Product/variant listings** | Forward sync (Amazon JSON Listings Feed, Shopify FS) and backward sync (BS) | `tbl_product_channels`, `tbl_product_channel_marketplaces`, `tbl_product_variant_channels`, `tbl_variant_marketplaces` (`sync_status`, `has_sync_errors`, `external_status`, `listing_errors`, `asin`, `is_data_updated`) |
| **Sync/listing errors** | Sync pipelines persist provider errors | `tbl_channel_product_errors` (severity, `mark_as_resolved`, `is_ignored`) |
| **Stock** | Warehouse stock updates, FBA inventory sync, channel stock fetch | `tbl_warehouse_products`, `tbl_channel_product_warehouses`, `tbl_amazon_fba_inventories`, `amazon_stock` / `external_stock` |
| **Prices** | Pricing tab edits, BS imports, Action Center remediation | `tbl_product_variant_prices` (`pricing_details` JSON) |
| **Readiness** | Readiness engine (worker), triggered by product edits | `tbl_readiness_values` (+ `tbl_readiness_configurations`) |
| **Orders** | Amazon/Shopify order sync | `tbl_orders`, `tbl_order_items` (business-impact boost only) |
| **Amazon events** | SP-API notifications → raw → scope → projection → `AmazonEventsActionCenterMaterializationService.evaluateProjectionGenerationContext` (needs `AMAZON_EVENTS_ACTION_CENTER_MATERIALIZER_ENABLED`). Writes materializations and may also update `tbl_action_center_issue_workflows` / `tbl_action_center_lifecycle_events` | §7.6 |
| **Sync correlation** | Syncs started from the drawer carry `actionCenterSyncContext` into `tbl_task_queues.raw_data` | `sync-correlation`, `lifecycle-candidate(s)` |
| **Buy Box** (Compound) | Buy Box / offers fetch | `tbl_product_current_offers` |
| **Action Engine settings** | Admin backend | `tbl_action_engine_settings` |

---

## 23. Reference: tables per section

| Section | Tables |
|---|---|
| Access & context (§4, §7.4 step 2) | `tbl_user_accounts`, `tbl_users`, `tbl_roles`, `tbl_user_channels`, `tbl_role_channels`, `tbl_attachments`; subscription via service |
| Account state / filters | `tbl_products`, `tbl_product_channels`, `tbl_channels`, `tbl_account_configurations`, `tbl_amazon_channels`, `tbl_amazon_channel_marketplaces`, `tbl_amazon_marketplaces` |
| Card content (name, SKU, image, category) | `tbl_product_attribute_groups`, `tbl_product_attributes`, `tbl_attributes`, `tbl_product_categories`, `tbl_categories`, `tbl_category_translations`, `tbl_product_media_linkers`, `tbl_digital_assets`, `tbl_product_variants` |
| Issue detection (sales) | `tbl_product_channels`, `tbl_product_channel_marketplaces`, `tbl_product_variant_channels`, `tbl_variant_marketplaces`, `tbl_shopify_channels`, `tbl_currencies`, `tbl_product_variant_prices`, `tbl_channel_product_warehouses`, `tbl_warehouse`, `tbl_warehouse_products`, `tbl_amazon_fba_inventories`, `tbl_channel_product_errors`, `tbl_readiness_values`, `tbl_readiness_configurations` |
| Business impact | `tbl_orders`, `tbl_order_items` |
| Channel blockers (§11) | `tbl_channels` |
| Workflow state (§15) | `tbl_action_center_issue_workflows` |
| Ownership (§16) | `tbl_action_center_issue_assignments` (+ user tables above) |
| History rail (§13), resolved variants (§19) | `tbl_action_center_lifecycle_events` (+ product/channel tables for labels) |
| Runtime settings | `tbl_action_engine_settings` |
| Amazon event rows (§7.6) | `tbl_amazon_event_action_center_materializations` |
| Listing issue detail | `tbl_channel_product_errors`, `tbl_product_attributes`, `tbl_product_variant_attributes`, `tbl_attributes`, `tbl_amazon_attribute_hierarchies`, `tbl_product_channel_marketplaces`, `tbl_variant_marketplaces`, `tbl_amazon_event_action_center_materializations`, `tbl_amazon_event_scope_identities`, `tbl_amazon_event_occurrences`, `tbl_amazon_event_normalized_observations`, `tbl_amazon_event_raw_messages` |
| Sync correlation / lifecycle candidates | `tbl_task_queues` (+ Amazon event tables) |
| Remediation save (§21.4) | `tbl_variant_marketplaces`, `tbl_product_variant_channels`, `tbl_product_variant_prices`, `tbl_product_channels`, `tbl_product_channel_marketplaces`, `tbl_currencies` |
| Compound only (§20) | `tbl_channel_error_definitions`, `tbl_product_current_offers` |

---

## 24. Reference: non-DB sources, caches, flags, browser storage

| Kind | Name | Effect |
|---|---|---|
| 🌐 env (API) | `ACTION_CENTER_COMPOUND_PROJECTION_ENABLED` | Turns on the Compound workspace (§20) |
| 🌐 env (API) | `AMAZON_EVENTS_LIVE_ENABLED`, `AMAZON_EVENTS_ACTION_CENTER_ENABLED`, `AMAZON_EVENTS_ACTION_CENTER_RENDER_ENABLED`, `…_MATERIALIZER_ENABLED`, `…_ACCOUNT_ALLOWLIST`, `…_CHANNEL_ALLOWLIST`, per-family flags | Amazon event rows (§7.6, §22) |
| 🧠 API memory | `progressiveSnapshotCache` (15 s) + in-flight map | Shares one snapshot build between lane/insight requests (per API process) |
| 🧠 API memory | table-availability checks (15 s) | Graceful behaviour when a migration is missing |
| 🧠 Redis (via ChannelService) | user channel scope | Channel access (§4.3) |
| 🌐 service | `AccountSubscriptionsService`, `SubscriptionCapabilityService` | Team capability, capability gates |
| 🌐 service | `ActionEngineSettingsResolver` | Runtime settings (+ code defaults) |
| 🌐 service | `ReadinessTriggerService` | Readiness recalculation after a remediation save |
| 💾 localStorage | `categra:action-center-saved-workflow-views:v1`, `categra:action-center-last-context:v1` | Saved views, last context |
| 💾 sessionStorage | `categra:action-center-return-target:v1` | Return target |
| 💾 sessionStorage | `categra:product-action-context:skipped-targets:*`, `…:target-progress:*`, `…:variant-workflow:*` | Drawer progress (§21) |
| Client state | Zustand `auth.store` (user, permissions, profile link), `global.store` (package resources), `navbar.store` (title/breadcrumbs) | Gates, avatars, chrome |

---

## 25. Reference: every click target

| # | Element | Where | Result |
|---|---|---|---|
| 1 | Sidebar "Action Center" | App shell | `/actionCenter` |
| 2 | Saved view (system/user) | Toolbar | Applies snapshot (client) |
| 3 | Save / Update / Delete view | Toolbar | 💾 localStorage |
| 4 | Refresh scope | Toolbar, empty states, issue page | Silent reload |
| 5 | All / Mine / Unassigned, teammate | Toolbar | Client filter + rail `assigneeUserId` |
| 6 | Filters → Apply / Clear all | Filters panel | Client filters |
| 7 | Category chip × / Clear issue filter | Toolbar / panel | Clears category |
| 8 | Snoozed | Toolbar | Snoozed dialog |
| 9 | Export to CSV | Toolbar | `action-center-workflow.csv` |
| 10 | Clear ownership filter | Ownership strip | Ownership = All |
| 11 | Open connection settings | Channel blockers | `/channelManagement?channelId=` |
| 12 | Card click | Board | Variants → issue page; else run action |
| 13 | Primary CTA / More routes option | Card | §14 → product tab or channel management |
| 14 | More routes → Review issue | Card | `/actionCenter/issue/{rowId}?issueType=&languageCode=` |
| 15 | Drag to lane / ⋮ Move to | Card | `POST workflow MOVE` (In progress via menu → reason dialog) |
| 16 | ⋮ Mark solved | Card (To do) | `POST workflow MARK_SOLVED` |
| 17 | ⋮ Snooze preset / Pick date | Card | `POST workflow SNOOZE` |
| 18 | ⋮ Unsnooze / Resume now | Card / Snoozed dialog | `POST workflow UNSNOOZE` |
| 19 | Assign to me / Unassign | Card ownership / ⋮ | `POST assignment` |
| 20 | Assign to teammate / Assign visible lane | Card / lane header → dialog | `POST assignment ASSIGN` |
| 21 | Load more / Show all | Lane footer | More cards rendered |
| 22 | Lane Retry | Lane | Reload that lane |
| 23 | Issue type row | Rail breakdown | Toggle category filter |
| 24 | Recently cleared item / Recent activity item | Rail | Review-variants modal |
| 25 | See all | Rail activity | Activity dialog (24) |
| 26 | Retry velocity / history / activity | Rail | Reload that card |
| 27 | Create product / Import | No-products state | `/products` / `/channelManagement` |
| 28 | Connect a sales channel / Go to channels | No-channels state | `/channelManagement` |
| 29 | Try again / Open Command Center | Unavailable state | Reload / `/command-center` |
| 30 | Back to Action Center | Issue page | `/actionCenter` (+ return target) |
| 31 | Move to waiting | Issue page, review modal, ⋮ | Reason dialog → `MOVE WAITING_BLOCKED` |
| 32 | All / Active / Resolved | Issue page variants | Client filter |
| 33 | Variant action / More routes | Issue page, review modal | §14 with the variant's action |

---

## 26. Observations

⚠️ = seen while reading, not verified at runtime.

1. ⚠️ **Drag vs menu to "In progress" behave differently.** Dragging a card into In progress sends `status: IN_PROGRESS` with no reason. The API only requires a reason for a raw `WAITING_BLOCKED`, so no reason is asked. The ⋮ menu "Move to In progress" always opens the reason dialog and sends `WAITING_BLOCKED`. Running an action also auto-moves to In progress with no reason.
2. ⚠️ **`SHOPIFY_UNPUBLISHED` / `SHOPIFY_INACTIVE` / `CHANNEL_ERROR` never come out of the base snapshot.** `decorateScopeItem` never sets `shopifyUnpublished`/`shopifyInactive`: the Shopify query only includes ACTIVE, published products. `channelError` is always forced to `false` (channel errors are folded into `syncFailed`). These types still appear in filters and settings.
3. ⚠️ **Three lane requests.** Each lane asks the API for the full snapshot and filters it. The 15 s cache is in-process, so with several API instances each instance may rebuild the heavy SQL.
4. ⚠️ **The issue detail page loads the full `/overview`** (all rows, breakdowns, recent activity) to find a single row.
5. ⚠️ **A card's click handler is named `handleOpenVariantsModal` but it navigates** to the issue page. The modal is only reached from history items and return targets.
6. ⚠️ **Hard-coded URL**: the drawer posts to `'action-center/lifecycle-candidates'` directly. That endpoint is not in `webapp/src/lib/api.ts` (the monorepo rule says URLs belong there).
7. ⚠️ **Permission split**: moving, snoozing and marking solved need only `actionCenter.view`. The remediation value save needs `actionCenter.assign` (route list), although it is a data edit, not an assignment.
8. ⚠️ `buildWorkspaceShellPayload` has a ternary whose two branches are identical (`channelIssueRows`).
9. ⚠️ The "Marked solved and moved to pending verification" toast is decided on the client (row has a channel). The server may instead clear the ticket or return it to To do (§15.3).
10. ⚠️ `lifecycle-candidates` (plural) returns 500 for every error, including bad input. The other endpoints map `HttpException` statuses.
11. ⚠️ **Mixed variant statuses collapse to one card.** When a row's variants are in different statuses, the API returns it in several lane responses (lane filtering uses per-status counts > 0). The webapp then keeps one copy (`dedupeWorkflowRows`) and places it by the row's display status (first non-zero of open → in progress → pending). Example: 2 variants in To do and 1 In progress show as one To do card. The In-progress variant has no card of its own on the board and is only visible on the issue page.
12. ⚠️ The live page never sends `includeSourceDataIssues`, so readiness/content issues never reach the board for any user. Only the drawer requests them.

---

## 27. Appendix: files nothing imports

These files exist under `actionCenter/` but nothing in `webapp/src` imports them (so they never render):

| File | Apparent purpose |
|---|---|
| `_components/action-center-header.tsx` | Old page header |
| `_components/action-center-hero.tsx` | Old hero banner |
| `_components/action-center-kpis.tsx` | KPI tiles |
| `_components/action-center-page-selector.tsx` | View switcher |
| `_components/action-center-actions-list-view.tsx` | List view of actions |
| `_components/action-center-backlog-list-view.tsx` | Backlog list view |
| `_components/action-center-operational-table.tsx` | Operational table |
| `_components/action-center-priority-list.tsx` | Priority list |
| `_components/action-center-product-matrix.tsx` | Product × channel matrix |
| `_components/action-center-breakdown-row.tsx` | Breakdown row |
| `_components/action-center-move-menu.tsx` | Old move menu |
| `_components/action-center-compound-priority-list.tsx` | Compound priority list |
| `_components/action-center-compound-queue.tsx` | Compound queue |
| `_components/action-center-compound-shell.tsx` | Compound shell |
| `_components/action-center-compound-summary.tsx` | Compound summary |
| `_lib/saved-views.ts` | Old saved-views helper |

Only used by the files above (so also never rendered): `_components/action-center-variant-drilldown.tsx`. `redesign.ts` also still exports list-view helpers (`sortRowsForActionList`, `partitionRowsForViews`, `buildBacklogGroups`, `buildContextualSavedViews`, `exportBacklogRowsToCsv`), and `normalizeActionCenterSubview` forces `KANBAN` ("List view is intentionally hidden for now").

---

## 28. File index

**Webapp**
- Route: `webapp/src/app/(index)/(menu-layout)/actionCenter/page.tsx`, `main.tsx`, `issue/[rowId]/page.tsx`
- Components (live): `_components/action-center-permission-gate.tsx`, `-view-toolbar.tsx`, `-saved-views-menu.tsx`, `-filters-panel.tsx`, `-channel-blockers-section.tsx`, `-kanban-board.tsx`, `-workflow-menu.tsx`, `-ownership-trigger.tsx`, `-value-popover.tsx`, `-inline-identifier.tsx`, `-asin-identifier.tsx`, `-product-thumbnail.tsx`, `-metadata-chip.tsx`, `-insights-panel.tsx`, `-pre-action-dialog.tsx`, `-snoozed-dialog.tsx`, `-variants-modal.tsx`, `-affected-variants-panel.tsx`, `-issue-detail-page.tsx`, `-empty-state.tsx`, `-unavailable-state.tsx`, `-compound-workspace.tsx`, `-compound-board.tsx`, `-compound-drawer.tsx`, `-compound-status-badge.tsx`, `health-badge.tsx`, `section-card.tsx`
- Lib: `_lib/types.ts`, `redesign.ts`, `navigation.ts`, `cta-routing.ts`, `cta-display.ts`, `execution-gate.ts`, `issue-detail-route.ts`, `continuity-state.ts`, `workflow-saved-views.ts`, `ownership-helpers.ts`, `affected-variants-modal.ts`, `snooze-presets.ts`, `presentation.ts`, `presentation-contract.ts`, `compound-presentation.ts`, `pending-reason-options.ts`, `avatar.ts`, `dropdown-interactions.ts`
- Shared: `webapp/src/lib/api.ts` (`GET_ACTION_CENTER_*`, `UPDATE_ACTION_CENTER_*`), `webapp/src/lib/action-center-permissions.ts`, `webapp/src/lib/action-center-remediation-navigation.ts`, `webapp/src/lib/subscription-restricted-mode.ts`, `webapp/src/components/restricted-mode/restricted-mode-banner.tsx`, `webapp/src/app/(index)/(menu-layout)/_components/nav-container.tsx`
- Product drawer: `webapp/src/app/(index)/(menu-layout)/products/[product]/_components/action-center/*`
- Entry from Command Center: `webapp/src/app/(index)/(menu-layout)/command-center/main.tsx` (`openActionCenter`)

**API** (`api/apps/api-main/src/`)
- `modules/app/system/operations/action-center/action-center.controller.ts`, `action-center.service.ts`, `action-center.module.ts`, `dto/action-center.dto.ts`
- `actionCenter.lifecycle.ts` (statuses, verification policies, identity), `actionCenter.priority-ordering.ts` (groups), `actionCenter.compound.ts` (families), `presentation/actionCenterPresentation.ts` (labels)
- `action-center-listing-issue-read-model.service.ts`, `action-center-remediation-policy.service.ts`, `action-center-sync-correlation.service.ts`, `action-center-sync-evaluation.service.ts`, `actionCenterSyncContext.ts`
- `modules/app/system/operations/configuration/action-engine-settings/*` (runtime settings)
- `modules/app/platforms/amazon/amazon-events-action-center-materialization/*`
- `common/security/permissions.data.ts` (route permissions), `modules/app/commerce/billing/subscription-capabilities/*`
- Entities: `actionCenterIssueAssignment.entity.ts`, `actionCenterLifecycleEvent.entity.ts`, `amazonEventActionCenterMaterialization*.entity.ts`, `ActionEngineSettings` (`tbl_action_engine_settings`). `tbl_action_center_issue_workflows` has **no entity** (raw SQL only; created by `db-migrations/src/database/migrations/20260327114000-create-action-center-issue-workflows.js`).

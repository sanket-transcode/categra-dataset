# Command Center — Attention Card Types

The "What needs attention now" section groups every detected problem into **one card per type** (called a *family*). There are **18 card types**. A card appears only when at least one item is affected, and its count is the number of distinct affected items (listings, channels, accounts, marketplaces).

## Severity and colour

Severity is fixed per card type. The API sends the severity; the webapp decides the colour.

| Severity | Colour | Cards of this severity |
|---|---|---|
| `critical` | Red | 12 |
| `warning` | Amber | 5 |
| `info` | Blue | 1 |

## Card types

| # | Card type | Label | Impact pill | Severity | Colour | Recommended step | What triggers it |
|---|---|---|---|---|---|---|---|
| 1 | Connection | Channel | Sync paused | critical | Red | 1 · Restore access | A channel's connection is lost or was never connected. Categra cannot read or write that channel, so catalog, stock and order updates stop. |
| 2 | Account health | Account | Selling at risk | critical | Red | 1 · Restore access | An Amazon account is at risk or deactivated, or a Shopify store is uninstalled, frozen, locked or inactive. Selling privileges are in danger. |
| 3 | Marketplace health | Marketplace | Marketplace at risk | critical | Red | 1 · Restore access | An Amazon marketplace is reported as critical or at risk because of account or performance problems. Each marketplace is judged on its own, and every affected marketplace is listed on the card. |
| 4 | Permission required | Permission | Health unavailable | warning | Amber | 1 · Restore access | An Amazon channel has not granted the Selling Partner Insights role, so account health cannot be read until it is granted. |
| 5 | Data trust | Data trust | Data untrusted | critical | Red | 1 · Restore access | A channel's stock sync or order sync is stale, so numbers that depend on that data cannot be trusted. Stock and order problems are counted separately. |
| 6 | Missing price | Pricing | Cannot sell | critical | Red | 2 · Restore sellability | A published listing has no usable price: no positive offer price and no active sale price to fall back on. Buyers cannot purchase it. |
| 7 | Out of stock | Inventory | Cannot sell | critical | Red | 2 · Restore sellability | A published listing has no sellable stock. |
| 8 | Low stock | Inventory | Running low | warning | Amber | 4 · Improve catalog | A published listing still has stock, but at or below its low-stock threshold. |
| 9 | Stock sync | Inventory sync | Stock untrusted | critical | Red | 4 · Improve catalog | A published listing has stock and is not low, but the quantity on the channel does not match Categra, or a stock push is failing. |
| 10 | Amazon listing blocker | Listings | Cannot sell | critical | Red | 3 · Fix listings | Amazon blocked or rejected a listing, so it cannot sell until the issue is fixed. The card breaks the issues down by kind. |
| 11 | Amazon not searchable | Discoverability | Cannot sell | critical | Red | 3 · Fix listings | An Amazon listing has a marketplace status that does not include "discoverable", so buyers cannot find it in search. |
| 12 | Shopify not active | Shopify | Not visible | critical | Red | 3 · Fix listings | A Shopify product's status is anything other than active, so customers cannot view or buy it. |
| 13 | Main image | Media | Cannot sell | critical | Red | 3 · Fix listings | An Amazon product or variant published to a channel has no main image, or its image format is one Amazon does not support, so buyers cannot view or buy it. Shopify is excluded because its products can be purchased with or without a main image. The published image is read from the sync baseline first, then from the live media. The card separates missing images from unsupported formats. |
| 14 | Amazon listing warning | Listings | Needs fixing | warning | Amber | 4 · Improve catalog | Amazon returned non-blocking warnings on a listing. It can still sell, but the warnings should be corrected. |
| 15 | Product type changed | Structure | Needs remap | info | Blue | 4 · Improve catalog | Amazon changed the product type of a listing, so Categra's attribute mapping no longer matches and catalog updates are paused until the new type is accepted. |
| 16 | Buy Box lost | Pricing | Buy Box lost | warning | Amber | 4 · Improve catalog | A published Amazon listing is currently not winning the Featured Offer, while another offer holds it. This is a competitive disadvantage, but the listing may still be sellable. |
| 17 | Buy Box at risk | Pricing | Buy Box at risk | warning | Amber | 4 · Improve catalog | A published Amazon listing currently wins the Featured Offer, but Categra's pricing or competitive signals indicate an elevated risk of losing it. |
| 18 | Buy Box ineligible | Pricing | Buy Box excluded | critical | Red | 3 · Fix listings | Amazon indicates that a published Amazon offer is not eligible for Featured Offer consideration. The card breaks down the issues by reason, including price-related ineligibility where established. |

## Rules that shape what appears

- **Published items only.** Inventory, pricing, listing, discoverability, image and product-type cards count only items that are published to a channel inside the current page filter.
- **Page filters apply.** The channel and marketplace filters narrow every card. When a marketplace is selected, marketplace-level problems are limited to that marketplace; channel-level problems (connection, account, permission) still count.
- **Stock cards never overlap.** Each published item is placed in at most one stock card, in this order: out of stock, then low stock, then stock sync. An item that is out of stock never also appears as low stock or a sync issue.
- **Listing blockers hide listing warnings.** Amazon issues are split into blockers and warnings. If any blocker exists in scope, only the blocker card is shown; the warning card appears only when there are no blockers.
- **Main image is Amazon-only.** Shopify products can be bought with or without a main image, so Shopify listings never count. For Amazon, the published image comes from the sync baseline when one exists, otherwise from the live media; a missing image or an unsupported file type both count.
- **One card per type.** Every affected channel or marketplace appears as a chip on that card instead of as a separate card.
- **Data trust and Stock sync are always red.** Both cards are fixed at `critical`. Individual stale-data issues are still marked `warning` or `critical` internally (more serious when they have lasted more than 24 hours or affect every marketplace of a channel), but that internal level does not change the card, which is always `critical`.
- **Data trust and Stock sync can share a root cause.** No rule dedupes them against each other: Data trust counts channels or marketplaces with a stale stock or order sync, while Stock sync counts published listings whose quantity mismatches or whose push is failing. A stale stock sync can therefore raise both cards, at channel level and at listing level. Stock sync is still subject to the stock-card exclusivity rule above.
- **Same label, different colour.** "Inventory" covers out of stock (red) and low stock (amber). "Listings" covers blockers (red) and warnings (amber). "Pricing" covers missing price and Buy Box ineligible (red) and Buy Box lost and at risk (amber).

## Order and the recommended sequence

Cards are ordered first by recommended step, then by severity, then by card type, then by number of affected items. A red card can therefore sit below a card from an earlier step.

| Step | Name | Card types |
|---|---|---|
| 1 | Restore access | Connection, Account health, Marketplace health, Permission required, Data trust |
| 2 | Restore sellability | Missing price, Out of stock |
| 3 | Fix listings | Amazon listing blocker, Amazon not searchable, Shopify not active, Main image, Buy Box ineligible |
| 4 | Improve catalog | Amazon listing warning, Low stock, Stock sync, Product type changed, Buy Box lost, Buy Box at risk |

The impact of each step in "Next best actions" is its worst card severity: critical gives High, warning gives Medium, info gives Low. Because Stock sync is now `critical`, step 4 "Improve catalog" is High whenever a Stock sync card is present; otherwise it is Medium when Low stock, Amazon listing warning, Buy Box lost or Buy Box at risk is present, or Low when only Product type changed is. Steps 2 and 3 are always High when present. Step 1 is High unless Permission required is its only card, in which case it is Medium.

## Display limits

- Up to 5 cards show by default: the top 2 as large cards, the rest as compact cards.
- If there are more than 5, a "Show all N" button in the section header reveals the rest, and "Show less" collapses back to 5.
- Each card's CTA opens Action Center, pre-filtered to that card type and the current channel and marketplace filters. For Main image, when no channel filter is set and the card affects a single channel, the CTA is pinned to that channel; with several Amazon channels Action Center cannot filter by channel type, so its list may also include Shopify items.

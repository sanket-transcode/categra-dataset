# Amazon SP-API Operations

Source: `api/apps/api-main/src/modules/app/platforms/amazon/transport/amazon.service.ts` (`AmazonConnectorService`) · Date: 2026-11-16

Covers the 32 first-level functions that call an SP-API route directly. Every one of them sends through `makeAmazonRequest`, which:
- waits for the operation's rate-limit slot (per seller, or app-wide where noted),
- re-sends a `429` after the cooldown, up to `AMAZON_API_MAX_RETRIES`,
- marks the channel `CONNECTION_LOST` and throws `UnrecoverableError` when Amazon says the LWA secret expired,
- returns the response or the error (it never throws on ordinary failures, so callers check `instanceof Error`).

> Caller paths below are relative to `api/apps/api-main/src/modules/app/`.

---

## Operations

### confirmShipment
- `POST /orders/v0/orders/{orderId}/shipmentConfirmation` with the package details (carrier code, tracking number, marketplace id).
- Used by `OrderService.confirmShipment` (`commerce/orders/order/order.service.ts`), behind the order controller's confirm-shipment endpoint.
- Purpose in Categra: when a seller fulfils an Amazon (MFN) order from the Categra Orders screen, this tells Amazon the order has shipped and gives the buyer the tracking number.

### createFeed
- `POST /feeds/2021-06-30/feeds` with the feed type, marketplace ids and the `inputFeedDocumentId` returned by `createFeedDocument`.
- Second step of every Amazon feed, called after the JSON content has been uploaded to the pre-signed URL.
- Used by:
  - Forward sync Q2 (`sync/productSyncing/amazon/forwardSync/amazon-product-fsq2.service.ts`): bulk publish of products/variants as a `JSON_LISTINGS_FEED`.
  - Close listing (`catalog/products/lifecycle/close-listing/close-listing.service.ts`): bulk `PATCH` messages that set the listing quantity to 0.
  - Product deletion (`catalog/products/lifecycle/deletion/product-deletion.service.ts`): bulk `DELETE` messages that remove listings from Amazon.
- Purpose in Categra: lets one request carry many SKU changes instead of one Listings API call per SKU; the returned `feedId` is stored and polled with `getFeed`.

### createFeedDocument
- `POST /feeds/2021-06-30/documents` with `{ contentType: 'application/json' }`.
- Returns a `feedDocumentId` and a pre-signed upload `url`; the feed JSON is then uploaded with `uploadFeedDocumentContent`.
- Used by the same three flows as `createFeed` (forward sync Q2, close listing, product deletion).
- Purpose in Categra: first step of a bulk feed; reserves the document that holds the Categra-built messages (header with seller id + `messages[]`).

### createReport
- `POST /reports/2021-06-30/reports` with a report type and marketplace ids; returns a `reportId`.
- Used by:
  - Seller performance (`platforms/amazon/account-health/amazon-seller-performance.service.ts`): `GET_V2_SELLER_PERFORMANCE_REPORT` per seller + marketplace, skipped when the stored result is still fresh.
  - Amazon category sync (`sync/catalogBootstrap/amazon-categories/amazon-category-sync.service.ts`): `GET_XML_BROWSE_TREE_DATA` per marketplace; on "not registered" it retries with a tenant channel registered in that marketplace.
  - Older order sync (`commerce/orders/order/amazon/amazon-order.service.ts` → `generateOrderReportForOlderSync`): `GET_FLAT_FILE_ALL_ORDERS_DATA_BY_ORDER_DATE_GENERAL` for a user-picked date range, tracked by an `ORDER / IMPORT` task queue row.
- Purpose in Categra: data that Amazon only gives as reports (account health metrics, the browse-node tree, historical orders) is requested asynchronously and later read with `getReport` → `getReportDocument`.

### createSubscription
- `POST /notifications/v1/subscriptions/{notificationType}` with the payload version and destination id.
- Called through `AmazonSubscriptionClient.createSubscription` by `AmazonSubscriptionProvisionerService` (`platforms/amazon/events/subscriptions/`).
- Purpose in Categra: subscribes a connected seller to Amazon notifications (listing status, order change, account status, buy box…) so Amazon pushes events to Categra's SQS / EventBridge destination for this environment.

### deleteListingsItem
- `DELETE /listings/2021-08-01/items/{sellerId}/{sku}?marketplaceIds=…`.
- Used by:
  - `ProductDeletionService.processAmazonExternalProductDeletion` (`catalog/products/lifecycle/deletion/`): delete a product in Categra and on Amazon.
  - `AmazonProductChannelService.deleteProductFromMarketplace` (`catalog/products/listings/amazon-product-channel.service.ts`): remove the listing when a marketplace is removed from a product.
  - `ProductService.deleteListingFromAmazonMarketplace` (`catalog/products/product.service.ts`): delete a single parent/variant listing from one marketplace.
- Purpose in Categra: removes a SKU's listing from an Amazon marketplace for single-item deletes; bulk deletes go through feeds instead.

### deleteSubscriptionById
- `DELETE /notifications/v1/subscriptions/{notificationType}/{subscriptionId}` — **grantless** (client-credentials token, scope `sellingpartnerapi::notifications`).
- Called through `AmazonSubscriptionClient.deleteGrantlessById` by `AmazonSubscriptionProvisionerService.disable`.
- Purpose in Categra: removes a notification subscription at Amazon when a subscription row is disabled, but only when no sibling channel of the same seller still uses the same provider subscription id.

### getCompetitivePricing
- `GET /products/pricing/v0/competitivePrice?MarketplaceId=…&Asins=…&ItemType=Asin`.
- Used by `AmazonChannelService.fetchProductPricing` (`catalog/channel/amazonChannel/amazon-channel.service.ts`).
- Purpose in Categra: returns the competitive (buy box / landed) price of an ASIN in the channel's first marketplace, so the user sees the current Amazon price when matching or creating a product from an existing ASIN.

### getCompetitiveSummary
- `POST /batches/products/pricing/2022-05-01/items/competitiveSummary`; batch of up to 20 ASINs, each asking for `featuredBuyingOptions`, `referencePrices` and `lowestPricedOffers` per item condition.
- Used by `BuyBoxRefreshService.fetchCompetitiveSummary` (`catalog/products/listings/buy-box/buy-box-refresh.service.ts`).
- Purpose in Categra: market-level buy box data per ASIN + condition (featured offer, lowest prices, reference prices); saved as market offers (source `COMPETITIVE_SUMMARY`) and used by the buy box evaluation and Action Center buy box tickets. Works for ASINs even when the listing is unpublished.

### getDefinitionsProductType
- `GET /definitions/2020-09-01/productTypes/{productType}?marketplaceIds=…[&sellerId=…][&locale=…]`.
- Uses one **app-wide** rate slot (`app:getDefinitionsProductType`) so calls from all sellers share Amazon's application-level pace.
- Used by:
  - Backward sync Q4 (`sync/productSyncing/amazon/backwardSync/amazon-product-bsq4.service.ts`): fetches the product type definition, then once per missing language (sequential, for the rate limit); the schema JSON itself is downloaded with `fetchProductTypeSchemaDocument`.
  - `AmazonChannelShippingTemplateService.fetchShippingTemplate`: reads the seller-specific schema to extract the seller's shipping templates (`merchant_shipping_group`).
  - `AmazonProductTypeDefinitionsInspector.inspect` (definitions refresh + checksum backfill): compares Amazon's schema checksum with the stored one to detect changed product type definitions.
- Purpose in Categra: builds the Amazon attribute schema (attributes, groups, options, required rules, per-language labels) that drives the product forms, readiness and forward-sync payloads.

### getFeaturedOfferExpectedPriceBatch
- `POST /batches/products/pricing/2022-05-01/offer/featuredOfferExpectedPrice`; one request per SKU + marketplace.
- Used by `BuyBoxRefreshService.fetchExpectedPrice`.
- Purpose in Categra: tells the seller what price their offer needs to become the featured offer (win the buy box); stored per seller SKU and used to flag a listing as buy box "at risk" and to suggest a price.

### getFeed
- `GET /feeds/2021-06-30/feeds/{feedId}`.
- Used by forward sync Q3 (`amazon-product-fsq3.service.ts`), close listing and product deletion, each with a retry when the first read fails.
- Purpose in Categra: polls a submitted feed until its `processingStatus` is `DONE` / `FATAL` / `CANCELLED`, then hands the `resultFeedDocumentId` to `getFeedDocument`.

### getFeedDocument
- `GET /feeds/2021-06-30/documents/{resultFeedDocumentId}`; returns the download URL of the processing report.
- Used by forward sync Q3, close listing and product deletion.
- Purpose in Categra: reads Amazon's per-message results of a feed so each SKU can be marked published / closed / deleted, or its errors and warnings stored against the product (sync errors in Categra).

### getInventorySummaries
- `GET /fba/inventory/v1/summaries?details=true&granularityType=Marketplace&granularityId=…&sellerSku=…&marketplaceIds=…`.
- Used by `AmazonFbaInventoryService.fetchFbaInventory` (`sync/productSyncing/amazon/amazonFbaInventory/`).
- Purpose in Categra: reads FBA stock for a SKU (fulfillable, inbound working/shipped/receiving, reserved, researching, unfulfillable) so Categra can show and store the quantity Amazon holds in its warehouses.

### getListingOffers
- `GET /products/pricing/v0/listings/{sku}/offers?MarketplaceId=…&ItemCondition=…&CustomerType=Consumer`.
- Used by `BuyBoxRefreshService.fetchListingOffers` when a batch holds a single SKU.
- Purpose in Categra: the seller's view of all offers on their SKU (buy box winner, competing prices, whether the seller is buy box eligible); single-SKU form of `getListingOffersBatch`.

### getListingOffersBatch
- `POST /batches/products/pricing/v0/listingOffers`; one `GET …/listings/{sku}/offers` request per SKU + condition.
- Used by `BuyBoxRefreshService.fetchListingOffers` for batches of 2+ SKUs.
- Purpose in Categra: same data as `getListingOffers` for many SKUs in one call; results feed the seller offer rows (source `OFFERS`) used by buy box status and Action Center buy box tickets.

### getListingsItem
- `GET /listings/2021-08-01/items/{sellerId}/{sku}?marketplaceIds=…[&includedData=…]`.
- Used by:
  - Backward sync (`amazon-product-bs.service.ts` → `refreshAndPersistAmazonListingStateBySku`): reads `attributes,issues,summaries` to store the listing state and its issues.
  - Backward sync Q1 (`amazon-product-bsq1.service.ts`): reads the parent SKU to get the real parent ASIN, since child listings sometimes report a wrong parent.
  - Listing notifications (`platforms/amazon/events/handlers/listing/listing-state.refresher.ts`): refreshes a listing after a listing-status notification, falling back to the notification's own issues.
  - Stock (`catalog/products/stock/product-stock.service.ts` → `fetchAmazonMarketplaceStock`) and order stock deduction (`amazon-order.service.ts` → `deductAmazonStock`): read `fulfillmentAvailability` (fulfillment channel code + quantity).
  - `ProductService.recheckProductType`: reads `productTypes` to check whether the listing's Amazon product type still matches Categra's.
- Purpose in Categra: the single-SKU source of truth for what Amazon currently holds for a listing (status, issues, attributes, quantity, product type).

### getMarketplaceParticipations
- `GET /sellers/v1/marketplaceParticipations`.
- Successful responses are cached in Redis for 10 minutes (`amazon:marketplaces:v1:{region}:{scope}`); `bypassCache` forces a live call; failures are never cached.
- Used by:
  - `AmazonChannelService.validateTokenForRegion`: live check (cache bypassed) while connecting a channel, to validate the token and detect a region mismatch.
  - `AmazonMarketplaceService.getUserMarketplaceParticipation`: returns the seller's marketplaces to the webapp for marketplace selection.
  - `AmazonSubscriptionReconcileService` (through the subscription client): resolves which marketplaces the seller participates in when reconciling notification subscriptions.
  - `AmazonSystemCredentialsService` (category sync): checks a credential is registered in the marketplace before requesting its browse tree.
- Purpose in Categra: tells Categra which Amazon marketplaces a connected seller can sell in, and doubles as the cheapest call to prove a token works.

### getMyFeesEstimateForSKU
- `POST /products/fees/v0/listings/{sellerSku}/feesEstimate` (SKU URL-encoded in the path) with `{ FeesEstimateRequest: { MarketplaceId, IsAmazonFulfilled, PriceToEstimateFees, Identifier } }`.
- Response: `{ payload: { FeesEstimateResult } }`, the same `FeesEstimateResult` object (`Status`, `FeesEstimateIdentifier` with `IdType: SellerSKU` + `IdValue`, `FeesEstimate`, `Error`) that `getMyFeesEstimates` returns per item.
- Used by `BuyBoxRefreshService.fetchFees` when a batch holds a single SKU (like `getListingOffers`); Amazon allows 1 request/second here vs 0.5 for the batch call.
- Purpose in Categra: same as `getMyFeesEstimates`, for one SKU at a time.

### getMyFeesEstimates
- `POST /products/fees/v0/feesEstimate`; top-level array of `{ FeesEstimateRequest, IdType: 'SellerSKU', IdValue: sku }` (marketplace, listing price + currency, FBA or not).
- Response: top-level array of `FeesEstimateResult`, one per requested SKU, each with its own `Status`.
- Used by `BuyBoxRefreshService.fetchFees` for batches of 2+ SKUs; only SKUs with a price are sent, and they are re-fetched when the price basis changes or the data is due.
- Purpose in Categra: stores the referral / FBA / closing fees per SKU so margins can be calculated (buy box guidance, Command Center margin cost stack).

### getOrder
- `GET /orders/2026-01-01/orders/{orderId}[?includedData=…]`.
- Used by:
  - `AmazonOrderService.fetchAmazonOrderFromNotification`: loads the full order after an `ORDER_CHANGE` notification.
  - Order Q1 processor (`commerce/orders/order/amazon/processors/amazon-order-q1.processor.ts`): re-fetches specific existing orders by id (recipient, proceeds, expense, fulfillment, packages, tax, payment…), and fetches a single order job.
- Purpose in Categra: keeps one Categra order in step with Amazon (status, items, amounts, fulfillment) without listing all orders.

### getReport
- `GET /reports/2021-06-30/reports/{reportId}`.
- Used by seller performance (under a Redis lock per report id) and Amazon category sync.
- Purpose in Categra: polls a report created with `createReport` until it is `DONE` (then reads `reportDocumentId`) or failed/cancelled.

### getReportDocument
- `GET /reports/2021-06-30/documents/{reportDocumentId}`; returns the download URL (and compression) of the report file.
- Used by seller performance (parses account health metrics) and Amazon category sync (parses the XML browse tree into category nodes; zero nodes is treated as an error).
- Purpose in Categra: last step of the report flow; turns a finished report into data Categra stores (account health, Amazon categories).

### getSubscription
- `GET /notifications/v1/subscriptions/{notificationType}` with the seller's token.
- Called through `AmazonSubscriptionClient.getSubscription` by `AmazonSubscriptionProvisionerService` (`verify`, `disable`).
- Purpose in Categra: finds the seller's existing subscription for a notification type when Categra has no stored provider id, so it reuses it instead of creating a duplicate, or marks the row `NOT_SUBSCRIBED`.

### getSubscriptionById
- `GET /notifications/v1/subscriptions/{notificationType}/{subscriptionId}` — **grantless**.
- Called through `AmazonSubscriptionClient.getGrantlessById` by `AmazonSubscriptionProvisionerService` (verify / reuse checks).
- Purpose in Categra: confirms a stored provider subscription id still exists and points to this environment's destination; a `404` marks the row `NOT_SUBSCRIBED` so it can be re-created.

### patchListingsItem
- `PATCH /listings/2021-08-01/items/{sellerId}/{sku}?marketplaceIds=…` with JSON-patch operations.
- Used by:
  - `AmazonProductFSService.patchProductOnAmazon`: partial publish of changed attributes only.
  - `AmazonProductFSService.updatePriceOnAmazon`: price-only update.
  - Forward sync Q1 `removeDeletedImagesFromAmazon`: deletes image attributes that were removed in Categra.
- Purpose in Categra: updates part of an existing listing without resending the whole item, keeping attributes Categra didn't change untouched.

### patchListingsItem_20200901
- `PATCH /listings/2020-09-01/items/{sellerId}/{sku}?marketplaceIds=…` (older API version).
- Used by `AmazonProductFSService.updateStockOnAmazon`.
- Purpose in Categra: pushes a stock / fulfillment availability change for one SKU.

### putListingsItem
- `PUT /listings/2021-08-01/items/{sellerId}/{sku}?marketplaceIds=…` with the full item (product type, requirements, attributes).
- Used by `AmazonProductFSService.syncProductOnAmazon` (parent / standalone) and `AmazonProductVariantFSService.syncProductVariantOnAmazon` (variant).
- Purpose in Categra: creates a listing on Amazon or fully replaces it during a single-item forward sync (publish).

### searchCatalogItems
- `GET /catalog/2022-04-01/items?…` with identifiers (ASIN / EAN / UPC…), `includedData` and marketplace ids.
- Used by:
  - `AmazonChannelService.searchAmazonProduct`: search Amazon by an external id when creating/matching a product.
  - Product search by ASIN Q1 (`catalog/products/listings/productSearchByAsin/amazon-product-cbaq1.service.ts`): imports an ASIN with summaries, attributes, images, product types and relationships (variation family).
  - Backward sync Q1 (`amazon-product-bsq1.service.ts`): enriches the seller's listings with catalog data in ASIN batches.
  - Catalog refresh (`sync/productSyncing/amazon/catalogRefresh/amazon-catalog-refresh.service.ts`): one call per ASIN batch + marketplace, fanned out to every account holding the ASIN.
  - `ProductChannelService` and `AmazonChannelProductTypeSchemaService`: read an ASIN's catalog attributes to pre-fill Amazon attributes when a product type / marketplace is linked.
  - Listing status handler (`platforms/amazon/events/handlers/listing/listing-status.handler.ts`): refreshes title and images after a listing notification.
- Purpose in Categra: Amazon's catalog data for an ASIN (title, images, brand, product type, parent/child relationships), independent of the seller's own listing.

### searchDefinitionsProductTypes
- `GET /definitions/2020-09-01/productTypes?marketplaceIds=…`.
- Used by `AmazonChannelService.getUserProductTypes` (once per channel marketplace) and `AmazonProductTypeService.fetchProductTypes` (upserts new names into the product types table).
- Purpose in Categra: lists the Amazon product types available in a marketplace so the user can pick one for a product and Categra keeps its product type list current.

### searchListingsItems
- `GET /listings/2021-08-01/items/{sellerId}?marketplaceIds=…&…` paged with `pageToken`.
- Used by backward sync Q1 (`amazon-product-bsq1.service.ts`).
- Purpose in Categra: pages through all of a seller's listings in a marketplace during backward sync (import from Amazon), the starting point for creating/updating products in Categra.

### searchOrders
- `GET /orders/2026-01-01/orders?marketplaceIds=…&createdAfter=…&maxResultsPerPage=100&includedData=…` paged with `NextToken`.
- Used by the order Q1 processor (order import) and backward sync Q6 (`amazon-product-bsq6.service.ts`, orders step of backward sync).
- Purpose in Categra: imports a seller's orders for a period into Categra's orders table (with recipient, proceeds, fees, fulfillment, packages, tax), feeding the orders screen, stock deduction and sales metrics.

---

## Summary

| # | Operation | Method | Route |
|---|---|---|---|
| 1 | confirmShipment | POST | `/orders/v0/orders/{orderId}/shipmentConfirmation` |
| 2 | createFeed | POST | `/feeds/2021-06-30/feeds` |
| 3 | createFeedDocument | POST | `/feeds/2021-06-30/documents` |
| 4 | createReport | POST | `/reports/2021-06-30/reports` |
| 5 | createSubscription | POST | `/notifications/v1/subscriptions/{notificationType}` |
| 6 | deleteListingsItem | DELETE | `/listings/2021-08-01/items/{sellerId}/{sku}` |
| 7 | deleteSubscriptionById | DELETE | `/notifications/v1/subscriptions/{notificationType}/{subscriptionId}` (grantless) |
| 8 | getCompetitivePricing | GET | `/products/pricing/v0/competitivePrice` |
| 9 | getCompetitiveSummary | POST | `/batches/products/pricing/2022-05-01/items/competitiveSummary` |
| 10 | getDefinitionsProductType | GET | `/definitions/2020-09-01/productTypes/{productType}` |
| 11 | getFeaturedOfferExpectedPriceBatch | POST | `/batches/products/pricing/2022-05-01/offer/featuredOfferExpectedPrice` |
| 12 | getFeed | GET | `/feeds/2021-06-30/feeds/{feedId}` |
| 13 | getFeedDocument | GET | `/feeds/2021-06-30/documents/{feedDocumentId}` |
| 14 | getInventorySummaries | GET | `/fba/inventory/v1/summaries` |
| 15 | getListingOffers | GET | `/products/pricing/v0/listings/{sku}/offers` |
| 16 | getListingOffersBatch | POST | `/batches/products/pricing/v0/listingOffers` |
| 17 | getListingsItem | GET | `/listings/2021-08-01/items/{sellerId}/{sku}` |
| 18 | getMarketplaceParticipations | GET | `/sellers/v1/marketplaceParticipations` |
| 19 | getMyFeesEstimateForSKU | POST | `/products/fees/v0/listings/{sellerSku}/feesEstimate` |
| 20 | getMyFeesEstimates | POST | `/products/fees/v0/feesEstimate` |
| 21 | getOrder | GET | `/orders/2026-01-01/orders/{orderId}` |
| 22 | getReport | GET | `/reports/2021-06-30/reports/{reportId}` |
| 23 | getReportDocument | GET | `/reports/2021-06-30/documents/{reportDocumentId}` |
| 24 | getSubscription | GET | `/notifications/v1/subscriptions/{notificationType}` |
| 25 | getSubscriptionById | GET | `/notifications/v1/subscriptions/{notificationType}/{subscriptionId}` (grantless) |
| 26 | patchListingsItem | PATCH | `/listings/2021-08-01/items/{sellerId}/{sku}` |
| 27 | patchListingsItem_20200901 | PATCH | `/listings/2020-09-01/items/{sellerId}/{sku}` |
| 28 | putListingsItem | PUT | `/listings/2021-08-01/items/{sellerId}/{sku}` |
| 29 | searchCatalogItems | GET | `/catalog/2022-04-01/items` |
| 30 | searchDefinitionsProductTypes | GET | `/definitions/2020-09-01/productTypes` |
| 31 | searchListingsItems | GET | `/listings/2021-08-01/items/{sellerId}` |
| 32 | searchOrders | GET | `/orders/2026-01-01/orders` |

All routes are relative to the regional host: `https://sellingpartnerapi-{na|eu|fe}.amazon.com`.

**Not listed (not first-level SP-API route calls):** `makeAmazonRequest` / `makeAmazonGrantlessRequest` (transport helpers), `exchangeLwaToken` / `getLwaClientCredentialsToken` (LWA token endpoint `api.amazon.com/auth/o2/token`), `fetchProductTypeSchemaDocument` / `uploadFeedDocumentContent` (Amazon pre-signed S3 URLs, not SP-API routes).

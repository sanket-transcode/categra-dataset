#### High level objective

- It should show for a product (SKU):
  - How many active offers present currently?
  - Your landed price
  - Lowest landed price
  - Gap Vs. Lowest landed
  - Eligibility
  - FBA pick and pack fees
  - Current competitors (their landed price (item + shipping), fulfillment type (ex. FBA), prime or not)
  - A status for buy box, ex. Winning, At risk
- Command Center
  - Cumulative count of products whichever are buy box lost, at risk and ineligible for buy box

#### Important clarifications

- Lowest landed price is not necessarily the Buy Box-winning price. Featured Offer selection can depend on multiple factors.
- Eligibility is not the same as winning. An eligible offer may not currently hold the Featured Offer.
- Winning chance is an estimate, not a guaranteed Amazon metric.

#### Amazon Resources

- SP APIs
  - **getListingOffers (**https://developer-docs.amazon/sp-api/reference/getlistingoffers**)\*\*
    - Already supporting but enhance it to grab specific info from it and structure it well
  - **getListingOffersBatch (**https://developer-docs.amazon/sp-api/reference/getlistingoffersbatch**)\*\*
    - Use this in case of huge amount of calls for a single channel (Up to 20 SKUs)
  - **getMyFeesEstimateForSKU (**https://developer-docs.amazon/sp-api/reference/getmyfeesestimateforsku**)\*\*
    - Already supporting but enhance it to grab specific info from it and structure it well
  - **getMyFeesEstimates (**https://developer-docs.amazon/sp-api/reference/getmyfeesestimates**)\*\*
    - Use this in case of huge amount of calls for a single channel (Up to 20 SKUs)
  - **getCompetitiveSummary (**https://developer-docs.amazon/sp-api/reference/getcompetitivesummary**)\*\*
    - Use this everytime, compulsary ASIN (Up to 20 ASINs)
  - **getFeaturedOfferExpectedPriceBatch (**https://developer-docs.amazon/sp-api/reference/getfeaturedofferexpectedpricebatch**)\*\*
    - Use this by SKU (Up to 40 SKUs)
- Event Notifications
  - ANY_OFFER_CHANGED
  - PRICING_HEALTH

- Use **CustomerType** as **Consumer** fixed
  - For **getListingOffers, getListingOffersBatch, getCompetitiveSummary**

- Current Buy Box Statuses
  - COMPETITIVE
  - NOT_COMPETITIVE
  - NO_BUY_BOX
  - INELIGIBLE
  - WINNING
  - AT_RISK
  - UNKNOWN
- Suggested Buy Box Statuses (will completely replace the current ones)

| Status        | Seller-specific meaning                                                                                                                                         | How Categra computes/gathers it                                                                                                                                                                                                       |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `WINNING`     | You are currently winning the Buy Box. Your offer holds the Amazon Featured Offer position.                                                                     | Confirm your offer is identified as the Featured Offer winner using fresh `getCompetitiveSummary` / `getListingOffers` data or a relevant `ANY_OFFER_CHANGED` notification.                                                           |
| `AT_RISK`     | You are winning now, but may lose the Buy Box. Signals indicate your current position is at risk.                                                               | Your offer is currently winning, and Categra's risk rules detect signals such as a worsening landed-price gap, an unfavorable expected-price signal from `getFeaturedOfferExpectedPriceBatch`, or a relevant competitor price change. |
| `NOT_WINNING` | Your offer is in the competitive range, but another offer is winning. Your price is competitive, but you do not currently hold the Buy Box.                     | Confirm another offer holds the Featured Offer position and your offer does not. Check your landed price against comparable offers using `getCompetitiveSummary` and `getListingOffers`.                                              |
| `INELIGIBLE`  | Amazon is not allowing your offer to compete for the Buy Box. This may be due to price-related or another eligibility issue.                                    | Use `PRICING_HEALTH`, explicit offer-eligibility information where available, and supporting pricing/listing data. Store the reason separately, such as `PRICE_TOO_HIGH` or `LISTING_ISSUE`.                                          |
| `UNAVAILABLE` | Your offer is not currently available for purchase, or its required current pricing information cannot be established.                                          | Check listing/offer availability and price fields. Distinguish confirmed unavailable offers from missing prices or incomplete responses using internal reason codes; do not infer unavailability from a missing price alone.          |
| `UNKNOWN`     | Categra cannot yet determine your Buy Box status. The product may not have been published to Amazon, or the required data may still be loading or insufficient. | Use when the listing is not yet published, initial synchronization is in progress, data is stale or incomplete, or API calls fail to establish a reliable status. Track publication and data-fetch states internally.                 |

#### Attention Family Types

- Buy Box Lost - warning → count of products having `NOT_WINNING`
- Buy Box at risk - warning → count of products having `AT_RISK`
- Buy Box ineligible - critical → count of products having `INELIGIBLE`

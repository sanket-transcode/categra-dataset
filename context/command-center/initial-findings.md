This doc represents the list of plan for the command center and action center data items.

1. **Inventory (FBM , FBA , Easy Ship)**
   1. Based on cron job daily iterate over every product check their stock(FBM , FBA, Easy Ship) in categra make list of near threshold in 3 day and send it to command center , action center.
   2. FBA_INVENTORY_AVAILABILITY_CHANGES Notifcaiton
      1. Detect FBA availability updates and show update in command center or as notification
   3. LISTINGS_ITEM_MFN_QUANTITY_CHANGE Notification
      1. Detect seller-fulfilled quantity changes & show update in command center as notification
2. **Account Health**
   1. Subscribe to OnACCOUNT_STATUS_CHANGED notification & on receive run the following report and pull necessary data and update in account health table
      1. GET_V1_SELLER_PERFORMANCE_REPORT
      2. GET_V2_SELLER_PERFORMANCE_REPORT
   2. Also write cron to periodically pull this data without notification and save and if there is any major drop is there show it on the command center \=\> account center.
3. **Listing issues \=\>** suppression , buyable status change using notification handle
   1. When product turns into non buyable state \=\> LISTINGS_ITEM_STATUS_CHANGE
   2. When product turns into suppression \=\>LISTINGS_ITEM_ISSUES_CHANGE
   3. On notification update in product as well as show it on command center , action center
4. **Competitiveness** \=\> Price , shipping , offers
   1. ANY_OFFER_CHANGED Notification gives 2 type info
      1. COMPETITIVE_CHANGE (Detect changes in top offers; batch fetch offers)
      2. BUY_BOX_CHANGED(Detect Buy Box winner/price change; enrich with competitive summary)
      3. Volatility Detection \=\> Detect frequent Buy Box changes
      4. Competitive Pressure Change \=\> Detect changes in offer count/price spread
      5. Price Gap Detection \=\> Compare own vs lowest vs Buy Box price
         1. Target Price Insight , Fetch expected Buy Box price
         2. Used to update price automatically to stay in competition
      6. Based on buybox data give Price Opportunity Signal=\> suggest potential price (Detect potential margin increase)
         1. Margin increse
   2. PRICING_HEALTH Notification
      1. PRICE_UNCOMPETITIVE \=\> Detect non-competitive pricing; enrich with FOEP
   3. For Shipping / Prime / Fulfilment change detection we need to store historical data and compare it using cron.
5. **Traffic & Sales**
   1. Increasing/decreasing traffic & sales trend identification
      1. GET_SALES_AND_TRAFFIC_REPORT \=\> give Sales and traffic performance data for the seller’s business, including **ordered product sales, ordered units, sessions, page views, buy box percentage, and unit session percentage**.
      2. Also other brand analytics related reports available that can be used if brand is registered.
      3. Cron to fetch this report periodically & save the states in our db and show declining traffic/revenue products into command center/action center.
6. **Profit**
   1. Revenue
   2. Product Cost
   3. Amazon cost \=\> ads , storage etc
   4. Profit calculation
7. **Ads**
   1. We need to explore amazon ads api
      1. https://advertising.amazon.com/API/docs/en-us/reference/api-overview

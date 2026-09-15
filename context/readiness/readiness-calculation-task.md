## api & webapp repo only

Task 1: Centralize readiness values with correct constraints for products and variants

### Description

- For products and variants, exactly below languages must be included as part of readiness
  - For master: all account enabled languages strait list (account base language + tbl_account_languages records for account)
  - For shopify: all current channel associated languages (tbl_shopify_language_mappings for current account + channel)
  - For amazon: each product/variant will return those languages list for whatever marketplaces they are enabled (actual language live in tbl_amazon_channel_marketplaces, product enable status in tbl_product_channel_marketplaces, variant enable status in tbl_variant_marketplaces)
    - In case of a specific marketplace filter is receievd, you return the readiness for that marketplace language only

### Context

- api\apps\api-main\src\modules\app\catalog\products\productList\productListQueryBuilder.ts (route: /products/get-products-list-raw)
- api\apps\api-main\src\modules\app\catalog\products\product.service.ts -> productVariantsListRaw, (route: /products/product-variants-list-raw)
- webapp\src\app\(index)\(menu-layout)\products_components\list\List.tsx

### Constraints

- respect is_deleted flag for entities wherever it exists

- In terms of any channel or in products or variants, the readiness values array key should be consistent, handle accessing the key properly into FE side

- In case of readiness value not exist for a language (in case the language is within the scope), it should be considered as 0, but the key point is the langauge must not be excluded into calculation

- Remove returning singleton readiness value, ex. an average readines key like currently has "readiness_value", there should be only the array of readiness values for current scope languages

- It will be the frontend's job to show the computed average of received readiness records

- BE will return exact value without rounding up, every calculation will also include exact value but in terms of showing, the value will be shown as rounded up

- In FE, if average value itself is 0 then on hover, it shouldn't show cursor as pointer and the tooltip won't be shown as well for all languages readiness

- In case of amazon channel, if a marketplace filter is selected then just show that marketplace language specific value on UI and on hover, no cursor pointer and hover tooltip

- At any point if there is a little need for single or multiple clarifications then you must ask for it rather than implementing purely on assumptions

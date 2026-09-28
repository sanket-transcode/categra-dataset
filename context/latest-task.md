## api repo only

Task 1: When it is calling suggestions for parent (not variant) then as part of the payload values, it should fill the payload with values with from active channel-marketplace variants

### Description

- within suggestAmazonGroupAttributeValues, currently as part of input payload values, it is levaraging only parent, but from now you'll going to pass values for those keys for which parent doesn't have value but maybe some of the variants have
  - Check 

### Context

- api\apps\api-main\src\modules\app\catalog\products\product.service.ts -> suggestAmazonGroupAttributeValues

### Constraints

- Only active variants should be considered (is_active TRUE from tbl_product_variant_channels and status TRUE from tbl_variant_marketplaces)
- A single attribute key + language should be unique within the payload, meaning per key -> either parent or one variant will get chance to contribute for that key
  - You can filter out from the database directly to not return those records which have no value for all keys
- For whatever attribute key + language, the parent has the value and to be contributed within the payload, any of it's variant won't interfere in that as variants will be levaraged only if parent lacks the value per (attribute + language)

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

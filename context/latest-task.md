## api repo only

Task 1: Amazon product catalog values should be created as suggestions for attributes and during local AI call shouldn't overwrite those

### Description

- While crafting dataset of type ExternalProductData, set another key similar to listingAttributes -> catalogAttributes which will be specifically got from the catalog response (within file api\apps\api-main\src\modules\app\sync\productSyncing\amazon\backwardSync\amazon-product-bsq1.service.ts)

- out of the attributes, there are 2 primary attributes to be extracted: item_name means Product Name, product_description means Product Description
  - The rest are structured AMAZON group attributes
- I would suggest you to add primary attributes suggestions within api\apps\api-main\src\modules\app\sync\productSyncing\amazon\backwardSync\amazon-product-bsq2.service.ts and group attribute suggestions within api\apps\api-main\src\modules\app\sync\productSyncing\amazon\backwardSync\amazon-product-bsq4.service.ts
  - For resolving exact attribute id, review existing approach into the shared file and generate an optimized and simple approach

- While upserting attribute suggestions (upsertAttributeSuggestedValues), skip those records completely for whose there are already records present into the table having same scope (product + variant + channel + language) and is_catalog_sourced as true

### Context

- Shared BS files

### Constraints

- The suggestions should be stored for current channel id as well as master channel as well
- A record will overwrite existing same scoped value so normal upsertion no issue, the differentiation is where the current record is AI generated and not received from amazon catalog then it doesn't overwrite the one marked as catalog sourced
  - All these should be handled within the single universal function upsertAttributeSuggestedValues

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

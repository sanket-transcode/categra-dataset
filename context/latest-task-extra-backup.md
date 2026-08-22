## api repo only

Task 1: Pass correct keys as part of partial product publish for amazon Forward Sync

### Context

api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\forwardsync\amazonProductFSQ1.service.ts
api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\forwardsync\amazonProductFSQ2.service.ts
api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\forwardsync\amazonProductFSQ3.service.ts
api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\forwardsync\amazonProductVariantFS.service.ts
api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\amazonProductFS.service.ts

### Constraints

- Things will be passed in the structure of patches (for both the approaches: direct PATCH listing and Amazon Feeds)

- You pass things which are either came as selected or supposed to be passed (use op as 'replace' for that key) or something is there which is supposed to be deleted (use op as 'delete' for that key)

- Every top level property is build using buildNestedStructure, so for partial publish you don't care about all the properties, you build the relevant properties only

- Here is the full explanation of what top level keys should be passed and how according to the selection received for an entity SKU (product and variant)

1. attributes
  - Whatever 'attributeIds' are come, they must be primary attributes, here are the actual amazon keys for primary attributes
    - for 'Product Name' -> 'item_name'
    - for 'Product Description' -> 'product_description'
    - for 'Identity Type' -> 'externally_assigned_product_identifier' -> 'type'
    - for 'Identity Value' -> 'externally_assigned_product_identifier' -> 'value'
    - You will receive above 4 categra attributes only that should be passed to amazon if received as selected, in case of 'Identity Type' or 'Identity Value' either is received as selected you have to pass the complementary attribute
  - Whatever 'hierarchyIds' are received, they are top level attributes or keys, so they are straint forward, build nested structure for the receievd hierarchy keys only and pass into payload for patches
2. If 'pricing' received as true, you build two absolute full top level properties 'purchasable_offer', 'list_price' and pass into payload for patches
3. If 'media' received as true, there are changes the media is either added, deleted or position changed, you handle this smartly; the keys are within CONSTANTS.AMAZON_MEDIA_PAYLOAD_KEYS
4. For variant -> variantStructure is true
  - You need to pass keys: variation_theme, parentage_level and child_parent_sku_relationship
  - As part of child_parent_sku_relationship -> parent_sku, in case valid wrapper's SKU is not there so the self sku is being passed as parent sku, this should work the same no change
5. For parent -> in case wrapper variant structure is supposed to be published
  - you need to pass variation_theme and parentage_level
6. In case for a variant -> variantAttributes is true
  - First figure out the attached wrapper -> amazonCurrentVariationTheme, using extractRequiredFields you get the accumulated top level properties directly that should be passed to the payload
7. In case of miscTypes -> below 3 should be taken care of:
  - AMAZON_CONDITION_TYPE -> property as condition_type
  - AMAZON_SHIPPING_TEMPLATE -> property as condition_type
  - AMAZON_CONDITION_TYPE -> property as merchant_shipping_group
  - AMAZON_CATEGORY -> a conditional path is there, the possible evaluations are: recommended_browse_nodes and item_type_keyword

- At any point if there is a strong need for single or multiple clarifications then you must ask for it rather than implementing purely on assumptions

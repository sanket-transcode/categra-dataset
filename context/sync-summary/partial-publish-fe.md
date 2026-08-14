### Types of Publish

1. Publish to External Channel (Full Publish)
2. Publish Changes (Partial Publish)

### Places to show publish options:

1. Product list
   1. Full Publish
      1. Always Show for individual + Bulk
   2. Partial Publish - on click open the dialog for selected products + their corresponding variants
      1. Individual Product/Variant
         1. If entity is OUT_OF_SYNC into current channel/any one market
      2. Bulk Product/Variant
         1. If any of the selected entity is OUT_OF_SYNC into current channel/any one market
2. Variants tab
   1. Full Publish - As it is flow
      1. Always Show for individual + Bulk
   2. Partial Publish - on click open the dialog for selected variants
      1. Individual Variant
         1. If variant is OUT_OF_SYNC into current channel/marketplace
      2. Bulk Product/Variant
         1. If any of the selected variant is OUT_OF_SYNC into current channel/marketplace
3. Channels tab Publish
   1. Publish button is disabled if (parent + all enabled variants) have sync status as SUCCESS
   2. On click publish, a simple confirmation dialog will be opened mentioning below things
      1. Unpublished entities whose full data will be published
      2. Published entities, whose changes will be published
      3. Published entities which have no changes to publish
   3. Within existing API, add extra param within body: **`isIncrementalSync`** that will act into backend to push only changes that are required

- Opened the changes review dialog will call API /sync-review/get-change-summary with selected products structure (include variants) + channel id + optional amazon marketplace id
- On Submit, in case at least one change is selected then enable button “Publish Changes” and call an API that will pass the whole product structure (marketplace wise + variant wise selected changes) + channel id

### Selection Scope

1. Attributes
   1. Hierarchy (for amazon group attributes)
   2. Individual (for rest)
2. Pricing - Single
3. Media - Single
4. Variant Structure - Single
5. Variant Attributes - Single
6. Misc - Per type

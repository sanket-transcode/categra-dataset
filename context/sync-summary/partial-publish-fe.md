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
    2. On click Publish or Publish All, a simple confirmation dialog will be opened mentioning below things marketplace wise
        1. Unpublished entities whose full data will be published
        2. Published entities, whose only changes will be published
        3. Published entities which have no changes to publish
    3. In case for a market, it has at least one entity which is published but changes exist, provide a button for modify changes and the smart dialog will be opened, on save from the dialog will close it and return the latest selected payload back to the first dialog
    4. On Confirm, call the existing API '/sync/product-syncing/sync-product-to-channel’, add extra param within body: **`isIncrementalSync`** that will act into backend to push only changes that are required + the selection structure also and filter out those products/variants from selection structure which are out of synced but 0 changes selected for them from

- Opened the changes review dialog will call API  /sync-review/get-change-summary  with selected products structure (include variants) + channel id + optional amazon marketplace id
- Dialog Constraints:
    - If you want to select variant attributes then you must select variant structure also as it is dependent whereas the reverse flow is not dependent
    - In case you select variant structure then for that product + marketplace, it will be selected for all variants visible into screen and internally for the parent as well
    - On submit, first the backend will check the same condition: If variant structure is selected then any sibling found which is not selected for sync then show warning message and simply say you need to publish all products data as there is a variant structure change
    - In UI, show a message that media is shared across all marketplaces of this channel and a single state to toggle it
    - A Single state for toggling all variant structure for selected marketplace and within payload, it lives directly at outer level - Show message into UI to appear it as shared
    - On Submit, in case at least one change is selected then enable button “Publish Changes” and call API 'sync/product-syncing/publish-products-to-channel’, pass the whole selection structure for product(s), filter out those products/variants from main product map and selection dataset which are synced but 0 changes selected for them

### Selection Scope

1. Attributes
    1. Hierarchy (for amazon group attributes)
    2. Individual (for rest)
2. Pricing - Single
3. Media - Single
4. Variant Structure - Single
5. Variant Attributes - Single
6. Misc - Per type
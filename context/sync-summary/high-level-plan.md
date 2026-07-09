Sync Status:

1. In Sync → “SUCCESS”
2. Out of Sync → “OUT_OF_SYNC”
3. Not Published → “NOT_PUBLISHED” or empty

Enabled entities: which has boolean key true into respective tables

Note: Amazon has concept of marketplaces so it will add a grouping layer between product & channel where as other type of channels not

1. Non-amazon channel: parent has is_active key into tbl_product_channels & variant has key is_active into tbl_product_variant_channels
2. AMAZON channel: same as above + additionally parent has key status into tbl_product_channel_marketplaces & variant has key status into tbl_variant_marketplaces

1. Variants tab
    1. Individual variant
        1. Show “Publish Changes” → if “Out of Sync”
        2. Show “Publish to External Channel” always
    2. Bulk variants
        1. Show “Publish Changes” → if at least one selected variant is “Out of Sync”
        2. Show “Publish to External Channel” always
    3. On click “Publish to External Channel”, flow as it is, no changes
    4. On click “Publish Changes“, call /sync-review/get-change-counts passing product id with selected variant ids + channel id + amazon marketplace id (for amazon only)
        1. If received empty array, call an API in which pass product id + selected variant ids and alter the sync status in state for those variants from “Out of Sync” to “In Sync”
        2. otherwise open the change UI dialog, in which only one product will be there with selected, call /sync-review/get-change-summary with product id + selected variant ids + channel id + amazon marketplace id
            1. On submit from the dialog, call the /sync/product-syncing/publish-products-to-channel by passing the selected things
2. Products list
    1. At non-amazon channel level + Amazon with selected marketplace level
        1. Individual product/variant
            1. Show “Publish Changes” → if “Out of Sync”
            2. Show “Publish to External Channel” always
        2. Bulk products/variants
            1. Show “Publish Changes” → if at least one selected entity (product or variant) is “Out of Sync”
            2. Show “Publish to External Channel” always
    2. At amazon with all markets level
        1. Individual product/variant
            1. Show “Publish Changes” → if at least for one marketplace “Out of Sync”
            2. Show “Publish to External Channel” always
        2. Bulk products/variants
            1. Show “Publish Changes” → if at least one selected entity (product or variant) for one marketplace is “Out of Sync”
            2. Show “Publish to External Channel” always
        3. On click “Publish to External Channel”, flow as it is, no changes
        4. On click “Publish Changes“, call /sync-review/get-change-counts passing product ids with their selected variant ids (if parent itself selected then empty array) + channel id + amazon marketplace id (for amazon only & if marketplace filter applied)
            1. If received empty array, call an API in which pass product ids + selected variant ids (same empty variants if parent selected) and on success alter the sync status in state for those parents or variants from “Out of Sync” to “In Sync”
            2. otherwise open the change UI dialog, in which by default 1st product will be selected and call /sync-review/get-change-summary with product id + selected variant ids + channel id + amazon marketplace id
                1. On submit from the dialog, call the /sync/product-syncing/publish-products-to-channel by passing the selected things
3. Channels tab
    - Dynamic working “Publish” button → No label change as of now
        - Standalone → Button will be disabled in case of “In Sync”
        - Product With variants → Button will be disabled in case of all enabled variants + parent itself (enabled variants and product will be considered in channel/marketplace) are “In Sync”
    - On click Publish
        - first call /sync-review/get-change-counts by passing current product id only with empty array + channel id + marketplace id (if market level button)
        - If received empty array
            - if at least one variant or parent is “Not Published”, call the API as working now no changes: sync/product-syncing/sync-product-to-channel
            - otherwise call an API in which pass product id +empty array and alter the sync status in state for enabled parent + variants into channel/marketplace from “Out of Sync” to “In Sync”
        - otherwise open the change UI dialog, in which by default 1st product will be selected and call /sync-review/get-change-summary with product id + selected variant ids + channel id + amazon marketplace id
            1. On submit from the dialog, call the /sync/product-syncing/publish-products-to-channel by passing the selected things
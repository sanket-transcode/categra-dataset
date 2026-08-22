## api repo only

Task 1: During Amazon Forward sync using Amazon SP API feed document method: as part of each feed object: use operation type as PATCH instead of PARTIAL_UPDATE, and create patches for affected keys only the similar way it has been done for sync standalone product (syncStandAloneProductToMarketplace) where a dedicated PATCH listing API is being used -> make sure you don't duplicate the code and use common utility functions for generating patches and related functionalities

### Context

api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\forwardsync\amazonProductFSQ1.service.ts
api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\forwardsync\amazonProductFSQ2.service.ts
api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\forwardsync\amazonProductFSQ3.service.ts
api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\forwardsync\amazonProductVariantFS.service.ts
api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\amazonProductFS.service.ts

### Constraints

- At any point if there is a strong need for single or multiple clarifications then you must ask for it rather than implementing purely on assumptions

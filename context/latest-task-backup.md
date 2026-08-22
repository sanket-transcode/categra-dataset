## api repo only

Task 1: Few things mentioned into constraints are not working properly as part of partial sync but working perfectly for full sync, make sure it works for partial publish as well

### Context

api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\forwardsync\amazonProductFSQ1.service.ts
api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\forwardsync\amazonProductFSQ2.service.ts
api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\forwardsync\amazonProductFSQ3.service.ts
api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\forwardsync\amazonProductVariantFS.service.ts
api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\amazonProductFS.service.ts

### Constraints

- The issue lies within the publish through Amazon Feeds, the standalone product publish (full and partial both) (syncStandAloneProductToMarketplace) are working perfectly

- The primary issues identified are regarding to the 'rawData' we are storing for taskQueue entity, exactly at what paths the issues present are below:
  - details[].wrappers[].variants[] -> at this level: missing keys are: name, variantId, wrapperId -> during partial publish they are missing, for full publish this is working (refer from there)
  - details[].wrappers[].variants[].marketplaces -> here unique id based object should be there which have id, status, countryCode, payload, error as keys for corresponding marketplace -> but rather for partial publish, it has generated 3 object chunks within marketplaces array, each object contains keys: (status, payload, error and other keys) separately across each object -> (refer from how it is during full publish, do the same for partial publish as well to have the correct behaviour)
  - make sure for parent and variants dataset, the common keys and raw data structure between full publish and partial publish should stay consistent, only few additional keys should be different conditionally

- At any point if there is a strong need for single or multiple clarifications then you must ask for it rather than implementing purely on assumptions

## api repo only

Task 1: Within amazon FS batches, pass necessary thing in case of 'op' is 'delete'

### Context
 
api\apps\api-main\src\modules\app\sync\syncing\productSyncing\incremental\partialPayloadScope.service.ts -> buildScopedPatches, buildPatchOperations, buildFeedMessageBody
api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\forwardsync\amazonProductFSQ1.service.ts -> syncStandAloneProductToMarketplace
api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\forwardsync\amazonProductFSQ1.service.ts
api\apps\api-main\src\modules\app\sync\syncing\productSyncing\amazon\forwardsync\amazonProductVariantFS.service.ts

### Constraints

- A single key within the payload contains an array of objects, within each object it may have 'marketplace_id' and may have 'language_tag' and other data keys which are dynamic

- Currently in case of op is 'delete', you're not passing any 'value' key unlike op as 'replace', which is causing issue, here is how you resolve that:
  - for a single patch, whenever op is 'delete', you need to pass the 'value' key as well but within each array of object, you pass (marketplace_id if schema has it and language_tag if schema has it) no those dynamic keys should be passed
  - For every delete operation you must need to do that because omitting 'value' field is not working
- Make sure the existing full publish flow doesn't break at all the change should be done at a consistent and specific places

- At any point if there is a strong need for single or multiple clarifications then you must ask for it rather than implementing purely on assumptions

## api repo only

Task 1: Create a full suggested plan within amazon Backward Sync (Import) by identifing common flaws and for making the process consistent and atomic

### Description

- As of now Amazon BS has total 6 queues which are divided into their corresponding files and a common referencing generic bs service
- Each queue processor function has a high level error and failure management (failing there will mark the worker as fail and accordingly to the worker max retires it will restart the processor function) and another heavy cleanup and failure handling is at the products batch level
- The so called "cleanup" runs at last of every successful batch of products, which includes updating some redis data, increment/decrement/update something into DB, emitting sockets, logging important things. The same way in case some error occurs in between the batch processing, it does the same (updating tracking and redis, database update, logging, emitting sockets), so I want you to plan the changes as per below constraints
  - Take care for the sequence of operations and dependencies between them the most, there should be no compromise in that (ex. in api\apps\api-main\src\modules\app\sync\productSyncing\amazon\backwardSync\amazon-product-bsq2.service.ts, it must run setProductChannelMarketplaceLoading before the productSuccess as when the product is presented on UI, it must have the necessary loaders be started by default)
  - At most of the cases, things must be atomic, whenever common transactions can be maintained then do that and most importantly for sockets: in case the trasaction environment is there then it must use afterCommit transaction callback (currently it is not being used anywhere)
  - Some cleanup operations and most of the error handling operations are kind of that which shouldn't throw errors themselves, so handle those cases wisely
    - Use tryCatchWrapper the optimal way as per the usecase
  - Handling in case data is unavailable from redis in between
  - Use parallel executions for operations wherever needed only, don't use them forcefully
  - I faced a blocker issue where for a batch of products, every SKU of the current batch is stored within SYNCED_KEYS_KEY as well as in FAILED_KEYS_KEY (which is absolutely wrong, the processed SKUs must be within exactly one key among these 2)

### Context

- api\apps\api-main\src\modules\app\sync\productSyncing\amazon\backwardSync\amazon-product-bs.service.ts
- api\apps\api-main\src\modules\app\sync\productSyncing\amazon\backwardSync\amazon-product-bsq1.service.ts to api\apps\api-main\src\modules\app\sync\productSyncing\amazon\backwardSync\amazon-product-bsq6.service.ts

### Constraints

- You can change the code structure, create common functions as per the needs, consistency and optimal code structure, you may need to alter the code and operations but the end behaviour shouldn't change (If you find some behaviour absolutely wrong then you can add on the fix into that also or in case of critical conditions you must take my input)

- The plan should be created within directory categra-dataset\context\bs

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

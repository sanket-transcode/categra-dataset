## api & webapp repo only

Task 1: Move the whole out of sync calculation for 6 tyeps of baselines into queue based mechanism

### Description

- example for attributes calculation is processAttributeOutOfSync
- There should be a single universal trigger be there with a common pattern for scheduling an out of sync operation and the trigger will be responsible for further orchestration of the requested task
  - similar to this._readinessTriggerService.trigger

### Context

- api\apps\api-main\src\modules\app\sync\queue\queue.service.ts

### Constraints

- Do I make a single queue that will be responsible for all kind of baselines calculation or 6 separate queues for each, tell me with genuine short reasons

- Follow every common queue standards and code patterns as per currently have

- ask questions agreesively to make sure any tiny confusion doesn't remain at the feature level

- first consturct the plan only into an md file at categra-dataset\context where you define the clear plan but before that inform me about the first constrait choice and bring my choice first then write down the full low level implementation plan

- At any point if there is a little need for single or multiple clarifications then you must ask for it rather than implementing purely on assumptions

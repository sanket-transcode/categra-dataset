## webapp & api & db-migrations repo only

Task 1: There are problems regarding to forward sync and baselines, so think and find a crucial solution plan that resolves the conflicts and the system should stay consistent

### Description

- Requirement for Forward Sync (FS) (Publish):

It should forward the values state which was at the time of publish task creation
It is allowed to make changes while FS is in progress for the product, but those changes must not be passed into the payload to be published
It is not working currently because while FS task is making payload then it gathers values from the normal tables, what if someone has changed a value in between the task is created and payload is created

An issue occurs also when reverting a baseline while FS is in progress
A value changed from Green to Blue then publish task started then we revert the baseline then it deletes the baseline and goes back to Green but the external channel will have Blue
Same case occurs If I change from Green to Blue before the publish task is started and change from Blue to Red after the task has started, then ideally it should have passed Blue to publish payload and on revert it will go back to Green

Another issue is if product is not published and publish task has started, after that whatever changes I make those are absolutely not recorded as baselines


In order to prevent that I have below initial solution:


We need to block the revert as well as the field modification of those attributes/values which had baselines before the publish task has started
Because of the revert problem
Manually editing and writing back the baseline value means the same was reverting

When a product is Not Published
After publish has started someone do a change, then
Create the baseline
Set the sync status as Out of Sync

In case of FS failure
It will reset the sync status as Not Published
It will delete all entity's baselines

In case of FS succeed
It will keep the baseline as it was created after the publish task is added
The status will also remain as Out of Sync


In case of product is In Sync then 
While gathering payload for the FS
If baseline created after the task has scheduled then pick baseline value
Otherwise pick from the normal tables



I have another solution in mind:


In that we will maintain an extra value state which will be storing changed values while FS is in progress for that product

### Context

- Create the plan and dump at an md file at categra-dataset\context\product

### Constraints

- 

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

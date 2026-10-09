## api & webapp & db-migrations repo only

Task 1: Remove the requested stalled tables/keys from database, their legacy implementation within code

### Description

- Execute the requested things only from the planned file: context/amazon-events/amazon-events-analysis.md

- Remove tbl_amazon_event_destinations as there is no scope of increasing the destinations, there will be fixed number of static entries and the destination ids are already within .env + additionally remove the code around it
- Remove tbl_amazon_channel_subscriptions and code around it (it is legacy and not needed)
- Follwing keys within tbl_amazon_event_subscriptions
  - destinationId (the table itself will be removed so)
  - accountId (as it will be derived from amazonChannel)
  - channelId (as it will be derived from amazonChannel)
  - notificationLabel (use the notificationType instead as label at the accessing places)

- Apart from (tbl_amazon_event_raw_deliveries, tbl_amazon_event_subscriptions) -> remove other tbl_amazon_event tables out of total 21
  - According to the analysis file, whatever dataset should be absorbed or merged should be done in that way, In case of major working feature affected due to the deletion, clarify first before actual implementation
  - Kindly check admin portal feature explicitly, in case something major affects then also clarify first
  
- In case of final entities/keys removal, the related coding areas should also be removed

### Context

- 

### Constraints

- You create separate migration files for different kind of operations, but not too many, related things/entities can be grouped into one file
- Provided instructions will be overwritten and for rest you follow the plan file for execution
- Those suggested improvements within the analysis file which don't contain any table/key deletion should be remained later on and not part of the current task
  - But the improvements which should be done before the requested entities/keys are deleted must be done as part of this task
- In case something is related with webapp or admin repos then do the required cleanup from there also

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

## webapp & api & db-migrations repo only

Task 1: 

### Description

- issue_type_rank and priority are needed or not, what those column indicates and how they are useful? because priority can be derived from app constants, consider a
  case where in future the sorting of tickets by issue type becomes dynamic
- isn't there sub issue type within the tickets table, ex. what if an issue type is account health related (mimic from command center) but for C1 it is account health
at risk and for C2 it is account de-activated -> and explain how a practical sub issue will vary between variants of a same tickets and is there any case happen that
a ticket which has no variants can have multiple sub issues. Also consider in mind that whenever we are redirected from command center for channel health related
button then all kinds of related tickets should be filtered even if the different channsl under the hood holds slightly different issues
- why do we need column status_changed_at (anyway it will be recorded within the events tracking table), same goes for assigned_at
- I think we don't need assigned_by_user_id as the UI don't care about who assigned the ticket
- Why do we need detected_at at this stage
- why do we need issue_type into events table, isn't it already present within the tickets
- should we make the feature to publish directly from action center ticket, because we need to handle the sockets, block the button with conditions similar to product list has, will it be better the button will redirect to the product page with q=channels, where we will provide the context bar and user will publish on his own

### Context

- 

### Constraints

- 

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

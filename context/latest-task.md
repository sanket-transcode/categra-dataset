## webapp & api & db-migrations repo only

Task 1: Create plan for handling amazon competition and buy box related things at more of a system level, feature level is already decided

### Description

- The base requirements mentioned the objectives and which exactly APIs and Event notifications to handle
  - You need to figure out how we will store things into system
  - Deeply study the schema and examples responses of the mentioned endpoints and notifications and figure out common structure out of them that will be handled consistently among APIs as well as through notifications
  - In case of some static fields or computed fields you evaluate from the amazon APIs/notifications which are very good enough to store at database fields then you are allowed to do that regardless of currently requirements include it or not
  - In case of some info is there which doesn't need to be structured but is useful enough for future features then you can keep them into JSON as well rather than introducing way many table fields
- The buy box statuses should be stored within database per (product id + variant id (nullable) + channel id + amazon marketplace id), they will no longer be computed
  - The base requirements already mentiones the new statuses to be introduced for buy box

### Context

- categra-dataset\context\buy-box\base-requirements.md

- Product Overview
  - Product Variants (Buy Box status)
  - Performance By Channel (BUY BOX & PRICING)
  - Variant Performance (BUY BOX & PRICING)

### Constraints

- Keep the terminologies at everyplace unambiguous and not too vague
- only one table currently is having related info: tbl_product_current_offers, which is just storing JSON, do not need to preserve it, your new database architecture can absolutely demolish it
  - It is fine, you will need to reshape the existing code resolves around buy box
- Do not care about action-center regarding buy box and do not consider it for anything (it will be reshaped later on), but the new changes will shape it if needed
- This competition thing will strictly be for amazon only so don't care about shopify
- Write down end to end plan within categra-dataset\context\buy-box

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

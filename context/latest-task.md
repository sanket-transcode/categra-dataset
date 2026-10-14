## webapp & api & db-migrations repo only

Task 1: Remove stalled legacy keys from the table

### Description

- from table tbl_product_media_linkers, remove these keys that are not being used currently, they are legacy keys
  - language_code
  - channel_marketplace_id
- Previously media were stored marketplace and language wise but now not, remove such code that does that way during creation/modification and access time

### Context

- Command Center -> What needs attention now
- Inspiration UI: "C:\Users\Promise\Downloads\image (2).png"

### Constraints

- Create a single migration file but don't execute
- In case of webapp is affected anyway then remove from there also
- In case you find somewhere commented code around it that looks legacy, remove that code also

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

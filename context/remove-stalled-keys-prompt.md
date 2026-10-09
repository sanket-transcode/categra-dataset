## api & webapp & db-migrations repo only

Task 1: Remove the requeted stalled (legacy) fields from the requested tables

### Description

- tbl_products
  - isTempSku
  - validSku
  - isArchive
  - allowStockTracking
  - consumedAiToken
  - attachment
  - supplierId

### Context

- 

### Constraints

- In case of any major working is there which actually affects for the keys that are requested for removal then inform first before the actual removal 
- Just create migration files per entity, don't execute them actually
- Remove from api repo where the keys are accessed/modified, do not remain any place
- In case of webapp is having the references or structures around these stalled keys then do the cleanup there as well

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

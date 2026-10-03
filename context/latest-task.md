## api & webapp repo only

Task 1: For variant, convert the 6 levels of inheritance into absolute 3 levels of normal inheritance the same way as per the parent + additional requirements

### Description

- Currently at some places, the variant is having max 6 levels of inheritance like this:
  - Variant + Channel + current language
  - Variant + Master + current language
  - Product + Channel + current language
  - Product + Master + current language
  - Variant + Master + base language
  - Product + Master + base language
- This should be converted to the same way as all places like parent:
  - Variant + Channel + current language
  - Variant + Master + current language
  - Variant + Master + base language
- Remove returning parent values, parent fallback values, parent base values within GetProductAttributesWithRawQuery
- Within details tab, stop showing the switches as well for variant's master + base language (it has some specific conditions, so stop treating variant's master + base as special and "can inherit" way, it should be exactly the same as generic (channel current -> master current -> master base) rule (regardless or product/parent) or variant)
- On hover of inheritance switch for attributes in details tab, message should be different when I’m on master and in channel
  - Because for channel it inherits from the same langauge but from master channels cope whereas within master, it inherits from the master's base langauge to master's current selected language

### Context

- 

### Constraints

- Need to implement this at literally every place of the project within api and webapp: during baseline calculation (if affects), readiness calculation (if affects), getting primary or group attributes for details tab (if affects), during forward sync time and other affecting places as well, no place should remain

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

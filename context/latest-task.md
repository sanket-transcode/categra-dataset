## webapp & api & db-migrations repo only

Task 1: Make sure the expected behaviour is there, if not then do the required changes + remove the unused code

### Description

- In case of amazon attribute (AM_)
  - It must not translate main lables from AI
    - The main labels will going to be translated for the 4 interface languages only from schema fetching (which is already in place)
  - In case of options, only those options should be translated from AI in which the attribute is having 'customTextAllowed' as true and option object isn't having 'marketplaceLanguage'
    - Because having marketplaceLanguage means that option is given by amazon retricted to a specific marketplace, so we don't touch it and customTextAllowed as false attribute will definitely have all marketplaceLanguage bound options only

### Context

- 

### Constraints

- 

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

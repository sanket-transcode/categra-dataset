## webapp & api & db-migrations repo only

Task 1: Make sure the expected behaviour is there, if not then do the required changes + remove the unused code

### Description

- In case of attribute group labels translation
  - For non-amazon groups, It should translate into 4 interface languages + account enabled languages from AI
  - For amazon groups: It should write down the translations while fetching schema into the 4 interface languages so lables into those 4 interface languages only, amazon groups need not to be translated into account languages

- Make sure into details tab, it should present the group label as into current interface language (fallback chain: account base language -> tbl_attribute_groups - label -> tbl_attribute_groups - name)
  - for all kinds of attribute groups (master, amazon, shopify)

### Context

- 

### Constraints

- 

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

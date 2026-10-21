## api & webapp & db-migrations repo only

Task 1: Provide me a suggested high-medium level flow that is user/seller friendly and genuinly helps seller but don't contain conflicts

### Description

- While activating a variant for an amazon channel and for an amazon language/marketplace: if that specific variant has whatever common variant attributes from any other marketplace of the current channel or other channel then they should pre-populate values from the other marketplace

- I think the API should be called in parallel when the edit dialog opens to fetch the suggested values from backend

- The suggested variant attribute values pre-population should only be run during the time or marketplace activation and edit dialog opened (not on normal edit)

### Context

- webapp\src\app\(index)\(menu-layout)\products\[product]\_components\variants\_components\edit-variant.tsx

### Constraints

- The key that will be containing the values from the other marketplace should be something like ;'suggested values' or compatible name
- It shoudl finalize a single marketplace and values structure for that marketplace to be given as suggestions, (color - red is received from M1 and size - XL is received from M2 --> this shouldn't happen)
  - The priority of finalizing the single marketplace can be the most relevant marketplace from the current one, activated marketplace for that variant over inactive one + good values assigned within a marketplace and not missing
- There are possibilities that the received suggested values are not in scope of the current marketplace/wrapper then it will not be pre-populated simply - user will select manually the value for that attribute

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

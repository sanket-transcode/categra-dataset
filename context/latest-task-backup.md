## webapp repo only

Task 1: Implement revised 3. Improvements -> #1 and #2

### Description

- implement localizeProductAttributeValues as the plan itself proposes
- implement #2 as the plan itself proposes, if affects any revised approach then add on that
- implement suggestAmazonGroupAttributeValues as per my proposed plans for combining master + channel call into one
  - For those 3 bugs and fixes you've found:
    - for 1. Variants which are enabled into marketplace will share the common call for master + those master variants which are not enabled into current channel marketplace but are enabled into any other channel's such marketplace which has the same language code as currently processing then those calls will be extra only for master only, those master variants which are not enabled into any of the channel's such amazon marketplace which has the current language code will be ignored simply
  - for 2. as per your recommendations
  - for 3. go with whatever is appropriate and consistent

### Context

- categra-dataset\context\llm\amazon-bs-q5-input-token-reduction-plan.md

### Constraints

- 

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

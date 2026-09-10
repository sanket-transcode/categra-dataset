## webapp repo only

Task 1: Into product details: for primary attributes specifically: for master & Shopify: Product Name is required and for Amazon: Product name and description both are required, so perform below tasks regarding them:

- Show star icon for labels which are required at a certain point (includes the identity value as well)
- Show red border and message in case of required but missing
- The save will also be disabled in case of an active error exist

### Context

webapp\src\app\(index)\(menu-layout)\products\[product]\_components\details\product-structure-item.tsx and related files

### Constraints

- In case of disablePrimaryAttributes, then no red border and required error should be there, skip the validation check as well, for identity type (it evaluates automatically but won't show any error)

- At any point if there is a strong need for single or multiple clarifications then you must ask for it rather than implementing purely on assumptions

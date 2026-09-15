## webapp and api repo only

Task 1: remove handling of route /readiness/product-values completely from the FE side, do not remove the route itself from BE

### Description

- remove flag fetchSeparateReadiness completely
  - alter the implementations as below whereever it is being used currently
    - as part of webapp\src\app\(index)\(menu-layout)\products\[product]\_components\productOverView\_components\productSummaryCards.tsx, do the changes within the basic info getting API and render the same way but the data will be received from basicInfo
    - as part of webapp\src\app\(index)\(menu-layout)\products\[product]\_components\details\product-essentials.tsx, put the necessary calculation into BE side /products/get-product, and return the array dataset of readiness
    - for both above APIs, already the product id compulsary, variant id correctly and channel id correctly passed, so use those from the API
    - Optimize the readiness gathering within both the APIs but the working must be the same as per the rules mentioned into categra-dataset\context\latest-task.md

### Context

- 

### Constraints

- 

- At any point if there is a little need for single or multiple clarifications then you must ask for it rather than implementing purely on assumptions

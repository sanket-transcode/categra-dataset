## api & webapp repo only

Task 1: within channels tab, do the correct calculation of readiness and return in consistent manner from API side, and render correctly into UI as per the given standards

### Description

- Figure out from which API, all channels (master and non master channels within webapp\src\app\(index)\(menu-layout)\products\[product]\_components\channels\channels-tab.tsx) is coming, within that for each kind of channel the calculation should be the same way as the referenced task md
  - Here in channel's tab we care about the parent product values only (explicitly variant id as null in DB queries)
  - For master, amazon and shopify, all those channel level readiness will show cursor pointer on hover show all languages readiness the same wasy as the referenced task describes
- For amazon only, there are readiness coming for active marketplace wise also, so there also the calculation should be returned correctly: /product-channel/get-product-channel-data
  - Here there will be a single value only so no cursor pointer and hover on UI (parent + amazon channel + corresponding active market language)

### Context

- webapp\src\app\(index)\(menu-layout)\products\[product]\_components\channels\channels-tab.tsx and descendents

### Constraints

- Apart from the strict overwrites for this task, the readiness rules will be the same as categra-dataset\context\latest-task.md

- At any point if there is a strong need for single or multiple clarifications then you must ask for it rather than implementing purely on assumptions

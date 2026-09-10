## api repo only

Task 1: Execute requested Findings and improvements as per following current constraints from file QUEUE4-BACKWARD-SYNC-PERFORMANCE-ANALYSIS.md

## Description

- Execute points: F, G, I, J, K

### Context

### Constraints

- For G: You don't pass product ids or variant ids as the intention itself is to find such errors which have SKUs but don't have product ids, so you pass the accumulated SKUs exist into current batch so that the find scope narrows down

- For I: treat as a common solution for this query, for that use utility functions if needed and remove previous utility functions if they are unused now

- For J: You just create necessary migrations as per the repo conventions and scripts but don't execute them, leave the execution to me

- For K: Only do this task if the function is used within local file and other places (only if those other places fetch base language or necessary things or can fetch without causing any problem) otherwise keep the fetching isolated, and while passing base language to anywhere use the convention as 'baseLanguageCode'

- At any point if there is a little need for single or multiple clarifications then you must ask for it rather than implementing purely on assumptions

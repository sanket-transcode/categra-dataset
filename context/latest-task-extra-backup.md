## webapp & api & db-migrations repo only

Task: Fill the actual operational cost with real accurate researched figures

### Description

- The template is already made, you just need to fill it with real figures by going through the process step by step

### Context

- categra-dataset\context\budget\categra-operational-cost-template.xlsx

### Constraints

- The assumptions shouldn't be set blindly, you should decide each assumption to be filled by either analysing the codebase or reading from the database or to be directly clarified from me, no one should esacpe among these 3 paths
- Currently the database have single seller id for NA region and single seller id for EU region that's it spread across every amazon channel across every account, but for the estimates: you would forcefully consider distinct seller id per channel and with this assumption you go on
- Giving you the READ access for the current database, you can pick the credentials from .env
- For accurate pricing, search live to the respective services to find out accurate pricing for the selected assumptions
- Make sure not to ruin the format anyway, maintain consistency and durability of the figures and data throughout modification
- In case of it strongly requires changing the format, then only you're allowed to change the format throughout

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

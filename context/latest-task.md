## webapp & api & db-migrations repo only

Task 1: For command center, the issues calculation should be as per issue tickets and remove the dynamic calculation being done from now

### Description

- This should cover all the sections which are doing dynamic calculation behind the figures, it will be computed from tickets onwards

### Context

- 

### Constraints

- Make sure to respect current applied filters at command center
- The cards call sellable counts (standalone + variants) whereas the tickets are per parent product, so handle that accordingly: variants specific items are sub items per ticket

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

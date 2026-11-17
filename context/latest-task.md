## webapp & api & db-migrations repo only

Task: Fill the actual operational cost with real accurate researched figures

### Description

- The template is already made, you just need to fill it with real data by going through each major point and sub-points with internal and external anlysis

### Context

- categra-dataset\context\amazon-signals\categra-amazon-signals-plan-template.xlsx

### Constraints

- Giving you the READ access for the current database, you can pick the credentials from .env
- Make sure not to ruin the format anyway, maintain consistency and durability of the figures and data throughout modification
- As part of internal anaysis, you can go through to Categra api, webapp, db-migrations, admin repo codes; additionally database as well
- As part of external analysis, you can visit the Amazon SP resources, 
- Most importantly analyse the style of presenting things by those items which are already being handled in categra regarding amazon signals and opportunities: ex. Buy Box & Competition, Amazon issues (listing errors and warnings), Seller account's health, Amazon marketplace health and basically all issue types handled within command center and action center
- It doesn't mean you have to fit all the cases within the existing Categra UI pages and sections, in case something needs extra section or dialog you are free to introduce that if it strictly doesn't fit within the exiting place somewhere
- Kindly levarage the in app notifications for categra, for those events in which user should be notified for something important occurs at that moment
  - Include already handled things also in this for which the notifications are not handled and they requires it to be sent within app
  - We have email functionality as well, make sure to mention those also, they shouldn't be as frequent as notifications, only specific scenarios (you think about them) should be handled through email as well

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

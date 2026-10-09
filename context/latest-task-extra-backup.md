## api & webapp & db-migrations repo only

Task 1: Craft Amazon Events handling code improvements plan

### Description

- Stalled entities and related code cleanup is already done as part of the prompt: context/latest-task-backup.md and done as part of current chat, now it's time to plan the coding level improvements for all the amazon events related stuff
- The improvements are already there within context/amazon-events/amazon-events-analysis.md but I want you to create a new refreshed plan not to copy from this file but take quick references and as per current coding/database state design the plan

- Primary thing is remove ambiguous code and redundancy from the code and migrate to common/unambiguous/centralized approaches.
- Follow a hierarchical structure properly in a way that a major module should be one, other sub modules come within it, and so on.
- Take inspiration from the existing modules
  - Readiness master module: api\apps\api-main\src\modules\app\catalog\products\readiness\readiness.module.ts -> there are dozens of complexity there but grouped correctly into files and everything assembles into a single readiness module
  - LLM module: api\apps\api-main\src\core\llm\llm.module.ts -> there is a very good orchestrator and factory design patterns followed to handle the working

### Context

- 

### Constraints

- The removal/cleanup is already done, you just need to refactor/optimize/migrate current code structure regarding to the amazon events and related stuff but make sure no major working breaks (in case whatever major is affected you inform me before implementation)
- Write down the plan categra-dataset\context\amazon-events, do not execute anything yet

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

## api repo only

Task 1: Refactor as suggested for issue-tickets module files

### Description

- Review files
  - Whatever singular constants are defined within the files should be moved to api\apps\api-main\src\core\constants\constants.ts -> CONSTANTS object
  - Whatever constant object or mappings are defined should be moved to either api\apps\api-main\src\core\constants\global-enums.ts or api\apps\api-main\src\core\constants\constants.ts accordingly
  - Whatever utility functions are defined should be moved to api\apps\api-main\src\common\lib\utils.ts
    - In case of funcitons which are very similar and already present into utils, you can levarage those instead of appending new functions to the file
  - Whatever types and interfaces are defined should be moved to corresponding file within api\apps\api-main\src\core\types
    - In case of new files should be created to distinguish as separate module then you are free to do so

### Context

- api\apps\api-main\src\modules\app\system\operations\issue-tickets

### Constraints

- 

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

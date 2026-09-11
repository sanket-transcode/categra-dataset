## webapp and api and db-migrations repo only

Task 1: cleanup requested things and the wherever they are used at the whole codebase and in webapp side also

### Description

- remove is_inherited from tbl_attribute_translations
- remove history module + service, product-history.service.ts and related code as history module is a legacy module not used anymore
- remove complete live-insight module from BE, its controller and service everything related to it but first ensure no route mentioned there is being used at webapp side

### Context

### Constraints

- Create migration files for them but don't actually execute them

- At any point if there is a little need for single or multiple clarifications then you must ask for it rather than implementing purely on assumptions

## webapp and api repo only

Task 1: For amazon only, while creating and editing a variant dialog should contain Condition Type, Shipping Template and Category so that values for those 3 can be chosen directly within the edit dialog

### Description

- Make sure to have a consistent UI into the edit variant dialog for amazon
  - For shipping template and condition type, it will be a dropdown just, no additional dialog triggers for them
  - For category, you'll going to open the same category dialog but you may need to change the implementation slightly because now it will no longer be an immediate update, it will be lazy update or even create variant
- Change the update variant API in a way that supports from now updating these 3 things
- Do not forgot to have the exact conditions for these 3 things have
  - ex. based on a flag we omit the Shipping template in case it is not supported
  - ex. Category has conditional selection, so respect that
  - Other cases, cover all of them that revolves around these 3
- These 3 are absolute required, so save button will be blocked due to missing values for them unless any additional flag for any of them has that says corresponding field is not required

### Context

- webapp\src\app\(index)\(menu-layout)\products\[product]\_components\variants\_components\edit-variant.tsx
- webapp\src\app\(index)\(menu-layout)\products\[product]\_components\variants\_components\nonMasterChannel\amazonChannel\amazon-variants-core.tsx

### Constraints

- Do not touch the separate value choosing system for those 3 types, this is an additional functionality and not a modification for the existing

- ask questions agreesively to make sure any tiny confusion doesn't remain at the feature level

- At any point if there is a little need for single or multiple clarifications then you must ask for it rather than implementing purely on assumptions

## webapp repo only

Task 1: Within the modify category dialog for variants only proposed changes and bug fixes

### Description

Issue:

1. Product Type Categories & Favourite Categories: Search is not working.

2. Unable to mark a category as Favourite.

3. Already selected category is not shown as selected (it should come only for non item type keyword supported categories, should come as selected from all 3 tabs).

4. Favourite Categories: Unable to select a category.

5. Sometimes Browse Amazon Categories: “Oops, something went wrong” error is displayed.

### Context

- webapp\src\app\(index)\(menu-layout)\products\[product]\_components\variants\_components\categoryEditableField.tsx

### Constraints

- Things should work the same way as webapp\src\app\(index)\(menu-layout)\products\[product]\_components\channels\channel-details\nonMasterChannels\amazonChannel\categorizeYourProductDialog\categorizeYourProductDialog.tsx (you can reference things into this component)
- But the referenced one is for selecting for parent category where as this one is for variant

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

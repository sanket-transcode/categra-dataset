## webapp repo only

Task 1: On Publish button click, for Non-amazon typed channel, keep the flow as it is but for amazon typed channel -> onclicking Publish per market or Publish All at channel level, You open a dialog which has detailed breakdown on how entities will be published followed by a confirm button that will end up calling the current publish API

### Constraints

- Take reference for the dialog overall style from webapp\src\app\(index)\(menu-layout)\products\[product]\_components\variants_components\misc-sync-baseline-reset.tsx
- The dialog content will show parts for parent and variants that if something to be fully published, only the changes to be published, for some entity they are already in sync so nothing to be published -> find the actual examples mentioned within categra-dataset\context\sync-summary\publish-messages.md
- Sync status (parent, variant) is SUCCESS = In Sync and no changes to be published, OUT_OF_SYNC = Published but exist some local changes that need to be published, neither among both of them are not published and surely be published with full payload
- If at least one parent or variant is OUT_OF_SYNC then you show something special: In a useEffect open the dialog through openSyncReviewDialog in which pass amazon marketplace id accordingly and products as {[productId]: []}, but introduce a new substate within syncReviewDialog -> hideDialogContent (default false), you pass here as true and it's side effects will run but the dialog content will not be visible, while this API is in progress you'll show the standard Loader component within our main confirmation dialog
- Modify the sync change dialog that maintain the selected changes and the items to publish within a store state that will be accessed to outside rather than local dialog component
- within confirmation dialog, you'll show exactly how many changes 4/10 to publish for parent (conditional) and 3 variants (conditional), user can modify it (provide a modify button there, clicking on it will set the flag as true hideDialogContent) so that the dialog will be visible
- modify the dialog webapp\src\components\sync-review\sync-change-review-dialog.tsx -> 1. accept a flag to distinguish between the normal behaviour and this behaviour, 2. On this behaviour you'll change the action button label from Publish Changes to something that fit in current situation, 3. In this behaviour you'll not call the API directly on the action button, rather set the flag hideDialogContent as false and don't close the dialog, then within our main dialog you'll show exactly how many changes 4/10 to publish for parent (conditional) and 3 variants (conditional), user can again modify it normally
- Then on API call (current API no new API), you'll pass additionally the following things: isIncrementalSync: true, selectedChanges (build from the selected state) -> on success you'll close both the dialogs (one current visible dialog and one another invisible sync change dialog)

- At any point if there is a strong need for single or multiple clarifications then you must ask for it rather than implementing purely on assumptions

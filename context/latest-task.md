## webapp & api & db-migrations repo only

Task 1: Generate 2 HTML files that will cover up everything that needed for action center V2

### Description

#### Core purpose
- Building action center V2, it should add 'Action Center V2' tab item into sidebar + having v2 route for that and the new screen that is inspired from the original action center will be there within v2
  - All file names, function names, variable names should contain V2 to differentiate from older one
  - Except the table entities and actual fields shoudln't be marked with v2, their names should be clear, consistent, unambiguous

- The older action center working (which was before the current command center was built) is within: categra-dataset\context\action-center\action-center-v1-explained.html
- The 3 questions solutions are mentioned within the file categra-dataset\context\action-center\action-center-v2-solutions.html

- My original suggested refinements on it:
System Level
Re-structure each issue types or sub issue types according to command center or simply re-use the same dataset for that
ex. main issue type as 'Account related trouble' and the actual sub issue maybe CONNECTION_LOST or DEACTIVATED

By default no threshold or constants should be picked from dynamic admin panel, they should be from constants
One ticket should re-present a (product x channel x marketplace (for amazon only) x issue type) that means parent + all it's variants will be within it
Issues will be purely based on external channel and not based on categra state
Remove modes: SALES_OPERATIONS, CATALOG_PREPARATION
Craft a universal consistent id format, out of that an individual work item can be identified: ex. productId:12||channelId:4||amazonMarketplaceId:2||issueType:MISSING_STOCK
Sorting should be issue type ASC -> number of variants for product DESC -> product id ASC -> channel id ASC -> amazon marketplace id ASC
Sub issue type sorting will be based on just variant ids ASC

There should be 3 statuses for priority: Red, Yellow, Blue that matches exactly with command center
The main table status will be there for managing per item status, if an item is cleared it will be deleted from it
Removed recently cleared items completely
Separate table will be there for tracking snoozed items per user 
It will be separate table for auditing the activity behind the work items (that will include major changes, ex. status, when cleared, re-generated)

UI Level
Remove periodic refresh and unnecessary full page refresh on doing micro interactions
It will be a dialog for viewing a ticket's details and not a separate page
The above bar will be very similar to Jira
Left side:
searchbar -> on product name, variant name, product/variant SKU/ASIN, 
assignee list (unassigned + team members) only if at least two users are within the account
A single filter, clicking on it will open a dialog that has mentioned all kinds of filters which includes snoozed items also
In case of any filter is applied then 'Clear filters'

Right side:
Export to CSV


Works items should be filtered with the exact filters when coming from command center
All channels or selected channel filter
All markets or selected market filter
Exact issue type filter from where you've been redirected

For each, issue type it will be a primary action decided within a primary button, clicking it will go to the correct page (ex. Open Stock)
While navigating to the corresponding page
It won't open the right side issue drawer
Create a centralized format for showing context bar
In case of variants are there, then affected variants should be highlighted and filtered


Remove option for "Mark Resolved" completely, every item will be resolved by system check
In case of only 1 user is there then the Unassigned won't be there
On clicking "Unassigned", it should open users present in team who has permissions for channel and module then assignment should work
On 3 dot of an item, the options should be: Move to this stage (provide other than current stage), Snooze for Me
Remove move to waiting reason feature completely
On i button hover at work item: it should be showing SKU + ASIN for parent and variants within the common categra tooltip
Pick snooze date component should be the same one which is within the details tab
Fixed size of each item card, fixed cap for height and width for image
Remove Resolution velocity section currently
The category dropdown and below items should be opened within a drawer the same way as the current 'Product Action Context' and the trigger should also be the same at action center page stick to center right of the page
Remove 'Saved Views' and 'Refresh Scope' buttons completely
Product Action Context drawer should be exactly compatible with the new action center, the design and functionalities shape should be inspired from the new action center, of course some of the really good things should remain or refined from the current action context drawer but definitely should be shaped accordingly to adapt simplicity and consistency in UI and functionalities as well

### Context

- categra-dataset\context\action-center\action-center-v1-explained.html
- categra-dataset\context\action-center\action-center-v2-solutions.html

### Constraints

- The older one shouldn't be removed, while V2 is in progress, it should be preserved to navigate
- All current action center tables should not be used and completely new ones should be build for that

- It should not left leftovers/confusions, those have to be cleared in between while planning

- 1st HTML file will contain the schema structure required for the newer scalable action center
- 2nd HTML will contain the actual planning for the webapp repo and api repo side, it mentiones the high level requirement followed by the actual technical requirements (not too deep as while actually building each section one by one the actual technicals will be covered)
  - It should mention at last the optimal execution steps to build the whole action center V2

- Write down the full end to end solution guide into categra-dataset\context\action-center by creating html files with beautiful dark themed view

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

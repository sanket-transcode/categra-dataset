## webapp repo only

Task 1: design the route /command-center-v2 route that will be the new command center UI, within the navbar it will be placed just below the current Command Center item

### Description

- The referenced UI is this one: categra-dataset\context\command-center\categra_command_center_full_prototype_v3.html, the visuals should look nearly same as per this but you'll prioritize current project's theme, colors, components, code structure
  - You just need to mimic the UI but the code structure and conventions will be followed as current repo
- Here are the exact sections to be crafted as part of command center V2 page
  - Top content
    - The UI topbar is not should be replicated, the very to bar having page title as "Command Center" should be exactly mimic of the current command center as this page will be replaced
    - At below it there should be a row (not sure invisible or visible BG) but at right hand side -> there will be filter, Keep both the selectors UI as consistent with the rest app interface as we have channels and marketplaces filter at various places within products pages
      - Channel filter: List of channels, first option will be “All Channels” by default
      - In case of an amazon type of channel is selected, provide list of marketplaces enabled to that channel to be chosen, by default All marketplace will be selected
  - That 1st row 6 cards from the UI
  - Business pulse cards
  - What needs attention now 5 cards
    - 5 cards
    - Sorted by severity first, issue type second
    - severity card colors
      1. Red → Because of this, I cannot sell.
      2. Yellow → I can still sell, but I cannot trust or safely operate on the data.
      3. Black → I can sell, but something needs to be fixed or completed.
  - Next best actions
  - Channel & marketplace health
  - What changed?
- Put a random delay on revealing the sections, during that time, show skeletons according to the section structures
  - Refer current command center page which has the skeletons
- Don't mimic current command center's yellow-brown theme, it must be white and with colors the other products related UI pages currently has

### Context

-

### Constraints

- Do not create other sections/cards which are not requested
- As of now things will be static, but later they will be dynamic based on APIs (that will be later on and not part of current task)

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

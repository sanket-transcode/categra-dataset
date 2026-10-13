## api & webapp & db-migrations repo only

Task 1: Provide the plan/estimate for 2 full sections (What needs attention now and Next best actions) 

### Description

- What needs attention no
  - There will be max 5 cards Sorted by severity first, issue type second
- severity card colors
    1. Red → Because of this, I cannot sell.
    2. Yellow → I can still sell, but I cannot trust or safely operate on the data.
    3. Black → I can sell, but something needs to be fixed or completed.
- All will mostly redirect to action center with the correct filter applied there
- It is fine in case some issue or pattern is not supported in action center currently, you just redirect with appropriate filters there
- I've figured out below cases that should be covered as part of these attention cards:
  - Account / Channel level health related things (group things based on major issues, ex. Connection lost and other kind of groupable issues)
  - Amazon channel marketplace level health related things
  - Missing price for published catalog
  - Missing Stock for published catalog
  - Low Stock for published catalog
  - In case more than 0 blocker listing issues (take inspiration from page's 1st row's 3rd card: Listings) -> decide how to show attention cards, based on classified issues or as a whole, which will be appropriate to showcase the seller
  - Missing main image for published catalog (a product and variant both should carry 1 main image)
    - Some may have main image but not supported type as per corresopnding channel type (so it is also counted as a trouble, you can have groupings like 5 have missing main image, 3 have unsupported main image)
  - For amazon published catalog only: based on external status: products/variants (only count sellable entities, exclude parents which have variants) whichever are not searchable (whose external_status doesn't include 'DISCOVERABLE')
  - For shopify published products which are not available for customers to show (product_channel level external_status != 'ACTIVE' so, but you count sellable entities, accumulate variants by checking the flag at parent level and for standalone count the parent product itself)
  - For those published amazon products whose product type is changed by AMAZON from what is sent from categra (those banners show at details tab of product, you can check out)
- Apart from above cases, you can count other important cases (do not put way much effort, only consider worth noticing and trackable or classified items)

- Next best actions
  - Here max 4 actions will be there, they will have no interaction at all
  - It will be related with the above attention data mostly, the underlying investigation will be similar for both the sections
  - Enrich it's UI from the original one to match with rest of command center

### Context

- command center v2

### Constraints

- Highly levarage the objective and those sections which are live into command center v2 to get better idea on what to showcase into the required sections
- I recommend gathering all attention items and action items from API side, the limit will be from frontend side so that later on in case we want to change the showing limit, we just change from the frontend side
- Decide the severities among attention items carefully, decide the priority sequence for same severity attention items and recommended actions carefully by looking at the objective of the items
- The features seems ambiguous, but try maximum to maintain a de-coupled unambiguous structure and make re-usable consistent code structure, leverage optimal design patterns and use appropriately
- Few things we don't support currently, they will be added later on so you write 'later on' items within the plan as well for future (ex. Conversion rate, Buy box lost)
- Any kind of obvious user friendly and seller important improvementes are automatically approved
- In case of any other improvements which are controversial but are genuine you can ask me before deciding
- Write the plan/estimation within an md file within categra-dataset\context\command-center

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

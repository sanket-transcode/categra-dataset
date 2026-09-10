## FE

- Show labels for those 4 criteria (they should live within those 5 languages), currently they are being showed as they are in camel case: lengthCheck

- Remove "System default" chip completely

- Remove showing dialog on clicking "Discard Changes", it should discard the changes immediately on button click, instead the dialog should be shown when someone tries to switch channel or other language, there the changes wili be lost dialog should be shown (you can alter the title and message of the dialog as per the use case)
- Altering channel or language should clear necessary previous channel+language dataset immediately and abort all running previous channel or language specific APIs (refer webapp\src\app\(index)\(menu-layout)\products\[product]\_components\details\details-tab.tsx for that)
- Cover whole module APIs with useApiRequestManager utility functions and wire those utilities around the side effected and APIs the same way as details tab
- Removed "Inheritance disabled" + "Direct rule" ships completely from the default rules section, they aren't relevant now
- Don't put static "5 rules", for amazon it has 7 default rules then still showing 5, resolve that

- Add Custom Rule dialog
  - The chips for channel and language should be next to the dialog title the same way as edit variant
  - Show the action button in secondary bg not in primary
  - All 4 filters below the searchbar, that should be of secondary color and not of primary color
  - The filter of sources should have two options only (User Attributes, Categra Generated) the same way as /attributes route page have the filter (make sure within the API the filter works correctly)
  - Hide the "All Sources" filter completely in case of selected channel is not a master channel
  - The attribute types filter should be almost the same design as route /attributes page have the types filter
  - Create a separate api in BE for getting attribute groups which have at least 1 valid associated attribute present, for selected master channel, it should show every attribute group, for a shopify channel, that domain specific groups should come for selection
  - The rule should add the rule for the selected channel (either master or specific channel) and selected language (must be exactly one) only and not more than that (do the change in BE API definition side if needed)
  - While action button is in progress, it should be opacity-60 and pointer-events-none and disabled
  - The dialog should be opened till the action button api is in progress, the refetch rules list api will show loader outside within the "Custom Rules" section
  - remove "Allow inherited/fallback values to satisfy this rule" completely, remove its usage and handling from everywhere

## BE

- Entity changes
  - remove field is_amazon_channel completely from tbl_readiness_configurations and the direct access of it also remove, in case you need to check for the channel type, you can do it by joining the channel itself
  - remove field product_channel_id completely from tbl_readiness_values, remove usage of it every where, in case product channel entity is required enough then it can be formed from product id + channel id
  - remove field count_inherited completely from tbl_readiness_configurations and entire usage of the field, within the calculation, it should basically consider what value is being presented (based on the inheritance flags for attributes, pricing -> look at the standard services for them)
  - Remove column meta_data completely from tbl_readiness_configurations, whatever is being stored currently there (cover FE also for accurate check), move those keys into configurations as sibling keys to "min" and "max"
    - For Stock and Price, currently the default rules are referencing their attribute ids, remove attaching attribute ids for those and start using a single key in configurations called "type", for that maintain an object defined in global enums: "stock" | "price" | "bullet_point" -> for stock and price migrate the affected places from attribute id to this configurations.type and for bullet points migrate from the old key "bulletPoints" from meta_data to the new configurations.type based approach

- While getting attributes for adding rule, in case there is master channel, there is no base filter you return literally all,
  - for amazon, attributes associated within groups whose type is AMAZON
  - for shopify channel, attributes for that specific channel (channel domain specific) should be returned (levarage how get attributes API is returning for a shopify channel)

- Remove usgae of static criteria everywhere (presenceCheck|lengthCheck|countCheck│valueCheck), add an object within api\apps\api-main\src\core\constants\global-enums.ts and use from that dataset

- While seeding default readiness rules where adding primary media, make sure you add default max as 10 (currently it is 9, alter that)

- Make sure while inserting/updating readiness configuration records, no unknown criteria is allowed, the criteria must be out of those 4 (the same way as in FE side also)

- Remove creating these 3 legacy primary attributes: Stock, Price, Category

- At every place where it needs to upsert records in tbl_readiness_values, create a common SQL query that upsert it based on uniqueness pairs (account id + product id + variant id nullable + channel id nullable + language code) that will be very similar query to upsertAttributeSuggestedValues, and use that query to upsert bulk records with a certain batch size

Research and To Do:

- Centralized calculation covering all rules as per new structure
- Minimal GET endpoints with meaningful and consistent route names

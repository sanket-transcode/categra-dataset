## api repo only

Task 1: Design the actual implementation plan for readiness triggers, performing scoring calculations and storing values by overriding the existing implementation wherever necessary

### Description

- From now it should have a single trigger destination where from every place, the readiness calculation request will get initiated with whatever scope(s)
  - Then the centralized trigger executor will orchestrate further process in optimal ways
- Flags to be respected before the actual execution:
  - ProductChannel -> isActive, isDeleted
  - VariantChannels -> isActive, isDeleted
  - ProductChannelMarketplace -> status, isDeleted
  - VariantMarketplace -> status, isDeleted
  - AmazonChannelMarketplace -> status
  - Product -> isDeleted
  - ProductVariant -> isDeleted
  - Channel -> isDeleted
- During the calculation, handle the following critical things with no mistake:
  - For predefined criteria name, description
    - Follow max 3 level inheritance to evaluate the correct value, check for other places where they are retrieved for products/variants
  - For predifined criteria: media
    - It also has an inheritance check flag (only check for non-master channel) to be found in ProductAttribute, if inherited then use master media otherwise channel media
  - For predifined criteria: stock
    - Investigate how stock final stock is evaluated at other places, for master, amazon and shopify
    - consider things like amazon FBA, Amazon FBM, Externally manage stock and similar things to have a final figure of stock to be checked for readiness calculation
  - For predifined criteria: price
    - You will only consider 'offerPrice' check, no other price worth to be checked
    - Respect the inheritance flag for offerPrice in case of channel, if inherited then use master value otherwise local channel value
  - For predifined criteria: bullet point
    - It will only be for amazon channel, the attribute that represents a bullet point will be named like: AM_COSTUME_OUTFIT_bullet_point_value, CONSTANTS.BULLET_POINT, CONSTANTS.VALUE, starts with AM_, in middle there is the product type name assigned at product channel level
    - It will surely be considered for calculation in case for amazon channel without checking whether attribute exist for product scope or not
  - For custom attributes
    - You'll only consider an attribute for a scope worth to be calculated for readiness is if the attribute is within the range of a product/variant + channel/marketplace
    - If an attribute is active as a variant attribute, then you won't consider it for readiness, for master: ProductVariantAttributes, for non-amazon: ProductVariantAttributesChannels, for amazon: WrapperVariantAttributes, for a variant of amazon: check the corresponding VariantMarketplace -> Wrapper -> WrapperVariantAttributes
    - For shopify, the channel specific ProductAttributeGroup, a record having attribute id must be present in ProductAttribute (variant id is null, channel id is null, language is base language), for amazon, the same product type based group must have a record present having the attribute id in ProductAttribute (variant id is null, channel id is null, language is base language), for master: check for universal channel type specific group for the product which has record present in ProductAttribute (variant id is null, channel id is null, language is base language)
    - Custom attributes are only considered for parents only in shopify, variants in shopify don't hold extra attributes
- Media is non-language bound, it is directly channel based so for every language for a channel, it's score will be the same
- The Stock and Price will be the same across every language for master channel and non-amazon channels, for amazon channel, each language is treated as a separate marketplace (PCM, VM, ACM) and both stock and price will be completely distinct for marketplaces (or languages) for amazon channel only

### Context

### Constraints

- You are fully allowed and requested to combine functions to centralize common similar operations, re-structure complex functions in structured manner, change terminologies of function/variable names to have unambiguous and simple names

- Use caching using redis only if it provides genuine benefits and realistic measured improvements and if the data could be structured well enough

- The planning file should be at the path: categra-dataset\context\readiness

- At any point if there is a little need for single or multiple clarifications then you must ask for it rather than implementing purely on assumptions


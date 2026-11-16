## webapp & api & db-migrations repo only

Task: Build an xlsx template for the task: Plan Amazon related signals and opportunities from identifying to showing into categra in a UX friendly way in order to make the solution reactive and proactive

### Description

1. Buy Box / Featured Offer Intelligence
  Featured Offer lost, at risk of being lost, or not currently won
  Identify possible reasons and recommend corrective actions
  Detect competitor price changes and pricing disadvantages
  Monitor Featured Offer eligibility and changes in competitive position

2. Listing Health & Protection
  Suppressed, inactive, stranded or unbuyable listings
  Amazon-initiated product type, category or attribute changes
  Missing or invalid product information and compliance issues
  Unexpected competing offers on listings marked as exclusive (potential hijacking)
  Listing content changes or discrepancies between Categra and Amazon
  Listing quality deterioration and emerging suppression risks

3. Inventory & Stock Intelligence
  Predict potential stockouts based on inventory and sales velocity
  Identify slow-moving and excess inventory
  Detect inventory discrepancies, stranded FBA inventory and replenishment risks
  Highlight products losing sales because of inventory availability

4. Pricing & Profitability
  Detect pricing errors, uncompetitive prices and unexpected price changes
  Identify potential margin erosion due to fees, costs or competitor pricing
  Flag pricing that may affect Featured Offer eligibility
  Recommend pricing adjustments within seller-defined limits

5. Sales & Performance Intelligence
  Detect unusual sales declines or changes in sales velocity
  Identify declining traffic, conversion rates or product performance
  Highlight products with growing demand that may need attention
  Connect sales changes to possible listing, stock or pricing issues

6. Account Health & Amazon Events
  Account health warnings, policy violations and performance risks
  Amazon notifications and operational events requiring action
  Listing restrictions, compliance deadlines and selling eligibility issues
  Failed or delayed synchronization and Amazon API errors

7. Proactive Command Center & Action Center
  Prioritize findings by severity, commercial impact and urgency
  Explain what changed, why it matters and what the seller should do
  Provide actionable recommendations rather than just reporting problems
  Create trackable actions with ownership, status and resolution history
  Support controlled execution where Amazon APIs permit it
  Verify whether corrective actions actually resolved the issue
  Avoid duplicate alerts and unnecessary notifications

For each proposed feature, I need something like this:
  Short description and business benefit
  Where it will appear in Categra (Command Center, Action Center or elsewhere)
  Required Amazon API, report or notification
  Implementation complexity and estimated development time
  Priority and proposed delivery month
  Any API limitations, dependencies or permissions required
  Whether reliably implement now or needs additional data or access

### Context

- 

### Constraints

- You can levarage already handled Amazon SP APIs, cron jobs around those APIs, notifications through /sqs and /event-bridge routes
- You can levarage how those already handled signals are crafted with UI insights and to be shown into Command Center and Action Center and other places as well
- Think extra to determine column/header/section names
- The current task includes only the template building, and not the actual research data and figures

**Before implementing, clarify any ambiguity or missing detail at the feature level. Do not make assumptions about intended behavior, even for small details, if they could affect the implementation. If there is any uncertainty or multiple reasonable interpretations, ask the user for clarification before proceeding.**

## api & if required then webapp repo only

Task 1: Do requested changes

### Description

- Remove creating these 3 legacy primary attributes: Stock, Price, Category

- Those shouldn't be treated as primary attributes from now, they are separately managed on their own entities or within supporting entities

- Remove those code and implementation wherever they are being used and treated as primary attributes

- Whichever attributes are already created for those, do not do anything for them, you just care about future so do the code level change only, no entity level or actual records deletion

- In case changes required in webapp then do that

### Context

### Constraints

- At any point if there is a strong need for single or multiple clarifications then you must ask for it rather than implementing purely on assumptions

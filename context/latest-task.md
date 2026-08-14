## api repo only

Task: endpoint /get-change-summary which is currently bringing only single product changes will start bringing multiple product changes, as result a top level grouping of product (id + other basic info)

Context:
api\apps\api-main\src\modules\app\syncBaseline\syncBaseline.controller.ts
api\apps\api-main\src\modules\app\syncBaseline\syncBaseline.service.ts -> getChangeSummary

Constraints:

- accept products having type required Record<string, number[]> instead of just product id , remove specificVariantIds in case of specific variants the corresponding product will have a variant ids array length

- As result it should return an array, each element object will have:
    productId,
    sku,
    name,
    "changeCounts": {
        "media": 1,
        "pricing": 4,
        "attributes": 0,
        "miscellaneous": 4,
        "variantStructure": 1,
        "variantAttributes": 0,
        "total": 10
    },
    data -> this will be current array of marketplace changes, no change in that, in case of non-amazon it will have a single element array having marketplace id as null

- Refer function getChangeCounts for getting per product info as described above

- you shouldn't be calling DB queries in loop for each product, imagine 6 type of baseline accumulation for 10 products meaning 60 queries to be fired, those queries should fetch all requested products (consider specific variants for specific products as per requested in body) instead of just one productId, then you can later group per product change accumulation (the whole point is the DB calls shouldn't be O(N) it should be O(1), you can decide whatever approach to achieve it)

- In between you get questions that need my clarification then I am ready for human in the loop

- Do not ask permission for terminal commands, I am granting permission for executing all commands required in this task
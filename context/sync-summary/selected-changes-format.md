## Format

AMAZON
│
└── channel
└── products[]
│
├── productId
│
└── marketplaces[]
│
├── amazonMarketplaceId
├── product
│ ├── attributes
│ ├── pricing
│ ├── media
│ └── miscTypes[]
│
└── variants[]
└── variant
├── variantId
├── attributes
├── pricing
├── media
├── miscTypes[]
├── variantStructure
└── variantAttributes

NON-AMAZON
│
└── channel
└── products[]
│
├── productId
├── product
│ ├── attributes
│ ├── pricing
│ ├── media
│ └── miscTypes[]
│
└── variants[]
└── variant
├── variantId
├── attributes
├── pricing
├── media
├── miscTypes[]
├── variantStructure
└── variantAttributes

## Examples

### Amazon Channel

```json
{
  "channelId": 7,
  "channelType": "AMAZON",
  "products": [
    {
      "productId": 101,

      "marketplaces": [
        {
          "amazonMarketplaceId": 3,

          "product": {
            "attributes": {
              "attributeIds": [12, 18],
              "hierarchyIds": [90]
            },
            "pricing": true,
            "media": true,
            "miscTypes": [
              "AMAZON_CONDITION_TYPE",
              "AMAZON_SHIPPING_TEMPLATE",
              "AMAZON_CATEGORY"
            ]
          },

          "variants": [
            {
              "variantId": 5001,

              "attributes": {
                "attributeIds": [40],
                "hierarchyIds": [77]
              },
              "pricing": true,
              "media": true,
              "miscTypes": ["AMAZON_CONDITION_TYPE", "AMAZON_CATEGORY"],
              "variantStructure": true,
              "variantAttributes": true
            },
            {
              "variantId": 5002,

              "attributes": {
                "attributeIds": [41],
                "hierarchyIds": [78]
              },
              "pricing": false,
              "media": true,
              "miscTypes": ["AMAZON_SHIPPING_TEMPLATE"],
              "variantAttributes": true
            },
            {
              "variantId": 5003,

              "attributes": {
                "attributeIds": [42]
              },
              "pricing": true,
              "media": false,
              "miscTypes": [],
              "variantAttributes": false
            }
          ]
        },

        {
          "amazonMarketplaceId": 4,

          "product": {
            "attributes": {
              "attributeIds": [12]
            },
            "pricing": true,
            "media": true,
            "miscTypes": ["AMAZON_CATEGORY"]
          },

          "variants": [
            {
              "variantId": 5001,
              "pricing": true,
              "media": true,
              "miscTypes": ["AMAZON_CATEGORY"],
              "variantStructure": true,
              "variantAttributes": true
            }
          ]
        }
      ]
    },

    {
      "productId": 102,

      "marketplaces": [
        {
          "amazonMarketplaceId": 3,

          "product": {
            "attributes": {
              "attributeIds": [15, 21],
              "hierarchyIds": [95]
            },
            "pricing": false,
            "media": true,
            "miscTypes": ["AMAZON_CATEGORY"]
          },

          "variants": [
            {
              "variantId": 6001,
              "attributes": {
                "attributeIds": [50]
              },
              "pricing": true,
              "media": true,
              "miscTypes": ["AMAZON_CONDITION_TYPE"],
              "variantAttributes": true
            },
            {
              "variantId": 6002,
              "attributes": {
                "attributeIds": [51]
              },
              "pricing": true,
              "media": false,
              "miscTypes": ["AMAZON_SHIPPING_TEMPLATE", "AMAZON_CATEGORY"],
              "variantStructure": true,
              "variantAttributes": true
            }
          ]
        }
      ]
    }
  ]
}
```

### Shopify Channel (Non-Amazon)

```json
{
  "channelId": 9,
  "channelType": "SHOPIFY",

  "products": [
    {
      "productId": 101,

      "product": {
        "attributes": {
          "attributeIds": [12, 34]
        },
        "media": true,
        "miscTypes": [
          "TAGS",
          "SHOPIFY_CATEGORY",
          "SHOPIFY_THEME_TEMPLATE",
          "SHOPIFY_EXTERNAL_PUBLICATIONS",
          "SHOPIFY_COLLECTIONS"
        ]
      },

      "variants": [
        {
          "variantId": 5001,

          "attributes": {
            "attributeIds": [40],
            "hierarchyIds": [77]
          },
          "pricing": true,
          "media": true,
          "miscTypes": ["TAGS", "SHOPIFY_CATEGORY"],
          "variantStructure": true,
          "variantAttributes": true
        },
        {
          "variantId": 5002,

          "attributes": {
            "attributeIds": [41]
          },
          "pricing": false,
          "media": true,
          "miscTypes": ["SHOPIFY_COLLECTIONS"],
          "variantStructure": true,
          "variantAttributes": true
        },
        {
          "variantId": 5003,

          "attributes": {
            "attributeIds": [42],
            "hierarchyIds": [78]
          },
          "pricing": true,
          "media": false,
          "miscTypes": ["TAGS", "SHOPIFY_THEME_TEMPLATE"],
          "variantAttributes": false
        }
      ]
    },

    {
      "productId": 102,

      "product": {
        "attributes": {
          "attributeIds": [15, 22]
        },
        "media": true,
        "miscTypes": ["SHOPIFY_CATEGORY", "SHOPIFY_COLLECTIONS"]
      },

      "variants": [
        {
          "variantId": 6001,

          "attributes": {
            "attributeIds": [50]
          },
          "pricing": true,
          "media": true,
          "miscTypes": ["TAGS"],
          "variantStructure": true,
          "variantAttributes": true
        },
        {
          "variantId": 6002,

          "attributes": {
            "attributeIds": [51]
          },
          "pricing": true,
          "media": false,
          "miscTypes": ["SHOPIFY_CATEGORY"],
          "variantStructure": false
        }
      ]
    }
  ]
}
```

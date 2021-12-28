# Dataset schema

## Basic properties

Dataset schema describes:
- Content of the dataset, i.e., the schema of objects that are allowed to be added
- Different views on how we can look at the data, AKA transformations
- Visualization of the View using predefined components (grid, table, ...), which improves the run view interface at Apify Console
  and also provides a better interface for datasets shared by Apify users

<img src="https://user-images.githubusercontent.com/594801/147474979-a224008c-8cba-43a6-8d2e-c24f6b0d5b37.png" width="500">

Basic properties:
- It's immutable
    - If you want to change the structure, then you need to create a new dataset
- It's weak
    - You can always push their additional properties, but schema will ensure that all the listed once are there with a correct type
    - This is to make actors more compatible, i.e., some actor expects dataset to contain certain fields but does not care about the additional ones

There are two ways how to create a dataset with schema:
- User can start the actor that has dataset schema linked from its
[OUTPUT_SCHEMA.json](./OUTPUT_SCHEMA.md)
- Or user can do it pragmatically via API by
  - either passing schema as payload to [create dataset](https://docs.apify.com/api#/reference/datasets/dataset-collection/create-dataset) API endpoint
  - or using the SDK:

```js
const dataset = await Apify.openDataset('my-new-dataset', { schema });
```

This also ensures that you are opening a dataset that is compatible with the actor as otherwise, you get an error:

```
Uncaught Error: Dataset schema is not compatible with a given schema
```

## Structure

```json
{
    "formatVersion": 2,
    "name": "Eshop products",
    "description": "Dataset containing the whole product catalog including prices and stock availability.",
    "fields": {
        "title": "String",  
        "priceUsd": "Number", 
        "manufacturer": "Object",
        "manufacturer.title": "String", 
        "manufacturer.url": "Number",
        "productVariants": "Array",
        "productVariants.color": "String",  
        ...
    },
    "views": {
        "overview": {
            "name": "Products overview",
            "description": "Displays only basic fields such as title and price",
            "transformation": {
                // Comma separated arrays such as "fields" and "pick" will be written as arrays
                "fields": [
                    "title",
                    "price",
                    "picture"
                ]
            },
            "visualisation": {
                "component": "grid",
                "options": { "width": 6 },
                "properties": {
                    "title": "$title",
                    // We will need some templating here
                    "description": "${price} USD",
                    "pictureUrl": "$picture"
                }
            }
        }
        "productVariants": {
            "name": "Product variants",
            "description": "Each product expanded into item per variant",
            "transformation": {
                "fields": [
                    "title",
                    "price",
                    "productVariants"
                ],
                "unwind": "productVariants"

            },
            "visualisation": {
                // Simply renders all the available fields
                "component": "table"
            }
        }
    },
}
```

## Fields

One big TODO: What schema definition are we going to support? The most powerful and standardized but
slow to write is [JSON Schema](https://json-schema.org/). We could start with
the [Easy JSON Schema](https://github.com/easy-json-schema/easy-json-schema) and add support for JSON
Schema later, but in this case, we will need another property saying what schema is used.

Here is a comparison of JSON Schema and Easy JSON Schema:

```json
{
  "id": "string",
  "*name": "string",
  "*email": "string",
  "arr": [{
    "site": "string",
    "url": "string"
  }]
}
```

```json
{
  "type": "object",
  "required": [
    "name",
    "email"
  ],
  "properties": {
    "id": {
      "type": "string"
    },
    "name": {
      "type": "string"
    },
    "email": {
      "type": "string"
    },
    "arr": {
      "type": "array",
      "items": {
        "type": "object",
        "required": [],
        "properties": {
          "site": {
            "type": "string"
          },
          "url": {
            "type": "string"
          }
        }
      }
    }
  }
}
```

## Views

Dataset view enables the user to explore certain data ways. For example the
[Google Search Scraper](https://apify.com/apify/google-search-scraper) enables the user to View 
the results in multiple ways:
- One item = one page with two embed arrays of organic results and ads
- One item = one organic results
- One item = one search result

The first View in the list is considered to be a default one.

### Views's transformation

Transformation is a combination of a 
[GET dataset items](https://docs.apify.com/api#/reference/datasets/item-collection/get-items)
API endpoint parameters. This makes View usable in both UI and API
where the users can use it to preset the parameters easily, for example:

```
https://api.apify.com/v2/datasets/[ID]/items?format=[FORMAT]&view=searchResults
```

instead of this complicated URL in the case of [Google Search Scraper](https://apify.com/apify/google-search-scraper#how-to-get-one-search-result-per-row):

```
https://api.apify.com/v2/datasets/[ID]/items?format=[FORMAT]&fields=searchQuery,organicResults&unwind=organicResults
```

And here is the description from the dataset schema:

```json
  "transformation": {
      "fields": [
          "searchQuery",
          "organicResults"
      ],
      "unwind": "organicResults"
  },
```

### View's visualization

It's a triplet of `component`, `options`, and `properties` (according to the ReactJS language) that maps dataset fields to template property names.

```
visualization: {
    component: 'grid',
    options: {
      columns: 6,
    },
    properties: {
        image: '$image.href',
        title: '$title',
        url: '$field1.field2.url,
    }
}
```

## TODOs

- Perhaps the visualization's `properties` should be called `itemProperties` as it's not the property of the whole component but one item
- JSON schema specification (full or simple) above
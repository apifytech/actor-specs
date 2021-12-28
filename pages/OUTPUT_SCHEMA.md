
# OUTPUT_SCHEMA.json

## Structure

```js
{
    "formatVersion": 1,
    "outputs": {
        // Default dataset contains all the scraped products
        "currentProducts": {
            "id": "@default",
            "type": "dataset",
            "schema": "./PRODUCTS_DATASET_SCHEMA"
        },

        // Actor uses named persistent request queue and named dataset to store all historical products
        "historicalProducts": {
            "id": "~historical-products",
            "type": "dataset",
            "schema": "./PRODUCTS_DATASET_SCHEMA"
        },
        "historicalProductsQueue": {
            "id": "~historical-products",
            "type": "requestQueue"
            // Does not enforce a schema for this storage
        },

        "productImages": {
            "type": "keyValueStore",
            "schema": "./PRODUCT_IMAGES_KEY_VALUE_STORE_SCHEMA.json"
        }

        // Live view
        "apiServer": {
            "type": "liveView",
            "schema": "TODO" // We should perhaps link a swagger file describing the API somehow?
        }
    }
}
```

## Examples of ideal actor run UI

- For the majority of actors, we want to see the dataset with new records being added in realtime
- For [Google Spreadsheet Import](https://apify.com/lukaskrivka/google-sheets), we want to first display Live View for the user to set up OAUTH, and once 
this is set up, then we want to display the log next time.
- For technical actors, it's log
- For [HTML to PDF convertor](https://apify.com/jancurn/url-to-pdf) it's a single record from key-value store
- For [Monitoring](https://apify.com/apify/monitoring-runner) it's log during the runtime and a single HTML record in an iframe in the end
- For an actor that has failed, it's the log

## How to define actor run UI

### Simple version

There will be a new tab on actor run detail for every actor with output schema called "Output".
This tab will be at the first position and displayed by default. Tab will show the following:
- Items from output schema with property `visible: true` will be rendered in the same order as they are in schema
- The live view will be displayed only when it's active
- If the dataset has more views then we should have some select or tabs to select the view

### Ideal most comprehensive state

- Default setup, i.e., what output components should be displayed at the default run tab
- Optionally, the setup for different states
- Be able to pragmatically changes this using API by actor itself

## TODOs
- We have a `title` and `description` in schemas. Perhaps it's not needed there, and we can have it just here?
- Should we enforce users to have at least one item with `visible: true`? Or what about doing it the opposite
way and letting the user choose what should not be visible?
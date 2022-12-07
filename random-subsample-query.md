# Ways to retrun a subset of the actual result set of a query

I'm looking for ways to return only parts of the hits of an Elasticsearch query, e.g. every second hit.

## Modulo Filtering on Numeric ID

If there is a (numeric?) ID field present, a filter on that field could filter out every second record.

### Example using `script` query on `id` field

This approach uses a `script` filter on a numeric `id` field. It calculates the modulus and e.g. only keeps the ones without remainder.
```
GET my-id-subsampling-test-index/_search
{
  "query": {
    "bool": {
      "filter": {
        "script": {
          "script": {
            "source": "doc['id'].value % 2 == 0"
          }
        }
      }
    }
  }
}
```
#### Pros
* works generally for every n-th record

#### Cons
* requires `id` field 
* hardcoded modulus always returns the same subset
* filters records, not hits: combined with a query may filter out results completely


## Example data

```
POST my-id-subsampling-test-index/_bulk
{ "index": {}}
{ "fruit": "Grapefruit", "id": 1}
{ "index": {}}
{ "fruit": "Jackfruit" , "id": 3}
{ "index": {}}
{ "fruit": "Apple", "id": 2}
{ "index": {}}
{ "fruit": "Banana" , "id": 4}
{ "index": {}}
{ "fruit": "Space Fruit", "id": 5}
{ "index": {}}
{ "fruit": "Cumquat" , "id": 6}
{ "index": {}}
{ "fruit": "Strawberry", "id": 7}
{ "index": {}}
{ "fruit": "Tomato" , "id": 8}
{ "index": {}}
{ "fruit": "Watermelon", "id": 9}
{ "index": {}}
{ "fruit": "Pineapple" , "id": 10}
```


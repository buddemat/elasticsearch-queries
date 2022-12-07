# Ways to retrun a subset of the actual result set of a query

I'm looking for ways to return only parts of the hits of an Elasticsearch query, e.g. every second hit. In an RMDBs, one could use a `ROW_COUNT()` or something like that, but Elasticsearch has no concept of numbered hits.

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
Note: due to the hardcoded modulus, this approach first basically filters out a set of documents (in this example those with an odd `id`). This may lead to no records being returned at all if the hits all are contained in the discarded subset (`"source": "doc['id'].value % 2 == 0"`) or all hits being returned, if they all are in the retained subset (`"source": "doc['id'].value % 2 == 1"`):
```
GET my-id-subsampling-test-index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "wildcard": {
            "fruit": {
              "value": "*fruit"
            }
          }
        }
      ], 
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
* requires continuous numeric `id` field 
* hardcoded modulus always returns the same subset
* filters records, not hits: combined with a query may filter out results completely or not at all


## Modulo Filtering on Random Number
Similar to the previous approach, but in this case, a random number in a certain range is generated and the modulus is calculated for that number.

An important aspect is, that the random number is _not_ generated once before the query is evaluated, but instead, a new random number is generated _for each document_. This makes the number of returned documents non-deterministic, i.e. in extreme cases, it might be all or none.

### Example using `script` query with `java.util.Random`

```
GET my-id-subsampling-test-index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "wildcard": {
            "fruit": {
              "value": "*fruit"
            }
          }
        }
      ],       
      "filter": {
        "script": {
          "script": {
            "source": 
            """
            java.util.Random rnd = new java.util.Random();
            // assign each document a random number 0 or 1 and only return if it is zero
            // could also be done with a larger range and modulo division 
            rnd.nextInt(2) == 0;
            """
          }
        }
      }
    }
  }
}
```
#### Pros
* requires no special field in index
* likely to return subset of queried documents  

#### Cons
* number of returned hits varies with subsequent executions of query 
* random number is generated per document: may filter out results completely or not at all 


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


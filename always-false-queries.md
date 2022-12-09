# Always False Queries
In some applications users may not be permitted to see certain result sets bases on data access policies. In the context of audit logging, it may be interesting to still log the query that the user intended to see. An easy way to do this, is modify the query with an always false condition.

In an SQL statement, one could e.g. append (or extend an existing) `WHERE` condition with `AND true = false`. In Elasticsearch there are different possibilites to achieve the same.

## Appending a `filter` with an always false condition

A boolean filter that requires the document to not have an `_id` field will be always false, since this is an internal field that Elasticsearch generates at indexing time.

### Example 

```
{
    "query":  {
        "bool": {
            "must": [{
                "match": {
                    "name": {
                        "query": "Doe"
                    }
                }
            }],
            "filter": [{
                "bool": {
                    "must_not": {
                        "exists": {
                            "field": "_id"
                        }
                    }
                }
            }]
        }
    }
}
```
## Overwriting the query with an URL query string 
TODO

## Setting `size: 0` 
TODO

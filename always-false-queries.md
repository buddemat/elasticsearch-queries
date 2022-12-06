# Always False Queries

TODO: e.g. to invalidate a query due to access restrictions while still being able to log the statement.

## Example

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

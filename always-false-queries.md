# Always False Queries
In some applications users may not be permitted to see certain result sets bases on data access policies. In the context of audit logging, it may be interesting to still log the query that the user intended to see. An easy way to do this, is modify the query with an always false condition.

In an SQL statement, one could e.g. append (or extend an existing) `WHERE` condition with `AND true = false`. In Elasticsearch there are different possibilites to achieve the same.

Depending on the approach, either the query body or the URL can be modified. The latter leaves the query body untouched, which, depending on the circumstances, could be either beneficial or not. The modyfied URL might not be part of logged data which would require changing the underlying mechanism. On the other hand, the original query in the body is invalidated without it being altered in that case.

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

An URL Lucene query string added with `?q=` overwrites the query in the body. An empty query string will always return no hits.

```
GET my-new-index/_search?q=
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "Doe"
          }
        }
      ]
    }
  }
}
```  


## Setting `size: 0` 

Another option to at least get no hits displayed is setting the parameter `size: 0`.

```
GET my-new-index/_search
{
  "size": 0,
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "Doe"
          }
        }
      ]
    }
  }
}
```

When a size is already set and that should be preserved as part of the query, additionally specifying `size=0` as URL parameter overwrites the setting.

```
GET my-new-index/_search?size=0
{
  "size": 10,
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "Doe"
          }
        }
      ]
    }
  }
}
```

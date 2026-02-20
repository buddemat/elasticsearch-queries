# Handling Elasticsearch Silent Failures / Partial Results

Elasticsearch can return incomplete results without obvious errors.
"Without obivious errors" in this context means that there may in fact be shard failures which also trigger a warning message, but the query nonetheless returns results and an application or user not checking this may receive partial results without noticing.
This document covers common failure modes, how to detect them, and remedies.

## Case 1: Queries with `sort` on aliases that group indices with mapping differences
When an alias points to multiple indices and you sort on a field that doesn't exist in all indices, you may get partial results -- or none at all.
This is a classic problem in multi-index alias situations.
The root cause is that

  - Shards where the field to `sort` on exists: Execute successfully, return results
  - Shards where the field to `sort` on is missing: FAIL and return 0 results from those shards

So in total, only results from the indices/shards where the sort field is mapped are returned.
If e.g. documents matching a query happen to live primarily in indices without the sort field mapping, this means no hits at all.
If documents are spread across multiple indices, some of which have the mapping, then those shards succeed and return hits, others fail, the query yields partial results.
Both results are wrong/incomplete.

### Example
The behavior described above can be reproduced easily.

First, create two indices with three fields each, two of which are present in both indices and one each that is unique to that respective index.
Define an alias that points to both of the created indices.
```json
# clean up, delete all previous test indices
DELETE my-test-index-*

# create index 1 with 3 fields
POST my-test-index-1/_bulk
{ "index": {}}
{ "name": "Alice", "id": 1, "hobby": "skydiving"}
{ "index": {}}
{ "name": "Bob", "id": 2, "hobby": "bobsled"}
{ "index": {}}
{ "name": "Charlie", "id": 3, "hobby": "reading"}

# create index 2 with 3 fields, 2 of which are also present in index 1, one with same data in one field
POST my-test-index-2/_bulk
{ "index": {}}
{ "name": "Charlie", "id": 4, "instrument": "guitar"}
{ "index": {}}
{ "name": "Daria", "id": 5, "instrument": "piano"}
{ "index": {}}
{ "name": "Erik", "id": 6, "instrument": "ukulele"}

# create alias that covers the two created indices
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "my-test-index-*",
        "alias": "my-test-indices"
      }
    }
  ]
}

# test: show full contents of indices
GET my-test-index-1/_search
GET my-test-index-2/_search
GET my-test-indices/_search
```

A regular `_search` query on the alias will return all documents from both indices:

```json
# query alias without sort -> returns all docs (hits: 6)
GET my-test-indices/_search
{ "size": 10000,
  "fields": [
    { "field": "name.keyword", 
      "include_unmapped": "true", 
      "format": null 
    },
    { "field": "id", 
      "include_unmapped": "true", 
      "format": null 
    },
    { "field": "hobby.keyword", 
      "include_unmapped": "true", 
      "format": null 
    },
    { "field": "instrument.keyword", 
      "include_unmapped": "true", 
      "format": null 
    }
  ]
}
```
Adding a `sort` statement on one of the exclusive fields to the above `_search` query on the alias will lead to a shard failure and the result will only return documents from the index that has the exclusive field. 
The result does show a failed shard and the info ` "reason" : { "type" : "query_shard_exception", "reason" : "No mapping found for [instrument.keyword] in order to sort on", ... }`. 
However, it still returns partial results, so if the "response health" is not checked, this may happen unnoticed.
The only circumstance such a query returns a definite error response is if _all_ shards fail.

```json
# query alias with sort -> shard failures, returns only docs from index 2 (hits: 3)
GET my-test-indices/_search
{ "size": 10000,
  "sort": [
    {
      "instrument.keyword": {
        "order": "desc"
      }
    }
  ], 
  "fields": [
    { "field": "name.keyword", 
      "include_unmapped": "true", 
      "format": null 
    },
    { "field": "id", 
      "include_unmapped": "true", 
      "format": null 
    },
    { "field": "hobby.keyword", 
      "include_unmapped": "true", 
      "format": null 
    },
    { "field": "instrument.keyword", 
      "include_unmapped": "true", 
      "format": null 
    }
  ]
}
```

Interestingly, `_search` queries with aggergation don't have this issue, since the sorting is not global but happens within the result buckets.

```json
# aggregated (grouped) query with sort on common field -> returns all docs, sorted within buckets (hits: 6)
GET my-test-indices/_search
{ "size": 0,
  "aggs": {
    "olap_buckets": {
      "composite": {
        "sources": [
          {
            "terms_name.keyword": {
              "terms": {
                "field": "name.keyword",
                "missing_bucket": true, 
                "order": "desc" 
              }
            }
          }
        ]
      }
    }
  }
}
```

This is irrespective of the `sort` field being present in all indices the alias points to or not. 
All documents are returned, no shards fail. 
Documents without the `sort` key are all aggregated in one bucket.

```json
# different aggregated (grouped) query with sort on field only in index 2 -> returns all docs, sorted within buckets (hits: 6)
GET my-test-indices/_search
{ "size": 0,
  "aggs": {
    "olap_buckets": {
      "composite": {
        "sources": [
          {
            "terms_instrument.keyword": {
              "terms": {
                "field": "instrument.keyword",
                "missing_bucket": true, 
                "order": "desc" 
              }
            }
          }
        ]
      }
    }
  }
}
```
### Remedy: `sort` with `unmapped_type`

There are several options to handle the problem described above.

On the data side, one could ensure ensure all new indices get consistent mappings
by using index templates and/or reindexing existing indices to add missing field mappings to them.
However, in some cases you do not control the cluster/mappings or there are other valid reasons against this approach.

On the query side, you can add the `unmapped_type` property.
This tells Elasticsearch: "If this field doesn't exist in an index, treat it as this type and sort those documents last (or first, depending on the optional `missing` parameter)."
This will then result in a response without shard failures based on sorting and the full number of documents in the result set.

```json
# query with sort and unmapped_type (_first or _last) -> all docs (hits: 6)
GET my-test-indices/_search
{ "size": 10000,
  "sort": [
    {
      "instrument.keyword": {
        "order": "desc",
        "unmapped_type": "keyword",
        "missing": "_first"
      }
    }
  ], 
  "fields": [
    { "field": "name.keyword", 
      "include_unmapped": "true", 
      "format": null 
    },
    { "field": "id", 
      "include_unmapped": "true", 
      "format": null 
    },
    { "field": "hobby.keyword", 
      "include_unmapped": "true", 
      "format": null 
    },
    { "field": "instrument.keyword", 
      "include_unmapped": "true", 
      "format": null 
    }
  ]
}
```


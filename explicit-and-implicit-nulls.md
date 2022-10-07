# Explicit and implicit nulls
Elasticsearch has two ways of representing `null` values: 

1. _Implicit_, i.e. by absence / non-existance of a key and 
1. _Explicit_, i.e. by defining a certain value as representing `null` in a the index mapping.

**Note:** The two are not easily combinable within one field, i.e. querying for `(not) null` is not a simple query in this case.

## Implicit `null` values
Implicit `null` values are simply keys that are not set in the document.

### Creation
Just omit the key that should e considered `null`.
#### Example
```
PUT my-null-index/_doc/3
{
}

PUT my-null-index/_doc/4
{
  "status_code": "123"
}
```

### Querying
Checking for `null` is done with an `exists` query in the implicit case. Either positive (i.e. `NOT NULL`) or, in the following negative (i.e. `NULL`):
```
GET my-null-index/_search
{
  "query": {
    "bool": {
      "must_not": {
        "exists": {
          "field": "status_code"
        }
      }
    }
  }
}
```
This will return record `3` from the example above.

## Explicit `null` values


### Index creation / mapping
Explicit `null` values are set per field when creating or modifying an index mapping by specifying the key `"null_value"`: 

```
PUT my-null-index
{
  "mappings": {
    "properties": {
      "status_code": {
        "type":       "keyword",
        "null_value": "NULL" 
      }
    }
  }
}
```
Documents that are indexed with the respective field being set to `null` will then be assigned the value of `null_value` instead.

#### Example
The following three documents have the field `status_code` set differently, once to the `null_value`, once to an empty list, and once not at all, i.e. _implicitly_ `null`.
```
PUT my-null-index/_doc/1
{
  "status_code": null
}

PUT my-null-index/_doc/2
{
  "status_code": [] 
}

PUT my-null-index/_doc/3
{ 
}
```

### Querying
A query for explicit `null` values is done with a `term` query:

```
GET my-null-index/_search
{
  "query": {
    "term": {
      "status_code": "NULL" 
    }
  }
}
```
This query to get the `null` records from the example data above will return record `1`, while paradoxically, the query for the _implicit_ `null` values above will return records `2` and `3`.

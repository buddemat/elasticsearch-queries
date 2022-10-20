# Ways to create unique ids in elasticsearch index

I'm looking for ways to auto generate unique IDs in elasticsearch indices. There are different approaches possible, each with its own set of benefits or drawbacks.

Possible characteristics:

- IDs may be numeric or not
- They may be ordered somehow or not
- Uniqueness may need to be ensured outside of elastic or not
- The approach may involve only Elasicsearch queries or additional tools (e.g. Logstash, Python, etc.)


## Update query 

The update approach always means that the id generation needs to be performed in a _separate_ step, i.e. not on indexing of a document. This can be done e.g. using `_reindex` or `_update_by_query`.

### Example using `script` query that copies elaticsearch `_id` field

This approach updates the `_source` with a new field built based on the auto-generated index field. It takes this `_id` and adds a new `id` field in the source with identical value (if it does not exist yet).
```
POST  my-id-field-test-index/_update_by_query?conflicts=proceed
{
  "script": {
    "source": "ctx._source.id = ctx._id",
    "lang": "painless"
  },
  "query": {
    "bool": {
      "must_not": {
        "exists": {
          "field": "id"
        }
      }
    }
  }
}
```
This approach does not work on indexing, since the internal `_id` field is only generated _at the very end_ of indexing and cannot be used in ingestion pipelines.

### Example using `script` query that generates id using `java.util.UUID.randomUUID()`

This approach updates the `_source` with a new field built based on UUID generation. It adds a new `uuid` field in the source with a
generated uuid.

```
POST  my-id-field-test-index/_update_by_query?conflicts=proceed
{
"script" : {
    "source": "ctx._source.uuid = java.util.UUID.randomUUID().toString()",
    "lang" : "painless"
  },
  "query": {
    "bool": {
      "must_not": {
        "exists": {
          "field": "uuid"
        }
      }
    }
  }
}
```

This approach can (slightly modified) also be used in ingestion pipelines, see below.

## ID generation in ingestion pipelines
ID generation while ingestion has the charm that it does not require an additional step. However, it limits the available approaches to what can be done with means available in elasticsearch queries.



## Example data

```
POST my-id-field-test-index/_bulk
{ "index": {}}
{ "fruit": "Strawberry", "amount": 1}
{ "index": {}}
{ "fruit": "Pineapple", "amount": 4}

POST my-id-field-test-index/_bulk
{ "index": {}}
{ "fruit": "Dragonfruit", "amount": 42}
{ "index": {}}
{ "amount": 3, "fruit": "Egg fruit" }

POST my-id-field-test-index/_bulk
{ "index": {}}
{ "fruit": "Grapefruit", "amount": 11}
{ "index": {}}
{ "fruit": "Jackfruit" , "amount": 22}

POST my-id-field-test-index/_bulk
{ "index": {}}
{ "fruit": "Whatever fruit" , "amount": 17}
{ "index": {}}
{ "fruit": "New fruit" , "amount": 122}
```
```
GET my-id-field-test-index/_search
```

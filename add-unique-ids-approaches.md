# Ways to create unique ids in elasticsearch index

I'm looking for ways to auto-generate unique IDs in elasticsearch indices. There are different approaches possible, each with its own set of benefits and drawbacks.

Possible characteristics:

- IDs may be numeric or not
- They may be ordered somehow or not
- Uniqueness may need to be ensured outside of elastic or not
- The approach may involve only Elasicsearch queries or additional tools (e.g. Logstash, Python, etc.)

Note: Read [this post](https://www.elastic.co/blog/efficient-duplicate-prevention-for-event-based-data-in-elasticsearch) before attempting to overwrite the native elasticsearch `_id` field. Specifically: 

> When Elasticsearch is allowed to assign the document identifier at indexing time, it can perform optimizations as it knows the generated identifier can not already exist in the index. This improves indexing performance. For identifiers generated externally and passed in with the document, Elasticsearch must treat this as a potential update and check whether the document identifier already exists in existing index segments, which requires additional work and therefore is slower. 

## Update query 

The update approach always means that the id generation needs to be performed in a _separate_ step, i.e. not on indexing of a document. This can be done e.g. using `_reindex` or `_update_by_query`.

### Example using `script` query that copies elaticsearch `_id` field

This approach updates the `_source` with a new field built based on the auto-generated index field. It takes this `_id` and adds a new `id` field in the source with identical value (if it does not exist yet).
```
POST  my-id-field-test-index/_update_by_query?conflicts=proceed&wait_for_completion=false&refresh=true
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
#### Pros
* the uniqueness of the generated `id` is *guaranteed* by Elasticsearch
* internal `_id` and `id` in source are identical

#### Cons
* this approach can not work while initially indexing a document, since the internal `_id` field is only generated _at the very end_ of indexing and could be used in ingestion pipelines
* since it can only be user in an `_reindex` or `_update_by_query` fashion, this is an extra step that is required to generate an `id` for existing documents


### Example using `script` query that generates id using `java.util.UUID.randomUUID()`

This approach updates the `_source` with a new field built based on UUID generation. It adds a new `uuid` field in the source with a
generated [version 4 (i.e. random type)](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random)) uuid.

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
#### Pros
* the chance of collisions [is _statistically_ very low](https://stackoverflow.com/a/20999821/14015737), so the uniqueness of the generated `uuid` is _likely_
* this approach can (slightly modified) also be used in ingestion pipelines, see below

#### Cons
* uniqeness is not _guaranteed_, only _statistically_ likely
* when used in an `_update_by_query` fashion, this is an extra step that is required to generate a `uuid` for existing documents


## ID generation in ingest pipelines
ID generation while ingestion has the charm that it does not require an additional step. However, it limits the available approaches to what can be done with means available in elasticsearch queries.

### Example using a painless `script` that generates ids using `java.util.UUID.randomUUID()`
The following approach defines a pipeline that uses a similar script as above to generate a `uuid` field with a [version 4 (i.e. random type)](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random)) uuid and attach it to incoming documents.

#### Simulate pipeline

```
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "processors": [
      {
        "script": {
          "description": "Generate (version 4, i.e. random) UUID and store in 'uuid' field.",
          "lang": "painless",
          "source": """
            ctx['uuid'] = java.util.UUID.randomUUID().toString()
          """,
          "params": {
            "delimiter": "-",
            "position": 1
          }
        }
      }
    ]
  },
  "docs": [
    {
      "_source": {
        "fruit": "banana",
        "amount": 10
      }
    }
  ]
}
```
#### Actually create pipeline and post data using it
```
PUT _ingest/pipeline/my-pipeline
{
  "description": "Generate UUID.",
  "processors": [
      {
        "script": {
          "description": "Generate (version 4, i.e. random) UUID and store in 'uuid' field.",
          "lang": "painless",
          "source": """
            ctx['uuid'] = java.util.UUID.randomUUID().toString()
          """,
          "params": {
            "delimiter": "-",
            "position": 1
          }
        }
      }
    ]
}

POST my-id-field-test-index/_bulk?pipeline=my-pipeline
{ "index": {}}
{ "fruit": "Whole new fruit" , "amount": 31}
{ "index": {}}
{ "fruit": "Never seen before fruit" , "amount": 67}
```
#### Pros
* the chance of collisions [is _statistically_ very low](https://stackoverflow.com/a/20999821/14015737), so the uniqueness of the generated `uuid` is *likely*
* the `uuid` is generated on indexing, no extra step is required

#### Cons
* uniqeness is not _guaranteed_, only _statistically_ likely


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

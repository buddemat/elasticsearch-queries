# Ways to create unique ids in elasticsearch index

I'm looking for ways to auto generate unique IDs in elasticsearch indices. There are different approaches possible, each with its own set of benefits or drawbacks.

Possible characteristics:

- IDs may be numeric or not
- Uniqueness may need to be ensured outside of elastic or not
- The approach may involve only Elasicsearch or additional tools (e.g. Logstash, Python, etc.)


## Update query 

The update approach always means that 

This approach updates the `_source` with a new field built based on the auto-generated index field. It takes this `_id` and adds a new `id` field in the source with identical value (if it does not exist yet).

### Example using `script` query
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

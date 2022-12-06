#

### Example



<table>
   <tr><th>Table <em>categories</em> </th><th>Table <em>items</em></th></tr>
<tr><td>

| id | category  |
|----|-----------|
| 1  | food      |
| 2  | furniture |
| 3  | movies    | 


</td><td>   
   
| item                  | cat_id |
|-----------------------|--------|
| spaghetti             | 1      |
| Die Hard              | 3      |
| Bridget Jones' Diary  | 3      |
| sofa                  | 2      |
| kitchen table         | 2      |

</td></tr> </table>   
   
1. Generate test data: create two indices, `items-source-index` and `categories-source-index`.

   ```
   POST _bulk
   { "index": { "_index" : "categories-source-index" } }
   { "id": 1, "category": "food"}
   { "index": { "_index" : "categories-source-index" } }
   { "id": 2, "category": "furniture"}
   { "index": { "_index" : "categories-source-index" } }
   { "id": 3, "category": "movies"}
   { "index": { "_index" : "items-source-index" } }
   { "item": "spaghetti", "cat_id": 1 }
   { "index": { "_index" : "items-source-index" } }
   { "item": "Die Hard", "cat_id": 3 }
   { "index": { "_index" : "items-source-index" } }
   { "item": "Bridget Jones' Diary", "cat_id": 3 }
   { "index": { "_index" : "items-source-index" } }
   { "item": "sofa", "cat_id": 2 }
   { "index": { "_index" : "items-source-index" } }
   { "item": "kitchen table", "cat_id": 2 }
   ```

1. Define `_enrich` policy

   ``` 
   PUT _enrich/policy/my-cat-enrichment
   {
     "match": {
       "indices": "categories-source-index",
       "match_field": "id",
       "enrich_fields": ["category"]
     }
   }
   ```

1. Execute `_enrich` policy

   ``` 
   POST _enrich/policy/my-cat-enrichment/_execute
   ```

1. Define `_ingest` pipeline

   ``` 
   PUT _ingest/pipeline/my-categorization-enrichment-pip
   {
     "processors": [
       {
         "enrich": {
           "policy_name": "my-cat-enrichment",
           "field": "cat_id",
           "target_field": "category"
         }
       },  
     ] 
   }
   ```

   If you do not want the result of the enrichment to be in a hierarchical field and/or do not want the original `match_field` as part of your data, you can use additional processors in the `_ingest` pipeline:

   ``` 
   PUT _ingest/pipeline/my-categorization-enrichment-pip
   {
     "processors": [
       {
         "enrich": {
           "policy_name": "my-cat-enrichment",
           "field": "cat_id",
           "target_field": "category"
         }
       },
       {
         "script": {
           "lang": "painless",
             "source": """
               ctx['categoryname'] = ctx['category']['category']
             """,
             "params": {
               "delimiter": "-",
               "position": 1
             }
         }
       },
       {
         "remove": {
           "field": "category"
         }
       },
       {
         "remove": {
           "field": "cat_id"
         }
       }
     ]
   }
   ```

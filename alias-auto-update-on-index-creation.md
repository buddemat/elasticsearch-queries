# Add newly created index to alias automatically based on index pattern

## Problem

Rolling indexes should be automatically added to alias with wildcard, but defining alias with wildcard only adds _existing_ indices.

### Example indices

```
POST my-index-2022-10-18/_bulk
{ "index": {}}
{ "fruit": "Strawberry", "amount": 1}
{ "index": {}}
{ "fruit": "Pineapple", "amount": 4}
{ "index": {}}
{ "fruit": "Dragonfruit", "amount": 42}
{ "index": {}}
{ "amount": 3, "fruit": "Egg fruit" }
{ "index": {}}
{ "fruit": "Grapefruit", "amount": 11}
{ "index": {}}
{ "fruit": "Jackfruit" , "amount": 22}

POST my-index-2022-10-19/_bulk
{ "index": {}}
{ "fruit": "Kaffir fruit" , "amount": 45}
{ "index": {}}
{ "fruit": "Miracle fruit" , "amount": 2}
{ "index": {}}
{ "fruit": "Passionfruit" , "amount": 3}
{ "index": {}}
{ "fruit": "Coco plum fruit", "amount": 7}
{ "index": {}}
{ "fruit": "Plum" , "amount": 21}
{ "index": {}}
{ "fruit": "Capital Fruit" , "amount": 10}
```
If now, an alias is defined like this:

```
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "my-index-*",
        "alias": "my-alias"
      }
    }
  ]
}
```
it will correctly point to the indices created above. However, if now, another index is created 

```
POST my-index-2022-10-20/_bulk
{ "index": {}}
{ "fruit": "Whatever fruit" , "amount": 4}
{ "index": {}}
{ "fruit": "New fruit" , "amount": 2022}
```
it will not be "picked up" by the alias, since the wildcard is "unrolled". 


## Solution

Create an index template

### Example 

After creating an index template like below, new indices that match the pattern will automatically be assigned the alias:

```
POST my-index-2022-09-21/_bulk
{ "index": {}}
{ "fruit": "Totally new fruit" , "amount": 4}
{ "index": {}}
{ "fruit": "Other fruit" , "amount": 2022}
```


#### Elasticsearch >= 7.8

From Elasticsearch 7.8 upwards, [composable index templates](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/index-templates.html) have been introduced, and the previous template syntax is deprecated (at least for index templates):

> #! Legacy index templates are deprecated in favor of composable templates.  
> #! Deprecated field [template] used, replaced by [index_patterns]

The syntax to create an index template (by the name of `my-index-template`) that automatically adds the alias `my-alias` to new indices that match the pattern `my-index-*` would be:

```elastic
PUT _index_template/my-index-template
{
  "index_patterns": ["my-index-*"],
  "template": {
    "aliases": {
      "my-alias": { }
    }
  }
}
```

As with the previous syntax, the behavior is identical: whenever a new index matches the pattern (or *patterns*, since it can be multiple ones) specified in the `index_patterns` parameter, the template will automatically be applied to it.

Besides adding an alias, composable index templates can be used for other operations, among them defining settings or mappings, just as in the past.

---

#### Elasticsearch < 7.8

For completeness sake, as already described in existing answers, the syntax before there were composable index templates was:
```
PUT /_template/my-index-template
{
    "template" : "my-index-*",
    "aliases" : {
        "my-alias" : { }
    }
}
```

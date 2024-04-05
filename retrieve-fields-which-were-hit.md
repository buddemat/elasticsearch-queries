# Ways to find which fields were hit by query

I'm looking for ways to find which fields were 'hit' by a `_search` query, ideally without returning the content of those fields.

## Example data

A quick test index to illustrate the following

```json
POST my_test_index/_bulk
{ "index": {} }
{ "name": "Jack Black", "color": "black", "description": "This is a very long text about something BLACK and BLACK and very BLACK which I only want to retrieve in a second query in case I have a hit here..." }
{ "index": {} }
{ "name": "Roy Rainbow", "color": "black", "description": "This is a very long text about something GREEN and RED and BLUE which I only want to retrieve in a second query in case I have a hit here..." }
{ "index": {} }
{ "name": "Jose Nero", "color": "black", "description": "This is a very long text about something BLACK and WHITE and BLACK which I only want to retrieve in a second query in case I have a hit here..." }
```
This index contains three documents, which have the search term `black` in different quantities in one or more of their three fields.

## Approach using `explain` API

Using the parameter `?explain=true`, one can get an `_explanation` section as part of the response, which will contain details on how the score was calculated. From that, one could in theory parse the fields that were hit. 

```json
GET my_test_index/_search?explain=true
{
  "query": {
    "query_string": {
      "query": "black",
      "fields": ["*"]
    }
  },
  "_source": false
}
```

The response to this consists of a detailed description of the components of each of the documents' scores, among them a `description` line that, for each field that contains a search hit, looks something like
```json
              "description" : "weight(color:black in 0) [PerFieldSimilarity], result of:",
```
From this, the field names could be extracted, e.g. using an appropriate regex with a capture group such as 
```regex
\s*\"description\" : "weight\((.*?):\S* in [0-9]+\) \[PerFieldSimilarity], result of:\",
```
This approach seems quite convoluted and potentially error prone, though.

## Approach using `highlight`

```json
GET my_test_index/_search
{
  "query": {
    "query_string": {
      "query": "black",
      "fields": ["*"]
    }
  },
  "_source":  false, 
  "highlight" : {
      "fields" : {
        "*" : {}
      }
    }
  }
}
```

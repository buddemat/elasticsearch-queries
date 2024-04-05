# Ways to find which fields were hit by query

I'm looking for ways to find which fields were 'hit' by a `_search` query. Ideally, this should work without returning the actual content of those fields, since some of them are potentially very long and I'd like to minimize traffic and only retrieve them on demand and if they contain a search hit.

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
\s*\"description\" : "weight\((\S+?):\S* in [0-9]+\) \[PerFieldSimilarity],.*
```
Since the search term should be known, the `\S*` part could also be substituted by the search term, at least if it is only a simple phrase. In case a more complex search term is used, such as `"(black OR blu~)"`, inserting the search term into the regex will not work. 

Overall, this approach seems quite convoluted and potentially error prone, though.

## Approach using `highlight`

Using _highlighters_ allows getting highlighted snippets within the search results to visually show where the query matches are. Requesting this is done by adding a `highlight` section to the query and will result in an according `highlight` section as part of the response.

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
```

This `highlight` section will by default only contain the fields where the query matches:

```json
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 0.9808291,
    "hits" : [
      {
        "_index" : "my_test_index",
        "_type" : "_doc",
        "_id" : "iTREqY4BvdRN5nPl3vyd",
        "_score" : 0.9808291,
        "highlight" : {
          "color" : [
            "<em>black</em>"
          ],
          "color.keyword" : [
            "<em>black</em>"
          ],
          "name" : [
            "Jack <em>Black</em>"
          ],
          "description" : [
            "This is a very long text about something <em>BLACK</em> and <em>BLACK</em> and very <em>BLACK</em> which I only want to retrieve"
          ]
        }
      },
      {
        "_index" : "my_test_index",
        "_type" : "_doc",
        "_id" : "izREqY4BvdRN5nPl3vyd",
        "_score" : 0.64825857,
        "highlight" : {
          "color" : [
            "<em>black</em>"
          ],
          "color.keyword" : [
            "<em>black</em>"
          ],
          "description" : [
            "This is a very long text about something <em>BLACK</em> and WHITE and <em>BLACK</em> which I only want to retrieve in a"
          ]
        }
      },
      {
        "_index" : "my_test_index",
        "_type" : "_doc",
        "_id" : "ijREqY4BvdRN5nPl3vyd",
        "_score" : 0.13353139,
        "highlight" : {
          "color" : [
            "<em>black</em>"
          ],
          "color.keyword" : [
            "<em>black</em>"
          ]
        }
      }
    ]
  }
}
```
However, this response still contains the full contents of the respective fields, which is what I was trying to avoid. A possibility to minimize the content while still retaining the information on which fields were hit can be achieved by using the `highlight` parameters `fragment_size` and `number_of_fragments`. The former defines the length of the returned fragments in characters (its default value is 100), the latter defines the maximum number of fragments that the highlighter should return. Setting both of them to `1` leads to a more or less minimal response that only contains the highlighted search term once for each field:
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
      "*" : {"fragment_size" : 1, "number_of_fragments" : 1}
    }
  }
}
```
Response:
```json
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 0.9808291,
    "hits" : [
      {
        "_index" : "my_test_index",
        "_type" : "_doc",
        "_id" : "iTREqY4BvdRN5nPl3vyd",
        "_score" : 0.9808291,
        "highlight" : {
          "color" : [
            "<em>black</em>"
          ],
          "color.keyword" : [
            "<em>black</em>"
          ],
          "name" : [
            "<em>Black</em>"
          ],
          "description" : [
            "<em>BLACK</em>"
          ]
        }
      },
      {
        "_index" : "my_test_index",
        "_type" : "_doc",
        "_id" : "izREqY4BvdRN5nPl3vyd",
        "_score" : 0.64825857,
        "highlight" : {
          "color" : [
            "<em>black</em>"
          ],
          "color.keyword" : [
            "<em>black</em>"
          ],
          "description" : [
            "<em>BLACK</em>"
          ]
        }
      },
      {
        "_index" : "my_test_index",
        "_type" : "_doc",
        "_id" : "ijREqY4BvdRN5nPl3vyd",
        "_score" : 0.13353139,
        "highlight" : {
          "color" : [
            "<em>black</em>"
          ],
          "color.keyword" : [
            "<em>black</em>"
          ]
        }
      }
    ]
  }
}
```
The length of the response could marginally be reduced further by setting the highlighting tags to an empty string, which will deliver basically the same response as above, but without the sourrounding `<em>` tags:
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
      "*" : {"fragment_size" : 1, "number_of_fragments" : 1}
    },
    "pre_tags": "",
    "post_tags": ""
  }
}
```



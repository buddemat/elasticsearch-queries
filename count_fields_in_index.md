# How to get the number of fields in an index

There is no soleley ES internal way to query this. However, there are different approaches to get (approximate or exact) counts with the help of other tools (e.g. `jq`, or `python`).
Most of this is taken [from this SO post and its answers](https://stackoverflow.com/q/40586020/14015737).

## Challenges

- Counting the occurrence of `type` may lead to wrong results for multi-type fields, e.g. `text/keyword`.
- Counting the occurrence of `type` may lead to wrong results for fields with the name `type` or `type` as part of the name, e.g. `content_type`.

## Examples

### Using `curl`, `grep` and `wc` 
From [this SO anwser](https://stackoverflow.com/a/40655667/14015737).

```
curl -s -XGET "https://<user>:<pass>@<elastic.url>:<port>/<index_name>/_mapping?pretty | grep '"type"' | wc -l
```

Pros:
- one-liner from shell
- `grep`, `wc`, and `curl` generally widely available

Cons:
- counting `type` occurences may count fields twice or even more (see challenges above)

### Using `curl` and `jq`

Using `curl`, the ES `_field_caps` endpoint and `jq` lets us count the fields. From [this SO answer](https://stackoverflow.com/a/54218379/14015737).

```
curl -s -XGET "https://<user>:<pass>@<elastic.url>:<port>/<index_name>/_field_caps?fields=*" | jq '.fields|length'       
```

Pros:
- one-liner from shell
- `jq` and `curl` generally widely available

Cons:
- counting `_fields.length` is not enough to know the number of fields, as there can be sub-fields as well?

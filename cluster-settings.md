
### Disable expensive queries

```
PUT _cluster/settings
{
  "transient": {
	"search.allow_expensive_queries": "false"
  }
}
```

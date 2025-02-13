# Create User and Role via API
Goal is to create a read only user quickly to test different permission setups.

## Elasticsearch

TODO

## OpenSearch

Add a read-only OpenSearch user with 
* only data `read` and `indices:admin/get` permissions on the index pattern `test-read*` 

```
PUT _plugins/_security/api/roles/USER_RO
{
  "cluster_permissions": [
  ],
  "index_permissions": [{
    "index_patterns": [
      "test-read*"
    ],
    "dls": "",
    "fls": [],
    "masked_fields": [],
    "allowed_actions": [
      "read",
      "indices:admin/get"
    ]
  }],
  "tenant_permissions": [{
    "tenant_patterns": [
    ],
    "allowed_actions": [
    ]
  }]
}
```

``` 
PUT _plugins/_security/api/internalusers/USER_RO
{
  "password": "whatever",
  "opendistro_security_roles": ["USER_RO"],
  "backend_roles": ["USER_RO"],
  "attributes": {
  }
}
``` 

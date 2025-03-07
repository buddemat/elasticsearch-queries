# Create User and Role via API
Goal is to create a read only user quickly to test different permission setups.

## Elasticsearch

TODO

## OpenSearch

Add a read-only OpenSearch user with 
* only data `read` and `indices:admin/get` permissions on the index pattern `test-read*` 

1. First add a new role

    ```
    PUT _plugins/_security/api/roles/ROLE_RO
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

1. Then add a new user and assign the created role

    ``` 
    PUT _plugins/_security/api/internalusers/USER_RO
    {
      "password": "whatever",
      "opendistro_security_roles": ["ROLE_RO"],
      "backend_roles": ["ROLE_RO"],
      "attributes": {
      }
    }
    ``` 

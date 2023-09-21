# Dumping and restoring ES data
Different ways of dumping and/or restoring data from/to an Elasticsearch cluster and their pros and cons.

| Tool      | Description | Advantages | Disadvantages              |
|-----------|-------------|------------|------------------------------|
| filebeats |...          | <ul><li>simple</li></ul>   | <ul><li>no dumping, only restoring</li><li>requires `docker` image</li></ul>  |


## Filebeat

Restoring a file based dump can easily be done using `filebeat`. If the documents are stored as newline-delimited JSON, the following `filebeat.yml` will configre filebeat to parse those files according to the input pattern and store all documents in a single index:

```
filebeat.inputs:
- type: filestream
  paths:
    - /input/*.json
  parsers:
    - ndjson:
      overwrite_keys: true 
      expand_keys: true

processors:
  - rename:
      fields:
        - from: "source"
          to: "orig_source"
        - from: "client"
          to: "orig_client"
      ignore_missing: true
      fail_on_error: true

output.elasticsearch:
  hosts: https://localhost:9200
  username: 'elastic'
  password: 'elastic'
  bulk_max_size: 0
  compression_level: 9
```

In order to use this configuration in the docker container, we need to adjust the file's permissions as follows, otherwise an error will occur (`Exiting: error loading config file: config file ("filebeat.yml") can only be writable by the owner but the permissions are "-rw-rw-r--"`):
```
chmod go-w filebeat.yml
```

This configuration can then either be executed by building an according Docker image with the YAML and the data inside, or simply directly from the command line (substitute file paths and version accordingly):

```
docker  run --rm --name filebeat -v $PWD/source_folder:/input -v $PWD/filebeat.yml:/usr/share/filebeat/filebeat.yml docker.elastic.co/beats/filebeat:7.17.3
```
:bangbang: The above 'filebeat.yml' will result in the data being stored according to the default index naming pattern, e.g. 
``` 
filebeat-7.17.3-2023.09.21-000001
``` 

To supply a custom index name, three additional steps are required:

1. The target index name or pattern needs to be added under `output.elasticsearch.index` and 
2. The options `setup.template.name` and `setup.template.pattern` have to be set if the index name is modified
3. ILM needs to be disabled or, alternatively `setup.ilm.policy_name` and `setup.ilm.rollover_alias` need to be set. Since for the simple case of importing a dump, ILM is probably not needed, the former will do.

Overall the yaml needs the following additional entries:
```
output:
  elasticsearch:
    index: "your-index-%{[agent.version]}-%{+yyyy.MM.dd}"
setup:
  template:
    name: 'your-index'
    pattern: 'your-index-*'
    enabled: false
  ilm: 
    enabled: false
```

:bangbang: The above config will make Elasticsearch 'guess' the mapping. To provide a mapping to filebeat, `setup.template.fields` can be set to point to a YAML file containing the fields.

For a reference of what should be in `your-index-fields.yml`, take a look at the `fields.yml` that comes installed in `/etc/filebeat/fields.yml` (from: https://discuss.elastic.co/t/filebeat-to-elasticsearch-configuration-mapping/271031).

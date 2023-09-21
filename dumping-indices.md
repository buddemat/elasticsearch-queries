# Dumping and restoring ES data
Different ways of dumping and/or restoring data from/to an Elasticsearch cluster and their pros and cons.

| Tool      | Description | Advantages | Disadvantages              |
|-----------|-------------|------------|------------------------------|
| filebeats |...          | <ul><li>simple</li></ul>   | <ul><li>no dumping, only restoring</li><li>requires `docker` image</li></ul>  |


## Filebeats

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
Note: The above 'filebeat.yml' will result in the data being stored according to the default index naming pattern, e.g. 
``` 
filebeat-7.17.3-2023.09.21-000001
``` 

To supply a custom index name, additional options are required.

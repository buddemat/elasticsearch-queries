# Dumping and restoring ES data
Different ways of dumping and/or restoring data from/to an Elasticsearch cluster and their pros and cons.

| Tool         | Description | Advantages | Disadvantages              |
|--------------|-------------|------------|------------------------------|
| elasticdump  |Command line tool, Open Source<br>https://github.com/elasticsearch-dump/elasticsearch-dump | <ul><li>simple / low-code</li><li>can also dump `mapping` and `analyzer`</li><li>also allows direct "dumping" from one ES cluster into another</li><li>install via npm</li><li>also supports OpenSearch</li><li>Can also be used to easily construct docker images with data in them, e.g. vor test / QA deployments.</li></ul>  | <ul><li>requires the dump to have been created with `elasticdump` in the first place</li></ul>  |
| logstash     |...          | <ul><li>processing and tranformation possible</li><li>supports sources and destinations other than files and Elasticsearch, e.g. RDBMs</li></ul>   | <ul><li>requires `docker` image</li><li>Designed for log ingestion / continuous ETL, so does not terminate when finished</li></ul>  |
| filebeat     |...          | <ul><li>processing and tranformation possible</li></ul>   | <ul><li>no dumping, only restoring</li><li>requires `docker` image</li><li>custom/complex mapping not straightforward to integrate</li><li>may add additional fields (e.g. `@timestamp`)</li><li>Designed for log ingestion / continuous ETL, so does not terminate when finished</li></ul>  |


## Logstash

Ingesting using logstash is controlled via `logstash.conf`:

```
input {                                       
    file {                                    
        codec => json                         
        path => ["/input/test_data.json"]     
        sincedb_path => [ "/dev/null" ]       
        start_position => "beginning"         
        max_open_files => 1                   
        close_older => "5 sec"                
    }                                         
}                                             
output {                                                             
    elasticsearch {                                                  
        hosts => ["https://localhost:9200"]
        ssl => true                                                  
        ssl_certificate_verification => false                        
        user => "elastic"                                            
        password => "password"                               
        index => "test_logstash"                                      
    }                                                                
    stdout { codec => rubydebug }                                    
}                                                                    
```
:bangbang: As codec in the `logstash.conf`, `json` must be used rather than `json_lines`. The latter is for _streamed_ JSON that is newline delimited, _not_ for files.


To execute, run a docker container with the configuration above
```
docker run --rm --name logstash -v $PWD:/input -v $PWD/logstash.conf:/usr/share/logstash/pipeline/logstash.conf docker.elastic.co/logstash/logstash:7.17.3
```

#### Remove extra fields

Similar to Filebeat, Logstash adds extra fields to the ingested documents. To remove those, add a `mutate` filter:

```
filter {                                                                        
    mutate {                                                                    
        remove_field => [ "host", "path", "@version", "@timestamp" ]            
    }                                                                           
}                                                                               
```
:bangbang: if your source data contains one of these fields, you should _not_ drop it.

The same can be achieved with a `prune` filter, whitelisting the fields to keep instead of specifying those to drop. This may become infeasible when the documents have a lot of fields.

```
filter {                                                                       
    prune {                                                                    
        interpolate => true                                                    
        whitelist_names => ["fieldtokeep1","fieldtokeep2"]
    }                                                                          
}                                                                              
```


#### Use custom mapping

To suppy a custom mapping to logstash, an index template can be created:

```
        template => "/mapping/mapping.json"   
        template_name => "test"               
        template_overwrite => true            
```

The index template file `mapping.json` must contain an `index_patterns` definition that matches the index name:
```
{
  "index_patterns": ["test_logstash"],
    "mappings" : {
      "properties" : {
        "name" : {
          "type" : "text"
        },
        "weirddate" : {
          "type" : "date",
          "format" : "yyyy.MM.dd"
        }
      }
    }
}
```

The mapping file also needs to be mounted into the docker container of course:
```
docker run --rm --name logstash -v $PWD:/input -v $PWD/logstash.conf:/usr/share/logstash/pipeline/logstash.conf -v /$PWD/mapping.json:/mapping/mapping.json docker.elastic.co/logstash/logstash:7.17.3
```

## Filebeat

Restoring a file based dump can (relatively) easily be done using `filebeat`. If the documents are stored as newline-delimited JSON, the following `filebeat.yml` will configre filebeat to parse those files according to the input pattern and store all documents in a single index:

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
  - drop_fields:                                        
      fields: ["input", "agent", "ecs", "host", "log"]  

output.elasticsearch:
  hosts: https://localhost:9200
  username: 'elastic'
  password: 'elastic'
  bulk_max_size: 0
  compression_level: 9
```
Since Filebeat adds custom fields, we drop those. The `@timestamp` field cannot be dropped.
:bangbang: if your source data contains one of these fields, you should _not_ drop it, as filebeat will not overwrite existing fields.

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

#### Use custom target index name
To supply a custom index name, three additional steps are required:

1. The target index name or pattern needs to be added under `output.elasticsearch.index` and 
2. The options `setup.template.name` and `setup.template.pattern` have to be set if the index name is modified
3. ILM needs to be disabled or, alternatively `setup.ilm.policy_name` and `setup.ilm.rollover_alias` need to be set. Since for the simple case of importing a dump, ILM is probably not needed, the former will do.

Overall the yaml needs the following additional/changed entries:
```
output:
  elasticsearch:
    index: "your-index-%{[agent.version]}-%{+yyyy.MM.dd}"
setup:
  ilm: 
    enabled: false
  template:
    name: 'your-index'
    pattern: 'your-index-*'
    enabled: false
```

#### Use custom mapping
:bangbang: The above config will make Elasticsearch 'guess' the mapping. To provide a mapping to filebeat, there are different options. 

1. One straight-forward option is to create the empty index with the desired mapping beforehand, e.g. via `curl` or the Kibana Dev Console. This would be a separate step though that needs to be done independently from filebeat.

1. Alternatively, `setup.template.append_fields` can be used to specify fields that should be overwritten. Change `setupt.template.enabled` to `true`:
   ```
   setup:
     template:
       enabled: true
       overwrite: true
       append_fields:
       - name: 'fieldname'
         type: keyword
   ```
   
   However, this has some limitations. It uses the standard dynamic template for all non-specified fields (i.e. mapping all strings to `keyword`) and does not support special mapping options, such as non-standard date formats for date type fields (although the [documentation auggests this should be possible](https://www.elastic.co/guide/en/beats/devguide/7.17/event-fields-yml.html). Also, since date detection is disabled for the standard template, all date fields must be explitly set, or they become `keyword` type fields as well.


1. `setup.template.fields` can be set to point to a YAML file containing the fields.

   ```
   setup:
     template:
    overwrite: true                                
    fields: "/usr/share/filebeat/custom_fields.yml"  
   ```

   The `custom_fields.yml` should contain the field definitions:

   ```
   - key: test                   
     title: test                 
     description: >              
       Custom fields.            
     fields:                     
       - name: 'name'       
         type: text         
         multi_fields:      
           - name: keyword  
             type: keyword  
       - name: 'email'      
         type: keyword      
       - name: 'birthday'   
         type: date         
   ```
                              
   The limitations are similar to above, except the standard fields are not automatically part of the mapping (since a custom `fields.yml` is used). But again, date detection is disabled by default, so you need to explicitly list all fields, and again, non-standard date formats cannot be specified as part of the mapping. See below for a workaround.                           

   For a reference of what should be in `custom_fields.yml`, take a look at the `fields.yml` that comes installed in `/etc/filebeat/fields.yml` (from: https://discuss.elastic.co/t/filebeat-to-elasticsearch-configuration-mapping/271031).

   The `custom_fields.yml` must also be mounted into the docker container:

   ```
   docker run --rm --name filebeat2 -v $PWD:/input -v $PWD/filebeat.yml:/usr/share/filebeat/filebeat.yml -v $PWD/custom_fields.yml:/usr/share/filebeat/custom_fields.yml docker.elastic.co/beats/filebeat:7.17.3
   ```

1. `setup.template.json` options can be set and the mapping can be provided through an Elasticsearch index template file.

   ```
   setup:
     template:
       json:
         enabled: true
         path: "template.json"
         name: "template-name"
         data_stream: false
   ```
   The `data_stream` option only exists from version 8.x on. 

   The index template file must contain an `index_pattern` definition:
   ```
   {
     "index_patterns": ["test_*"],
       "mappings" : {
         "properties" : {
           "name" : {
             "type" : "text"
           },
           "weirddate" : {
             "type" : "date",
             "format" : "yyyy.MM.dd"
           }
         }
       }
   }
   ```
   Finally, the index template file must of course also be copied or mounted into the docker container when running Filebeat.

   ```
   docker run --rm --name filebeat2 -v $PWD:/input -v $PWD/filebeat.yml:/usr/share/filebeat/filebeat.yml -v $PWD/template.json:/usr/share/filebeat/template.json docker.elastic.c o/beats/filebeat:7.17.3
   ```

#### Process fields
Using processors is a way to transform or filter data during ingestion with Filebeat. 
It is also an option to accomodate special date formats instead of adding an according format to the mapping, simply parsing them into correct dates through Filebeat.
Since date parsing in Filebeat is very convoluted at best, you need to add a `timestamp` processor and parse the date while the event flows through Filebeat (see [documentation](https://www.elastic.co/guide/en/beats/filebeat/7.17/processor-timestamp.html):
```
processors:                                             
  - timestamp:                                          
      field: weirddate                                  
      target_field: weirddate_conv                      
      layouts:                                          
        - '2006.01.02'                                  
      test:                                             
        - '2019.06.22'                                  
        - '2019.11.18'                                  
        - '2020.08.03'                                  
  - drop_fields:                                        
      fields: [weirddate]                               
  - rename:                                             
      fields:                                           
        - from: "weirddate_conv"                        
          to: "weirddate"                               
```

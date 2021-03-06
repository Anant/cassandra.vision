# NOTE: THIS IS A WORK IN PROGRESS! 
This is a reference for information about using Filebeat to ingest offline logs from Cassandra into Elasticsearch/Kibana. 

Note however that this is an older document, and the notes in other documentation is more up to date. **Much of what we do in cassandra.vision now no longer follows what is described in this document.** However, this document is kept here for reference.

# How to Analyze on offline BEK (Beats/Filebeat ElasticSearch Kibana) Setup

When you get a dump of the logs after unzipping all of the gc and log files, use this filebeat template to ingest the logs using a parser that is specialized
for this type of dump. 

This doc does not show you how to setup Filebeat / Kibana / Elastic but assumes you know how to do that. This is just focused on configuring filebeat.



## prepare by unzipping the files 

In the main directory with the host names as different folders , this bash script will unzip everything if it is rolled up. 

```
for z in * ; do echo $z ; cd $z/cassandra ; unzip '*.zip' ; cd ../../ ; done
```

May have to enter "A" or "Y" to override, etc. 


## fields.yml (usually sits in /etc/filebeat/)
This was added to the fields.yml but probably don't need it. 
```
.....
- key: cassandra
  title: Cassandra
  description: >
    Module for parsing Cassandra system log.
  fields:
    - name: ingest
    type: group
    description: >
      Parsed values of the ingestion tasks.
    fields:
    - name: timestamp
      type: date
      description: >
        The actual timestamp.
    - name: loglevel
      type: keyword
      description: >
        The loglevel.
    - name: message
      type: text
      description: >
        The actual message.
 ```

## filebeat.yml (usually sits in /etc/filebeat/)

Pay attention to all of the items under the inputs.. if you dont have the multiline / exclude files config it wont work right. 

```
###################### Filebeat Configuration Example #########################

#=========================== Filebeat inputs =============================

filebeat.inputs:
  - type: log
    tags: ["system","messages"]
    enabled: true
    paths:
      - /home/anant/Projects/Client/ClientName/incident.2020060919/Hostname/system/*.test
      #- /home/anant/Projects/Client/ClientName/incident.2020060919/Hostname/system/*
  - type: log
    tags: ["cassandra","main"]
    enabled: true
    paths:
      - /home/anant/Projects/Client/ClientName/incident.2020060919/Hostname1/cassandra/system.log*
      - /home/anant/Projects/Client/ClientName/incident.2020060919/Hostname2/cassandra/system.log*
    exclude_files: ['\.zip$']  
    multiline.pattern: ^TRACE|DEBUG|WARN|INFO|ERROR
    multiline.negate: true
    multiline.match: after
  - type: log
    tags: ["cassandra","java","gc"]
    enabled: true
    paths:
      - /home/anant/Projects/Client/ClientName/incident.2020060919/Hostname1/cassandra/gc.log*
      - /home/anant/Projects/Client/ClientName/incident.2020060919/Hostname2/cassandra/gc.log*
    exclude_files: ['\.zip$']
    multiline.pattern: ^TRACE|DEBUG|WARN|INFO|ERROR
    multiline.negate: true
    multiline.match: after

#==================== Elasticsearch template setting ==========================
setup.template.settings:
  index.number_of_shards: 1
  #index.codec: best_compression
  # watch out for setting this to false, can make it so kibana only shows an empty _source field in the Discover pane.
  _source.enabled: true
#================================ General =====================================
#name:
#tags: ["service-X", "web-tier"]
#fields:
#  env: staging

#============================== Dashboards =====================================
#setup.dashboards.enabled: true
#setup.dashboards.url:

#============================== Kibana =====================================
setup.kibana:
  host: "http://localhost:5601"
#================================ Outputs =====================================
#-------------------------- Elasticsearch output ------------------------------
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["localhost:9200"]

  # Protocol - either `http` (default) or `https`.
  #protocol: "https"

  # Authentication credentials - either API key or username/password.
  #api_key: "id:api_key"
  #username: "elastic"
  #password: "changeme"

# to debug comment out output.elasticseaarch and uncomment this
# output.console.pretty: true

#----------------------------- Logstash output --------------------------------
# We're sending directly to ES
# output.logstash:
  # The Logstash hosts
  # hosts: ["localhost:5044"]

  # Optional SSL. By default is off.
  # List of root certificates for HTTPS server verifications
  #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]

  # Certificate for SSL client authentication
  #ssl.certificate: "/etc/pki/client/cert.pem"

  # Client Certificate Key
  #ssl.key: "/etc/pki/client/cert.key"

#================================ Processors =====================================
# Configure processors to enhance or manipulate events generated by the beat.

processors:
  # ( since we are not using a live host.. these are irrelevant) 
  #- add_host_metadata: ~ 
  #- add_cloud_metadata: ~
  #- add_docker_metadata: ~ 
  #- add_kubernetes_metadata: 
  
#------------------------------------- system messages 
  - dissect:
      tokenizer: "%{timestamp} %{+timestamp->} %{+timestamp} %{+timestamp} %{system-host-name} %{component}: %{message}"
      field: "message"
      target_prefix: "ingest"

  - dissect:
      tokenizer: "/%{}/%{}/%{}/%{}/%{}/incident.%{incident-id}/%{host-name}/system/"
      field: "log.file.path"
      target_prefix: "ingest"

#-#------------------------------------ cassandra   
  - dissect:
      tokenizer: "%{loglevel} [%{component}] %{timestamp} %{+timestamp},%{} %{class}.%{}:%{} - %{message}"
      field: "message"
      target_prefix: "ingest"
  #- dissect:
  #    tokenizer: "%{componentname}:%{}"
  #    field: "ingest.component"
  #    target_prefix: "ingest"
  #- dissect:
  #    tokenizer: "%{streamdirection}/%{streamvector}"
  #    field: "ingest.component"
  #    target_prefix: "ingest"
  - dissect:
      tokenizer: "/%{}/%{}/%{}/%{}/%{}/incident.%{incident-id}/%{host-name}/cassandra/"
      field: "log.file.path"
      target_prefix: "ingest"
  - timestamp:
      field: "ingest.timestamp"
      layouts:
        - '2006 Jan _2 15:04:05'
        - '2006-01-02 15:04:05'
      test:
        - '2020 Jun  8 02:40:01'
        - '2001 Jan 17 15:04:01'
        - '2019-06-22 16:33:51'
        - '2020-06-13 12:57:30'
      ignore_failure: true
      target_field: "@timestamp"
  - include_fields:
      fields: [ "ingest", "message","log", "input", "tags" ]

 
```
### run file beat setup

```
sudo filebeat setup
```

### repeated testing / debugging 
Use this site to debug dissect statements first. https://dissect-tester.jorgelbg.me/
Then use the output.console to see that it dissects properly. 

Then and only then add to elasticsearch. 

If it gets bungled, you can always delete the filebeat index altogether or if you have gigabytes of data, you can do a selective delete. 
This is where tagging the inputs is useful. 

You can go to the kibana console at http://localhost:5601/app/kibana#/dev_tools/console and do something like this 

```
POST filebeat-7.7.1-2020.06.13-000001/_delete_by_query?conflicts=proceed
{
  "query": {
    "match": {
      "tags": "system"
    }
  }
}

```

### using scripted fields to extract numbers from the message post ingestion

https://www.elastic.co/guide/en/kibana/current/scripted-fields.html
https://stackoverflow.com/questions/32998228/elasticsearch-extract-number-from-a-field

enable painless regex support by putting the following in your elasticsearch.yaml:
```script.painless.regex.enabled: true```

restart elasticsearch
create a new scripted field in Kibana through Management -> Index Patterns -> Scripted Fields
select painless as the language and number as the type
create the actual script, for example:


```
def logMsg = params['_source']['message'];

if(logMsg == null) {
 return 0;
}

def m = /\shas ([0-9]+) dropped hints,\s/.matcher(logMsg);
if ( m.find() ) {
   return Integer.parseInt(m.group(1))
} else {
   return 0
}
```


you must reload the website completely for the new fields to be executed, simply re-doing a search on an open discover site will not pick up the new fields. (This almost made me quit trying to get this working -.-)
use the script in discover or visualizations

### cleaning up the registry after testing

sometimes you have to clean out the filebeat registry 
```
/var/lib/filebeat/registry/filebeat
```

https://discuss.elastic.co/t/solved-force-filebeat-to-reship-file/44415

### testing with output to console 

Sometime it is useful to see that everything is parsing properly before you send to elastic search. 

To do this in filebeats.yml  comment out 

```
#output.elasticsearch:
  # Array of hosts to connect to.
  #hosts: ["localhost:9200"]
```

and uncomment

```
output.console.pretty: true
```

and then run ````````` 
until you see that its parsing properly, you can continue deleting the index / cleaning the registry. 

```
curl -XDELETE 'http://localhost:9200/filebeat-*'
rm -rf /var/lib/filebeat/registry/filebeat/*
```

After its working, you need to revert back the elastic search output otherwise it wont go into elasticsearch

Sometimes you need to delete all the indexes because you fubarred the kibana / index pattern / meta data config. 

```
curl -X DELETE 'http://localhost:9200/_all'
```


After fixing the filebeat.yml you can then run 

```
filebeat -e -d "*"
```




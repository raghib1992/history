## Elastic Stack

### Create elasticsearch pod
```
kubectl create -f https://github.com/hisrarul/history/blob/master/elastic-stack/elasticsearch.yaml
kubectl get pod elastic-search -n elastic-stack -o yaml
```


### Create kibana pod
```
kubectl create -f https://github.com/hisrarul/history/blob/master/elastic-stack/kibana.yaml
kubectl get pod kibana -n elastic-stack -o yaml
```

### Create application pod to generate logs
```
kubectl create -f https://github.com/hisrarul/history/blob/master/elastic-stack/es-app.yaml
kubectl get pod app -n elastic-stack -o yaml
```

### Add a sidecar to send the log of container to Elasticsearch
```
kubectl create -f https://github.com/hisrarul/history/blob/master/elastic-stack/es-sidecar.yaml
```

## Backup and Restore in Elasticsearch

#### Take backup in form of snapshot in elasticsearch
```
PUT /_snapshot/your_s3_respository/your_backup_name/?pretty
{
  "indices": "index1, index2 ",
  "ignore_unavailable": true,
  "include_global_state": false
}
```

## Change replica count

#### Create template for replica count
```
#It is applicable on new indices starting with security-auditlog
curl -XPUT -k -u 'username:password' https://localhost:9200/_template/zeroreplicas -H 'Content-Type: application/json' -d '{"template" : "*", "index_patterns": ["security-auditlog-*"], "settings" : { "number_of_shards": 1, "number_of_replicas" : 0 } }}'
```

#### Change the replica count for existing index
```
curl -X PUT -k -u "$ELASTIC_USERNAME:$ELASTIC_PASSWORD"  https://localhost:9200/security-auditlog-2017.10.30/_settings -H 'Content-Type: application/json' -d '{ "index": {"number_of_replicas": 0 } }'
```

#### Using kibana dashboard
```
GET _cat/indices

GET logstash-2020.12.28/_settings

PUT logstash-2020.12.28/_settings 
{
  "index": {
    "number_of_replicas" : "0"
  }
}
```
---

### Error: Request entity too large
```
#Add this value in kibana.yml
server.maxPayloadBytes: 10000000
```
---

### Error: kibana is loading or wazuh plugin is not showing security and integrity events
+ [Detailed solution](https://github.com/hisrarul/history/blob/master/wazuh/README.md)
---

### Add your own SSL certificates to Open Distro for Elasticsearch
+ [Detailed Steps](https://github.com/hisrarul/history/blob/master/elastic-stack/renew_certificates.md)
---

### Check index pattern in .kibana 
```
GET .kibana_1/_search?_source=false&size=4
{
  "query": {
    "exists": {
      "field": "index-pattern"
    }
  }
}
```

### Error: [2019-03-26T16:40:43,447][WARN ][o.e.m.j.JvmGcMonitorService] [fycetJG] [gc][527283] overhead, spent [763ms] collecting in the last [1s]
Ref: [[1]](https://discuss.elastic.co/t/warn-message-elasticsearch/173975/3) [[2]](https://www.elastic.co/guide/en/elasticsearch/reference/current/tasks.html)
```
Possible solution: 
1. Increase the heap size from jvm.option
2. List the task and take appropriate action. GET _tasks?group_by=parents
``` 

#### [error] [out_es] could not pack/validate JSON response
Ref: [[1]](https://github.com/fluent/fluent-bit/issues/2078)
```
PUT _cluster/settings
{
    "persistent": {
        "action.auto_create_index": "true" 
    }
}
```

### Copy indices from one to another (Reindex)
**1. Create mapping**
```
PUT qa_test_amazon
{"mappings" : {
      "doc" : {
        "properties" : {
          "user" : {
            "type" : "keyword"
          },
          "offer" : {
            "type" : "keyword"
          },
          "cashback" : {
            "type" : "keyword"
          },
          "offer_id" : {
            "type" : "integer"
          },
          "product_id" : {
            "type" : "integer"
          },
          "score" : {
            "type" : "float"
          }
        }
      }
    }}
```
**2. Reindex from source**
```
POST _reindex
{
  "source": {
    "index": "qa_test_amazon"
  },
  "dest": {
    "index": "dev_test_amazon"
  }
}
```

**3. verify total hits**
```
GET dev_test_amazon/_search
{
  "size": 20
}
```

#### open indices in elasticsearch
```
curl -s -XGET 'http://localhost:9200/_cat/indices?h=status,index'
```

#### Configure snapshot for the es indices backup
```
#create snotshot
curl -XPUT 'http://localhost:9200/_snapshot/<s3_respository_name>?verify=false&pretty'  -H 'Content-Type: application/json' -d'
{
    "type" : "s3",
    "settings" : {
      "bucket" : "backup-elk-test",
      "region" : "ap-south-1"
    }
}'
```

#### Backup indices to s3 bucket
```bash
# using kibana dev tool
PUT /_snapshot/<s3_respository_name>/<snapshot-name>/?wait_for_completion=false
{
   "indices": "index_1,index_2,index3",
   "ignore_unavailable": true,
   "include_global_state": false
}

-or-

curl -XPOST http://localhost:9200/_snapshot/<s3_respository_name>/<snapshot-name>/?wait_for_completion=false -H 'Content-Type: application/json' -d'
{
   "indices": "index1,index2",
   "ignore_unavailable": true,
   "include_global_state": false
}'

-or-
# This step is required when we want to backup a lot of indices.
cat > indices-list-29th-june-2021.txt << EOF
{
   "indices": "index1,index2",
   "ignore_unavailable": true,
   "include_global_state": false
}
EOF

curl -XPOST http://localhost:9200/_snapshot/<s3_respository_name>/<snapshot-name>/?wait_for_completion=false -H 'Content-Type: application/json' -d @indices-list-29th-june-2021.txt
```

#### Restore snapshot from s3 bucket
```
curl -XPOST http://localhost:9200/_snapshot/<snapshot-repository>/<snapshot-name>/_restore?wait_for_completion=false -H 'Content-Type: application/json' -d'
{
  "indices": "*",
  "index_settings": {
    "index.number_of_replicas": 0
  }
}'

-or-

#dev tool
POST /_snapshot/<snapshot-repository>/<snapshot-name>/_restore?wait_for_completion=false
{
  "indices": "*",
  "index_settings": {
    "index.number_of_replicas": 0
  }
}

```

#### List repositories
```curl -XGET http://localhost:9200/_cat/repositories/```

#### List configured snapshot
```curl -XGET http://localhost:9200/_snapshot/```

#### List all snapshot/backup in es
```GET _cat/snapshots/s3_respository?v```

#### Read information about snapshot/backup
```
GET /_snapshot/s3_respository/snapshot-2021-02-25
```

#### Delete Snapshots
```
List the snapshot
curl -X DELETE --key ./admin.key --cert ./admin.pem --cacert ./root-ca.pem https://<elasticsearch-server>:9200/_snapshot/<s3_respository>/<snapshot_name>
```

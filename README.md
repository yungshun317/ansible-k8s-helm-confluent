# Ansible Kubernetes Helm Confluent

[![Work with Ansible](https://img.shields.io/badge/Work%20with-Ansible-brightgreen.svg)](https://img.shields.io/badge/Work%20with-Ansible-brightgreen.svg) [![Powered by Kubernetes](https://img.shields.io/badge/Powered%20by-Kubernetes-pink.svg)](https://img.shields.io/badge/Powered%20by-Kubernetes-pink.svg) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT) 

An Ansible playbook deploys Confluent Platform with Helm Charts on Kubernetes.

This is one of the most used playbooks I wrote when I worked for ITRI. It also plays an important role for my last project, Kafka benchmarking, there.

## Overview

Before running the playbook, make sure you have a Kubernetes cluster in anger ([ansible-kubeadm-cluster](https://github.com/yungshun317/ansible-kubeadm-cluster)) and Prometheus deployed ([ansible-k8s-helm-prometheus](https://github.com/yungshun317/ansible-k8s-helm-prometheus)).

In addition to the `cp-helm-charts`, the playbook deploys:
* `zookeeper-datadir-pv.yml`.
* `zookeeper-datalogdir-pv.yml`.
* `zookeeper-client.yml`.
* `kafka-pv.yml`.
* `kafka-client.yml`.

At that time, I followed a really bad approach which used `hostPath` volumes and set `Local` as my `StorageClass`. I even did not use `nodeSelector` but manually created empty directories for each node. The better approach is dynamically  provisioning persistent volumes with Ceph as the `StorageClass`. For more details please refer to my project, [k8s-ceph-nginx](https://github.com/yungshun317/k8s-ceph-nginx). Hence, in fact, all manifest files for persistent volumes are not necessary here.

The Helm Chart deplys:
* `cp-kafka`.
* `cp-kafka-connect`.
* `cp-kafka-rest`.
* `cp-ksql-server`.
* `cp-schema-registry`.
* `cp-zookeeper`.
If you want to deploy manually, just run `helm install --name confluent ./cp-helm-charts` to install the chart.

For benchmarking Kafka, I added these manifest files, which were not deployed by Ansible:
* `kafka-client-producer.yml`: sends messages with `kafka-console-producer` in a `while` loop, which is very resource-intensive.
* `ksql-client.yml`: acts as a consumer for demonstrations.
* `ksql-datagen.yml`: sends messages with `ksql-datagen quickstart=pageviews format=avro topic=pageviews schemaRegistryUrl=http://confluent-cp-schema-registry:8081 bootstrap-server=confluent-cp-kafka-headless:9092` using the `confluentinc/ksql-examples` image.

## Zookeeper Client

Kafka uses Zookeeper to coordinate the brokers/cluster topology. Zookeeper is a consistent file system for configuration information. Zookeeper gets used for leadership election for broker, topic, partition, leaders.

Use the `zookeeper-client` to run `zookeeper-shell`:
```sh
~$ kubectl exec -it zookeeper-client -- /bin/bash

root@zookeeper-client:/# zookeeper-shell confluent-cp-zookeeper:2181
Connecting to confluent-cp-zookeeper:2181
Welcome to ZooKeeper!
JLine support is enabled
[zk: confluent-cp-zookeeper:2181(CONNECTING) 0] 
WATCHER::

WatchedEvent state:SyncConnected type:None path:null
ls /brokers/ids
[0, 1, 2]
 
[zk: confluent-cp-zookeeper:2181(CONNECTED) 1] ls /brokers/topics
[confluent-cp-kafka-canary-topic, _schemas, confluent-cp-kafka-connect-config, confluent-cp-kafka-connect-status, confluent-cp-kafka-connect-offset, __confluent.support.metrics, __consumer_offsets, _confluent-ksql-confluent_command_topic]

[zk: confluent-cp-zookeeper:2181(CONNECTED) 2] get /brokers/ids/0
{"listener_security_protocol_map":{"PLAINTEXT":"PLAINTEXT","EXTERNAL":"PLAINTEXT"},"endpoints":["PLAINTEXT://confluent-cp-kafka-0.confluent-cp-kafka-headless.default:9092","EXTERNAL://10.236.1.2:31090"],"jmx_port":5555,"host":"confluent-cp-kafka-0.confluent-cp-kafka-headless.default","timestamp":"1553592646850","port":9092,"version":4}
cZxid = 0x100000054
ctime = Tue Mar 26 09:30:46 UTC 2019
mZxid = 0x100000054
mtime = Tue Mar 26 09:30:46 UTC 2019
pZxid = 0x100000054
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x2017d997fb90003
dataLength = 337
numChildren = 0
```

## Kafka Client

`kafka-console-producer` and `kafka-console-consumer` are frequently used for testing and demonstrations.

### kafka-console-producer

Create a topic and start `kafka-console-producer`.
```sh
~$ kubectl exec -it kafka-client -- /bin/bash

root@kafka-client:/# kafka-topics --zookeeper confluent-cp-zookeeper-headless:2181 --topic confluent-topic --create --partitions 1 --replication-factor 1 --if-not-exists
Created topic "confluent-topic".

root@kafka-client:/# kafka-console-producer --broker-list confluent-cp-kafka-headless:9092 --topic confluent-topic
>Wed Mar 27 01:56:47 UTC 2019
```

### kafka-console-consumer

Consume the messages and then delete the topic.
```sh
~$ kubectl exec -it kafka-client -- /bin/bash

root@kafka-client:/# kafka-console-consumer --bootstrap-server confluent-cp-kafka-headless:9092 --topic confluent-topic --from-beginning
Wed Mar 27 01:56:47 UTC 2019

root@kafka-client:/# kafka-topics --list --zookeeper confluent-cp-zookeeper-headless:2181
__confluent.support.metrics
__consumer_offsets
_confluent-ksql-confluent_command_topic
_schemas
confluent-cp-kafka-canary-topic
confluent-cp-kafka-connect-config
confluent-cp-kafka-connect-offset
confluent-cp-kafka-connect-status
confluent-topic

root@kafka-client:/# kafka-topics --zookeeper confluent-cp-zookeeper-headless:2181 --delete --topic confluent-topic
Topic confluent-topic is marked for deletion.
Note: This will have no impact if delete.topic.enable is not set to true.

root@kafka-client:/# kafka-topics --list --zookeeper confluent-cp-zookeeper-headless:2181
__confluent.support.metrics
__consumer_offsets
_confluent-ksql-confluent_command_topic
_schemas
confluent-cp-kafka-canary-topic
confluent-cp-kafka-connect-config
confluent-cp-kafka-connect-offset
confluent-cp-kafka-connect-status
```

## KSQL client

Confluent KSQL is the streaming SQL engine that enables real-time data processing against Apache Kafka. It provides an easy-to-use, yet powerful interactive SQL interface for stream processing on Kafka, without the need to write code.
```sh
~$ docker run -it confluentinc/cp-ksql-cli http://10.108.124.40:8088

# Or
~$ kubectl exec -it ksql-client -- ksql http://10.108.124.40:8088

ksql> SHOW TOPICS;

ksql> create stream pageviews_original (viewtime bigint, userid varchar, pageid varchar) with (kafka_topic='pageviews', value_format='DELIMITED');
 Message        
----------------
 Stream created 
----------------

ksql> SELECT pageid FROM pageviews_original LIMIT 3;
Page_90
Page_85
Page_21
Limit Reached
Query terminated

ksql> DROP STREAM pageviews_original;
 Message        
----------------
 Stream created 
----------------
```

The above commands were used for working with my `ksql-datagen` Deployment, which I scaled the producer Pods up to hundreds.

## Kafka Connect

One of my another researches about Kafka is integrating Kafka with PostgreSQL using the JDBC sink connector. Here is an example of `FileStreamSinkConnector`:
```sh
~$ kubectl get svc confluent-cp-kafka-connect
NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
confluent-cp-kafka-connect   ClusterIP   10.99.152.226   <none>        8083/TCP   16h

~$ curl 10.99.152.226:8083
{"version":"2.1.1-cp1","commit":"9aa84c2aaa91e392","kafka_cluster_id":"PVaQwafjQjm04xAnZeatSg"}

~$ curl 10.99.152.226:8083/connector-plugins
[{"class":"io.confluent.connect.activemq.ActiveMQSourceConnector","type":"source","version":"5.1.2"},{"class":"io.confluent.connect.elasticsearch.ElasticsearchSinkConnector","type":"sink","version":"5.1.2"},{"class":"io.confluent.connect.hdfs.HdfsSinkConnector","type":"sink","version":"5.1.2"},{"class":"io.confluent.connect.hdfs.tools.SchemaSourceConnector","type":"source","version":"2.1.1-cp1"},{"class":"io.confluent.connect.ibm.mq.IbmMQSourceConnector","type":"source","version":"5.1.2"},{"class":"io.confluent.connect.jdbc.JdbcSinkConnector","type":"sink","version":"5.1.2"},{"class":"io.confluent.connect.jdbc.JdbcSourceConnector","type":"source","version":"5.1.2"},{"class":"io.confluent.connect.jms.JmsSourceConnector","type":"source","version":"5.1.2"},{"class":"io.confluent.connect.s3.S3SinkConnector","type":"sink","version":"5.1.2"},{"class":"io.confluent.connect.storage.tools.SchemaSourceConnector","type":"source","version":"2.1.1-cp1"},{"class":"org.apache.kafka.connect.file.FileStreamSinkConnector","type":"sink","version":"2.1.1-cp1"},{"class":"org.apache.kafka.connect.file.FileStreamSourceConnector","type":"source","version":"2.1.1-cp1"}]

~$ curl 10.99.152.226:8083/connectors
[]

~$ curl -X POST -H "Content-Type: application/json" --data '{"name": "local-file-sink", "config": {"connector.class":"FileStreamSinkConnector", "tasks.max":"1", "file":"test.sink.txt", "topics":"connect-test" }}' http://10.99.152.226:8083/connectors
{"name":"local-file-sink","config":{"connector.class":"FileStreamSinkConnector","tasks.max":"1","file":"test.sink.txt","topics":"connect-test","name":"local-file-sink"},"tasks":[],"type":"sink"}

~$ curl 10.99.152.226:8083/connectors
["local-file-sink"]

~$ curl 10.99.152.226:8083/connectors/local-file-sink/tasks 
[{"id":{"connector":"local-file-sink","task":0},"config":{"file":"test.sink.txt","task.class":"org.apache.kafka.connect.file.FileStreamSinkTask","topics":"connect-test"}}]

~$ curl 10.99.152.226:8083/connectors/local-file-sink/status
{"name":"local-file-sink","connector":{"state":"RUNNING","worker_id":"10.244.1.33:8083"},"tasks":[{"id":0,"state":"RUNNING","worker_id":"10.244.1.33:8083"}],"type":"sink"}

~$ curl -X DELETE 10.99.152.226:8083/connectors/local-file-sink
```

### JDBC Sink Connector

The producer, `ksql-datagen`, generates messages with keys in string and values in Avro. We need to set `key.converter` as `org.apache.kafka.connect.storage.StringConverter` to convert the keys to Avro because Kafka Connect only works with Avro. This is tricky but is not documented well. 
```sh
~$ vim jdbc-sink.json
{
  "name": "jdbc-sink",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max": "1",
    "topics": "pageviews",
    "connection.url": "jdbc:postgresql://postgres-postgresql:5432/sink",
    "connection.user": "postgres",
    "connection.password": "1rAo45RfDM",
    "auto.create": "true",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter"
  }
}

~$ curl -X POST -H "Content-Type: application/json" --data @jdbc-sink.json http://10.104.55.60:8083/connectors
{"name":"jdbc-sink","config":{"connector.class":"io.confluent.connect.jdbc.JdbcSinkConnector","tasks.max":"1","topics":"pageviews","connection.url":"jdbc:postgresql://postgres-postgresql:5432/sink","connection.user":"postgres","connection.password":"1rAo45RfDM","auto.create":"true","name":"jdbc-sink"},"tasks":[{"connector":"jdbc-sink","task":0}],"type":"sink"}

~$ curl 10.104.55.60:8083/connectors
["jdbc-sink"]

~$ curl 10.104.55.60:8083/connectors/jdbc-sink/status
{"name":"jdbc-sink","connector":{"state":"RUNNING","worker_id":"10.244.1.154:8083"},"tasks":[{"id":0,"state":"RUNNING","worker_id":"10.244.1.154:8083"}],"type":"sink"}
```

## Schema Registry

Because `ksql-datagen` has its values in Avro, we can check the versions in Schema Registry. 
```sh
~$ curl 10.111.43.228:8081/subjects
["pageviews-value"]

~$ curl 10.111.43.228:8081/subjects/pageviews-value
{"error_code":405,"message":"HTTP 405 Method Not Allowed"}

~$ curl 10.111.43.228:8081/subjects/pageviews-value/versions
[1]

~$ curl 10.111.43.228:8081/subjects/pageviews-value/versions/1
{"subject":"pageviews-value","version":1,"id":1,"schema":"{\"type\":\"record\",\"name\":\"KsqlDataSourceSchema\",\"namespace\":\"io.confluent.ksql.avro_schemas\",\"fields\":[{\"name\":\"viewtime\",\"type\":[\"null\",\"long\"],\"default\":null},{\"name\":\"userid\",\"type\":[\"null\",\"string\"],\"default\":null},{\"name\":\"pageid\",\"type\":[\"null\",\"string\"],\"default\":null}]}"}
```

## Tech

This project uses:
* [Ansible](https://www.ansible.com/) - Ansible is open source software that automates software provisioning, configuration management, and application deployment.
* [Kubernetes](https://kubernetes.io/) - an open-source system for automating deployment, scaling, and management of containerized applications.
* [Confluent Platform](https://www.confluent.io/product/confluent-platform/) - an enterprise-ready event streaming platform, driving a new paradigm for application and data infrastructure, based on Apache Kafka.

## Todos
 - Create a client Deployment with Pods that export Kafka JMX metrics of producers or consumers to Prometheus and Grafana.
 - Get more familiar with Kafka Connect and Schema Registry.

## License
[Ansible Kubernetes Helm Confluent](https://github.com/yungshun317/ansible-k8s-helm-confluent) is released under the [MIT License](https://opensource.org/licenses/MIT) by [yungshun317](https://github.com/yungshun317).

# Ansible Kubernetes Helm Confluent
## Zookeeper Client
$ kubectl exec -it zookeeper-client -- /bin/bash
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

## Kafka Client
### kafka-console-producer
$ kubectl exec -it kafka-client -- /bin/bash
root@kafka-client:/# kafka-topics --zookeeper confluent-cp-zookeeper-headless:2181 --topic confluent-topic --create --partitions 1 --replication-factor 1 --if-not-exists
Created topic "confluent-topic".
root@kafka-client:/# kafka-console-producer --broker-list confluent-cp-kafka-headless:9092 --topic confluent-topic
>Wed Mar 27 01:56:47 UTC 2019

### kafka-console-consumer
$ kubectl exec -it kafka-client -- /bin/bash
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
root@kafka-client:/# kafka-topics --zookeeper confluent-cp-zookeeper-headless:2181 --delete --topic confluent-topicTopic confluent-topic is marked for deletion.
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

## Kafka Connect
$ kubectl get svc confluent-cp-kafka-connect
NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
confluent-cp-kafka-connect   ClusterIP   10.99.152.226   <none>        8083/TCP   16h

$ curl 10.99.152.226:8083
{"version":"2.1.1-cp1","commit":"9aa84c2aaa91e392","kafka_cluster_id":"PVaQwafjQjm04xAnZeatSg"}

$ curl 10.99.152.226:8083/connector-plugins
[{"class":"io.confluent.connect.activemq.ActiveMQSourceConnector","type":"source","version":"5.1.2"},{"class":"io.confluent.connect.elasticsearch.ElasticsearchSinkConnector","type":"sink","version":"5.1.2"},{"class":"io.confluent.connect.hdfs.HdfsSinkConnector","type":"sink","version":"5.1.2"},{"class":"io.confluent.connect.hdfs.tools.SchemaSourceConnector","type":"source","version":"2.1.1-cp1"},{"class":"io.confluent.connect.ibm.mq.IbmMQSourceConnector","type":"source","version":"5.1.2"},{"class":"io.confluent.connect.jdbc.JdbcSinkConnector","type":"sink","version":"5.1.2"},{"class":"io.confluent.connect.jdbc.JdbcSourceConnector","type":"source","version":"5.1.2"},{"class":"io.confluent.connect.jms.JmsSourceConnector","type":"source","version":"5.1.2"},{"class":"io.confluent.connect.s3.S3SinkConnector","type":"sink","version":"5.1.2"},{"class":"io.confluent.connect.storage.tools.SchemaSourceConnector","type":"source","version":"2.1.1-cp1"},{"class":"org.apache.kafka.connect.file.FileStreamSinkConnector","type":"sink","version":"2.1.1-cp1"},{"class":"org.apache.kafka.connect.file.FileStreamSourceConnector","type":"source","version":"2.1.1-cp1"}]

$ curl 10.99.152.226:8083/connectors
[]

$ curl -X POST -H "Content-Type: application/json" --data '{"name": "local-file-sink", "config": {"connector.class":"FileStreamSinkConnector", "tasks.max":"1", "file":"test.sink.txt", "topics":"connect-test" }}' http://10.99.152.226:8083/connectors
{"name":"local-file-sink","config":{"connector.class":"FileStreamSinkConnector","tasks.max":"1","file":"test.sink.txt","topics":"connect-test","name":"local-file-sink"},"tasks":[],"type":"sink"}

$ curl 10.99.152.226:8083/connectors
["local-file-sink"]

$ curl 10.99.152.226:8083/connectors/local-file-sink/tasks 
[{"id":{"connector":"local-file-sink","task":0},"config":{"file":"test.sink.txt","task.class":"org.apache.kafka.connect.file.FileStreamSinkTask","topics":"connect-test"}}]

$ curl 10.99.152.226:8083/connectors/local-file-sink/status
{"name":"local-file-sink","connector":{"state":"RUNNING","worker_id":"10.244.1.33:8083"},"tasks":[{"id":0,"state":"RUNNING","worker_id":"10.244.1.33:8083"}],"type":"sink"}

$ curl -X DELETE 10.99.152.226:8083/connectors/local-file-sink

# Ansible Kubernetes Helm Confluent
## Kafka Client

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



#work #kafka

# 启动Kafka
* zookeeper
```bash
export KAFKA_HOME=~/kafka_2.13-2.7.0
$KAFKA_HOME/bin/zookeeper-server-start.sh $KAFKA_HOME/config/zookeeper.properties
```
* kafka
```bash
export KAFKA_HOME=~/kafka_2.13-2.7.0
$KAFKA_HOME/bin/kafka-server-start.sh $KAFKA_HOME/config/server.properties
```

* kafdrop
```sh
export KAFKA_HOME=~/kafka_2.13-2.7.0
java -jar $KAFKA_HOME/kafdrop-3.27.0.jar --kafka.brokerConnect=localhost:9092
```
* connector
```sh
export KAFKA_HOME=~/kafka_2.13-2.7.0
$KAFKA_HOME/bin/connect-distributed.sh $KAFKA_HOME/config/connect-distributed.properties
```

### Kubernetes部署

* Chart: https://github.com/bitnami/charts/tree/master/bitnami/kafka
  * replicaCount
  * Using NodePort services
* Kafka server config: /opt/bitnami/kafka/config/server.properties
  * Listener
  * Log

```bash

```


### 性能测试

* Client
```bash
kubectl run my-release-kafka-client --restart='Never' --image docker.io/bitnami/kafka:2.7.0-debian-10-r68 --namespace kafka --command -- sleep infinity
```

* Test
```bash
  export KAFKAS=my-release-kafka-0.my-release-kafka-headless.kafka.svc.cluster.local:9092
  
  # 创建Topic
  kafka-topics.sh --bootstrap-server $KAFKAS --create --topic test-rep-one --partitions 6 --replication-factor 1
  
  # Producer
  kafka-producer-perf-test.sh --topic test-rep-one --num-records 50000000 --record-size 100 --throughput -1 --producer-props acks=1 bootstrap.servers=$KAFKAS  buffer.memory=67108864 batch.size=8196
  
  # Consumer
  kafka-consumer-perf-test.sh --bootstrap-server $KAFKAS --messages 50000000 --topic test-rep-one --threads 1
```



### 命令

* 查看Consumer Group

```sh
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group test-group
```

* 修改Partition

```sh
kafka-topics.sh --bootstrap-server localhost:9092 --alter --topic test --partitions 2
```



### Performance

- **Tune your consumer socket buffers for high-speed ingest.** In Kafka 0.10.x, the parameter is `receive.buffer.bytes`, which defaults to 64kB. In Kafka 0.8.x, the parameter is `socket.receive.buffer.bytes`, which defaults to 100kB. Both of these default values are too small for high-throughput environments, particularly if the network's [bandwidth-delay product](https://en.wikipedia.org/wiki/Bandwidth-delay_product) between the broker and the consumer is larger than a local area network (LAN). For high-bandwidth networks (10 Gbps or higher) with latencies of 1 millisecond or more, consider setting the socket buffers to 8 or 16 MB. If memory is scarce, consider 1 MB. You can also use a value of `-1`, which lets the underlying operating system tune the buffer size based on network conditions. However, the automatic tuning might not occur fast enough for consumers that need to start "hot."

### Monitoring Kafka while maintaining sanity

We have three very important things we now monitor and alert on for our Kafka clusters:

- **Retention:** How much data can we store on disk for each topic partition?
- **Replication:** How many copies of the data can we make?
- **Consumer Lag:** How do we monitor how far behind our consumer applications are from the producers?

### Connector

```sh
{
    "name": "inventory-connector",
    "config": {
        "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
        "connection.url": "jdbc:postgresql://172.16.0.130:5432/yanhuang",
        "connection.user": "postgresadmin",
        "connection.password": "admin123",
        "mode": "bulk",
        "topic.prefix": "test-"
    }
}
```



```json
{
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:postgresql://172.16.0.130:5432/yanhuang",
    "connection.user": "postgresadmin",
    "connection.password": "admin123",
    "mode": "bulk",
    "transforms": "setupYHP",
    "transforms.setupYHP.type": "com.yanhuangdata.kafka.connect.smt.SetYhpAttribute",
    "transforms.setupYHP.datatype": "json",
    "transforms.setupYHP.eventset": "test_eventset",
    "transforms.setupYHP.host": "test_host",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false"
}
```

```json
{
    "connector.class": "com.yanhuangdata.parana.connect.jdbc.JdbcSourceConnector",  
    "connection.url": "jdbc:postgresql://172.16.0.130:5432/yanhuang",
    "connection.user": "postgresadmin",
    "connection.password": "admin123",
    "mode": "incrementing",
    "table.whitelist": "test_result", 
    "incrementing.column.name": "id",
    "tasks.max": 1,
    "poll.interval.ms": 5000,
    "errors.tolerance": "all",
    "errors.log.enable":true,
    "errors.log.include.messages":true,
    "transforms": "setupYHP",
    "transforms.setupYHP.type": "com.yanhuangdata.parana.connect.smt.SetYhpAttribute",
    "transforms.setupYHP.datatype": "json",
    "transforms.setupYHP.eventset": "test_eventset",
    "transforms.setupYHP.host": "test_host",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false"
}
```

https://github.com/Comcast/MirrorTool-for-Kafka-Connect

```json
{
    "name": "kafka-connect-kafka-source-example2",
    "config": {
        "tasks.max": "2",
        "key.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
        "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
        "connector.class": "com.comcast.kafka.connect.kafka.KafkaSourceConnector",
        "source.bootstrap.servers": "kafka-dev.kafka.svc.cluster.local:9092",
        "source.topic.whitelist": "test-.*",
        "source.auto.offset.reset": "earliest",
        "source.group.id": "kafka-connect-testing1",
        "connector.consumer.reconnect.backoff.max.ms": "10000",
        "transforms": "setupYHP",
        "transforms.setupYHP.type": "com.yanhuangdata.parana.connect.smt.SetYhpAttribute",
        "transforms.setupYHP.datatype": "json",
        "transforms.setupYHP.eventset": "test_eventset",
        "transforms.setupYHP.host": "test_host"
    }
}
```



### Kafka Connector Transforms

https://kafka.apache.org/documentation/#connect_transforms




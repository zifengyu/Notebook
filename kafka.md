# 启动Kafka

* zookeeper

```sh
export KAFKA_HOME=~/kafka_2.13-2.7.0
$KAFKA_HOME/bin/zookeeper-server-start.sh $KAFKA_HOME/config/zookeeper.properties
```

* kafka

```sh
export KAFKA_HOME=~/kafka_2.13-2.7.0
$KAFKA_HOME/bin/kafka-server-start.sh $KAFKA_HOME/config/server.properties
```

* kafdrop

```sh
export KAFKA_HOME=~/kafka_2.13-2.7.0
java -jar $KAFKA_HOME/kafdrop-3.27.0.jar --kafka.brokerConnect=localhost:9092 --protobufdesc.directory=$KAFKA_HOME/protobuf_desc
```



### Kubernetes部署

* Chart: https://github.com/bitnami/charts/tree/master/bitnami/kafka
  * replicaCount
  * Using NodePort services
* Kafka server config: /opt/bitnami/kafka/config/server.properties
  * Listener
  * Log



### 性能测试

* Client
  ```sh
kubectl run my-release-kafka-client --restart='Never' --image docker.io/bitnami/kafka:2.7.0-debian-10-r68 --namespace kafka --command -- sleep infinity
  ```

* Test
  ```sh
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


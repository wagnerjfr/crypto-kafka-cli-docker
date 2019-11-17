# crypto-kafka-cli-docker

Simple example to take Bitcoin and Litecoin JSON data from [Bitstamp API](https://www.bitstamp.net/api/), publish them in different topics and read the values in different consumers. Basic steps for deploying Kafka along with Confluent Platform components in a Docker environment are also provided.

## Step 1: Create a Docker Network
Docker network is used to run the Confluent containers and is required to enable DNS resolution across your containers.
```
$ docker network create cryptonet
```
## Step2: Start ZooKeeper
Start ZooKeeper container and keep the service running:
```
$ docker run -d \
  --net=cryptonet \
  --name=zookeeper \
  -e ZOOKEEPER_CLIENT_PORT=2181 \
  confluentinc/cp-zookeeper:5.1.0
```
*[Optional]* Check the Docker container log to confirm that it has booted up successfully:
```
$ docker logs zookeeper
```
Wait for some seconds and you should see the below message at the end of the log output:
```console
[2019-01-05 11:33:04,754] INFO binding to port 0.0.0.0/0.0.0.0:2181 (org.apache.zookeeper.server.NIOServerCnxnFactory)
```
## Step 3: Start Kakfa
Start a single-node cluster Kafka with this command:
```
$ docker run -d \
  --net=cryptonet \
  --name=kafka \
  -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092 \
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  confluentinc/cp-kafka:5.1.0
```
*[Optional]* Check the logs to see whether the broker has started successfully:
```
$ docker logs kafka
```
Similiar messages should appear at the end of the log output command:
```console
[2019-01-05 11:39:06,528] DEBUG [Controller id=1001] Topics not in preferred replica for broker 1001 Map() (kafka.controller.KafkaController)
[2019-01-05 11:39:06,529] TRACE [Controller id=1001] Leader imbalance ratio for broker 1001 is 0.0 (kafka.controller.KafkaController)
```
Up to now we have two services running. To check it run:
```
$ docker ps -a
```
Something similar to this should appear:
```console
CONTAINER ID    IMAGE                             COMMAND                  CREATED              STATUS              PORTS                          NAMES
7bfb0e8084df    confluentinc/cp-kafka:5.1.0       "/etc/confluent/dock…"   36 seconds ago       Up 35 seconds       9092/tcp                       kafka
5b4857aae162    confluentinc/cp-zookeeper:5.1.0   "/etc/confluent/dock…"   About a minute ago   Up About a minute   2181/tcp, 2888/tcp, 3888/tcp   zookeeper
```

## Step 4: Create topics
We are going to create two topics, named BTC and LTC, and attribute to them just one partition and one replica.
```
for TYPE in BTC LTC
do
  docker run \
  --net=cryptonet \
  --rm confluentinc/cp-kafka:5.1.0 \
  kafka-topics --create --topic "$TYPE" --partitions 1 --replication-factor 1 \
  --if-not-exists --zookeeper zookeeper:2181
done
```
Console output:
```console
Created topic "BTC".
Created topic "LTC".
```
*[Optional]* Verify whether the topics were created successfully:
```
for TYPE in BTC LTC
do
  docker run \
  --net=cryptonet \
  --rm \
  confluentinc/cp-kafka:5.1.0 \
  kafka-topics --describe --topic $TYPE --zookeeper zookeeper:2181
done
```
Console output:
```console
Topic:BTC	PartitionCount:1	ReplicationFactor:1	Configs:
	Topic: BTC	Partition: 0	Leader: 1001	Replicas: 1001	Isr: 1001
Topic:LTC	PartitionCount:1	ReplicationFactor:1	Configs:
	Topic: LTC	Partition: 0	Leader: 1001	Replicas: 1001	Isr: 1001
```
*[Optional]* Command to list the kafka topics registered:
```
docker exec -it kafka kafka-topics --zookeeper zookeeper:2181 --list
```

## Step 5: Produce and consume data

### 1. Start consumers
The commands below will start two *built-in* Kafka Console Consumer containers, one assigned to consume data from the BTC topic and the other one from the LTC topic. Run the commands below in different terminals so we can see the data being consumed.

**BTC Consumer (terminal 1):**
```
$ docker run \
  --net=cryptonet \
  --name=consumer-btc \
  confluentinc/cp-kafka:5.1.0 \
  kafka-console-consumer --bootstrap-server kafka:9092 --topic BTC
```
**LTC Consumer (terminal 2)**
```
$ docker run \
  --net=cryptonet \
  --name=consumer-ltc \
  confluentinc/cp-kafka:5.1.0 \
  kafka-console-consumer --bootstrap-server kafka:9092 --topic LTC
```
Both consumers are now just waiting for the new messages from the broker.

### 2. Publish data to the new topics
Open a new terminal (terminal 3) and run the command below. It will first start the built-in Kafka Console Producer, grab Bitcoin (BTC) JSON data from Bitstamp API and finally publish it to the BTC topic. After that, Litecoin (LTC) JSON data will be grabbed and published to LTC topic. 

*P.S. The below command will iterate for 5 times with 3 seconds interval.* 
```
for i in {1..5}
do
  docker run \
  --net=cryptonet \
  --rm \
  confluentinc/cp-kafka:5.1.0 \
  bash -c "curl https://www.bitstamp.net/api/v2/ticker_hour/btcusd/ | kafka-console-producer \
    --request-required-acks 1 --broker-list kafka:9092 --topic BTC && \
    curl https://www.bitstamp.net/api/v2/ticker_hour/ltcusd/ | kafka-console-producer \
    --request-required-acks 1 --broker-list kafka:9092 --topic LTC"
  sleep 3s
done
```
### 3: Check data consumed

**BTC Consumer (terminal 1):**
```console
{"high": "3848.14", "last": "3845.61", "timestamp": "1546691445", "bid": "3841.65", "vwap": "3840.95", "volume": "115.46455836", "low": "3830.72", "ask": "3846.51", "open": "3838.02"}
{"high": "3848.14", "last": "3846.07", "timestamp": "1546691453", "bid": "3840.93", "vwap": "3840.99", "volume": "116.46455836", "low": "3830.72", "ask": "3846.05", "open": "3838.02"}
{"high": "3848.14", "last": "3846.07", "timestamp": "1546691460", "bid": "3840.75", "vwap": "3840.99", "volume": "116.46455836", "low": "3830.72", "ask": "3845.73", "open": "3838.02"}
{"high": "3848.14", "last": "3846.07", "timestamp": "1546691469", "bid": "3840.75", "vwap": "3841.01", "volume": "116.28799356", "low": "3831.41", "ask": "3845.40", "open": "3838.02"}
{"high": "3848.14", "last": "3846.07", "timestamp": "1546691477", "bid": "3840.75", "vwap": "3841.01", "volume": "116.28799356", "low": "3831.41", "ask": "3846.04", "open": "3838.02"}
```
**LTC Consumer (terminal 2)**
```console
{"high": "34.29", "last": "34.06", "timestamp": "1546691447", "bid": "33.98", "vwap": "33.90", "volume": "4965.13128531", "low": "33.21", "ask": "34.13", "open": "34.00"}
{"high": "34.29", "last": "34.06", "timestamp": "1546691454", "bid": "33.97", "vwap": "33.90", "volume": "4965.13128531", "low": "33.21", "ask": "34.10", "open": "34.00"}
{"high": "34.29", "last": "34.06", "timestamp": "1546691462", "bid": "33.97", "vwap": "33.90", "volume": "4965.13128531", "low": "33.21", "ask": "34.16", "open": "34.00"}
{"high": "34.29", "last": "34.06", "timestamp": "1546691469", "bid": "33.96", "vwap": "33.90", "volume": "4965.13128531", "low": "33.21", "ask": "34.13", "open": "34.00"}
{"high": "34.29", "last": "34.06", "timestamp": "1546691478", "bid": "33.97", "vwap": "33.90", "volume": "4965.13128531", "low": "33.21", "ask": "34.13", "open": "34.00"}
```
## Step 6: Cleanup
1. Stop running containers:
```
$ docker stop zookeeper kafka consumer-btc consumer-ltc
```
2. Remove stopped containers:
```
$  docker rm zookeeper kafka consumer-btc consumer-ltc
```
3. Remove Docker network:
```
$ docker network rm cryptonet
```

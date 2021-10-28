## Table of Contents
- [Kafka Producers](#Kafka-Producers)
  - [Overview](#Overview)
  - [Constructing](#Constructing)
  - [Sending a Message](#sending-a-message)
  - [Configuring Producers](#configuring-producers)
  - [Serializers](#serializers)

## Kafka Producers

### Overview

1. `ProducerRecord` is created by application to be sent as messages to Kafka broker. `ProducerRecord` contains:
    1. Topic
    2. Partition (optional)
    3. Key (optional)
    4. Value

2. `ProducerRecord` is serialized to ByteArrays

3. Data is sent to partitioner, partition is selected (either based on partition specifed or based on key)

4. Record is added to batch of records that will be sent to same topic and partition

5. Batch of messages is sent to broker. If successful, it will return a `RecordMetadata` object with topic, partition, and offset. If failed, it will send an error, potentially triggering a retry

### Constructing

Three mandatory properties for constructing a producer:
1. `bootstrap.servers` - List of host:port pairs of brokers that producer will use to connect to Kafka cluster (recommended to include at least 2)
2. `key.serializer` - Name of class that will be used to serialize the keys of records. class must implement `org.apache.kafka.common.serialization.Serializer` interface.
    - Kafka client includes `ByteArraySerializer`, `StringSerializer`, and `IntegerSerializer`
2. `value.serializer` - Name of class that will serialize the values of records

Example of creating Producer:
```
private Properties kafkaProps = new Properties();
kafkaProps.put("bootstrap.servers", "broker1:9092,broker2:9092");
kafkaProps.put("key.serializer",
 "org.apache.kafka.common.serialization.StringSerializer");
kafkaProps.put("value.serializer",
 "org.apache.kafka.common.serialization.StringSerializer");
producer = new KafkaProducer<String, String>(kafkaProps);
```

### Sending A Message

Primary methods of sending messages:
1. Fire-and-forget - send a message and don't care if it's successful or not
```
ProducerRecord<String, String> record =
 new ProducerRecord<>("CustomerCountry", "Precision Products",
 "France");
try {
 producer.send(record);
} catch (Exception e) {
 e.printStackTrace();
}
```
2. Synchronous send - send a message, send() method returns a Future object, use get() to wait on the future's success
```
ProducerRecord<String, String> record =
 new ProducerRecord<>("CustomerCountry", "Precision Products", "France");
try {
 producer.send(record).get();
} catch (Exception e) {
 e.printStackTrace();
}
```
3. Asynchronous send - we call send() method with callback which gets triggered when it receives a response from Kafka broker
```
private class DemoProducerCallback implements Callback {
 @Override
 public void onCompletion(RecordMetadata recordMetadata, Exception e) {
 if (e != null) {
 e.printStackTrace();
 }
 }
}
ProducerRecord<String, String> record =
 new ProducerRecord<>("CustomerCountry", "Biomedical Materials", "USA");
producer.send(record, new DemoProducerCallback());
```

### Configuring Producers

`acks` - controls how many partition replicas must receive record before producer considers the write successful
    - `acks=0` - producer will not wait for the broker to assume success, can be used to achieve high throughput if success is not critical
    - `acks=1` - producer will receive success response from the broker when leader receives the message. safer but no without potential for failure, increased latency
    - `acks=all` - producer will receive success response once all in-sync replicas received message. safest mode, highest latency

`buffer.memory` - sets amount of memory the producer will use to buffer messages waiting to be set. if messages are sent faster than they can be delivered, producer may run out of space and send() may block or throw exception based on `block.on.buffer.full` param (or `max.block.ms` which allows for blocking a certain amount of time)

`compression.type` - by default messages are uncompressed. param can be set to `snappy`, `gzip`, or `lz4`. snappy is good for low CPU overhead, gzip is good for cases where network bandwidth is more restricted

`retries` - how many times client will try re-sending message if error is transient, before giving up. default is to wait 100ms between tries (can be set with `retry.backoff.ms`)

`batch.size` - param controls the amount of memory in bytes that will be sent per batch. when batch is full, all messages will be sent. producer will not necessarily wait for batch to be full before sending

`linger.ms` - amount of time to wait for additional messages before sending current batch. by default, producer will send messages as soon as a sender thread is available unless set to higher than 0. increases latency, but also increases throughput

`client.id` - any string used to identify messages from client

`max.in.flight.requests.per.connection` - controls how many messages the producer will send to the server without receiving responses. setting this high can increase memory but improve throughput. too high can reduce throughput as batching becomes less efficient. setting to 1 will guarantee that messages will be written in the order they were sent

`timout.ms`, - controls how long producer will wait for a reply for server when sending data

`request.timeout.ms` - controls how long producer will wait for a reply for server when sending data

`metadata.fetch.timeout.ms` - how long producer will wait when requesting metadata such as current leader for partition

`max.block.ms` - how long producer will block when calling send() and when requesting metadata via partitionsFor()

`max.request.size` - controls size of a produce request, caps largest message that can be sent and number of messages that can be sent

`receive.buffer.bytes` and `send.buffer.bytes` -  sizes of the TCP send and receive buffers used by sockets when reading and writing data. if set to -1, OS defaults will be used. Good idea to increase when producers/consumers communicate with brokers in a different datacenter due to higher latency/lower bandwidtch

ordering gaurantees - setting retries paraemeter to > 0 and max.in.flight.requests.per.session to more than one means that messages may not be in exact order. recommend to set in.flight.requests.per.session=1 to make sure order is preserved


## Serializers

Not recommended to create custom serializer. Instead use serialization library like Avro, Thrift, or Protobuf

Serializing using Apache Avro
- language neutral data serialization format
- described in a language-independent schema (JSON)
- typically serialized to binary files
- allows for upgrading schema over time
- reading and writing data must use compatible schema

Schema Registry - not part of Apache Kafka but there are several open source options to choose from
    - store all schemas to write data
    - store the identifier for the schema in the record we produce
    - consumers use identifier to pull record from registry and deserialize data
    - handled in serializer/deserializer
use `schema.registry.url` in properties when creating Producer
can also provide a custom schema with `GenericData.Record(schema)`

## Partitions


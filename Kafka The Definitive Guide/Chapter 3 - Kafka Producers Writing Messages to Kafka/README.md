# Chapter 3 - Kafka Producers Writing Messages to Kafka 

## Producer Overview

There are many reasons an application might need to write messages to Kafka: recording user activities for auditing or analysis, recording metrics, storing log messages, recording information from smart appliances, communicating asynchronously
with other applications, buffering information before writing to a database, and much more.

Those diverse use cases also imply diverse requirements: is every message critical, or can we tolerate loss of messages? Are we OK with accidentally duplicating messages? Are there any strict latency or throughput requirements we need to support?

We start producing messages to Kafka by creating a ProducerRecord, which must include the topic we want to send the record to and a value. Optionally, we can also specify a key and/or a partition. Once we send the ProducerRecord, the first thing the producer will do is serialize 
the key and value objects to ByteArrays so they can be sent over the network.

Next, the data is sent to a partitioner. If we specified a partition in the ProducerRecord, the partitioner doesn’t do anything and simply returns the partition we specified. If we didn’t, the partitioner will choose a partition for us, usually based on the ProducerRecord key. 
Once a partition is selected, the producer knows which topic and partition the record will go to. It then adds the record to a batch of records that will also be sent to the same topic and partition.

When the broker receives the messages, it sends back a response. If the messages were successfully written to Kafka, it will return a RecordMetadata object with the topic, partition, and the offset of the record within the partition. If the broker failed
to write the messages, it will return an error.

When the producer receives an error, it may retry sending the message a few more times before giving up and returning an error.

## Constructing a Kafka Producer

A Kafka producer has three mandatory properties:

**bootstrap.servers**

List of host:port pairs of brokers that the producer will use to establish initial connection to the Kafka cluster.
it is recommended to include at least two, so in case one broker goes down, the producer will still be able to connect to the cluster.

**key.serializer**

Name of a class that will be used to serialize the keys of the records we will produce to Kafka. Kafka brokers expect byte arrays as keys and values of messages.

**value.serializer**

Name of a class that will be used to serialize the values of the records we will produce to Kafka.

The following code snippet shows how to create a new producer by setting just the mandatory parameters and using defaults for everything else:

```java
private Properties kafkaProps = new Properties();
kafkaProps.put("bootstrap.servers", "broker1:9092,broker2:9092");
kafkaProps.put("key.serializer","org.apache.kafka.common.serialization.StringSerializer");
kafkaProps.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer"); 
producer = new KafkaProducer<String, String>(kafkaProps);
```

With such a simple interface, it is clear that most of the control over producer behavior is done by setting the correct configuration properties.

Once we instantiate a producer, it is time to start sending messages. There are three primary methods of sending messages:
- Fire-and-forget: We send a message to the server and don’t really care if it arrives succesfully or not.
- Synchronous send: We send a message, the send() method returns a Future object, and we use get() to wait on the future and see if the send() was successful or not.
- Asynchronous send: We call the send() method with a callback function, which gets triggered when it receives a response from the Kafka broker.

While all the examples in this chapter are single threaded, a producer object can be used by multiple threads to send messages. You will probably want to start with one producer and one thread. If you need better throughput, you can add more threads that use the same producer.

## Sending a Message to Kafka

The simplest way to send a message is as follows:

```java
ProducerRecord<String, String> record =
new ProducerRecord<>("CustomerCountry", "Precision Products", "France");
try {
producer.send(record);
} catch (Exception e) {
e.printStackTrace();
}
```

## Sending a Message Synchronously

The simplest way to send a message synchronously is as follows:

```java
ProducerRecord<String, String> record =
new ProducerRecord<>("CustomerCountry", "Precision Products", "France");
try {
producer.send(record).get();
} catch (Exception e) {
e.printStackTrace();
}
```

Here, we are using Future.get() to wait for a reply from Kafka. This method will throw an exception if the record is not sent successfully to Kafka. If there were no errors, we will get a RecordMetadata object that we can use to retrieve the offset the message was written to.

KafkaProducer has two types of errors. Retriable errors are those that can be resolved by sending the message again.
Some errors will not be resolved by retrying.

## Sending a Message Asynchronously

Suppose the network roundtrip time between our application and the Kafka cluster is 10ms. If we wait for a reply after sending each message, sending 100 messages will take around 1 second.
On the other hand, if we just send all our messages and not wait for any replies, then sending 100 messages will barely take any time at all.

In order to send messages asynchronously and still handle error scenarios, the producer supports adding a callback when sending a record. Here is an example of how we use a callback:

```java
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

## Configuring Producers

**acks**

The acks parameter controls how many partition replicas must receive the record before the producer can consider the write successful.

- If acks=0, the producer will not wait for a reply from the broker before assuming the message was sent successfully. This means that if something went wrong and the broker did not receive the message, the producer will not know about it and the message will be lost. 
However, because the producer is not waiting for any response from the server, it can send messages as fast as the network will support, so this setting can be used to achieve very high throughput.
- If acks=1, the producer will receive a success response from the broker the moment the leader replica received the message. If the message can’t be written to the leader (e.g., if the leader crashed and a new leader was not elected yet), 
the producer will receive an error response and can retry sending the message, avoiding potential loss of data.
- If acks=all, the producer will receive a success response from the broker once all in-sync replicas received the message. This is the safest mode since you can make sure more than one broker has the message and that the message will survive even in the case of crash

**buffer.memory**

This sets the amount of memory the producer will use to buffer messages waiting to be sent to brokers. If messages are sent by the application faster than they can be delivered to the server, the producer may run out of space and additional send() calls will either block or throw an exception.

**compression.type**

By default, messages are sent uncompressed. This parameter can be set to snappy, gzip, or lz4, in which case the corresponding compression algorithms will be used to compress the data before sending it to the brokers.
Snappy compression was invented by Google to provide decent compression ratios with low CPU overhead and good performance, so it is recommended in cases where both performance and bandwidth are a concern.
Gzip compression will typically use more CPU and time but result in better compression ratios, so it recommended in cases where network bandwidth is more restricted. By enabling compression, you reduce network utilization and storage, which is often a bottleneck when sending messages to Kafka.

**retries**

When the producer receives an error message from the server, the error could be transient (e.g., a lack of leader for a partition). In this case, the value of the retries parameter will control how many times the producer will retry sending the message before giving up and notifying the client of an issue.

**batch.size**

When multiple records are sent to the same partition, the producer will batch them together. This parameter controls the amount of memory in bytes (not messages!) that will be used for each batch.

**linger.ms**

linger.ms controls the amount of time to wait for additional messages before sending the current batch. KafkaProducer sends a batch of messages either when the current batch is full or when the linger.ms limit is reached.

**client.id**
 
This can be any string, and will be used by the brokers to identify messages sent from the client. It is used in logging and metrics, and for quotas.

**max.in.flight.requests.per.connection**

This controls how many messages the producer will send to the server without receiving responses. Setting this high can increase memory usage while improving throughput, but setting it too high can reduce throughput as batching becomes less efficient.

**timeout.ms, request.timeout.ms, and metadata.fetch.timeout.ms**

These parameters control how long the producer will wait for a reply from the server when sending data.
If the timeout is reached without reply, the producer will either retry sending or respond with an error.

**max.block.ms**

This parameter controls how long the producer will block when calling send() and when explicitly requesting metadata via partitionsFor().

**max.request.size**

This setting controls the size of a produce request sent by the producer. It caps both the size of the largest message that can be sent and the number of messages that the producer can send in one request.

**receive.buffer.bytes and send.buffer.bytes**

These are the sizes of the TCP send and receive buffers used by the sockets when writing and reading data.

## Ordering Guarantees

Apache Kafka preserves the order of messages within a partition. This means that if messages were sent from the producer in a specific order, the broker will write them to a partition in that order and all consumers will read them in that order.

Usually, setting the number of retries to zero is not an option in a reliable system, so if guaranteeing order is critical, we recommend setting in.flight.requests.per.session=1 to make sure that while a batch of messages is retrying, additional messages will not be sent.


## Serializers

producer configuration includes mandatory serializers.

## Custom Serializers

When the object you need to send to Kafka is not a simple string or integer, you have a choice of either using a generic serialization library like Avro, Thrift, or Protobuf to create records, or creating a custom serialization for objects you are already using. 
We highly recommend using a generic serialization library.

## Serializing Using Apache Avro

Avro data is described in a language-independent schema. The schema is usually described in JSON and the serialization is usually to binary files, although serializing to JSON is also supported. Avro assumes that the schema is present when reading and writing files, 
usually by embedding the schema in the files themselves.

One of the most interesting features of Avro, and what makes it a good fit for use in a messaging system like Kafka, is that when the application that is writing messages switches to a new schema, the applications reading the data can continue processing messages without requiring any change or update.

Suppose the original schema was:

```json
{"namespace": "customerManagement.avro",
"type": "record",
"name": "Customer",
"fields": [
{"name": "id", "type": "int"},
{"name": "name", "type": "string""},
{"name": "faxNumber", "type": ["null", "string"], "default": "null"}
]
}
```
Now suppose that we decide that in the new version, we will upgrade to the twenty-first century and will no longer include a fax number field and will instead use an email field.


## Partitions

Kafka messages are key-value pairs and while it is possible to create a ProducerRecord with just a topic and a value, with the key set to null by default, most applications produce records with keys. Keys serve two goals:
- they are additional information that gets stored with the message
- they are also used to decide which one of the topic partitions the message will be written to.

All messages with the same key will go to the same partition. This means that if a process is reading only a subset of the partitions in a topic all the records for a single key will be read by the same process.

create a ProducerRecord as follows:

```java
ProducerRecord<Integer, String> record =
new ProducerRecord<>("CustomerCountry", "Laboratory Equipment", "USA");
```

When creating messages with a null key, you can simply leave the key out:

```java
ProducerRecord<Integer, String> record =
new ProducerRecord<>("CustomerCountry", "USA");
```

When the key is null and the default partitioner is used, the record will be sent to one of the available partitions of the topic at random. A round-robin algorithm will be used to balance the messages among the partitions.

## Implementing a custom partitioning strategy

Kafka does not limit you to just hash partitions, and sometimes there are good reasons to partition data differently.
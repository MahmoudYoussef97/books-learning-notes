# Chapter 2 - Installing Kafka

## Choosing an Operating System

Apache Kafka is a Java application, and can run on many operating systems. This includes Windows, MacOS, Linux, and others.
Linux is the most common OS on which it is installed. This is also the recommended OS for deploying Kafka for general use.

## Installing Java

Prior to installing either Zookeeper or Kafka, you will need a Java environment set up and functioning. This should be a Java 8 version, and can be the version provided by your OS or one directly downloaded from [java.com](http://java.com/).

## Installing Zookeeper

Apache Kafka uses Zookeeper to store metadata about the Kafka cluster, as well as consumer client details. While it is possible to run a Zookeeper server using scripts contained in the Kafka distribution, it is trivial to install a full version of Zookeeper from the distribution.

## Zookeeper ensemble

A Zookeeper cluster is called an ensemble. Due to the algorithm used, it is recommended that ensembles contain an odd number of servers (e.g., 3, 5, etc.) as a majority of ensemble members (a quorum) must be working in order for Zookeeper to respond to requests.
This means that in a three-node ensemble, you can run with one node missing. With a five-node ensemble, you can run with two nodes missing.

## Sizing Your Zookeeper Ensemble

Consider running Zookeeper in a five-node ensemble. In order to make configuration changes to the ensemble, including swapping a node, you will need to reload nodes one at a time. If your ensemble cannot tolerate more than one node being down, doing maintenance work introduces additional risk.
It is also not recommended to run more than seven nodes, as performance can start to degrade due to the nature of the consensus protocol.
To configure Zookeeper servers in an ensemble, they must have a common configuration that lists all servers, and each server needs a myid file in the data directory that
specifies the ID number of the server.

```
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
initLimit=20
syncLimit=5
server.1=zoo1.example.com:2888:3888
server.2=zoo2.example.com:2888:3888
server.3=zoo3.example.com:2888:3888
```

the initLimit is the amount of time to allow followers to connect with a leader. The syncLimit value limits how out-of-sync followers can be with the leader.

The configuration also lists each server in the ensemble. The servers are specified in the format server.X=hostname:peerPort:leaderPort, with the following parameters:
**X**
The ID number of the server. This must be an integer, but it does not need to be zero-based or sequential.
**hostname**
The hostname or IP address of the server.
**peerPort**
The TCP port over which servers in the ensemble communicate with each other.
**leaderPort**
The TCP port over which leader election is performed.

## Installing a Kafka Broker

Once Java and Zookeeper are configured, you are ready to install Apache Kafka.

## Broker Configuration

The example configuration provided with the Kafka distribution is sufficient to run a standalone server as a proof of concept, but it will not be sufficient for most installations. There are numerous configuration options for Kafka that control all aspects of setup and tuning.

## General Broker

There are several broker configurations that should be reviewed when deploying Kafka for any environment other than a standalone broker on a single server.

**broker.id**

Every Kafka broker must have an integer identifier, which is set using the broker.id configuration. By default, this integer is set to 0, but it can be any value.
The most important thing is that the integer must be unique within a single Kafka cluster.
A good guideline is to set this value to something intrinsic to the host so that when performing maintenance it is not onerous to map broker ID numbers to hosts.

**port**

The example configuration file starts Kafka with a listener on TCP port 9092. This can be set to any available port by changing the port configuration parameter. Keep
in mind that if a port lower than 1024 is chosen, Kafka must be started as root.

**zookeeper.connect**

The location of the Zookeeper used for storing the broker metadata is set using the zookeeper.connect configuration parameter.

- hostname, the hostname or IP address of the Zookeeper server.
- port, the client port number for the server.
- /path, an optional Zookeeper path to use as a chroot environment for the Kafka cluster. If it is omitted, the root path is used. If a chroot path is specified and does not exist, it will be created by the broker when it starts up.

**log.dirs**

Kafka persists all messages to disk, and these log segments are stored in the directories specified in the log.dirs configuration. This is a comma-separated list of paths on the local system.
If more than one path is specified, the broker will store partitions on them in a “least-used” fashion with one partition’s log segments stored within the same path.

**num.recovery.threads.per.data.dir**

Kafka uses a configurable pool of threads for handling log segments. Currently, this thread pool is used:
- When starting normally, to open each partition’s log segments
- When starting after a failure, to check and truncate each partition’s log segments
- When shutting down, to cleanly close log segments

By default, only one thread per log directory is used.
This means that if num.recovery.threads.per.data.dir is set to 8, and there are 3 paths specified in log.dirs, this is a total of 24 threads.

**auto.create.topics.enable**

The default Kafka configuration specifies that the broker should automatically create a topic under the following circumstances:
- When a producer starts writing messages to the topic
- When a consumer starts reading messages from the topic
- When any client requests metadata for the topic

If you are managing topic creation explicitly, whether manually or through a provisioning system, you can set the auto.create.topics.enable configuration to false.

## Topic Defaults

The Kafka server configuration specifies many default configurations for topics that are created. Several of these parameters, including partition counts and message retention.

**num.partitions**

The num.partitions parameter determines how many partitions a new topic is created with. Default is 1
partitions are the way a topic is scaled within a Kafka cluster, which makes it important to use partition counts that will balance the message load across the entire cluster as brokers are added.

## How to Choose the Number of Partitions

- What is the throughput you expect to achieve for the topic?
- What is the maximum throughput you expect to achieve when
consuming from a single partition?
- If you are sending messages to partitions based on keys,
adding partitions later can be very challenging, so calculate
throughput based on your expected future usage, not the current
usage.
- Consider the number of partitions you will place on each
broker and available diskspace and network bandwidth per
broker.
- Avoid overestimating, as each partition uses memory and
other resources on the broker and will increase the time for
leader elections.

if I want to be able to write and read 1 GB/sec from a topic, and I know each consumer can only process 50 MB/s, then I know I need at least 20 partitions. This way, I can have 20 consumers reading from the topic and achieve 1 GB/sec.

**log.retention.ms**

The most common configuration for how long Kafka will retain messages is by time. The default is specified in the configuration file using the log.retention.hours
parameter, and it is set to 168 hours, or one week.
The recommended parameter to use is log.retention.ms, as the smaller unit size will take precedence if more than one is specified.

**log.retention.bytes**

Another way to expire messages is based on the total number of bytes of messages retained.

This value is set using the log.retention.bytes parameter, and it is applied per-partition. This means that if you have a topic with 8 partitions, and log.retention.bytes is set to 1 GB, the amount of data retained for the topic will be 8 GB at most.

**log.segment.bytes**

Once the log segment has reached the size specified by the log.segment.bytes parameter, which defaults to 1 GB, the log segment is closed and a new one is opened. Once a log segment has been closed, it can be considered for expiration.

**log.segment.ms**

Another way to control when log segments are closed is by using the [log.segment.ms](http://log.segment.ms/) parameter

**message.max.bytes**

The Kafka broker limits the maximum size of a message that can be produced, configured by the message.max.bytes parameter, which defaults to 1000000, or 1 MB. A producer that tries to send a message larger than this will receive an error back from the broker.
This configuration deals with compressed message size, which means that producers can send messages that are much larger than this value uncompressed

## Coordinating Message Size Configurations

The message size configured on the Kafka broker must be coordinated with the fetch.message.max.bytes configuration on consumer clients. If this value is smaller than message.max.bytes, then consumers that encounter larger messages will fail to fetch those messages, resulting in a situation where the consumer gets stuck and cannot proceed.

## Hardware Selection

Selecting an appropriate hardware configuration for a Kafka broker can be more art than science. Kafka itself has no strict requirement on a specific hardware configuration, and will run without issue on any system.

## Disk Throughput

The obvious decision when it comes to disk throughput is whether to use traditional spinning hard drives (HDD) or solid-state disks (SSD). SSDs have drastically lower
seek and access times and will provide the best performance.
HDDs, on the other hand, are more economical and provide more capacity per unit.

## Disk Capacity

If the broker is expected to receive 1 TB of traffic each day, with 7 days of retention, then the broker will need a minimum of 7 TB of useable storage for log segments. You should also factor in at least 10% overhead for other files, in addition to any buffer that you wish to maintain for fluctuations in traffic or growth over time.
The decision on how much disk capacity is needed will also be informed by the replication strategy chosen for the cluster.

## Memory

Kafka consumer is reading from the end of the partitions, where the consumer is caught up and lagging behind the producers very little, if at all. In this situation, the messages the consumer is reading are optimally stored in the system’s page cache,  resulting in faster reads than if the broker has to reread the messages from disk.

Kafka itself does not need much heap memory configured for the Java Virtual Machine (JVM).

## Networking

The available network throughput will specify the maximum amount of traffic that Kafka can handle.

## CPU

Processing power is not as important as disk and memory, but it will affect overall performance of the broker to some extent.
Ideally, clients should compress messages to optimize network and disk usage.
The Kafka broker must decompress all message batches, to validate the checksum of the individual messages and assign offsets. It then needs to recompress the message batch in order to store it on disk.

## Kafka in the Cloud

this will mean that for AWS either the m4 or r3 instance types are a common choice. The m4 instance will allow for greater retention periods, but the throughput to the disk will be less because it is on elastic block storage. The r3 instance will have much better throughput with local SSD drives, but those drives will limit the amount of data that can be retained.
For the best of both worlds, it is necessary to move up to either the i2 or d2 instance types, which are significantly more expensive.

## Kafka Clusters

A single Kafka server works well for local development work, or for a proof-ofconcept system, but there are significant benefits to having multiple brokers configured
as a cluster.
The biggest benefit is the ability to scale the load across multiple servers. A close second is using replication to guard against data loss due to single system failures. Replication will also allow for performing maintenance work on Kafka or the underlying systems while still maintaining availability for
clients.

## How Many Brokers?

The appropriate size for a Kafka cluster is determined by several factors. The first factor to consider is how much disk capacity is required for retaining messages and how much storage is available on a single broker.
If the cluster is required to retain 10 TB of data and a single broker can store 2 TB, then the minimum cluster size is five brokers.
In addition, using replication will increase the storage requirements by at least 100%, depending on the replication factor chosen. This means that this same cluster, configured with replication, now needs to contain at least 10 brokers.
The other factor to consider is the capacity of the cluster to handle requests. For example, what is the capacity of the network interfaces, and can they handle the client traffic if there are multiple consumers of the data or if the traffic is not consistent over the retention period of the data

## Broker Configuration

There are only two requirements in the broker configuration to allow multiple Kafka brokers to join a single cluster. The first is that all brokers must have the same configuration for the zookeeper.connect parameter. This specifies the Zookeeper ensemble and path where the cluster stores metadata.

## OS Tuning

While most Linux distributions have an out-of-the-box configuration for the kerneltuning parameters that will work fairly well for most applications, there are a few
changes that can be made for a Kafka broker that will improve performance.

- Virtual Memory
- Disk
- Networking

## Production Concerns

Once you are ready to move your Kafka environment out of testing and into your production operations, there are a few more things to think about that will assist with
setting up a reliable messaging service.
- Garbage Collector Options
- Datacenter Layout
- Colocating Applications on Zookeeper
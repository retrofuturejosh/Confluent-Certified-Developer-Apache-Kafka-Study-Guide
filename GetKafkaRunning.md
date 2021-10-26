## Table of Contents
- [Get Kafka Running](#get-kafka-running)
  - [Zookeeper](#zookeeper)
  - [Kafka Broker](#kafka-broker)
  - [Hardware Selection](#hardware-selection)
  - [In the Cloud](#in-the-cloud)
  - [Kafka Clusters](#kafka-clusters)
  - [Production Concerns](#production-concerns)



## Get Kafka Running

### Zookeeper
- stores metadata about the Kafka cluster, as well as
consumer client details
- Zookeeper cluster is called an ensemble
  - ensemble should be odd number
  - 5 is recommended for ability to upgrade servers without degrading performance
  - must have a common configuration that lists all servers
  - each server needs a myid file in the data directory that specifies the ID number of the server

Example configuration:
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
-  initLimit is the amount of time to allow followers to con‐
nect with a leader.
- syncLimit value limits how out-of-sync followers can be with
the leader
- Both values are a number of tickTime units
- Servers are specified in the format server.X=hostname:peerPort:leaderPort
  - X - the ID number of the server. This must be an integer, but it does not need to be
zero-based or sequential.
  - hostname - hostname or IP address of the server.
  - peerPort - TCP port over which servers in the ensemble communicate with each other.
  - leaderPort - TCP port over which leader election is performed


### Kafka Broker
#### Broker Configurations
- broker.id - integer identifier, default is 0, van be any value, must be unique within cluster
- port - 9092 in example, ideally higher than 1024, so it doesn't have to run as root
- zookeeper.connect - location of zookeeper, semicolon-separated list of hostname:port/path
  - /path is optional Zookeeper path to use as a chroot environment
  - good to use chroot path, allows Zookeeper ensemble to  be shared with other applications
- log.dirs - kafka persists messages to disk in log segments, specified in log.dirs configuration
  - comma-separated list of paths on the local system
  - if more than one, broker stores in "least-used" fashion
  - one partitions log segments are stored in the same path
  - broker places new partition in path with least number of partitions, not least amount of disk space
- num.recovery.threads.per.data -  configurable pool of threads for handling log segment, used for:
  1. starting normally, open each partitions log segments
  2. starting after failure, check and truncate each partitions log segments
  3. shutting down, cleanly close log segments
  - default is one thread per log directory
  - large number can be used to parallelize operations if broker restarts
  - number is multiplied by number of log directories (e.g. if number is 8 and there are 3 paths in log.dirs, there are 24 threads)
- auto.create.topics.enable - default configuration specifies that broker should automatically create topic when:
  1. producer starts writing messages to topic
  2. consumer starts reading messages from topic
  3. any client requests metadata for topic
#### Topic Defaults
- num.partitions - determines how many partitions a new topic is crated with (when auto creation is enabled), default is 1
  - note: num of partitions can only be increased, never decreased
  - typical pattern is to have num partitions equal to or multiple of number of brokers
  - suggested to limit size of partition on the disk to less than 6 GB per day of retention

- message retention - most common configuration for how long Kafka will retain messages is by time, default is specified using log.retention.hours set to 168 hours (one week)
  - log.retention.minutes and log.retention.ms are also available
  - smaller unit of size takes precedence, so recommended parameter is log.retention.ms

- log.retention.bytes - expiration of messages based on the total number of bytes of messages retained, applied per partition

- log retention can use both time and byes when either criteria is met

- log.segment.bytes - retention settings operate on log segments, not individual messages
  - helpful to configure if topics have a low produce rate, so messages are not grouped together from very different times

- log.segment.ms - specifies the amount of time after which a log segment should be closed
  - can also be used with log.segment.bytes, whichever is reached first

- message.max.bytes - limits max size of a message that can be produced, defaults to 1MB
  - must be coordinated with fetch.message.max.bytes configuration on consumer or consumer can get stuck
  - must be coordinated with replica.fetch.max.bytes

### Hardware Selection

- Disk throughput
  - most direct influence on performance
  - SSD is preferable of HDD
  - Increase performance of HDD by useing more than one in broker (multiple directories or RAID)
- Disk capacity
  - determined by how many messages need to be retained at a given time
  - informed by replication strategy
- Memory
  - more memory improves performance for consumer clients, since they can read from system's page cache
  - JVM does not need much heap memory (5gb can handle high number of messages)
  - Ideally, Kafka should not be on a system with another significant application so as to not share page cache
- Networking
  - specifies maximum amount traffic, inbound and outbound
- CPU
  - not as important as disk/memory, not a primary factor in choosing hardware

### In the Cloud

- how to decide
  - start with the amount of data retention
required, followed by the performance needed from the producers
- m4 or r3 instance types are a common choice.
  - m4 instance will allow for greater retention periods, but the
throughput to the disk will be less because it is on elastic block storage
  - The r3 instance will have much better throughput with local SSD drives, but those drives will limit the amount of data that can be retained
- For the best of both worlds, use the i2 or d2 instance types, which are significantly more
expensive

### Kafka Clusters

single server works for dev work or POC

multiple brokers provides significant benefits for prod
  - biggest benefit is scaling across multiple servers
  - replication protects against data loss from system failures
  - allows for performing maintenance

How many brokers?
  - how much disk capacity is required and how much storage is available on a single broker
  - capacity for handling requests

Broker Configuration
  - requirements for multiple brokers:
    1. all brokers must have the same configuration for zookeeper.connect
    2. each must have unique broker.id

OS Tuning
  - configured in /etc/systcl.conf: set vm.swappiness to a very low value (such as 1). it is better to reduze the size of page cache rather than swap

Disk
  - In file system, set the noatime mount option, atime increases disk writes and is not needed by kafka

Networking
  - tune networking stack for high amount of network traffic

### Production Concerns

Garbage Collection
  - MaxGCPauseMillis - default is 200 ms, can be set to 20 ms
  - InitiatingHeapOccupancyPercent - default is 45, can be set to 35
  - these are declared as environment variables when using the start command for kafka


Datacenter Layout
  - best practice is to have each Kafka broker in a cluster installed in a different rack
  - no shared points of failure for infra


Colocating Apps on Zookeeper
  - single Zookeeper ensemble can support multiple Kafka clusters (using chroot path)
  - recommended for consumers to use Kafka for committing offsets rather than zookeeper
  - do not share zookeeper with other applications

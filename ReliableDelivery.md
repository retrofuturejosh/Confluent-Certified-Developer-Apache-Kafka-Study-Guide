## Table of Contents
- [Reliable Data Delivery](#Reliable-Data-Delivery)
  - [Reliability Guarantees](#Reliability-Guarantees)
  - [Replication](#Replication)
  - [Broker Configuration](#Broker-Configuration)
  - [Using Producers](#Using-Producers)
  - [Using Consumers](#Using-Consumers)
  - [Validating Reliability](#Validating-Reliability)

## Reliability Guarantees

- Order guarantee of messages in a partition
- Produced messages are committed when written to partition and in-sync replicas
- Messages that are committed will not be lost as long as one replica remains alive
- Consumers can only read messages that are committed


## Replication

Multiple replicas per partition is core of reliability guarantees

Each topic is broken down into partitions (basic data blocks). Partition is stored on a single disk. Partition can have multiple replicas (one is leader). Events are producted/consumed from leader. Other replicas stay in sync with leader

Replicas are in-sync if
- sending heartbeats to Zookeeper (6 sec default)
- fetched messages from leader in last 10 secs
- fetched most recent messages from leader in last 10 secs

## Broker Configuration

### Replication Factor

`replication.factor` for topic-level config
`default.replication.factor` for broker-level config

replication factor of n, allows you to lose n-1 brokers and requires at least n brokers

### Unclean Leader Election

`unclean.leader.election.emable` - only available at broker level, default is true

should be set to false if data consistency and quality are more important than availability

### Minimum In-sync replicas

`min.insync.replicas` - both topic and broker level config

If min insync is not met, producers will get `NotEnoughReplicasException` when attempting to produce, and the leader (if available) will become read only until more brokers are back online

## Using Producers

For reliability, pay attentions to:
- correct acks configuration
- handle errors correctly in configuration and code

### Send Acknowledgments

acks=0 - message is 'successful' if producer sent it over network, won't know about many failures (leader election, etc)

acks=1 - leader sends acknowledgement that it wrote the message (or an error if it didn't). this means a producer will know about `LeaderNotAvailableException` if leader election is taking place

acks=all - leader will wait until all in-sync replicas got message before sending acknowledgment or error, used in conjunction with `min.insync.replica`. most reliable option, but slowest

### Configuring Producer Retries

Must handle retriable errors, e.g. `LEADER_NOT_AVAILABLE`

Keep tring to send messages if error is retriable. Configure how long you will retry

Some retries may cause messages to be duplicated

Kafka guarantees that messages will be sent at least once, NOT exactly once

Must use a unique identifier to check for duplicates in application

Could also make request idempotent (2 of the same messages has no negative consquence)

### Additional Error Handling

Built in producer retries automatically

Developer has to handle
- nonretriable errors (message size, authorization errorts, etc)
- errors that occur before message was sent (serialization)
- errors that occur when producer exhausted all retry attemptsor when producer is filled to limit with stored messages while retrying

## Using Consumers

Consumers read messages that have been written to all in-sync replicas

Consumers keep track of what they have consumed by committing offsets

### Important Consumer Config

`group.id` - consumers with the same id are the same app

`auto.offset.reset` - controls what consumer will do when no offsets were committed or when consumer asks for offsets that don't exist
    - `earliest` - start from beginning of partition
    - `latest` - start from end of partition

`enables.auto.commit` - consumer automatically commits on a schedule. no control over duplicate records, may commit offsets for records the consumer has read but did not process

`auto.commit.interval.ms` - how frequently consumer autocommits default is 5 seconds

### Explicitly Committing Offsets

Always commit offsets AFTER events were processed
    - commit at the end of the poll loop

Commit frequency is trade-off between performance and number of duplicates in the event of crash
    - can commit after every message, after one loop, after multiple loops, etc.

Make sure you know exactly what offsets you are committing
    - only commit messages that were processed, otherwise consumer may miss messages

Rebalances
    - consumer rebalances will happen. commit offsets before partitions are revoked and clean state when assigned new partitions

Consumers may need to retry
    - If error is retriable, commit last successful record, and put others in retriable buffer
    - OR write to a separate topic and continue (similar to dead letter queue)

Consumers may need to maintain state
    - Best option is to use Kafka Streams

Handling long processing times
    - hand off data to a thread pool with multiple threads to speed things
    - hand off data to multi threads and pause consumer (keep polling without fetching data) until data is processed

Exactly-once delivery
    - easiest way is to write results to a system with support for unique keys resulting in idempotent writes
    - write records and offsets in the same transaction to stay in sync. retrieve offsets of latest records when starting up consumer using consumer.seek() method

## Validating System Reliability

### Validating Configuration

- does configuration meet requirements?
- reason through expected behavior

use `VerifiableProducer` and `VerifiableConsumer` which can be used CLI or in testing framework to produce and read a sequence of messages with configuration of your consumer

### Validating Applications

Test things like:
- client loses connection to server
- leader election
- rolling restart of brokers
- rolling restart of consumers
- rolling restart of producers

### Monitoring in Production

For producers, look at error-rate and retry-rate per record

For consumers, look for consumer lag


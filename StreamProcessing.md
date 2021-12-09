## Table of Contents
- [Stream Processing](#Stream-Processing)
  - [Concepts](#Concepts)
  - [Design Patterns](#Design-Patterns)
  - [Kafka Streams](#Kafka-Streams)
  - [Kafka Streams: Architecture](#Kafka-Streams-Architecture)

## Stream Processing

### What is it?

a data stream is an abstraction representing an unbounded (infinite and growing) dataset. e.g. stream of credit card transactions, stock trades, package deliveries

Event streams are ordered

Event streams have immutable data records

Event streams are replayable

### Stream processing

Request-response: lowest latency processing paradigm. usually blocking. e.g. POS system

Batch processing: high latency/high-throughput. e.g. cron job that runs on the hour

Stream processing: nonblocking option. Somewhere between request/response and batch processing latency. good use case for alerting suspicious activity, adjusting prices based on supply/demand, tracking package deliveries

## Concepts

### Time

Most important concept in stream processing

Event time: time that events occured. Kafka automatically adds current time to producer records when created. Event time can also be added as a field in the data

Log append time: time the event arrived to Kafka broker and was stored there. Less important

Processing time: time at which the stream-processing application received the event

### State

Some processes are stateless and can be processed individually

Some take multiple events and join them together to create an enriched stream of info: stateful

Types of state in a streaming app:
1. local or internal state: state only accessible by the app, usually in memory. fast but limited
2. External state: maintained in external datastore (often NoSQL). Unlimited size and accessible from multiple instances

### Stream-Table Duality

Streams represent every event that took place over time

Tables contain the current state of events

In order to convert a table to a stream, we need to capture changes that modify the table (insert, update, delete events)

In order to convert a stream to a table, we need to apply the changes the stream contains. Also called materializing the stream

### Time Windows

most operations on streams are windowed operations, existing in slices of time

consider the following:
- size of the window
- how often the window moves (advance interval)
- how long the window remains updatable

## Design Patterns

### Single Event Processing

process each event in isolation (map/filter pattern). Events are processed/transformed and put into an output stream

Example: Read messages from a stream and write ERROR events into a high-priority stream

### Processing with Local State

Aggregating information, esp within a time window. Requires maintaining state for the stream

Example: find a min/max value in a stream

Consider:
- memory usage
- persistence. Handled in Kafka Streams by using embedded RocksDB, persisting data to disk for quick recovery, and sending aggregate state to a kafka topic
- Rebalancing: instance can lose or gain state

### Multiphase Processing / Repartitioning

Example: top 10 stocks. multiple applications reduce to a smaller stream. this stream is read by a single application which can calculate the final result

### Processing with External Lookup: Stream-Table Join

Example: enriching click data with user info from a DB. Can't keep calling DB or it would be too slow. Must manage cache information in the application. Best way to manage cache is to use a stream of db changes to notify if cache should be updated. This is called a stream-table join

### Streaming Join

Also called a windowed-join.

Example: one stream with search queries and on stream with clicks (including clicks on search results). If user_id is the key for both streams, Kafka Streams can read from correct partitions and perform window-join in embedded RocksDB cache

### Out-of-Sequence Events

Stream apps have to know how to handle out-of-sequence events:
- Recognize that event is out of sequence (examine time)
- Define a time period during which it will reconcile these events (e.g. throw away records over 3 days old)
- have capability to reconcile the event
- Be aple to update results

### Reprocessing

Needed if we have a new version of our app or if our app is buggy and requires fixes with re-runs

In Kafka, with new app, you can read from earliest possible offset

## Kafka Streams

2 streams APIS: low-level Processor API and high-level Streams DSL (which covers most use cases

App that uses DSL API starts with using `StreamBuilder` to create a processing topology (directed graph) of transformations that are applied to events in the stream.

Then you create `KafkaStreams` execution object from the topology. This starts multi threads processing

Kafka Streams application must have a unique app ID

KS always reads data from topics and write output to topics. Thusly, must provide boostrap servers

Must always provide serializer and deserializer. default is Serde classes

## Kafka Streams: Architecture

### Building a Topology

Every stream application implements/executes at least one topology: a set of operations and transitions that every event moves through from input to output. Think filter, map, aggregate, etc.

Topology always starts with one or more source processors and finishes with one or more sink processors

### Scaling the Topology

Kafka Streams automatically balances work between multiple threads or instances.

Streams engine splits topology into tasks. Each task is responsible for a subset of the partitions. The task subscribes to those partitions and consumes events. Task applies all the processing and writes to sink.

You can have as many tasks as you have partitions in the topics you are processing.

Kafka requires that all topics that participate in a join operation have the same number of partitions and be partitioned on the join key

If you need to use a different key, you can repartition, and write the events to a new topic. Then you can read from the repartitioned topic for processing

### Surviving Failures

Will look up its last position in the stream if app restarts

If a task failed but there are other instances/threads that are active, they will take on the task

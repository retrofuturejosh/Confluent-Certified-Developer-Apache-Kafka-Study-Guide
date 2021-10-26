## Table of Contents
- [Kafka Ecosystem](#Kafka-Ecosystem)
  - [Kafka Connect](#Kafka-Connect)
  - [Schema Registry](#Schema-Registry)
  - [Kafka Streams](#Kafka-Streams)
  - [ksqlDB](#ksqlDB)
  - [Kafka APIs](#Kafka-APIs)

## Kafka Ecosystem
General notes
- do NOT write your own framework
- use existing infrastructure from the kafka community
- only write applications that have domain-specific, business value

#### Kafka Connect
- system for connecting non-Kafka systems to Kafka
- external client process, NOT on broker
- declarative
- scalable
- fault-tolerant
- removes need to write code
- configured with some JSON
- Connectors
  - pluggable software component
  - interfaces with external system and kafka
  - also exist as runtime entities
  - Source connectors act as Producers
  - Sink connectors act as Consumers

#### Schema Registry
- Problem:
  - new consumers will emerge for topics
  - schemas evolve with business
- Schema Registry solves this:
  - server process external to Kafka brokers
  - maintains a database of schemas
  - high availability option available
  - Consumer/Producer API component (check schema compatability)
  - defines schema compatability rules per topic
  - Producer API prevents incompatible messages from being produced
  - Consumer API prevents incompatible messages from being consumed
- Supported formats
  - JSON schema
  - Avro
  - Protocol buffers

#### Kafka Streams
- Functional Java API
- filtering, grouping, aggregating, joining
- scalable, fault-tolerant state management
- scalable computation based on Consumer groups
- integrates within your services as Library
- runs in context of your application
- does not require special infra

#### ksqlDB
- database optimized for stream processing
- runs on its own scalable, fault-tolerant cluster adjacent to kafka cluster
- stream processing programs written in SQL
- command line interface
- REST API for application integration
- Java library
- Kafka Connect integration

#### Kafka APIs
- Admin API
  - manage and inspect topics, brokers, and Kafka objects
- Producer API
  - write a stream of events to one or more topics
- Consumer API
  - read and process one or more topics' events
- Kafka Streams API
  - implement stream processing applications and microservice
  - performs transformations, stateful operations (e.g. aggregations and joins, windowing, processing based on event-time)
  - read from one or more topics, generate output to one or more topics, transforming input streams to output streams
- Kafka Connect API
  - data import/export connectors that read/write streams of events from/to external systems

## Table of Contents
- [Kafka Connect](#Kafka-Connect)
  - [Running Connect](#Runnning-Connect)
  - [Deeper Look](#deeper-look)

## Kafka Connect

scalable and reliable way to move data between Kafka and other data stores. Provides APIs and runtime to run `connector` plugins

## Running Connect

Should be run on separate servers (not same as brokers)

Start script: ` bin/connect-distributed.sh config/connect-distributed.properties`

Key configurations
- `bootstrap.servers` - a list of brokers that onnect will work with. data will be piped to/from these brokers
- `group.id` - workers with the same group id are part of the same connect cluster
- `key.converter` and `value.converter` - default is JSONConverter, can also be set to AvroConverter (part of Confluent Schema Registry)

`key.converter.schema.enable=true` and  `value.converter.schema.enable` can set whether messages include a schema or not

`rest.host.name` and `rest.port` are configured through REST API of Kafka Connect

## Deeper Look

run a cluster of workers and then start/stop connectors

### Connectors and tasks

Connector API includes two parts:
- Connectors
    - determines how many tasks will run for the connector
    - decides how to split the data-copying work between the tasks
    - gets config for tasks from workers and passes it along
- Tasks
    - responsible for actually getting data in/our of Kafka
    - receives context from worker

## Workers

worker processes are the "container" processes that execute connectors and tasks

handle HTTP requests to define connectors and config, stores connector config, starts connectors/tasks, passes along config

if worker crashes, other works in Connect cluster will recognize (using heartbeats) and reassign connectors and tasks to remaining workers

if new worker joines, other workers will assign connectors or tasks and balance load

workers are responsible for committing offsets

### Offset management

Source connectors use a logical partition and logical offset depending on the source data. e.g. table and id for record in a JDBC source

Sink Connectors commit offsets similar to Consumer, using topic/partition/offset identifiers


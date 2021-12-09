Notes from Practice Tests

- Consumers are able to see a message when high watermark has advanced (NOT when message has been replicated to all replicas)

- If offset is committed before message, guarantee is At-Most-Once

- If reducing a value in a KTable, set cleanup.policy = compact

- Consumers do not directly write to the __consumer_offsets topic, they instead interact with a broker that has been elected to manage that topic, which is the Group Coordinator broker

- find all partitions without the leader by using `kafka-topics.sh --bootstrap-server localhost:9092 --describe --unavailable-partitions`

- In Kafka Streams, the application.id is also the underlying group.id for your consumers, and the prefix for all internal topics (repartition and state)

- Controller is a broker that in addition to usual broker functions is responsible for partition leader election. The election of that broker happens thanks to Zookeeper and at any time only one broker can be a controller


- KSQL is based on Kafka Streams and allows you to express transformations in the SQL language that get automatically converted to a Kafka Streams program in the backend


- subscribe() and assign() cannot be called by the same consumer, subscribe() is used to leverage the consumer group mechanism, while assign() is used to manually control partition assignment and reads assignment


- KafkaConsumer is NOT thread-safe, KafkaProducer IS thread safe

- stateless operators:
  - branch
  - filter
  - inverse filter
  - flatmap
  - foreach
  - groupbykey
  - groupby
  - map
  - peek
  - print
  - selectkey
  - table to stream

- stateful operations:
  - Aggregate
  - Count
  - Reduce
  - Join

- time windows:
  - Tumbling: Fixed-size, non-overlapping, gap-less windows
  - Hopping: Fixed-size, overlapping windows
  - Sliding: Fixed-size, overlapping windows that work on differences between record timestamps
  - Session: Dynamically-sized, non-overlapping, data-driven windows


- review avro schema changes

- JDBC connector allows one task per table

-

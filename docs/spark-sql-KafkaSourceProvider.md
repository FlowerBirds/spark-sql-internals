# KafkaSourceProvider

[[shortName]]
`KafkaSourceProvider` is a <<spark-sql-DataSourceRegister.md#, DataSourceRegister>> and registers itself to handle *kafka* data source format.

`KafkaSourceProvider` uses `META-INF/services/org.apache.spark.sql.sources.DataSourceRegister` file for the registration (available in the [source code]({{ spark.github }}/external/kafka-0-10-sql/src/main/resources/META-INF/services/org.apache.spark.sql.sources.DataSourceRegister) of Apache Spark).

`KafkaSourceProvider` is a <<createRelation-RelationProvider, RelationProvider>> and a <<createRelation-CreatableRelationProvider, CreatableRelationProvider>>.

```scala
scala> val fromKafkaTopic1 = spark.
  read.
  format("kafka").
  option("subscribe", "topic1").  // subscribe, subscribepattern, or assign
  option("kafka.bootstrap.servers", "localhost:9092").
  load("gauge_one")
```

`KafkaSourceProvider` <<sourceSchema, uses a fixed schema>> (and makes sure that a user did not set a custom one).

[source, scala]
----
import org.apache.spark.sql.types.StructType
val schema = new StructType().add($"id".int)
scala> spark
  .read
  .format("kafka")
  .option("subscribe", "topic1")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .schema(schema) // <-- defining a custom schema is not supported
  .load
org.apache.spark.sql.AnalysisException: kafka does not allow user-specified schemas.;
  at org.apache.spark.sql.execution.datasources.DataSource.resolveRelation(DataSource.scala:307)
  at org.apache.spark.sql.DataFrameReader.load(DataFrameReader.scala:178)
  at org.apache.spark.sql.DataFrameReader.load(DataFrameReader.scala:146)
  ... 48 elided
----

[NOTE]
====
`KafkaSourceProvider` is also a `StreamSourceProvider`, a `StreamSinkProvider`, a `StreamWriteSupport` and a `ContinuousReadSupport` that are contracts used in Spark Structured Streaming.

You can find more on Spark Structured Streaming in my gitbook https://jaceklaskowski.gitbooks.io/spark-structured-streaming/[Spark Structured Streaming].
====

=== [[createRelation-RelationProvider]] Creating BaseRelation -- `createRelation` Method (from RelationProvider)

[source, scala]
----
createRelation(
  sqlContext: SQLContext,
  parameters: Map[String, String]): BaseRelation
----

NOTE: `createRelation` is part of <<spark-sql-RelationProvider.md#createRelation, RelationProvider Contract>> to create a <<spark-sql-BaseRelation.md#, BaseRelation>> (for reading or writing).

`createRelation` starts by <<validateBatchOptions, validating the Kafka options (for batch queries)>> in the input `parameters`.

`createRelation` collects all ``kafka.``-prefixed key options (in the input `parameters`) and creates a local `specifiedKafkaParams` with the keys without the `kafka.` prefix (e.g. `kafka.whatever` is simply `whatever`).

`createRelation` <<getKafkaOffsetRangeLimit, gets the desired KafkaOffsetRangeLimit>> with the `startingoffsets` offset option key (in the given `parameters`) and <<spark-sql-KafkaOffsetRangeLimit.md#EarliestOffsetRangeLimit, EarliestOffsetRangeLimit>> as the default offsets.

`createRelation` makes sure that the <<spark-sql-KafkaOffsetRangeLimit.md#, KafkaOffsetRangeLimit>> is not <<spark-sql-KafkaOffsetRangeLimit.md#EarliestOffsetRangeLimit, EarliestOffsetRangeLimit>> or throws an `AssertionError`.

`createRelation` <<getKafkaOffsetRangeLimit, gets the desired KafkaOffsetRangeLimit>>, but this time with the `endingoffsets` offset option key (in the given `parameters`) and <<spark-sql-KafkaOffsetRangeLimit.md#LatestOffsetRangeLimit, LatestOffsetRangeLimit>> as the default offsets.

`createRelation` makes sure that the <<spark-sql-KafkaOffsetRangeLimit.md#, KafkaOffsetRangeLimit>> is not <<spark-sql-KafkaOffsetRangeLimit.md#EarliestOffsetRangeLimit, EarliestOffsetRangeLimit>> or throws a `AssertionError`.

In the end, `createRelation` creates a <<spark-sql-KafkaRelation.md#creating-instance, KafkaRelation>> with the <<strategy, subscription strategy>> (in the given `parameters`), <<failOnDataLoss, failOnDataLoss>> option, and the starting and ending offsets.

=== [[validateBatchOptions]] Validating Kafka Options (for Batch Queries) -- `validateBatchOptions` Internal Method

[source, scala]
----
validateBatchOptions(caseInsensitiveParams: Map[String, String]): Unit
----

`validateBatchOptions` <<getKafkaOffsetRangeLimit, gets the desired KafkaOffsetRangeLimit>> for the [startingoffsets](datasource/kafka/options.md#startingoffsets) option in the input `caseInsensitiveParams` and with <<spark-sql-KafkaOffsetRangeLimit.md#EarliestOffsetRangeLimit, EarliestOffsetRangeLimit>> as the default `KafkaOffsetRangeLimit`.

`validateBatchOptions` then matches the returned <<spark-sql-KafkaOffsetRangeLimit.md#, KafkaOffsetRangeLimit>> as follows:

. <<spark-sql-KafkaOffsetRangeLimit.md#EarliestOffsetRangeLimit, EarliestOffsetRangeLimit>> is acceptable and `validateBatchOptions` simply does nothing

. <<spark-sql-KafkaOffsetRangeLimit.md#LatestOffsetRangeLimit, LatestOffsetRangeLimit>> is not acceptable and `validateBatchOptions` throws an `IllegalArgumentException`:
+
```
starting offset can't be latest for batch queries on Kafka
```

. <<spark-sql-KafkaOffsetRangeLimit.md#SpecificOffsetRangeLimit, SpecificOffsetRangeLimit>> is acceptable unless one of the offsets is <<spark-sql-KafkaOffsetRangeLimit.md#LATEST, -1L>> for which `validateBatchOptions` throws an `IllegalArgumentException`:
+
```
startingOffsets for [tp] can't be latest for batch queries on Kafka
```

NOTE: `validateBatchOptions` is used exclusively when `KafkaSourceProvider` is requested to <<createRelation-RelationProvider, create a BaseRelation>> (as a <<spark-sql-RelationProvider.md#createRelation, RelationProvider>>).

=== [[createRelation-CreatableRelationProvider]] Writing DataFrame to Kafka Topic -- `createRelation` Method (from CreatableRelationProvider)

[source, scala]
----
createRelation(
  sqlContext: SQLContext,
  mode: SaveMode,
  parameters: Map[String, String],
  df: DataFrame): BaseRelation
----

NOTE: `createRelation` is part of the <<spark-sql-CreatableRelationProvider.md#createRelation, CreatableRelationProvider Contract>> to write the rows of a structured query (a DataFrame) to an external data source.

`createRelation` gets the [topic](datasource/kafka/options.md#topic) option from the input `parameters`.

`createRelation` gets the <<kafkaParamsForProducer, Kafka-specific options for writing>> from the input `parameters`.

`createRelation` then uses the `KafkaWriter` helper object to <<spark-sql-KafkaWriter.md#write, write the rows of the DataFrame to the Kafka topic>>.

In the end, `createRelation` creates a fake <<spark-sql-BaseRelation.md#, BaseRelation>> that simply throws an `UnsupportedOperationException` for all its methods.

`createRelation` supports <<spark-sql-CreatableRelationProvider.md#Append, Append>> and <<spark-sql-CreatableRelationProvider.md#ErrorIfExists, ErrorIfExists>> only. `createRelation` throws an `AnalysisException` for the other save modes:

```
Save mode [mode] not allowed for Kafka. Allowed save modes are [Append] and [ErrorIfExists] (default).
```

=== [[sourceSchema]] `sourceSchema` Method

[source, scala]
----
sourceSchema(
  sqlContext: SQLContext,
  schema: Option[StructType],
  providerName: String,
  parameters: Map[String, String]): (String, StructType)
----

`sourceSchema`...FIXME

[source, scala]
----
val fromKafka = spark.read.format("kafka")...
scala> fromKafka.printSchema
root
 |-- key: binary (nullable = true)
 |-- value: binary (nullable = true)
 |-- topic: string (nullable = true)
 |-- partition: integer (nullable = true)
 |-- offset: long (nullable = true)
 |-- timestamp: timestamp (nullable = true)
 |-- timestampType: integer (nullable = true)
----

NOTE: `sourceSchema` is part of Structured Streaming's `StreamSourceProvider` Contract.

=== [[getKafkaOffsetRangeLimit]] Getting Desired KafkaOffsetRangeLimit (for Offset Option) -- `getKafkaOffsetRangeLimit` Object Method

[source, scala]
----
getKafkaOffsetRangeLimit(
  params: Map[String, String],
  offsetOptionKey: String,
  defaultOffsets: KafkaOffsetRangeLimit): KafkaOffsetRangeLimit
----

`getKafkaOffsetRangeLimit` tries to find the given `offsetOptionKey` in the input `params` and converts the value found to a <<spark-sql-KafkaOffsetRangeLimit.md#, KafkaOffsetRangeLimit>> as follows:

* `latest` becomes <<spark-sql-KafkaOffsetRangeLimit.md#LatestOffsetRangeLimit, LatestOffsetRangeLimit>>

* `earliest` becomes <<spark-sql-KafkaOffsetRangeLimit.md#EarliestOffsetRangeLimit, EarliestOffsetRangeLimit>>

* For a JSON text, `getKafkaOffsetRangeLimit` uses the `JsonUtils` helper object to <<spark-sql-JsonUtils.md#partitionOffsets, read per-TopicPartition offsets from it>> and creates a <<spark-sql-KafkaOffsetRangeLimit.md#SpecificOffsetRangeLimit, SpecificOffsetRangeLimit>>

When the input `offsetOptionKey` was not found, `getKafkaOffsetRangeLimit` returns the input `defaultOffsets`.

[NOTE]
====
`getKafkaOffsetRangeLimit` is used when:

* `KafkaSourceProvider` is requested to <<validateBatchOptions, validate Kafka options (for batch queries)>> and <<createRelation-RelationProvider, create a BaseRelation>> (as a <<spark-sql-RelationProvider.md#createRelation, RelationProvider>>)

* (Spark Structured Streaming) `KafkaSourceProvider` is requested to `createSource` and `createContinuousReader`
====

=== [[strategy]] Getting ConsumerStrategy per Subscription Strategy Option -- `strategy` Internal Method

[source, scala]
----
strategy(caseInsensitiveParams: Map[String, String]): ConsumerStrategy
----

`strategy` finds one of the strategy options: [subscribe](datasource/kafka/options.md#subscribe), [subscribepattern](datasource/kafka/options.md#subscribepattern) and [assign](datasource/kafka/options.md#assign).

For [assign](datasource/kafka/options.md#assign), `strategy` uses the `JsonUtils` helper object to <<spark-sql-JsonUtils.md#partitions-String-Array, deserialize TopicPartitions from JSON>> (e.g. `{"topicA":[0,1],"topicB":[0,1]}`) and returns a new <<spark-sql-ConsumerStrategy.md#AssignStrategy, AssignStrategy>>.

For [subscribe](datasource/kafka/options.md#subscribe), `strategy` splits the value by `,` (comma) and returns a new <<spark-sql-ConsumerStrategy.md#SubscribeStrategy, SubscribeStrategy>>.

For [subscribepattern](datasource/kafka/options.md#subscribepattern), `strategy` returns a new <<spark-sql-ConsumerStrategy.md#SubscribePatternStrategy, SubscribePatternStrategy>>

`strategy` is used when:

* `KafkaSourceProvider` is requested to <<createRelation-RelationProvider, create a BaseRelation>> (as a <<spark-sql-RelationProvider.md#createRelation, RelationProvider>>)

* (Spark Structured Streaming) `KafkaSourceProvider` is requested to `createSource` and `createContinuousReader`

=== [[failOnDataLoss]] `failOnDataLoss` Internal Method

[source, scala]
----
failOnDataLoss(caseInsensitiveParams: Map[String, String]): Boolean
----

`failOnDataLoss`...FIXME

NOTE: `failOnDataLoss` is used when `KafkaSourceProvider` is requested to <<createRelation-RelationProvider, create a BaseRelation>> (and also in `createSource` and `createContinuousReader` for Spark Structured Streaming).

=== [[kafkaParamsForDriver]] Setting Kafka Configuration Parameters for Driver -- `kafkaParamsForDriver` Object Method

[source, scala]
----
kafkaParamsForDriver(specifiedKafkaParams: Map[String, String]): java.util.Map[String, Object]
----

`kafkaParamsForDriver` simply sets the <<kafkaParamsForDriver-Kafka-parameters, additional Kafka configuration parameters>> for the driver.

[[kafkaParamsForDriver-Kafka-parameters]]
.Driver's Kafka Configuration Parameters
[cols="1m,1m,1m,2",options="header",width="100%"]
|===
| Name
| Value
| ConsumerConfig
| Description

| key.deserializer
| org.apache.kafka.common.serialization.ByteArrayDeserializer
| KEY_DESERIALIZER_CLASS_CONFIG
| [[key.deserializer]] Deserializer class for keys that implements the Kafka `Deserializer` interface.

| value.deserializer
| org.apache.kafka.common.serialization.ByteArrayDeserializer
| VALUE_DESERIALIZER_CLASS_CONFIG
| [[value.deserializer]] Deserializer class for values that implements the Kafka `Deserializer` interface.

| auto.offset.reset
| earliest
| AUTO_OFFSET_RESET_CONFIG
a| [[auto.offset.reset]] What to do when there is no initial offset in Kafka or if the current offset does not exist any more on the server (e.g. because that data has been deleted):

* `earliest` -- automatically reset the offset to the earliest offset

* `latest` -- automatically reset the offset to the latest offset

* `none` -- throw an exception to the Kafka consumer if no previous offset is found for the consumer's group

* _anything else_ -- throw an exception to the Kafka consumer

| enable.auto.commit
| false
| ENABLE_AUTO_COMMIT_CONFIG
| [[enable.auto.commit]] If `true` the Kafka consumer's offset will be periodically committed in the background

| max.poll.records
| 1
| MAX_POLL_RECORDS_CONFIG
| [[max.poll.records]] The maximum number of records returned in a single call to `Consumer.poll()`

| receive.buffer.bytes
| 65536
| MAX_POLL_RECORDS_CONFIG
| [[receive.buffer.bytes]] Only set if not set already
|===

[[ConfigUpdater-logging]]
[TIP]
====
Enable `DEBUG` logging level for `org.apache.spark.sql.kafka010.KafkaSourceProvider.ConfigUpdater` logger to see updates of Kafka configuration parameters.

Add the following line to `conf/log4j.properties`:

```
log4j.logger.org.apache.spark.sql.kafka010.KafkaSourceProvider.ConfigUpdater=DEBUG
```

Refer to spark-logging.md[Logging].
====

[NOTE]
====
`kafkaParamsForDriver` is used when:

* `KafkaRelation` is requested to <<spark-sql-KafkaRelation.md#buildScan, build a distributed data scan with column pruning>> (as a <<spark-sql-TableScan.md#, TableScan>>)

* (Spark Structured Streaming) `KafkaSourceProvider` is requested to `createSource` and `createContinuousReader`
====

=== [[kafkaParamsForExecutors]] `kafkaParamsForExecutors` Object Method

[source, scala]
----
kafkaParamsForExecutors(
  specifiedKafkaParams: Map[String, String],
  uniqueGroupId: String): java.util.Map[String, Object]
----

`kafkaParamsForExecutors`...FIXME

NOTE: `kafkaParamsForExecutors` is used when...FIXME

=== [[kafkaParamsForProducer]] `kafkaParamsForProducer` Object Method

[source, scala]
----
kafkaParamsForProducer(parameters: Map[String, String]): Map[String, String]
----

`kafkaParamsForProducer`...FIXME

NOTE: `kafkaParamsForProducer` is used when...FIXME

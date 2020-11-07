# Kafka Data Source Options

[[options]]
.Kafka Data Source Options
[cols="1m,1,2",options="header",width="100%"]
|===
| Option
| Default
| Description

| assign
|
| [[assign]] One of the three subscription strategy options (with [subscribe](options.md#subscribe) and [subscribepattern](options.md#subscribepattern))

See [KafkaSourceProvider.strategy](../../spark-sql-KafkaSourceProvider.md#strategy)

| endingoffsets
|
| [[endingoffsets]]

| failondataloss
|
| [[failondataloss]]

| kafkaConsumer.pollTimeoutMs
|
| [[kafkaConsumer.pollTimeoutMs]] See <<spark-sql-KafkaRelation.md#pollTimeoutMs, kafkaConsumer.pollTimeoutMs>>

| startingoffsets
|
| [[startingoffsets]]

| subscribe
|
| [[subscribe]] One of the three subscription strategy options (with [subscribepattern](options.md#subscribepattern) and [assign](options.md#assign))

See <<spark-sql-KafkaSourceProvider.md#strategy, KafkaSourceProvider.strategy>>

| subscribepattern
|
| [[subscribepattern]] One of the three subscription strategy options (with [subscribe](options.md#subscribe) and [assign](options.md#assign))

See [KafkaSourceProvider.strategy](../../spark-sql-KafkaSourceProvider.md#strategy)

| topic
|
a| [[topic]] *Required* for writing a DataFrame to Kafka

Used when:

* `KafkaSourceProvider` is requested to <<spark-sql-KafkaSourceProvider.md#createRelation-CreatableRelationProvider, write a DataFrame to a Kafka topic and create a BaseRelation afterwards>>

* (Spark Structured Streaming) `KafkaSourceProvider` is requested to `createStreamWriter` and `createSink`
|===

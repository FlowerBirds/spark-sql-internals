# DataSource &mdash; Pluggable Data Provider Framework

`DataSource` paves the way for **Pluggable Data Provider Framework** in Spark SQL.

Together with the <<providers, provider interfaces>>, `DataSource` allows Spark SQL integrators to use external data systems as data sources and sinks in structured queries in Spark SQL (incl. Spark Structured Streaming).

[[providers]]
.Provider Interfaces
[cols="1,3",options="header",width="100%"]
|===
| Interface
| Description

| [CreatableRelationProvider](CreatableRelationProvider.md)
| [[CreatableRelationProvider]] Saves the result of a structured query per save mode and returns the schema

| [FileFormat](FileFormat.md)
a| [[FileFormat]]

| RelationProvider.md[RelationProvider]
| [[RelationProvider]] Supports schema inference and can be referenced in SQL's `USING` clause

| spark-sql-SchemaRelationProvider.md[SchemaRelationProvider]
| [[SchemaRelationProvider]] Requires a user-defined schema

| StreamSinkProvider
| [[StreamSinkProvider]] Used in Structured Streaming

| StreamSourceProvider
| [[StreamSourceProvider]] Used in Structured Streaming

|===

NOTE: Data source is also called a *table provider*.

`DataSource` requires an <<className, alias or a fully-qualified class name>> of the data source provider (among <<creating-instance, other optional parameters>>). `DataSource` uses the name  to <<lookupDataSource, load the Java class>> (available as <<providingClass, providingClass>> internally). Eventually, `DataSource` uses the Java class to <<resolveRelation, resolve a relation>> (the [BaseRelation](BaseRelation.md)) to represent the data source in logical plans (using LogicalRelation.md[LogicalRelation] leaf logical operator).

`DataSource` also requires a <<sparkSession, SparkSession>> for the configuration properties to <<lookupDataSource, resolve the data source provider>>.

`DataSource` is <<creating-instance, created>> when:

* `HiveMetastoreCatalog` is requested to hive/HiveMetastoreCatalog.md#convertToLogicalRelation[convert a HiveTableRelation to a LogicalRelation over a HadoopFsRelation]

* `DataFrameReader` is requested to [load data from a data source (Data Source V1)](DataFrameReader.md#loadV1Source)

* `DataFrameWriter` is requested to [save to a data source (Data Source V1)](DataFrameWriter.md#saveToV1Source)

* [CreateDataSourceTableCommand](logical-operators/CreateDataSourceTableCommand.md), [CreateDataSourceTableAsSelectCommand](logical-operators/CreateDataSourceTableAsSelectCommand.md), [InsertIntoDataSourceDirCommand](logical-operators/InsertIntoDataSourceDirCommand.md), [CreateTempViewUsing](logical-operators/CreateTempViewUsing.md) commands are executed

* [FindDataSourceTable](logical-analysis-rules/FindDataSourceTable.md) and [ResolveSQLOnFile](logical-analysis-rules/ResolveSQLOnFile.md) logical evaluation rules are executed

* For Spark Structured Streaming's `FileStreamSource`, `DataStreamReader` and `DataStreamWriter`

`DataSource` takes a list of <<paths, file system paths that hold data>>. The list is empty by default, but can be different per data source:

* The [location URI](CatalogTable.md#location) of a [HiveTableRelation](hive/HiveTableRelation.md) (when `HiveMetastoreCatalog` is requested to [convert a HiveTableRelation to a LogicalRelation over a HadoopFsRelation](hive/HiveMetastoreCatalog.md#convertToLogicalRelation))

* The table name of a <<UnresolvedRelation.md#, UnresolvedRelation>> (when [ResolveSQLOnFile](logical-analysis-rules/ResolveSQLOnFile.md) logical evaluation rule is executed)

* The files in a directory when Spark Structured Streaming's `FileStreamSource` is requested for batches

As a Spark SQL developer (_user_), you interact with `DataSource` by [DataFrameReader](DataFrameReader.md) (when you execute SparkSession.md#read[spark.read] or SparkSession.md#readStream[spark.readStream]) or SQL's `CREATE TABLE USING`.

[source, scala]
----
// Batch reading
val people: DataFrame = spark.read
  .format("csv")
  .load("people.csv")

// Streamed reading
val messages: DataFrame = spark.readStream
  .format("kafka")
  .option("subscribe", "topic")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .load
----

When requested to <<resolveRelation, resolve a batch (non-streaming) FileFormat>>, `DataSource` creates a [HadoopFsRelation](HadoopFsRelation.md) with the optional [bucketing specification](#bucketSpec).

=== [[creating-instance]][[apply]] Creating DataSource Instance

`DataSource` takes the following to be created:

* [[sparkSession]] SparkSession.md[SparkSession]
* [[className]] Fully-qualified class name or an alias of the data source provider (aka _data source format_)
* [[paths]] Data paths (default: empty)
* [[userSpecifiedSchema]] (optional) User-specified [schema](StructType.md) (default: undefined)
* [[partitionColumns]] (optional) Names of the partition columns (default: empty)
* [[bucketSpec]] (optional) spark-sql-BucketSpec.md[Bucketing specification] (default: undefined)
* [[options]] (optional) Options (default: empty)
* [[catalogTable]] (optional) [CatalogTable](CatalogTable.md) (default: undefined)

`DataSource` initializes the <<internal-properties, internal properties>>.

NOTE: Only the <<sparkSession, SparkSession>> and the <<className, fully-qualified class name of the data source provider>> are required to create an instance of `DataSource`.

=== [[lookupDataSource]] Loading Java Class Of Data Source Provider -- `lookupDataSource` Utility

[source, scala]
----
lookupDataSource(
  provider: String,
  conf: SQLConf): Class[_]
----

[[lookupDataSource-provider1]]
`lookupDataSource` first finds the given `provider` in the <<backwardCompatibilityMap, backwardCompatibilityMap>> internal registry, and falls back to the `provider` name itself when not found.

NOTE: The `provider` argument can be either an alias (a simple name, e.g. `parquet`) or a fully-qualified class name (e.g. `org.apache.spark.sql.execution.datasources.parquet.ParquetFileFormat`).

`lookupDataSource` then uses the given [SQLConf](SQLConf.md) to decide on the class name of the provider for ORC and Avro data sources as follows:

* For `orc` provider and [native](SQLConf.md#ORC_IMPLEMENTATION), `lookupDataSource` uses the new ORC file format [OrcFileFormat](OrcFileFormat.md) (based on Apache ORC)

* For `orc` provider and [hive](SQLConf.md#ORC_IMPLEMENTATION), `lookupDataSource` uses `org.apache.spark.sql.hive.orc.OrcFileFormat`

* For `com.databricks.spark.avro` and [spark.sql.legacy.replaceDatabricksSparkAvro.enabled](SQLConf.md#replaceDatabricksSparkAvroEnabled) configuration enabled (default), `lookupDataSource` uses the built-in (but external) [Avro data source](AvroFileFormat.md) module

[[lookupDataSource-provider2]]
`lookupDataSource` uses `DefaultSource` as the class name (in the <<lookupDataSource-provider1, provider1>> package) as another provider name variant, i.e. `[provider1].DefaultSource`.

[[lookupDataSource-serviceLoader]]
`lookupDataSource` uses Java's [ServiceLoader]({{ java.api }}/java/util/ServiceLoader.html) service-provider loading facility to find all data source providers of type [DataSourceRegister](DataSourceRegister.md) on the Spark CLASSPATH.

NOTE: [DataSourceRegister](DataSourceRegister.md) is used to register a data source provider by a short name (_alias_).

`lookupDataSource` tries to find the `DataSourceRegister` provider classes (by their [alias](DataSourceRegister.md#shortName)) that match the <<lookupDataSource-provider1, provider1>> name (case-insensitive, e.g. `parquet` or `kafka`).

If a single `DataSourceRegister` provider class is found, `lookupDataSource` simply returns the instance of the data source provider.

If no `DataSourceRegister` provider class could be found by the short name (alias), `lookupDataSource` tries to load the <<lookupDataSource-provider1, provider1>> name to be a fully-qualified class name. If not successful, `lookupDataSource` tries to load the <<lookupDataSource-provider2, provider2>> name (aka _DefaultSource_) instead.

!!! note
    [DataFrameWriter.format](DataFrameWriter.md#format) and [DataFrameReader.format](DataFrameReader.md#format) methods accept the name of the data source provider to use as an alias or a fully-qualified class name.

```text
import org.apache.spark.sql.execution.datasources.DataSource
val source = "parquet"
val cls = DataSource.lookupDataSource(source, spark.sessionState.conf)
```

CAUTION: FIXME Describe error paths (`case Failure(error)` and `case sources`).

`lookupDataSource` is used when:

* [DataFrameReader.load](DataFrameReader.md#load) operator is used (to create a source node)

* [DataFrameWriter.save](DataFrameWriter.md#save) operator is used (to create a sink node)

* (Structured Streaming) `DataStreamReader.load` operator is used

* (Structured Streaming) `DataStreamWriter.start` operator is used

* `AlterTableAddColumnsCommand` command is executed

* `DataSource` is requested (_lazily_) for the <<providingClass, providingClass>> internal registry

* [PreprocessTableCreation](logical-analysis-rules/PreprocessTableCreation.md) posthoc logical resolution rule is executed

=== [[createSource]] `createSource` Method

[source, scala]
----
createSource(
  metadataPath: String): Source
----

`createSource`...FIXME

NOTE: `createSource` is used when...FIXME

=== [[createSink]] `createSink` Method

[source, scala]
----
createSink(
  outputMode: OutputMode): Sink
----

`createSink`...FIXME

NOTE: `createSink` is used when...FIXME

=== [[sourceSchema]] `sourceSchema` Internal Method

[source, scala]
----
sourceSchema(): SourceInfo
----

`sourceSchema` returns the name and spark-sql-schema.md[schema] of the data source for streamed reading.

CAUTION: FIXME Why is the method called? Why does this bother with streamed reading and data sources?!

It supports two class hierarchies, i.e. [FileFormat](FileFormat.md) and Structured Streaming's `StreamSourceProvider` data sources.

Internally, `sourceSchema` first creates an instance of the data source and...

CAUTION: FIXME Finish...

For Structured Streaming's `StreamSourceProvider` data sources, `sourceSchema` relays calls to `StreamSourceProvider.sourceSchema`.

For [FileFormat](FileFormat.md) data sources, `sourceSchema` makes sure that `path` option was specified.

TIP: `path` is looked up in a case-insensitive way so `paTh` and `PATH` and `pAtH` are all acceptable. Use the lower-case version of `path`, though.

NOTE: `path` can use https://en.wikipedia.org/wiki/Glob_%28programming%29[glob pattern] (not regex syntax), i.e. contain any of `{}[]*?\` characters.

It checks whether the path exists if a glob pattern is not used. In case it did not exist you will see the following `AnalysisException` exception in the logs:

```
scala> spark.read.load("the.file.does.not.exist.parquet")
org.apache.spark.sql.AnalysisException: Path does not exist: file:/Users/jacek/dev/oss/spark/the.file.does.not.exist.parquet;
  at org.apache.spark.sql.execution.datasources.DataSource$$anonfun$12.apply(DataSource.scala:375)
  at org.apache.spark.sql.execution.datasources.DataSource$$anonfun$12.apply(DataSource.scala:364)
  at scala.collection.TraversableLike$$anonfun$flatMap$1.apply(TraversableLike.scala:241)
  at scala.collection.TraversableLike$$anonfun$flatMap$1.apply(TraversableLike.scala:241)
  at scala.collection.immutable.List.foreach(List.scala:381)
  at scala.collection.TraversableLike$class.flatMap(TraversableLike.scala:241)
  at scala.collection.immutable.List.flatMap(List.scala:344)
  at org.apache.spark.sql.execution.datasources.DataSource.resolveRelation(DataSource.scala:364)
  at org.apache.spark.sql.DataFrameReader.load(DataFrameReader.scala:149)
  at org.apache.spark.sql.DataFrameReader.load(DataFrameReader.scala:132)
  ... 48 elided
```

If [spark.sql.streaming.schemaInference](configuration-properties.md#spark.sql.streaming.schemaInference) is disabled and the data source is different than [TextFileFormat](TextFileFormat.md), and the input `userSpecifiedSchema` is not specified, the following `IllegalArgumentException` exception is thrown:

```text
Schema must be specified when creating a streaming source DataFrame. If some files already exist in the directory, then depending on the file format you may be able to create a static DataFrame on that directory with 'spark.read.load(directory)' and infer schema from it.
```

CAUTION: FIXME I don't think the exception will ever happen for non-streaming sources since the schema is going to be defined earlier. When?

Eventually, it returns a `SourceInfo` with `FileSource[path]` and the schema (as calculated using the <<inferFileFormatSchema, inferFileFormatSchema>> internal method).

For any other data source, it throws `UnsupportedOperationException` exception:

```
Data source [className] does not support streamed reading
```

NOTE: `sourceSchema` is used exclusively when `DataSource` is requested for the <<sourceInfo, sourceInfo>>.

## <span id="resolveRelation"> Resolving Relation

```scala
resolveRelation(
  checkFilesExist: Boolean = true): BaseRelation
```

`resolveRelation` resolves (creates) a [BaseRelation](BaseRelation.md).

Internally, `resolveRelation` creates an instance of the [providingClass](#providingClass) and branches based on type and whether the [user-defined schema](#userSpecifiedSchema) was specified or not.

.Resolving BaseRelation per Provider and User-Specified Schema
[cols="1,3",options="header",width="100%"]
|===
| Provider
| Behaviour

| spark-sql-SchemaRelationProvider.md[SchemaRelationProvider]
| Executes spark-sql-SchemaRelationProvider.md#createRelation[SchemaRelationProvider.createRelation] with the provided schema

| RelationProvider.md[RelationProvider]
| Executes RelationProvider.md#createRelation[RelationProvider.createRelation]

| [FileFormat](FileFormat.md)
| Creates a [HadoopFsRelation](BaseRelation.md#HadoopFsRelation)
|===

`resolveRelation` is used when:

* `DataSource` is requested to <<writeAndRead, write and read>> the result of a structured query (only when <<providingClass, providingClass>> is a [FileFormat](FileFormat.md))

* `DataFrameReader` is requested to [load data from a data source that supports multiple paths](DataFrameReader.md#load)

* `TextInputCSVDataSource` and `TextInputJsonDataSource` are requested to infer schema

* `CreateDataSourceTableCommand` runnable command is CreateDataSourceTableCommand.md#run[executed]

* `CreateTempViewUsing` logical command is requested to <<CreateTempViewUsing.md#run, run>>

* `FindDataSourceTable` is requested to [readDataSourceTable](logical-analysis-rules/FindDataSourceTable.md#readDataSourceTable)

* `ResolveSQLOnFile` is requested to convert a logical plan (when <<providingClass, providingClass>> is a [FileFormat](FileFormat.md))

* `HiveMetastoreCatalog` is requested to hive/HiveMetastoreCatalog.md#convertToLogicalRelation[convert a HiveTableRelation to a LogicalRelation over a HadoopFsRelation]

* Structured Streaming's `FileStreamSource` creates batches of records

=== [[buildStorageFormatFromOptions]] `buildStorageFormatFromOptions` Utility

[source, scala]
----
buildStorageFormatFromOptions(
  options: Map[String, String]): CatalogStorageFormat
----

`buildStorageFormatFromOptions`...FIXME

NOTE: `buildStorageFormatFromOptions` is used when...FIXME

## <span id="planForWriting"> Creating Logical Command for Writing (for CreatableRelationProvider and FileFormat Data Sources)

```scala
planForWriting(
  mode: SaveMode,
  data: LogicalPlan): LogicalPlan
```

`planForWriting` creates an instance of the [providingClass](#providingClass) and branches off per type as follows:

* For a [CreatableRelationProvider](CreatableRelationProvider.md), `planForWriting` creates a <<SaveIntoDataSourceCommand.md#creating-instance, SaveIntoDataSourceCommand>> (with the input `data` and `mode`, the `CreatableRelationProvider` data source and the <<caseInsensitiveOptions, caseInsensitiveOptions>>)

* For a [FileFormat](FileFormat.md), `planForWriting` [planForWritingFileFormat](#planForWritingFileFormat) (with the `FileFormat` format and the input `mode` and `data`)

* For other types, `planForWriting` simply throws a `RuntimeException`:

    ```text
    [providingClass] does not allow create table as select.
    ```

`planForWriting` is used when:

* `DataFrameWriter` is requested to [save (to a data source V1](DataFrameWriter.md#saveToV1Source)
* [InsertIntoDataSourceDirCommand](logical-operators/InsertIntoDataSourceDirCommand.md) logical command is executed

## <span id="writeAndRead"> Writing Data to Data Source (per Save Mode) Followed by Reading Rows Back (as BaseRelation)

```scala
writeAndRead(
  mode: SaveMode,
  data: LogicalPlan,
  outputColumnNames: Seq[String],
  physicalPlan: SparkPlan): BaseRelation
```

`writeAndRead`...FIXME

!!! note
    `writeAndRead` is also known as **Create Table As Select** (CTAS) query.

`writeAndRead` is used when [CreateDataSourceTableAsSelectCommand](logical-operators/CreateDataSourceTableAsSelectCommand.md) logical command is executed.

## <span id="planForWritingFileFormat"> Planning for Writing (to FileFormat-Based Data Source)

```scala
planForWritingFileFormat(
  format: FileFormat,
  mode: SaveMode,
  data: LogicalPlan): InsertIntoHadoopFsRelationCommand
```

`planForWritingFileFormat` takes the [paths](#paths) and the `path` option (from the [caseInsensitiveOptions](#caseInsensitiveOptions)) together and (assuming that there is only one path available among the paths combined) creates a fully-qualified HDFS-compatible output path for writing.

!!! note
    `planForWritingFileFormat` uses Hadoop HDFS's Hadoop [Path]({{ hadoop.api }}/org/apache/hadoop/fs/Path.html) to requests for the [FileSystem]({{ hadoop.api }}/org/apache/hadoop/fs/FileSystem.html) that owns it (using a [Hadoop Configuration](SessionState.md#newHadoopConf)).

`planForWritingFileFormat` [validates partition columns](spark-sql-PartitioningUtils.md#validatePartitionColumn) in the given [partitionColumns](#partitionColumns).

In the end, `planForWritingFileFormat` returns a new [InsertIntoHadoopFsRelationCommand](logical-operators/InsertIntoHadoopFsRelationCommand.md).

`planForWritingFileFormat` throws an `IllegalArgumentException` when there are more than one [path](#paths) specified:

```text
Expected exactly one path to be specified, but got: [allPaths]
```

`planForWritingFileFormat` is used when `DataSource` is requested for the following:

* [Writing data to a data source followed by "reading" rows back](#writeAndRead) (for [CreateDataSourceTableAsSelectCommand](logical-operators/CreateDataSourceTableAsSelectCommand.md) logical command)

* [Creating a logical command for writing](#planForWriting) (for [InsertIntoDataSourceDirCommand](logical-operators/InsertIntoDataSourceDirCommand.md) logical command and [DataFrameWriter.save](DataFrameWriter.md#save) operator with DataSource V1 data sources)

## <span id="getOrInferFileFormatSchema"> getOrInferFileFormatSchema

```scala
getOrInferFileFormatSchema(
  format: FileFormat,
  fileIndex: Option[InMemoryFileIndex] = None): (StructType, StructType)
```

`getOrInferFileFormatSchema`...FIXME

`getOrInferFileFormatSchema` is used when `DataSource` is requested for the [sourceSchema](#sourceSchema) and to [resolve a non-streaming FileFormat-based relation](#resolveRelation).

=== [[checkAndGlobPathIfNecessary]] `checkAndGlobPathIfNecessary` Internal Method

[source, scala]
----
checkAndGlobPathIfNecessary(
  checkEmptyGlobPath: Boolean,
  checkFilesExist: Boolean): Seq[Path]
----

`checkAndGlobPathIfNecessary`...FIXME

NOTE: `checkAndGlobPathIfNecessary` is used when...FIXME

=== [[createInMemoryFileIndex]] `createInMemoryFileIndex` Internal Method

[source, scala]
----
createInMemoryFileIndex(
  globbedPaths: Seq[Path]): InMemoryFileIndex
----

`createInMemoryFileIndex`...FIXME

NOTE: `createInMemoryFileIndex` is used when `DataSource` is requested to <<getOrInferFileFormatSchema, getOrInferFileFormatSchema>> and <<resolveRelation, resolve a non-streaming FileFormat-based relation>>.

=== [[internal-properties]] Internal Properties

[cols="30m,70",options="header",width="100%"]
|===
| Name
| Description

| providingClass
a| [[providingClass]] https://docs.oracle.com/javase/8/docs/api/java/lang/Class.html[java.lang.Class] that was <<lookupDataSource, loaded>> for the given <<className, data source provider>>

Used when:

* `DataSource` is requested to <<sourceSchema, sourceSchema>>, <<createSource, createSource>>, <<createSink, createSink>>, <<resolveRelation, resolveRelation>>, <<writeAndRead, writeAndRead>>, and <<planForWriting, planForWriting>>

* InsertIntoDataSourceDirCommand.md[InsertIntoDataSourceDirCommand] logical command and [ResolveSQLOnFile](logical-analysis-rules/ResolveSQLOnFile.md) logical evaluation rule are executed (to ensure that only [FileFormat](FileFormat.md)-based data sources are used)

| sourceInfo
| [[sourceInfo]] `SourceInfo`

Used when...FIXME

| caseInsensitiveOptions
| [[caseInsensitiveOptions]] FIXME

Used when...FIXME

| equality
| [[equality]] FIXME

Used when...FIXME

| backwardCompatibilityMap
| [[backwardCompatibilityMap]] Names of the data sources that are no longer available but should still be accepted (<<lookupDataSource, "resolvable">>) for backward-compatibility

|===

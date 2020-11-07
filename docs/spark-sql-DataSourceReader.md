# DataSourceReader

`DataSourceReader` is the <<contract, abstraction>> of <<implementations, data source readers>> in [DataSource V2](new-and-noteworthy/datasource-v2.md) that can <<planInputPartitions, plan InputPartitions>> and know the <<readSchema, schema for reading>>.

`DataSourceReader` is created to scan the data from a data source when:

* `DataSourceV2Relation` is requested to <<DataSourceV2Relation.md#newReader, create a new reader>>

* `ReadSupport` is requested to <<spark-sql-ReadSupport.md#createReader, create a reader>>

`DataSourceReader` is used to create `StreamingDataSourceV2Relation` and <<DataSourceV2ScanExec.md#, DataSourceV2ScanExec>> physical operator

NOTE: It _appears_ that all concrete <<implementations, data source readers>> are used in Spark Structured Streaming only.

[[contract]]
.DataSourceReader Contract
[cols="30m,70",options="header",width="100%"]
|===
| Method
| Description

| planInputPartitions
a| [[planInputPartitions]]

[source, java]
----
List<InputPartition<InternalRow>> planInputPartitions()
----

[InputPartitions](connector/InputPartition.md)

Used exclusively when `DataSourceV2ScanExec` leaf physical operator is requested for the <<DataSourceV2ScanExec.md#partitions, input partitions>> (and simply delegates to the underlying <<DataSourceV2ScanExec.md#reader, DataSourceReader>>) to create the input `RDD[InternalRow]` (`inputRDD`)

| readSchema
a| [[readSchema]]

```java
StructType readSchema()
```

[Schema](StructType.md) for reading (loading) data from a data source

Used when:

* `DataSourceV2Relation` factory object is requested to <<DataSourceV2Relation.md#create, create a DataSourceV2Relation>> (when `DataFrameReader` is requested to ["load" data (as a DataFrame)](DataFrameReader.md#load) from a data source with [ReadSupport](spark-sql-ReadSupport.md))

* `DataSourceV2Strategy` execution planning strategy is requested to [apply column pruning optimization](execution-planning-strategies/DataSourceV2Strategy.md#pruneColumns)

* Spark Structured Streaming's `MicroBatchExecution` stream execution is requested to run a single streaming batch

* Spark Structured Streaming's `ContinuousExecution` stream execution is requested to run a streaming query in continuous mode

* Spark Structured Streaming's `DataStreamReader` is requested to "load" data (as a DataFrame)

|===

[NOTE]
====
`DataSourceReader` is an `Evolving` contract that is evolving towards becoming a stable API, but is not a stable API yet and can change from one feature release to another release.

In other words, using the contract is as _"treading on thin ice"_.
====

[[implementations]]
[[extensions]]
.DataSourceReaders (Direct Implementations and Extensions Only)
[cols="30,70",options="header",width="100%"]
|===
| DataSourceReader
| Description

| ContinuousReader
| [[ContinuousReader]] `DataSourceReaders` for *Continuous Stream Processing* in Spark Structured Streaming

Consult https://jaceklaskowski.gitbooks.io/spark-structured-streaming/spark-sql-streaming-ContinuousReader.html[The Internals of Spark Structured Streaming]

| MicroBatchReader
| [[MicroBatchReader]] `DataSourceReaders` for *Micro-Batch Stream Processing* in Spark Structured Streaming

Consult https://jaceklaskowski.gitbooks.io/spark-structured-streaming/spark-sql-streaming-MicroBatchReader.html[The Internals of Spark Structured Streaming]

| <<spark-sql-SupportsPushDownFilters.md#, SupportsPushDownFilters>>
| [[SupportsPushDownFilters]] `DataSourceReaders` that can push down filters to the data source and reduce the size of the data to be read

| <<spark-sql-SupportsPushDownRequiredColumns.md#, SupportsPushDownRequiredColumns>>
| [[SupportsPushDownRequiredColumns]] `DataSourceReaders` that can push down required columns to the data source and only read these columns during scan to reduce the size of the data to be read

| [SupportsReportPartitioning](spark-sql-SupportsReportPartitioning.md)
| [[SupportsReportPartitioning]] `DataSourceReaders` that can report data partitioning and try to avoid shuffle at Spark side

| <<spark-sql-SupportsReportStatistics.md#, SupportsReportStatistics>>
| [[SupportsReportStatistics]] `DataSourceReaders` that can report statistics to Spark

| <<spark-sql-SupportsScanColumnarBatch.md#, SupportsScanColumnarBatch>>
| [[SupportsScanColumnarBatch]] `DataSourceReaders` that can output [ColumnarBatch](ColumnarBatch.md) and make the scan faster

|===

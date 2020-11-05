# DataSourceV2Utils Helper Object

[[extractSessionConfigs]]
`DataSourceV2Utils` is a helper object that is used exclusively to <<extractSessionConfigs, extract session configuration options>> (i.e. options with *spark.datasource* prefix for the keys in [SQLConf](SQLConf.md)) for <<spark-sql-DataSourceV2.md#, DataSourceV2>> data sources with <<spark-sql-SessionConfigSupport.md#, SessionConfigSupport>> in [DataSource V2](new-and-noteworthy/datasource-v2.md).

[source, scala]
----
extractSessionConfigs(
  ds: DataSourceV2,
  conf: SQLConf): Map[String, String]
----

NOTE: `extractSessionConfigs` supports data sources with <<spark-sql-SessionConfigSupport.md#, SessionConfigSupport>> only.

`extractSessionConfigs` requests the `SessionConfigSupport` data source for the <<spark-sql-SessionConfigSupport.md#keyPrefix, custom key prefix for configuration options>> that is used to find all configuration options with the keys in the format of *spark.datasource.[keyPrefix]* in the given [SQLConf](SQLConf.md#getAllConfs).

`extractSessionConfigs` returns the matching keys with the *spark.datasource.[keyPrefix]* prefix removed (i.e. `spark.datasource.keyPrefix.k1` becomes `k1`).

`extractSessionConfigs` is used when:

* `DataFrameReader` is requested to ["load" data as a DataFrame](DataFrameReader.md#load) (for a [DataSourceV2](spark-sql-DataSourceV2.md) with [ReadSupport](spark-sql-ReadSupport.md))

* `DataFrameWriter` is requested to [saves a DataFrame to a data source](DataFrameWriter.md#save) (for a [DataSourceV2](spark-sql-DataSourceV2.md) with [WriteSupport](spark-sql-WriteSupport.md))

* (Spark Structured Streaming) `DataStreamReader` is requested to load input data stream (for data sources with `MicroBatchReadSupport` or `ContinuousReadSupport`)

* (Spark Structured Streaming) `DataStreamWriter` is requested to start a streaming query (with data sources with `StreamWriteSupport`)

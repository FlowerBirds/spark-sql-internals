# JDBCRDD

`JDBCRDD` is a `RDD` of [InternalRow](../../InternalRow.md)s that represents a structured query over a table in a database accessed via JDBC.

!!! note
    `JDBCRDD` represents a `SELECT requiredColumns FROM table` query.

`JDBCRDD` is <<creating-instance, created>> exclusively when `JDBCRDD` is requested to <<scanTable, scanTable>> (when `JDBCRelation` is requested to [build a scan](JDBCRelation.md#buildScan)).

[[internal-registries]]
.JDBCRDD's Internal Properties (e.g. Registries, Counters and Flags)
[cols="1,2",options="header",width="100%"]
|===
| Name
| Description

| `columnList`
| [[columnList]] Column names

Used when...FIXME

| `filterWhereClause`
| [[filterWhereClause]] <<filters, Filters>> as a SQL `WHERE` clause

Used when...FIXME
|===

=== [[compute]] Computing Partition (in TaskContext) -- `compute` Method

[source, scala]
----
compute(thePart: Partition, context: TaskContext): Iterator[InternalRow]
----

NOTE: `compute` is part of Spark Core's `RDD` Contract to compute a partition (in a `TaskContext`).

`compute`...FIXME

=== [[resolveTable]] `resolveTable` Method

[source, scala]
----
resolveTable(options: JDBCOptions): StructType
----

`resolveTable`...FIXME

NOTE: `resolveTable` is used exclusively when `JDBCRelation` is requested for the <<spark-sql-JDBCOptions.md#schema, schema>>.

=== [[scanTable]] Creating RDD for Distributed Data Scan -- `scanTable` Object Method

[source, scala]
----
scanTable(
  sc: SparkContext,
  schema: StructType,
  requiredColumns: Array[String],
  filters: Array[Filter],
  parts: Array[Partition],
  options: JDBCOptions): RDD[InternalRow]
----

`scanTable` takes the <<spark-sql-JDBCOptions.md#url, url>> option.

`scanTable` finds the corresponding JDBC dialect (per the `url` option) and requests it to quote the column identifiers in the input `requiredColumns`.

`scanTable` uses the `JdbcUtils` object to <<spark-sql-JdbcUtils.md#createConnectionFactory, createConnectionFactory>> and <<pruneSchema, prune columns>> from the input `schema` to include the input `requiredColumns` only.

In the end, `scanTable` creates a new <<creating-instance, JDBCRDD>>.

NOTE: `scanTable` is used exclusively when `JDBCRelation` is requested to <<datasources/jdbc/JDBCRelation.md#buildScan, build a distributed data scan with column pruning and filter pushdown>>.

## Creating Instance

`JDBCRDD` takes the following to be created:

* [[sc]] `SparkContext`
* [[getConnection]] Function to create a `Connection` (`() => Connection`)
* [[schema]] [Schema](../../StructType.md)
* [[columns]] Array of column names
* [[filters]] Array of [Filter predicates](../../Filter.md)
* [[partitions]] Array of Spark Core's `Partitions`
* [[url]] Connection URL
* [[options]] [JDBCOptions](JDBCOptions.md)

=== [[getPartitions]] `getPartitions` Method

[source, scala]
----
getPartitions: Array[Partition]
----

NOTE: `getPartitions` is part of Spark Core's `RDD` Contract to...FIXME

`getPartitions` simply returns the <<partitions, partitions>> (this `JDBCRDD` was created with).

=== [[pruneSchema]] `pruneSchema` Internal Method

[source, scala]
----
pruneSchema(schema: StructType, columns: Array[String]): StructType
----

`pruneSchema`...FIXME

NOTE: `pruneSchema` is used when...FIXME

=== [[compileFilter]] Converting Filter Predicate to SQL Expression -- `compileFilter` Object Method

[source, scala]
----
compileFilter(f: Filter, dialect: JdbcDialect): Option[String]
----

`compileFilter`...FIXME

[NOTE]
====
`compileFilter` is used when:

* `JDBCRelation` is requested to <<datasources/jdbc/JDBCRelation.md#unhandledFilters, find unhandled Filter predicates>>

* `JDBCRDD` is <<filterWhereClause, created>>
====

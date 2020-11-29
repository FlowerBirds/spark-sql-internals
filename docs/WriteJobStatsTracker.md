# WriteJobStatsTracker

`WriteJobStatsTracker` is the <<contract, abstraction>> of <<implementations, WriteJobStatsTrackers>> that can <<newTaskInstance, create a new WriteTaskStatsTracker>> and <<processStats, processStats>>.

[[contract]]
.WriteJobStatsTracker Contract
[cols="1m,2",options="header",width="100%"]
|===
| Method
| Description

| newTaskInstance
a| [[newTaskInstance]]

[source, scala]
----
newTaskInstance(): WriteTaskStatsTracker
----

Creates a new <<spark-sql-WriteTaskStatsTracker.md#, WriteTaskStatsTracker>>

Used when `EmptyDirectoryWriteTask`, `SingleDirectoryWriteTask` and `DynamicPartitionWriteTask` are requested for the `statsTrackers`

| processStats
a| [[processStats]]

[source, scala]
----
processStats(stats: Seq[WriteTaskStats]): Unit
----

Used when...FIXME
|===

[[implementations]]
NOTE: [BasicWriteJobStatsTracker](BasicWriteJobStatsTracker.md) is the one and only known implementation of the <<contract, WriteJobStatsTracker Contract>> in Apache Spark.

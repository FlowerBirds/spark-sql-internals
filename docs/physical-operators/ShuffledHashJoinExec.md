# ShuffledHashJoinExec Binary Physical Operator for Shuffled Hash Join

`ShuffledHashJoinExec` is a SparkPlan.md#BinaryExecNode[binary physical operator] to <<doExecute, execute>> a *shuffled hash join*.

`ShuffledHashJoinExec` performs a hash join of two child relations by first shuffling the data using the join keys.

`ShuffledHashJoinExec` is <<creating-instance, selected>> to represent a Join.md[Join] logical operator when [JoinSelection](../execution-planning-strategies/JoinSelection.md) execution planning strategy is executed and [spark.sql.join.preferSortMergeJoin](../configuration-properties.md#spark.sql.join.preferSortMergeJoin) configuration property is off.

!!! note
   [JoinSelection](../execution-planning-strategies/JoinSelection.md) execution planning strategy (and so Spark Planner) prefers [sort merge join](SortMergeJoinExec.md) over shuffled hash join based on [spark.sql.join.preferSortMergeJoin](../configuration-properties.md#spark.sql.join.preferSortMergeJoin) configuration property.

   In other words, you will _hardly_ see shuffled hash joins in your structured queries unless you turn `spark.sql.join.preferSortMergeJoin` on.

Beside the `spark.sql.join.preferSortMergeJoin` configuration property one of the following requirements has to hold:

* (For a right build side, i.e. `BuildRight`) [canBuildRight](../execution-planning-strategies/JoinSelection.md#canBuildRight), [canBuildLocalHashMap](../execution-planning-strategies/JoinSelection.md#canBuildLocalHashMap) for the right join side and finally the right join side is [at least three times smaller](../execution-planning-strategies/JoinSelection.md#muchSmaller) than the left side

* (For a right build side, i.e. `BuildRight`) Left join keys are *not* SortMergeJoinExec.md#orderable[orderable], i.e. cannot be sorted

* (For a left build side, i.e. `BuildLeft`) [canBuildLeft](../execution-planning-strategies/JoinSelection.md#canBuildLeft), [canBuildLocalHashMap](../execution-planning-strategies/JoinSelection.md#canBuildLocalHashMap) for left join side and finally left join side is [at least three times smaller](../execution-planning-strategies/JoinSelection.md#muchSmaller) than right

[TIP]
====
Enable `DEBUG` logging level for `org.apache.spark.sql.catalyst.planning.ExtractEquiJoinKeys` logger to see the join condition and the left and right join keys.
====

[source, scala]
----
// Use ShuffledHashJoinExec's selection requirements
// 1. Disable auto broadcasting
// JoinSelection (canBuildLocalHashMap specifically) requires that
// plan.stats.sizeInBytes < autoBroadcastJoinThreshold * numShufflePartitions
// That gives that autoBroadcastJoinThreshold has to be at least 1
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 1)

scala> println(spark.sessionState.conf.numShufflePartitions)
200

// 2. Disable preference on SortMergeJoin
spark.conf.set("spark.sql.join.preferSortMergeJoin", false)

val dataset = Seq(
  (0, "playing"),
  (1, "with"),
  (2, "ShuffledHashJoinExec")
).toDF("id", "token")
// Self LEFT SEMI join
val q = dataset.join(dataset, Seq("id"), "leftsemi")

val sizeInBytes = q.queryExecution.optimizedPlan.stats.sizeInBytes
scala> println(sizeInBytes)
72

// 3. canBuildLeft is on for leftsemi

// the right join side is at least three times smaller than the left side
// Even though it's a self LEFT SEMI join there are two different join sides
// How is that possible?

// BINGO! ShuffledHashJoin is here!

// Enable DEBUG logging level
import org.apache.log4j.{Level, Logger}
val logger = "org.apache.spark.sql.catalyst.planning.ExtractEquiJoinKeys"
Logger.getLogger(logger).setLevel(Level.DEBUG)

// ShuffledHashJoin with BuildRight
scala> q.explain
== Physical Plan ==
ShuffledHashJoin [id#37], [id#41], LeftSemi, BuildRight
:- Exchange hashpartitioning(id#37, 200)
:  +- LocalTableScan [id#37, token#38]
+- Exchange hashpartitioning(id#41, 200)
   +- LocalTableScan [id#41]

scala> println(q.queryExecution.executedPlan.numberedTreeString)
00 ShuffledHashJoin [id#37], [id#41], LeftSemi, BuildRight
01 :- Exchange hashpartitioning(id#37, 200)
02 :  +- LocalTableScan [id#37, token#38]
03 +- Exchange hashpartitioning(id#41, 200)
04    +- LocalTableScan [id#41]
----

[[requiredChildDistribution]]
.ShuffledHashJoinExec's Required Child Output Distributions
[cols="1,1",options="header",width="100%"]
|===
| Left Child
| Right Child

| [HashClusteredDistribution](HashClusteredDistribution.md) (per <<leftKeys, left join key expressions>>)
| [HashClusteredDistribution](HashClusteredDistribution.md) (per <<rightKeys, right join key expressions>>)
|===

=== [[doExecute]] Executing Physical Operator (Generating RDD[InternalRow]) -- `doExecute` Method

[source, scala]
----
doExecute(): RDD[InternalRow]
----

`doExecute` is part of the [SparkPlan](SparkPlan.md#doExecute) abstraction.

`doExecute` requests [streamedPlan](HashJoin.md#streamedPlan) physical operator to SparkPlan.md#execute[execute] (and generate a `RDD[InternalRow]`).

`doExecute` requests [buildPlan](HashJoin.md#buildPlan) physical operator to SparkPlan.md#execute[execute] (and generate a `RDD[InternalRow]`).

`doExecute` requests [streamedPlan](HashJoin.md#streamedPlan) physical operator's `RDD[InternalRow]` to zip partition-wise with [buildPlan](HashJoin.md#buildPlan) physical operator's `RDD[InternalRow]` (using `RDD.zipPartitions` method with `preservesPartitioning` flag disabled).

[NOTE]
====
`doExecute` generates a `ZippedPartitionsRDD2` that you can see in a RDD lineage.

[source, scala]
----
scala> println(q.queryExecution.toRdd.toDebugString)
(200) ZippedPartitionsRDD2[8] at toRdd at <console>:26 []
  |   ShuffledRowRDD[3] at toRdd at <console>:26 []
  +-(3) MapPartitionsRDD[2] at toRdd at <console>:26 []
     |  MapPartitionsRDD[1] at toRdd at <console>:26 []
     |  ParallelCollectionRDD[0] at toRdd at <console>:26 []
  |   ShuffledRowRDD[7] at toRdd at <console>:26 []
  +-(3) MapPartitionsRDD[6] at toRdd at <console>:26 []
     |  MapPartitionsRDD[5] at toRdd at <console>:26 []
     |  ParallelCollectionRDD[4] at toRdd at <console>:26 []
----
====

`doExecute` uses `RDD.zipPartitions` with a function applied to zipped partitions that takes two iterators of rows from the partitions of `streamedPlan` and `buildPlan`.

For every partition (and pairs of rows from the RDD), the function <<buildHashedRelation, buildHashedRelation>> on the partition of `buildPlan` and [join](HashJoin.md#join) the `streamedPlan` partition iterator, the [HashedRelation](HashedRelation.md), <<numOutputRows, numOutputRows>> and <<avgHashProbe, avgHashProbe>> SQL metrics.

=== [[buildHashedRelation]] Building HashedRelation for Internal Rows -- `buildHashedRelation` Internal Method

[source, scala]
----
buildHashedRelation(iter: Iterator[InternalRow]): HashedRelation
----

`buildHashedRelation` creates a [HashedRelation](HashedRelation.md#apply) (for the input `iter` iterator of `InternalRows`, [buildKeys](HashJoin.md#buildKeys) and the current `TaskMemoryManager`).

NOTE: `buildHashedRelation` uses `TaskContext.get()` to access the current `TaskContext` that in turn is used to access the `TaskMemoryManager`.

`buildHashedRelation` records the time to create the `HashedRelation` as <<buildTime, buildTime>>.

`buildHashedRelation` requests the `HashedRelation` for [estimatedSize](../KnownSizeEstimation.md#estimatedSize) that is recorded as <<buildDataSize, buildDataSize>>.

NOTE: `buildHashedRelation` is used exclusively when `ShuffledHashJoinExec` is requested to <<doExecute, execute>> (when [streamedPlan](HashJoin.md#streamedPlan) and [buildPlan](HashJoin.md#buildPlan) physical operators are executed and their RDDs zipped partition-wise using `RDD.zipPartitions` method).

=== [[creating-instance]] Creating ShuffledHashJoinExec Instance

`ShuffledHashJoinExec` takes the following when created:

* [[leftKeys]] Left join key expressions/Expression.md[expressions]
* [[rightKeys]] Right join key expressions/Expression.md[expressions]
* [[joinType]] [Join type](../joins.md#join-types)
* [[buildSide]] `BuildSide`
* [[condition]] Optional join condition expressions/Expression.md[expression]
* [[left]] Left SparkPlan.md[physical operator]
* [[right]] Right SparkPlan.md[physical operator]

## <span id="metrics"> Performance Metrics

Key            | Name (in web UI)        | Description
---------------|-------------------------|---------
 numOutputRows | number of output rows   | Number of output rows
 avgHashProbe  | avg hash probe          |
 buildDataSize | data size of build side |
 buildTime     | time to build hash map  |

!ShuffledHashJoinExec in web UI (Details for Query)(../images/spark-sql-ShuffledHashJoinExec-webui-query-details.png)

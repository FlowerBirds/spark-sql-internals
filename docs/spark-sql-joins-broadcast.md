# Broadcast Joins

Spark SQL uses **broadcast join** (_broadcast hash join_) instead of hash join to optimize join queries when the size of one side data is below [spark.sql.autoBroadcastJoinThreshold](configuration-properties.md#spark.sql.autoBroadcastJoinThreshold).

Broadcast join can be very efficient for joins between a large table (fact) with relatively small tables (dimensions) that could then be used to perform a *star-schema join*. It can avoid sending all data of the large table over the network.

You can use spark-sql-functions.md#broadcast[broadcast] function or SQL's [broadcast hints](new-and-noteworthy/hint-framework.md#broadcast-hints) to mark a dataset to be broadcast when used in a join query.

NOTE: According to the article http://dmtolpeko.com/2015/02/20/map-side-join-in-spark/[Map-Side Join in Spark], *broadcast join* is also called a *replicated join* (in the distributed system community) or a *map-side join* (in the Hadoop community).

`CanBroadcast` object matches a spark-sql-LogicalPlan.md[LogicalPlan] with output small enough for broadcast join.

NOTE: Currently statistics are only supported for Hive Metastore tables where the command `ANALYZE TABLE [tableName] COMPUTE STATISTICS noscan` has been run.

[JoinSelection](execution-planning-strategies/JoinSelection.md) execution planning strategy uses [spark.sql.autoBroadcastJoinThreshold](configuration-properties.md#spark.sql.autoBroadcastJoinThreshold) configuration property to control the size of a dataset before broadcasting it to all worker nodes when performing a join.

```text
val threshold =  spark.conf.get("spark.sql.autoBroadcastJoinThreshold").toInt
scala> threshold / 1024 / 1024
res0: Int = 10

val q = spark.range(100).as("a").join(spark.range(100).as("b")).where($"a.id" === $"b.id")
scala> println(q.queryExecution.logical.numberedTreeString)
00 'Filter ('a.id = 'b.id)
01 +- Join Inner
02    :- SubqueryAlias a
03    :  +- Range (0, 100, step=1, splits=Some(8))
04    +- SubqueryAlias b
05       +- Range (0, 100, step=1, splits=Some(8))

scala> println(q.queryExecution.sparkPlan.numberedTreeString)
00 BroadcastHashJoin [id#0L], [id#4L], Inner, BuildRight
01 :- Range (0, 100, step=1, splits=8)
02 +- Range (0, 100, step=1, splits=8)

scala> q.explain
== Physical Plan ==
*BroadcastHashJoin [id#0L], [id#4L], Inner, BuildRight
:- *Range (0, 100, step=1, splits=8)
+- BroadcastExchange HashedRelationBroadcastMode(List(input[0, bigint, false]))
   +- *Range (0, 100, step=1, splits=8)

spark.conf.set("spark.sql.autoBroadcastJoinThreshold", -1)
scala> spark.conf.get("spark.sql.autoBroadcastJoinThreshold")
res1: String = -1

scala> q.explain
== Physical Plan ==
*SortMergeJoin [id#0L], [id#4L], Inner
:- *Sort [id#0L ASC NULLS FIRST], false, 0
:  +- Exchange hashpartitioning(id#0L, 200)
:     +- *Range (0, 100, step=1, splits=8)
+- *Sort [id#4L ASC NULLS FIRST], false, 0
   +- ReusedExchange [id#4L], Exchange hashpartitioning(id#0L, 200)

// Force BroadcastHashJoin with broadcast hint (as function)
val qBroadcast = spark.range(100).as("a").join(broadcast(spark.range(100)).as("b")).where($"a.id" === $"b.id")
scala> qBroadcast.explain
== Physical Plan ==
*BroadcastHashJoin [id#14L], [id#18L], Inner, BuildRight
:- *Range (0, 100, step=1, splits=8)
+- BroadcastExchange HashedRelationBroadcastMode(List(input[0, bigint, false]))
   +- *Range (0, 100, step=1, splits=8)

// Force BroadcastHashJoin using SQL's BROADCAST hint
// Supported hints: BROADCAST, BROADCASTJOIN or MAPJOIN
val qBroadcastLeft = """
  SELECT /*+ BROADCAST (lf) */ *
  FROM range(100) lf, range(1000) rt
  WHERE lf.id = rt.id
"""
scala> sql(qBroadcastLeft).explain
== Physical Plan ==
*BroadcastHashJoin [id#34L], [id#35L], Inner, BuildRight
:- *Range (0, 100, step=1, splits=8)
+- BroadcastExchange HashedRelationBroadcastMode(List(input[0, bigint, false]))
   +- *Range (0, 1000, step=1, splits=8)

val qBroadcastRight = """
 SELECT /*+ MAPJOIN (rt) */ *
 FROM range(100) lf, range(1000) rt
 WHERE lf.id = rt.id
"""
scala> sql(qBroadcastRight).explain
== Physical Plan ==
*BroadcastHashJoin [id#42L], [id#43L], Inner, BuildRight
:- *Range (0, 100, step=1, splits=8)
+- BroadcastExchange HashedRelationBroadcastMode(List(input[0, bigint, false]))
   +- *Range (0, 1000, step=1, splits=8)
```

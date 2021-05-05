# BroadcastNestedLoopJoinExec Binary Physical Operator

`BroadcastNestedLoopJoinExec` is a [binary physical operator](BinaryExecNode.md) (with two child <<left, left>> and <<right, right>> physical operators) that is <<creating-instance, created>> (and converted to) when [JoinSelection](../execution-planning-strategies/JoinSelection.md) physical plan strategy finds a Join.md[Join] logical operator that meets either case:

* [canBuildRight](../execution-planning-strategies/JoinSelection.md#canBuildRight) join type and `right` physical operator [broadcastable](../execution-planning-strategies/JoinSelection.md#canBroadcast)

* [canBuildLeft](../execution-planning-strategies/JoinSelection.md#canBuildLeft) join type and `left` [broadcastable](../execution-planning-strategies/JoinSelection.md#canBroadcast)

* non-``InnerLike`` join type

!!! note
    `BroadcastNestedLoopJoinExec` is the default physical operator when no other operators have matched [selection requirements](../execution-planning-strategies/JoinSelection.md#join-selection-requirements).

[NOTE]
====
[canBuildRight](../execution-planning-strategies/JoinSelection.md#canBuildRight) join types are:

* CROSS, INNER, LEFT ANTI, LEFT OUTER, LEFT SEMI or Existence

[canBuildLeft](../execution-planning-strategies/JoinSelection.md#canBuildLeft) join types are:

* CROSS, INNER, RIGHT OUTER
====

```text
val nums = spark.range(2)
val letters = ('a' to 'c').map(_.toString).toDF("letter")
val q = nums.crossJoin(letters)

scala> q.explain
== Physical Plan ==
BroadcastNestedLoopJoin BuildRight, Cross
:- *Range (0, 2, step=1, splits=Some(8))
+- BroadcastExchange IdentityBroadcastMode
   +- LocalTableScan [letter#69]
```

[[requiredChildDistribution]]
.BroadcastNestedLoopJoinExec's Required Child Output Distributions
[cols="1m,2,2",options="header",width="100%"]
|===
| BuildSide
| Left Child
| Right Child

| BuildLeft
| [BroadcastDistribution](BroadcastDistribution.md) (uses `IdentityBroadcastMode` broadcast mode)
| [UnspecifiedDistribution](UnspecifiedDistribution.md)

| BuildRight
| [UnspecifiedDistribution](UnspecifiedDistribution.md)
| [BroadcastDistribution](BroadcastDistribution.md) (uses `IdentityBroadcastMode` broadcast mode)
|===

=== [[creating-instance]] Creating BroadcastNestedLoopJoinExec Instance

`BroadcastNestedLoopJoinExec` takes the following when created:

* [[left]] Left SparkPlan.md[physical operator]
* [[right]] Right SparkPlan.md[physical operator]
* [[buildSide]] `BuildSide`
* [[joinType]] spark-sql-joins.md#join-types[Join type]
* [[condition]] Optional join condition expressions/Expression.md[expressions]

## <span id="metrics"> Performance Metrics

Key             | Name (in web UI)        | Description
----------------|-------------------------|---------
numOutputRows   | number of output rows   | Number of output rows

![BroadcastNestedLoopJoinExec in web UI (Details for Query)](../images/spark-sql-BroadcastNestedLoopJoinExec-webui-details-for-query.png)

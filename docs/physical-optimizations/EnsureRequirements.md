---
title: EnsureRequirements
---

# EnsureRequirements Physical Optimization

[[apply]]
`EnsureRequirements` is a *physical query optimization* (aka _physical query preparation rule_ or simply _preparation rule_) that `QueryExecution` [uses](../QueryExecution.md#preparations) to optimize the physical plan of a structured query by transforming the following physical operators (up the plan tree):

. Removes two adjacent ShuffleExchangeExec.md[ShuffleExchangeExec] physical operators if the child partitioning scheme guarantees the parent's partitioning

. For other non-``ShuffleExchangeExec`` physical operators, <<ensureDistributionAndOrdering, ensures partition distribution and ordering>> (possibly adding new physical operators, e.g. BroadcastExchangeExec.md[BroadcastExchangeExec] and ShuffleExchangeExec.md[ShuffleExchangeExec] for distribution or SortExec.md[SortExec] for sorting)

Technically, `EnsureRequirements` is just a catalyst/Rule.md[Catalyst rule] for transforming SparkPlan.md[physical query plans], i.e. `Rule[SparkPlan]`.

`EnsureRequirements` is part of [preparations](../QueryExecution.md#preparations) batch of physical query plan rules and is executed when `QueryExecution` is requested for the [optimized physical query plan](../QueryExecution.md#executedPlan) (i.e. in *executedPlan* phase of a query execution).

[[conf]]
`EnsureRequirements` takes a [SQLConf](../SQLConf.md) when created.

[source, scala]
----
val q = ??? // FIXME
val sparkPlan = q.queryExecution.sparkPlan

import org.apache.spark.sql.execution.exchange.EnsureRequirements
val plan = EnsureRequirements(spark.sessionState.conf).apply(sparkPlan)
----

=== [[createPartitioning]] `createPartitioning` Internal Method

CAUTION: FIXME

=== [[defaultNumPreShufflePartitions]] `defaultNumPreShufflePartitions` Internal Method

CAUTION: FIXME

## <span id="ensureDistributionAndOrdering"> Enforcing Partition Requirements (Distribution and Ordering) of Physical Operator

```scala
ensureDistributionAndOrdering(
  operator: SparkPlan): SparkPlan
```

`ensureDistributionAndOrdering` takes the following from the input physical `operator`:

* SparkPlan.md#requiredChildDistribution[required partition requirements] for the children

* SparkPlan.md#requiredChildOrdering[required sort ordering] per the required partition requirements per child

* child physical plans

NOTE: The number of requirements for partitions and their sort ordering has to match the number and the order of the child physical plans.

`ensureDistributionAndOrdering` matches the operator's required partition requirements of children (`requiredChildDistributions`) to the children's SparkPlan.md#outputPartitioning[output partitioning] and (in that order):

. If the child satisfies the requested distribution, the child is left unchanged

. For [BroadcastDistribution](../physical-operators/BroadcastDistribution.md), the child becomes the child of BroadcastExchangeExec.md[BroadcastExchangeExec] unary operator for BroadcastHashJoinExec.md[broadcast hash joins]

. Any other pair of child and distribution leads to ShuffleExchangeExec.md[ShuffleExchangeExec] unary physical operator (with proper <<createPartitioning, partitioning>> for distribution and with [spark.sql.shuffle.partitions](../configuration-properties.md#spark.sql.shuffle.partitions) number of partitions)

NOTE: ShuffleExchangeExec.md[ShuffleExchangeExec] can appear in the physical plan when the children's output partitioning cannot satisfy the physical operator's required child distribution.

If the input `operator` has multiple children and specifies child output distributions, then the children's SparkPlan.md#outputPartitioning[output partitionings] have to be compatible.

If the children's output partitionings are not all compatible, then...FIXME

`ensureDistributionAndOrdering` <<withExchangeCoordinator, adds ExchangeCoordinator>> (only when [Adaptive Query Execution](../adaptive-query-execution/index.md) is enabled which is not by default).

NOTE: At this point in `ensureDistributionAndOrdering` the required child distributions are already handled.

`ensureDistributionAndOrdering` matches the operator's required sort ordering of children (`requiredChildOrderings`) to the children's SparkPlan.md#outputPartitioning[output partitioning] and if the orderings do not match, SortExec.md#creating-instance[SortExec] unary physical operator is created as a new child.

In the end, `ensureDistributionAndOrdering` [sets the new children](../catalyst/TreeNode.md#withNewChildren) for the input `operator`.

`ensureDistributionAndOrdering` is used when `EnsureRequirements` physical optimization is [executed](#apply).

=== [[reorderJoinPredicates]] `reorderJoinPredicates` Internal Method

[source, scala]
----
reorderJoinPredicates(plan: SparkPlan): SparkPlan
----

`reorderJoinPredicates`...FIXME

NOTE: `reorderJoinPredicates` is used when...FIXME

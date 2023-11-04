---
title: WindowFrameCoercion
---

# WindowFrameCoercion Type Coercion Logical Rule

`WindowFrameCoercion` is a `TypeCoercionRule` that [cast the data types of the boundaries of a range window frame to the data type of the order specification in a WindowSpecDefinition](#coerceTypes) in a [logical plan](../logical-operators/LogicalPlan.md).

```text
import java.time.LocalDate
import java.sql.Timestamp
val sales = Seq(
  (Timestamp.valueOf(LocalDate.of(2018, 9, 1).atStartOfDay), 5),
  (Timestamp.valueOf(LocalDate.of(2018, 9, 2).atStartOfDay), 10),
  // Mind the 2-day gap
  (Timestamp.valueOf(LocalDate.of(2018, 9, 5).atStartOfDay), 5)
).toDF("time", "volume")
scala> sales.show
+-------------------+------+
|               time|volume|
+-------------------+------+
|2018-09-01 00:00:00|     5|
|2018-09-02 00:00:00|    10|
|2018-09-05 00:00:00|     5|
+-------------------+------+

scala> sales.printSchema
root
 |-- time: timestamp (nullable = true)
 |-- volume: integer (nullable = false)

// FIXME Use Catalyst DSL
// rangeBetween with column expressions
// data type of orderBy expression is date
// data types of range frame boundaries is interval
// WindowSpecDefinition(_, Seq(order), SpecifiedWindowFrame(RangeFrame, lower, upper))
import org.apache.spark.unsafe.types.CalendarInterval
val interval = lit(CalendarInterval.fromString("interval 1 days"))
import org.apache.spark.sql.expressions.Window
val windowSpec = Window.orderBy($"time").rangeBetween(currentRow(), interval)

val q = sales.select(
  $"time",
  (sum($"volume") over windowSpec) as "sum",
  (count($"volume") over windowSpec) as "count")
val plan = q.queryExecution.logical
scala> println(plan.numberedTreeString)
00 'Project [unresolvedalias('time, None), sum('volume) windowspecdefinition('time ASC NULLS FIRST, specifiedwindowframe(RangeFrame, currentrow$(), interval 1 days)) AS sum#156, count('volume) windowspecdefinition('time ASC NULLS FIRST, specifiedwindowframe(RangeFrame, currentrow$(), interval 1 days)) AS count#158]
01 +- AnalysisBarrier
02       +- Project [_1#129 AS time#132, _2#130 AS volume#133]
03          +- LocalRelation [_1#129, _2#130]

import spark.sessionState.analyzer.ResolveReferences
val planWithRefsResolved = ResolveReferences(plan)

import spark.sessionState.analyzer.ResolveAliases
val planWithAliasesResolved = ResolveReferences(planWithRefsResolved)

// FIXME Looks like nothing changes in the query plan with regard to WindowFrameCoercion

import org.apache.spark.sql.catalyst.analysis.TypeCoercion.WindowFrameCoercion
val afterWindowFrameCoercion = WindowFrameCoercion(planWithRefsResolved)
scala> println(afterWindowFrameCoercion.numberedTreeString)
00 'Project [unresolvedalias(time#132, None), sum(volume#133) windowspecdefinition(time#132 ASC NULLS FIRST, specifiedwindowframe(RangeFrame, currentrow$(), interval 1 days)) AS sum#156L, count(volume#133) windowspecdefinition(time#132 ASC NULLS FIRST, specifiedwindowframe(RangeFrame, currentrow$(), interval 1 days)) AS count#158L]
01 +- AnalysisBarrier
02       +- Project [_1#129 AS time#132, _2#130 AS volume#133]
03          +- LocalRelation [_1#129, _2#130]
```

```text
import java.time.LocalDate
import java.sql.Date
val sales = Seq(
  (Date.valueOf(LocalDate.of(2018, 9, 1)), 5),
  (Date.valueOf(LocalDate.of(2018, 9, 2)), 10),
  // Mind the 2-day gap
  (Date.valueOf(LocalDate.of(2018, 9, 5)), 5)
).toDF("time", "volume")
scala> sales.show
+----------+------+
|      time|volume|
+----------+------+
|2018-09-01|     5|
|2018-09-02|    10|
|2018-09-05|     5|
+----------+------+

scala> sales.printSchema
root
 |-- time: date (nullable = true)
 |-- volume: integer (nullable = false)

// FIXME Use Catalyst DSL
// rangeBetween with column expressions
// data type of orderBy expression is date
// WindowSpecDefinition(_, Seq(order), SpecifiedWindowFrame(RangeFrame, lower, upper))
import org.apache.spark.sql.expressions.Window
val windowSpec = Window.orderBy($"time").rangeBetween(currentRow(), lit(1))

val q = sales.select(
  $"time",
  (sum($"volume") over windowSpec) as "sum")
val plan = q.queryExecution.logical
scala> println(plan.numberedTreeString)
00 'Project [unresolvedalias('time, None), sum('volume) windowspecdefinition('time ASC NULLS FIRST, specifiedwindowframe(RangeFrame, currentrow$(), 1)) AS sum#238]
01 +- AnalysisBarrier
02       +- Project [_1#222 AS time#225, _2#223 AS volume#226]
03          +- LocalRelation [_1#222, _2#223]

import spark.sessionState.analyzer.ResolveReferences
val planWithRefsResolved = ResolveReferences(plan)

import spark.sessionState.analyzer.ResolveAliases
val planWithAliasesResolved = ResolveReferences(planWithRefsResolved)

// FIXME Looks like nothing changes in the query plan with regard to WindowFrameCoercion

import org.apache.spark.sql.catalyst.analysis.TypeCoercion.WindowFrameCoercion
val afterWindowFrameCoercion = WindowFrameCoercion(planWithAliasesResolved)
scala> println(afterWindowFrameCoercion.numberedTreeString)
00 'Project [unresolvedalias(time#132, None), sum(volume#133) windowspecdefinition(time#132 ASC NULLS FIRST, specifiedwindowframe(RangeFrame, currentrow$(), interval 1 days)) AS sum#156L, count(volume#133) windowspecdefinition(time#132 ASC NULLS FIRST, specifiedwindowframe(RangeFrame, currentrow$(), interval 1 days)) AS count#158L]
01 +- AnalysisBarrier
02       +- Project [_1#129 AS time#132, _2#130 AS volume#133]
03          +- LocalRelation [_1#129, _2#130]
```

=== [[coerceTypes]] Coercing Types in Logical Plan -- `coerceTypes` Method

[source, scala]
----
coerceTypes(plan: LogicalPlan): LogicalPlan
----

`coerceTypes` is part of the `TypeCoercionRule` abstraction.

`coerceTypes` <<catalyst/QueryPlan.md#transformAllExpressions, traverses all Catalyst expressions>> (in the input <<spark-sql-LogicalPlan.md#, LogicalPlan>>) and replaces the <<spark-sql-Expression-WindowSpecDefinition.md#frameSpecification, frameSpecification>> of every <<spark-sql-Expression-WindowSpecDefinition.md#, WindowSpecDefinition>> with a `RangeFrame` window frame and the single <<spark-sql-Expression-WindowSpecDefinition.md#orderSpec, order specification>> expression <<expressions/Expression.md#resolved, resolved>> with the lower and upper window frame boundary expressions cast to the <<expressions/Expression.md#dataType, data type>> of the order specification expression.

=== [[createBoundaryCast]] `createBoundaryCast` Internal Method

[source, scala]
----
createBoundaryCast(boundary: Expression, dt: DataType): Expression
----

`createBoundaryCast` returns a <<expressions/Expression.md#, Catalyst expression>> per the input `boundary` <<expressions/Expression.md#, Expression>> and the `dt` [DataType](../types/DataType.md) (in the order of execution):

* The input `boundary` expression if it is a `SpecialFrameBoundary`

* The input `boundary` expression if the `dt` data type is [DateType](../types/DataType.md#DateType) or [TimestampType](../types/DataType.md#TimestampType)

* `Cast` unary operator with the input `boundary` expression and the `dt` data type if the <<expressions/Expression.md#dataType, result type>> of the `boundary` expression is not the `dt` data type, but the result type can be cast to the `dt` data type

* The input `boundary` expression

NOTE: `createBoundaryCast` is used exclusively when `WindowFrameCoercion` type coercion logical rule is requested to <<coerceTypes, coerceTypes>>.

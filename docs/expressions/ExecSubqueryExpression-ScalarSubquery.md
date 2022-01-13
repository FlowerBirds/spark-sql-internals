# ScalarSubquery (ExecSubqueryExpression)

`ScalarSubquery` is an [ExecSubqueryExpression](ExecSubqueryExpression.md) that <<updateResult, can give exactly one value>> (i.e. the value of executing <<plan, SubqueryExec>> subquery that can result in a single row and a single column or `null` if no row were computed).

IMPORTANT: Spark SQL uses the name of `ScalarSubquery` twice to represent an `ExecSubqueryExpression` (this page) and a [SubqueryExpression](ScalarSubquery.md). It _is_ confusing and you should _not_ be anymore.

`ScalarSubquery` is <<creating-instance, created>> when [PlanSubqueries](../physical-optimizations/PlanSubqueries.md) physical optimization is executed (and plans a [ScalarSubquery](ScalarSubquery.md) expression).

[source, scala]
----
// FIXME DEMO
import org.apache.spark.sql.execution.PlanSubqueries
val spark = ...
val planSubqueries = PlanSubqueries(spark)
val plan = ...
val executedPlan = planSubqueries(plan)
----

[[Unevaluable]]
`ScalarSubquery` is an [unevaluable expression](Unevaluable.md).

[[dataType]]
`ScalarSubquery` uses...FIXME...for the <<Expression.md#dataType, data type>>.

[[internal-registries]]
.ScalarSubquery's Internal Properties (e.g. Registries, Counters and Flags)
[cols="1,2",options="header",width="100%"]
|===
| Name
| Description

| `result`
| [[result]] The value of the single column in a single row after collecting the rows from executing the <<plan, subquery plan>> or `null` if no rows were collected.

| `updated`
| [[updated]] Flag that says whether `ScalarSubquery` was <<updateResult, updated>> with collected result of executing the <<plan, subquery plan>>.
|===

=== [[creating-instance]] Creating ScalarSubquery Instance

`ScalarSubquery` takes the following when created:

* [[plan]] SubqueryExec.md[SubqueryExec] plan
* [[exprId]] Expression ID (as `ExprId`)

=== [[updateResult]] Updating ScalarSubquery With Collected Result -- `updateResult` Method

[source, scala]
----
updateResult(): Unit
----

NOTE: `updateResult` is part of spark-sql-Expression-ExecSubqueryExpression.md#updateResult[ExecSubqueryExpression Contract] to fill an Catalyst expression with a collected result from executing a subquery plan.

`updateResult` requests <<plan, SubqueryExec>> physical plan to SubqueryExec.md#executeCollect[execute and collect internal rows].

`updateResult` sets <<result, result>> to the value of the only column of the single row or `null` if no row were collected.

In the end, `updateResult` marks the `ScalarSubquery` instance as <<updated, updated>>.

`updateResult` reports a `RuntimeException` when there are more than 1 rows in the result.

```
more than one row returned by a subquery used as an expression:
[plan]
```

`updateResult` reports an `AssertionError` when the number of fields is not exactly 1.

```
Expects 1 field, but got [numFields] something went wrong in analysis
```

=== [[eval]] Evaluating Expression -- `eval` Method

[source, scala]
----
eval(input: InternalRow): Any
----

`eval` is part of the [Expression](Expression.md#eval) abstraction.

`eval` simply returns <<result, result>> value.

`eval` reports an `IllegalArgumentException` if the `ScalarSubquery` expression has not been <<updated, updated>> yet.

=== [[doGenCode]] Generating Java Source Code (ExprCode) For Code-Generated Expression Evaluation -- `doGenCode` Method

[source, scala]
----
doGenCode(ctx: CodegenContext, ev: ExprCode): ExprCode
----

NOTE: `doGenCode` is part of <<Expression.md#doGenCode, Expression Contract>> to generate a Java source code (ExprCode) for code-generated expression evaluation.

`doGenCode` first makes sure that the <<updated, updated>> flag is on (`true`). If not, `doGenCode` throws an `IllegalArgumentException` exception with the following message:

```
requirement failed: [this] has not finished
```

`doGenCode` then creates a <<spark-sql-Expression-Literal.md#create, Literal>> (for the <<result, result>> and the <<dataType, dataType>>) and simply requests it to <<spark-sql-Expression-Literal.md#doGenCode, generate a Java source code>>.

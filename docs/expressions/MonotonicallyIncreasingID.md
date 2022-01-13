# MonotonicallyIncreasingID

`MonotonicallyIncreasingID` is a [non-deterministic](Nondeterministic.md) [leaf expression](Expression.md#LeafExpression) that is the internal representation of the `monotonically_increasing_id` [standard](../spark-sql-functions.md#monotonically_increasing_id) and [SQL](../FunctionRegistry.md#monotonically_increasing_id) functions.

As a `Nondeterministic` expression, `MonotonicallyIncreasingID` requires explicit <<initializeInternal, initialization>> (with the current partition index) before <<evalInternal, evaluating a value>>.

`MonotonicallyIncreasingID` <<doGenCode, generates Java source code (as ExprCode) for code-generated expression evaluation>>.

```text
import org.apache.spark.sql.catalyst.expressions.MonotonicallyIncreasingID
val monotonicallyIncreasingID = MonotonicallyIncreasingID()

import org.apache.spark.sql.catalyst.expressions.codegen.{CodegenContext, ExprCode}
val ctx = new CodegenContext
// doGenCode is used when Expression.genCode is executed
val ExprCode(code, _, _) = monotonicallyIncreasingID.genCode(ctx)

// Helper methods
def trim(code: String): String = {
  code.trim.split("\n").map(_.trim).filter(line => line.nonEmpty).mkString("\n")
}
def prettyPrint(code: String) = println(trim(code))
// END: Helper methods

scala> println(trim(code))
final long value_0 = partitionMask + count_0;
count_0++;
```

[[dataType]]
`MonotonicallyIncreasingID` uses [LongType](../types/DataType.md#LongType) as the <<Expression.md#dataType, data type>> of the result of evaluating itself.

[[nullable]]
`MonotonicallyIncreasingID` is never [nullable](Expression.md#nullable).

[[prettyName]]
`MonotonicallyIncreasingID` uses *monotonically_increasing_id* for the <<Expression.md#prettyName, user-facing name>>.

[[sql]]
`MonotonicallyIncreasingID` uses *monotonically_increasing_id()* for the <<Expression.md#sql, SQL representation>>.

`MonotonicallyIncreasingID` is <<creating-instance, created>> when <<spark-sql-functions.md#monotonically_increasing_id, monotonically_increasing_id>> standard function is used in a structured query.

`MonotonicallyIncreasingID` is <<FunctionRegistry.md#expressions, registered>> as `monotonically_increasing_id` SQL function.

[[creating-instance]]
`MonotonicallyIncreasingID` takes no input parameters when created.

[[internal-registries]]
.MonotonicallyIncreasingID's Internal Properties (e.g. Registries, Counters and Flags)
[cols="1,2",options="header",width="100%"]
|===
| Name
| Description

| `count`
| [[count]] Number of <<evalInternal, evalInternal>> calls, i.e. the number of rows for which `MonotonicallyIncreasingID` was evaluated

Initialized when `MonotonicallyIncreasingID` is requested to <<initializeInternal, initialize>> and used to <<evalInternal, evaluate a value>>.

| `partitionMask`
| [[partitionMask]] Current partition index shifted 33 bits left

Initialized when `MonotonicallyIncreasingID` is requested to <<initializeInternal, initialize>> and used to <<evalInternal, evaluate a value>>.
|===

=== [[doGenCode]] Generating Java Source Code (ExprCode) For Code-Generated Expression Evaluation -- `doGenCode` Method

[source, scala]
----
doGenCode(ctx: CodegenContext, ev: ExprCode): ExprCode
----

NOTE: `doGenCode` is part of <<Expression.md#doGenCode, Expression Contract>> to generate a Java source code (ExprCode) for code-generated expression evaluation.

`doGenCode` requests the `CodegenContext` to [add a mutable state](../whole-stage-code-generation/CodegenContext.md#addMutableState) as `count` name and `long` Java type.

`doGenCode` requests the `CodegenContext` to [add an immutable state (unless exists already)](../whole-stage-code-generation/CodegenContext.md#addImmutableStateIfNotExists) as `partitionMask` name and `long` Java type.

`doGenCode` requests the `CodegenContext` to [addPartitionInitializationStatement](../whole-stage-code-generation/CodegenContext.md#addPartitionInitializationStatement) with `[countTerm] = 0L;` statement.

`doGenCode` requests the `CodegenContext` to [addPartitionInitializationStatement](../whole-stage-code-generation/CodegenContext.md#addPartitionInitializationStatement) with `[partitionMaskTerm] = ((long) partitionIndex) << 33;` statement.

In the end, `doGenCode` returns the input `ExprCode` with the `code` as follows and `isNull` property disabled (`false`):

```text
final [dataType] [value] = [partitionMaskTerm] + [countTerm];
      [countTerm]++;
```

=== [[initializeInternal]] Initializing Nondeterministic Expression -- `initializeInternal` Method

[source, scala]
----
initializeInternal(input: InternalRow): Long
----

NOTE: `initializeInternal` is part of <<Nondeterministic.md#initializeInternal, Nondeterministic Contract>> to initialize a `Nondeterministic` expression.

`initializeInternal` simply sets the <<count, count>> to `0` and the <<partitionMask, partitionMask>> to `partitionIndex.toLong << 33`.

[source, scala]
----
val partitionIndex = 1
val partitionMask = partitionIndex.toLong << 33
scala> println(partitionMask.toBinaryString)
1000000000000000000000000000000000
----

=== [[evalInternal]] Evaluating Nondeterministic Expression -- `evalInternal` Method

[source, scala]
----
evalInternal(input: InternalRow): Long
----

NOTE: `evalInternal` is part of <<Nondeterministic.md#evalInternal, Nondeterministic Contract>> to evaluate the value of a `Nondeterministic` expression.

`evalInternal` remembers the current value of the <<count, count>> and increments it.

In the end, `evalInternal` returns the sum of the current value of the <<partitionMask, partitionMask>> and the remembered value of the <<count, count>>.

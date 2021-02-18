# Nondeterministic Expression Contract

`Nondeterministic` is a <<contract, contract>> for [Catalyst expressions](Expression.md) that are non-[deterministic](#deterministic) and non-[foldable](#foldable).

`Nondeterministic` expressions require explicit <<initialize, initialization>> (with the current partition index) before <<eval, evaluating a value>>.

[[contract]]
[source, scala]
----
package org.apache.spark.sql.catalyst.expressions

trait Nondeterministic extends Expression {
  // only required methods that have no implementation
  protected def initializeInternal(partitionIndex: Int): Unit
  protected def evalInternal(input: InternalRow): Any
}
----

.Nondeterministic Contract
[cols="1,2",options="header",width="100%"]
|===
| Method
| Description

| `initializeInternal`
| [[initializeInternal]] Initializing the `Nondeterministic` expression

Used exclusively when `Nondeterministic` expression is requested to <<initialize, initialize>>

| `evalInternal`
| [[evalInternal]] Evaluating the `Nondeterministic` expression

Used exclusively when `Nondeterministic` expression is requested to <<eval, evaluate a value>>
|===

NOTE: `Nondeterministic` expressions are the target of `PullOutNondeterministic` logical plan rule.

[[implementations]]
.Nondeterministic Expressions
[cols="1,2",options="header",width="100%"]
|===
| Expression
| Description

| `CurrentBatchTimestamp`
| [[CurrentBatchTimestamp]]

| `InputFileBlockLength`
| [[InputFileBlockLength]]

| `InputFileBlockStart`
| [[InputFileBlockStart]]

| `InputFileName`
| [[InputFileName]]

| expressions/MonotonicallyIncreasingID.md[MonotonicallyIncreasingID]
| [[MonotonicallyIncreasingID]]

| `NondeterministicExpression`
| [[NondeterministicExpression]]

| `Rand`
| [[Rand]]

| `Randn`
| [[Randn]]

| `RDG`
| [[RDG]]

| `SparkPartitionID`
| [[SparkPartitionID]]
|===

[[internal-registries]]
.Nondeterministic's Internal Properties (e.g. Registries, Counters and Flags)
[cols="1,2",options="header",width="100%"]
|===
| Name
| Description

| [[deterministic]] Expression.md#deterministic[deterministic]
| Always turned off (i.e. `false`)

| [[foldable]] Expression.md#foldable[foldable]
| Always turned off (i.e. `false`)

| [[initialized]] `initialized`
| Controls whether a `Nondeterministic` expression has been <<initialize, initialized>> before <<eval, evaluation>>.

Turned off by default.
|===

=== [[initialize]] Initializing Expression -- `initialize` Method

[source, scala]
----
initialize(partitionIndex: Int): Unit
----

Internally, `initialize` <<initializeInternal, initializes>> itself (with the input partition index) and turns the internal <<initialized, initialized>> flag on.

`initialize` is used when [InterpretedProjection](InterpretedProjection.md#initialize) and `InterpretedMutableProjection` are requested to `initialize` themselves.

=== [[eval]] Evaluating Expression -- `eval` Method

[source, scala]
----
eval(input: InternalRow): Any
----

`eval` is part of the [Expression](Expression.md#eval) abstraction.

`eval` is just a wrapper of <<evalInternal, evalInternal>> that makes sure that <<initialize, initialize>> has already been executed (and so the expression is initialized).

Internally, `eval` makes sure that the expression was <<initialized, initialized>> and calls <<evalInternal, evalInternal>>.

`eval` reports a `IllegalArgumentException` exception when the internal <<initialized, initialized>> flag is off, i.e. <<initialize, initialize>> has not yet been executed.

```text
requirement failed: Nondeterministic expression [name] should be initialized before eval.
```

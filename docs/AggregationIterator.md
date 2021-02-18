title: AggregationIterator

# AggregationIterator -- Generic Iterator of UnsafeRows for Aggregate Physical Operators

`AggregationIterator` is the base for <<implementations, iterators>> of <<UnsafeRow.md, UnsafeRows>> that...FIXME

> Iterators are data structures that allow to iterate over a sequence of elements. They have a `hasNext` method for checking if there is a next element available, and a `next` method which returns the next element and discards it from the iterator.

[[implementations]]
.AggregationIterator's Implementations
[width="100%",cols="1,2",options="header"]
|===
| Name
| Description

| <<spark-sql-ObjectAggregationIterator.md#, ObjectAggregationIterator>>
| Used when [ObjectHashAggregateExec](physical-operators/ObjectHashAggregateExec.md) physical operator is executed

| <<spark-sql-SortBasedAggregationIterator.md#, SortBasedAggregationIterator>>
| Used exclusively when `SortAggregateExec` physical operator is SortAggregateExec.md#doExecute[executed].

| TungstenAggregationIterator.md[TungstenAggregationIterator]
a| Used when [HashAggregateExec](physical-operators/HashAggregateExec.md) physical operator is executed

|===

[[internal-registries]]
.AggregationIterator's Internal Registries and Counters
[cols="1,2",options="header",width="100%"]
|===
| Name
| Description

| [[aggregateFunctions]] `aggregateFunctions`
| spark-sql-Expression-AggregateFunction.md[Aggregate functions]

Used when...FIXME

| [[allImperativeAggregateFunctions]] `allImperativeAggregateFunctions`
| spark-sql-Expression-ImperativeAggregate.md[ImperativeAggregate] functions

Used when...FIXME

| [[allImperativeAggregateFunctionPositions]] `allImperativeAggregateFunctionPositions`
| Positions

Used when...FIXME

| [[expressionAggInitialProjection]] `expressionAggInitialProjection`
| `MutableProjection`

Used when...FIXME

| `generateOutput`
a| [[generateOutput]] Function used to generate an <<UnsafeRow.md#, unsafe row>> (i.e. `(UnsafeRow, InternalRow) => UnsafeRow`)

Used when:

* `ObjectAggregationIterator` is requested for the <<spark-sql-ObjectAggregationIterator.md#next, next unsafe row>> and <<spark-sql-ObjectAggregationIterator.md#outputForEmptyGroupingKeyWithoutInput, outputForEmptyGroupingKeyWithoutInput>>

* `SortBasedAggregationIterator` is requested for the <<spark-sql-SortBasedAggregationIterator.md#next, next unsafe row>> and <<spark-sql-SortBasedAggregationIterator.md#outputForEmptyGroupingKeyWithoutInput, outputForEmptyGroupingKeyWithoutInput>>

* `TungstenAggregationIterator` is requested for the <<TungstenAggregationIterator.md#next, next unsafe row>> and <<TungstenAggregationIterator.md#outputForEmptyGroupingKeyWithoutInput, outputForEmptyGroupingKeyWithoutInput>>

| [[groupingAttributes]] `groupingAttributes`
| Grouping spark-sql-Expression-Attribute.md[attributes]

Used when...FIXME

| [[groupingProjection]] `groupingProjection`
| [UnsafeProjection](expressions/UnsafeProjection.md)

Used when...FIXME

| [[processRow]] `processRow`
| `(InternalRow, InternalRow) => Unit`

Used when...FIXME
|===

=== [[creating-instance]] Creating AggregationIterator Instance

`AggregationIterator` takes the following when created:

* [[groupingExpressions]] Grouping expressions/NamedExpression.md[named expressions]
* [[inputAttributes]] Input spark-sql-Expression-Attribute.md[attributes]
* [[aggregateExpressions]] [Aggregate expressions](expressions/AggregateExpression.md)
* [[aggregateAttributes]] Aggregate spark-sql-Expression-Attribute.md[attributes]
* [[initialInputBufferOffset]] Initial input buffer offset
* [[resultExpressions]] Result expressions/NamedExpression.md[named expressions]
* [[newMutableProjection]] Function to create a new `MutableProjection` given expressions and attributes

`AggregationIterator` initializes the <<internal-registries, internal registries and counters>>.

NOTE: `AggregationIterator` is a Scala abstract class and cannot be <<creating-instance, created>> directly. It is created indirectly for the <<implementations, concrete AggregationIterators>>.

=== [[initializeAggregateFunctions]] `initializeAggregateFunctions` Internal Method

[source, scala]
----
initializeAggregateFunctions(
  expressions: Seq[AggregateExpression],
  startingInputBufferOffset: Int): Array[AggregateFunction]
----

`initializeAggregateFunctions`...FIXME

NOTE: `initializeAggregateFunctions` is used when...FIXME

=== [[generateProcessRow]] `generateProcessRow` Internal Method

[source, scala]
----
generateProcessRow(
  expressions: Seq[AggregateExpression],
  functions: Seq[AggregateFunction],
  inputAttributes: Seq[Attribute]): (InternalRow, InternalRow) => Unit
----

`generateProcessRow`...FIXME

NOTE: `generateProcessRow` is used when...FIXME

=== [[generateResultProjection]] `generateResultProjection` Method

[source, scala]
----
generateResultProjection(): (UnsafeRow, InternalRow) => UnsafeRow
----

`generateResultProjection`...FIXME

[NOTE]
====
`generateResultProjection` is used when:

* `AggregationIterator` is <<generateOutput, created>>

* `TungstenAggregationIterator` is requested for the <<TungstenAggregationIterator.md#generateResultProjection, generateResultProjection>>
====

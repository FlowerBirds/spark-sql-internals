# ExpressionEncoder

`ExpressionEncoder[T]` is a generic [Encoder](Encoder.md) of JVM objects of the type `T` to and from [internal binary rows](InternalRow.md).

`ExpressionEncoder[T]` uses expressions/Expression.md[expressions] for a <<serializer, serializer>> and a <<deserializer, deserializer>>.

NOTE: `ExpressionEncoder` is the only supported implementation of `Encoder` which is explicitly enforced when `Dataset` Dataset.md#exprEnc[is created] (even though `Dataset` data structure accepts a _bare_ `Encoder[T]`).

[source, scala]
----
import org.apache.spark.sql.catalyst.encoders.ExpressionEncoder
val stringEncoder = ExpressionEncoder[String]
scala> val row = stringEncoder.toRow("hello world")
row: org.apache.spark.sql.catalyst.InternalRow = [0,100000000b,6f77206f6c6c6568,646c72]

import org.apache.spark.sql.catalyst.expressions.UnsafeRow
scala> val unsafeRow = row match { case ur: UnsafeRow => ur }
unsafeRow: org.apache.spark.sql.catalyst.expressions.UnsafeRow = [0,100000000b,6f77206f6c6c6568,646c72]
----

`ExpressionEncoder` uses <<serializer, serializer expressions>> to encode (aka _serialize_) a JVM object of type `T` to an [InternalRow](InternalRow.md).

NOTE: It is assumed that all serializer expressions contain at least one and the same [BoundReference](expressions/BoundReference.md).

`ExpressionEncoder` uses a <<deserializer, deserializer expression>> to decode (aka _deserialize_) a JVM object of type `T` from [InternalRow](InternalRow.md).

`ExpressionEncoder` is <<flat, flat>> when <<serializer, serializer>> uses a single expression (which also means that the objects of a type `T` are not created using constructor parameters only like `Product` or `DefinedByConstructorParams` types).

Internally, a `ExpressionEncoder` creates a [UnsafeProjection](expressions/UnsafeProjection.md) (for the input serializer), a [InternalRow](InternalRow.md) (of size `1`), and a safe `Projection` (for the input deserializer). They are all internal lazy attributes of the encoder.

[[properties]]
.ExpressionEncoder's (Lazily-Initialized) Internal Properties
[cols="1,3",options="header",width="100%"]
|===
| Property
| Description

| [[constructProjection]] `constructProjection`
a| `Projection` generated for the <<deserializer, deserializer>> expression

Used exclusively when `ExpressionEncoder` is requested for a <<fromRow, JVM object from a Spark SQL row>> (i.e. `InternalRow`).

| [[extractProjection]] `extractProjection`
a| `UnsafeProjection` [generated](whole-stage-code-generation/GenerateUnsafeProjection.md#generated) for the <<serializer, serializer>> expressions

Used exclusively when `ExpressionEncoder` is requested for an <<toRow, encoded version of a JVM object as a Spark SQL row>> (i.e. `InternalRow`).

| [[inputRow]] `inputRow`
a| `GenericInternalRow` (with the underlying storage array) of size 1 (i.e. it can only store a single JVM object of any type).

Used...FIXME
|===

NOTE: `Encoders` object contains the default `ExpressionEncoders` for Scala and Java primitive types, e.g. `boolean`, `long`, `String`, `java.sql.Date`, `java.sql.Timestamp`, `Array[Byte]`.

=== [[apply]] Creating ExpressionEncoder -- `apply` Method

[source, scala]
----
apply[T : TypeTag](): ExpressionEncoder[T]
----

CAUTION: FIXME

=== [[creating-instance]] Creating ExpressionEncoder Instance

`ExpressionEncoder` takes the following when created:

* [[schema]] [Schema](StructType.md)
* [[flat]] Flag whether `ExpressionEncoder` is flat or not
* [[serializer]] Serializer expressions/Expression.md[expressions] (to convert objects of type `T` to internal rows)
* [[deserializer]] Deserializer expressions/Expression.md[expression] (to convert internal rows to objects of type `T`)
* [[clsTag]] Scala's http://www.scala-lang.org/api/current/scala/reflect/ClassTag.html[ClassTag] for the JVM type `T`

=== [[deserializerFor]][[ScalaReflection-deserializerFor]] Creating Deserialize Expression -- `ScalaReflection.deserializerFor` Method

[source, scala]
----
deserializerFor[T: TypeTag]: Expression
----

`deserializerFor` creates an expressions/Expression.md[expression] to deserialize from [InternalRow](InternalRow.md) to a Scala object of type `T`.

```text
import org.apache.spark.sql.catalyst.ScalaReflection.deserializerFor
val timestampDeExpr = deserializerFor[java.sql.Timestamp]
scala> println(timestampDeExpr.numberedTreeString)
00 staticinvoke(class org.apache.spark.sql.catalyst.util.DateTimeUtils$, ObjectType(class java.sql.Timestamp), toJavaTimestamp, upcast(getcolumnbyordinal(0, TimestampType), TimestampType, - root class: "java.sql.Timestamp"), true)
01 +- upcast(getcolumnbyordinal(0, TimestampType), TimestampType, - root class: "java.sql.Timestamp")
02    +- getcolumnbyordinal(0, TimestampType)

val tuple2DeExpr = deserializerFor[(java.sql.Timestamp, Double)]
scala> println(tuple2DeExpr.numberedTreeString)
00 newInstance(class scala.Tuple2)
01 :- staticinvoke(class org.apache.spark.sql.catalyst.util.DateTimeUtils$, ObjectType(class java.sql.Timestamp), toJavaTimestamp, upcast(getcolumnbyordinal(0, TimestampType), TimestampType, - field (class: "java.sql.Timestamp", name: "_1"), - root class: "scala.Tuple2"), true)
02 :  +- upcast(getcolumnbyordinal(0, TimestampType), TimestampType, - field (class: "java.sql.Timestamp", name: "_1"), - root class: "scala.Tuple2")
03 :     +- getcolumnbyordinal(0, TimestampType)
04 +- upcast(getcolumnbyordinal(1, DoubleType), DoubleType, - field (class: "scala.Double", name: "_2"), - root class: "scala.Tuple2")
05    +- getcolumnbyordinal(1, DoubleType)
```

Internally, `deserializerFor` calls the recursive internal variant of <<deserializerFor-recursive, deserializerFor>> with a single-element walked type path with `- root class: "[clsName]"`

!!! tip
    Read up on Scala's `TypeTags` in [TypeTags and Manifests](http://docs.scala-lang.org/overviews/reflection/typetags-manifests.html).

NOTE: `deserializerFor` is used exclusively when `ExpressionEncoder` <<creating-instance, is created>> for a Scala type `T`.

==== [[deserializerFor-recursive]] Recursive Internal `deserializerFor` Method

[source, scala]
----
deserializerFor(
  tpe: `Type`,
  path: Option[Expression],
  walkedTypePath: Seq[String]): Expression
----

.JVM Types and Deserialize Expressions (in evaluation order)
[cols="1,1",options="header",width="100%"]
|===
| JVM Type (Scala or Java)
| Deserialize Expressions

| `Option[T]`
|

| `java.lang.Integer`
|

| `java.lang.Long`
|

| `java.lang.Double`
|

| `java.lang.Float`
|

| `java.lang.Short`
|

| `java.lang.Byte`
|

| `java.lang.Boolean`
|

| `java.sql.Date`
|

| `java.sql.Timestamp`
|

| `java.lang.String`
|

| `java.math.BigDecimal`
|

| `scala.BigDecimal`
|

| `java.math.BigInteger`
|

| `scala.math.BigInt`
|

| `Array[T]`
|

| `Seq[T]`
|

| `Map[K, V]`
|

| `SQLUserDefinedType`
|

| User Defined Types (UDTs)
|

| [[DefinedByConstructorParams]] `Product` (including `Tuple`) or `DefinedByConstructorParams`
|
|===

=== [[serializerFor]][[ScalaReflection-serializerFor]] Creating Serialize Expression -- `ScalaReflection.serializerFor` Method

[source, scala]
----
serializerFor[T: TypeTag](inputObject: Expression): CreateNamedStruct
----

`serializerFor` creates a <<spark-sql-Expression-CreateNamedStruct.md#creating-instance, CreateNamedStruct>> expression to serialize a Scala object of type `T` to [InternalRow](InternalRow.md).

```text
import org.apache.spark.sql.catalyst.ScalaReflection.serializerFor

import org.apache.spark.sql.catalyst.expressions.BoundReference
import org.apache.spark.sql.types.TimestampType
val boundRef = BoundReference(ordinal = 0, dataType = TimestampType, nullable = true)

val timestampSerExpr = serializerFor[java.sql.Timestamp](boundRef)
scala> println(timestampSerExpr.numberedTreeString)
00 named_struct(value, input[0, timestamp, true])
01 :- value
02 +- input[0, timestamp, true]
```

Internally, `serializerFor` calls the recursive internal variant of <<serializerFor-recursive, serializerFor>> with a single-element walked type path with `- root class: "[clsName]"` and _pattern match_ on the result expressions/Expression.md[expression].

CAUTION: FIXME the pattern match part

TIP: Read up on Scala's `TypeTags` in http://docs.scala-lang.org/overviews/reflection/typetags-manifests.html[TypeTags and Manifests].

NOTE: `serializerFor` is used exclusively when `ExpressionEncoder` <<creating-instance, is created>> for a Scala type `T`.

==== [[serializerFor-recursive]] Recursive Internal `serializerFor` Method

[source, scala]
----
serializerFor(
  inputObject: Expression,
  tpe: `Type`,
  walkedTypePath: Seq[String],
  seenTypeSet: Set[`Type`] = Set.empty): Expression
----

`serializerFor` creates an expressions/Expression.md[expression] for serializing an object of type `T` to an internal row.

CAUTION: FIXME

=== [[toRow]] Encoding JVM Object to Internal Binary Row Format -- `toRow` Method

[source, scala]
----
toRow(t: T): InternalRow
----

`toRow` encodes (aka _serializes_) a JVM object `t` as an [InternalRow](InternalRow.md).

Internally, `toRow` sets the only JVM object to be `t` in  <<inputRow, inputRow>> and converts the `inputRow` to a UnsafeRow.md[unsafe binary row] (using <<extractProjection, extractProjection>>).

In case of any exception while serializing, `toRow` reports a `RuntimeException`:

```
Error while encoding: [initial exception]
[multi-line serializer]
```

[NOTE]
====
`toRow` is _mostly_ used when `SparkSession` is requested for:

* SparkSession.md#createDataset[Dataset from a local dataset]

* SparkSession.md#createDataFrame[DataFrame from RDD[Row\]]
====

=== [[fromRow]] Decoding JVM Object From Internal Binary Row Format -- `fromRow` Method

[source, scala]
----
fromRow(row: InternalRow): T
----

`fromRow` decodes (aka _deserializes_) a JVM object from a `row` [InternalRow](InternalRow.md) (with the required values only).

Internally, `fromRow` uses <<constructProjection, constructProjection>> with `row` and gets the 0th element of type `ObjectType` that is then cast to the output type `T`.

In case of any exception while deserializing, `fromRow` reports a `RuntimeException`:

```
Error while decoding: [initial exception]
[deserializer]
```

[NOTE]
====
`fromRow` is used for:

* `Dataset` operators, i.e. `head`, `collect`, `collectAsList`, `toLocalIterator`

* Structured Streaming's `ForeachSink`
====

=== [[tuple]] Creating ExpressionEncoder For Tuple -- `tuple` Method

[source, scala]
----
tuple(encoders: Seq[ExpressionEncoder[_]]): ExpressionEncoder[_]
tuple[T](e: ExpressionEncoder[T]): ExpressionEncoder[Tuple1[T]]
tuple[T1, T2](
  e1: ExpressionEncoder[T1],
  e2: ExpressionEncoder[T2]): ExpressionEncoder[(T1, T2)]
tuple[T1, T2, T3](
  e1: ExpressionEncoder[T1],
  e2: ExpressionEncoder[T2],
  e3: ExpressionEncoder[T3]): ExpressionEncoder[(T1, T2, T3)]
tuple[T1, T2, T3, T4](
  e1: ExpressionEncoder[T1],
  e2: ExpressionEncoder[T2],
  e3: ExpressionEncoder[T3],
  e4: ExpressionEncoder[T4]): ExpressionEncoder[(T1, T2, T3, T4)]
tuple[T1, T2, T3, T4, T5](
  e1: ExpressionEncoder[T1],
  e2: ExpressionEncoder[T2],
  e3: ExpressionEncoder[T3],
  e4: ExpressionEncoder[T4],
  e5: ExpressionEncoder[T5]): ExpressionEncoder[(T1, T2, T3, T4, T5)]
----

`tuple`...FIXME

NOTE: `tuple` is used when...FIXME

=== [[resolveAndBind]] `resolveAndBind` Method

[source, scala]
----
resolveAndBind(
  attrs: Seq[Attribute] = schema.toAttributes,
  analyzer: Analyzer = SimpleAnalyzer): ExpressionEncoder[T]
----

`resolveAndBind`...FIXME

[source, scala]
----
// A very common use case
case class Person(id: Long, name: String)
import org.apache.spark.sql.Encoders
val schema = Encoders.product[Person].schema

import org.apache.spark.sql.catalyst.encoders.{RowEncoder, ExpressionEncoder}
import org.apache.spark.sql.Row
val encoder: ExpressionEncoder[Row] = RowEncoder.apply(schema).resolveAndBind()

import org.apache.spark.sql.catalyst.InternalRow
val row = InternalRow(1, "Jacek")

val deserializer = encoder.deserializer

scala> deserializer.eval(row)
java.lang.UnsupportedOperationException: Only code-generated evaluation is supported
  at org.apache.spark.sql.catalyst.expressions.objects.CreateExternalRow.eval(objects.scala:1105)
  ... 54 elided

import org.apache.spark.sql.catalyst.expressions.codegen.CodegenContext
val ctx = new CodegenContext
val code = deserializer.genCode(ctx).code
----

`resolveAndBind` is used when:

* `Dataset` is requested for the Dataset.md#deserializer[deserializer expression] (to convert internal rows to objects of type `T`)

* `TypedAggregateExpression` is spark-sql-Expression-TypedAggregateExpression.md#apply[created]

* `JdbcUtils` is requested to spark-sql-JdbcUtils.md#resultSetToRows[resultSetToRows]

* Spark Structured Streaming's `FlatMapGroupsWithStateExec` physical operator is requested for the state deserializer (i.e. `stateDeserializer`)

* Spark Structured Streaming's `ForeachSink` is requested to add a streaming batch (i.e. `addBatch`)

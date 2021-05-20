# UserDefinedFunction

`UserDefinedFunction` represents a **user-defined function**.

`UserDefinedFunction` is <<creating-instance, created>> when:

* [udf](spark-sql-functions.md#udf) function is executed

* `UDFRegistration` is requested to [register a Scala function as a user-defined function](UDFRegistration.md#register) (in `FunctionRegistry`)

```text
import org.apache.spark.sql.functions.udf
val lengthUDF = udf { s: String => s.length }

scala> :type lengthUDF
org.apache.spark.sql.expressions.UserDefinedFunction

val r = lengthUDF($"name")

scala> :type r
org.apache.spark.sql.Column
```

`UserDefinedFunction` can have an optional <<withName, name>>.

```text
val namedLengthUDF = lengthUDF.withName("lengthUDF")
scala> namedLengthUDF($"name")
res2: org.apache.spark.sql.Column = UDF:lengthUDF(name)
```

`UserDefinedFunction` is *nullable* by default, but can be changed as <<asNonNullable, non-nullable>>.

[source, scala]
----
val nonNullableLengthUDF = lengthUDF.asNonNullable
assert(nonNullableLengthUDF.nullable == false)
----

[[deterministic]]
`UserDefinedFunction` is *deterministic* <<_deterministic, by default>>, i.e. produces the same result for the same input. `UserDefinedFunction` can be changed to be <<asNondeterministic, non-deterministic>>.

[source, scala]
----
assert(lengthUDF.deterministic)
val ndUDF = lengthUDF.asNondeterministic
assert(ndUDF.deterministic == false)
----

[[internal-registries]]
.UserDefinedFunction's Internal Properties (e.g. Registries, Counters and Flags)
[cols="1m,3",options="header",width="100%"]
|===
| Name
| Description

| _deterministic
a| [[_deterministic]] Flag that controls whether the function is <<deterministic, deterministic>> (`true`) or not (`false`).

Default: `true`

* Use <<asNondeterministic, asNondeterministic>> to change it to `false`

Used when `UserDefinedFunction` is requested to <<apply, execute>>

|===

=== [[apply]] Executing UserDefinedFunction (Creating Column with ScalaUDF Expression) -- `apply` Method

```scala
apply(
  exprs: Column*): Column
```

`apply` creates a [Column](Column.md) with [ScalaUDF](expressions/ScalaUDF.md) expression.

```text
import org.apache.spark.sql.functions.udf
scala> val lengthUDF = udf { s: String => s.length }
lengthUDF: org.apache.spark.sql.expressions.UserDefinedFunction = UserDefinedFunction(<function1>,IntegerType,Some(List(StringType)))

scala> lengthUDF($"name")
res1: org.apache.spark.sql.Column = UDF(name)
```

`apply` is used when...FIXME

=== [[asNonNullable]] Marking UserDefinedFunction as NonNullable -- `asNonNullable` Method

[source, scala]
----
asNonNullable(): UserDefinedFunction
----

`asNonNullable`...FIXME

`asNonNullable` is used when...FIXME

=== [[withName]] Naming UserDefinedFunction -- `withName` Method

[source, scala]
----
withName(name: String): UserDefinedFunction
----

`withName`...FIXME

NOTE: `withName` is used when...FIXME

## Creating Instance

`UserDefinedFunction` takes the following when created:

* [[f]] A Scala function (as Scala's `AnyRef`)
* [[dataType]] Output [data type](types/DataType.md)
* [[inputTypes]] Input [data types](types/DataType.md) (if available)

`UserDefinedFunction` initializes the <<internal-registries, internal registries and counters>>.

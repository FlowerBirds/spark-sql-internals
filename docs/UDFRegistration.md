title: UDFRegistration

# UDFRegistration -- Session-Scoped FunctionRegistry

`UDFRegistration` is an interface to the session-scoped <<functionRegistry, FunctionRegistry>> to register user-defined functions (UDFs) and <<register, user-defined aggregate functions>> (UDAFs).

`UDFRegistration` is available using SparkSession.md#udf[SparkSession].

[source, scala]
----
import org.apache.spark.sql.SparkSession
val spark: SparkSession = ...
spark.udf
----

[[functionRegistry]]
[[creating-instance]]
`UDFRegistration` takes a <<FunctionRegistry.md#, FunctionRegistry>> when created.

`UDFRegistration` is <<creating-instance, created>> exclusively for SessionState.md#creating-instance[SessionState].

=== [[register]] Registering UserDefinedFunction (with FunctionRegistry) -- `register` Method

[source, scala]
----
register(name: String, func: Function0[RT]): UserDefinedFunction
register(name: String, func: Function1[A1, RT]): UserDefinedFunction
...
register(name: String, func: Function22[A1, A2, A3, A4, A5, A6, A7, A8, A9, A10, A11, A12, A13, A14, A15, A16, A17, A18, A19, A20, A21, A22, RT]): UserDefinedFunction
----

`register`...FIXME

NOTE: `register` is used when...FIXME

=== [[register-UserDefinedFunction]] Registering UserDefinedFunction (with FunctionRegistry) -- `register` Method

[source, scala]
----
register(name: String, udf: UserDefinedFunction): UserDefinedFunction
----

`register`...FIXME

NOTE: `register` is used when...FIXME

=== [[register-UserDefinedAggregateFunction]] Registering UserDefinedAggregateFunction (with FunctionRegistry) -- `register` Method

[source, scala]
----
register(
  name: String,
  udaf: UserDefinedAggregateFunction): UserDefinedAggregateFunction
----

`register` FunctionRegistry.md#registerFunction[registers a UserDefinedAggregateFunction] under `name` with <<functionRegistry, FunctionRegistry>>.

`register` creates a expressions/ScalaUDAF.md[ScalaUDAF] internally to register a UDAF.

NOTE: `register` gives the input `udaf` aggregate function back after the function has been registered with <<functionRegistry, FunctionRegistry>>.

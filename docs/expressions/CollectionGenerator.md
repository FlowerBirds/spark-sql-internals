title: CollectionGenerator

# CollectionGenerator Generator Expression Contract

`CollectionGenerator` is the <<contract, contract>> in Spark SQL for spark-sql-Expression-Generator.md[Generator expressions] that <<collectionType, generate a collection object>> (i.e. an array or map) and (at execution time) GenerateExec.md#doConsume[use a different path for whole-stage Java code generation] (while executing `GenerateExec` physical operator with spark-sql-whole-stage-codegen.md[Whole-Stage Java Code Generation] enabled).

[[contract]]
[source, scala]
----
package org.apache.spark.sql.catalyst.expressions

trait CollectionGenerator extends Generator {
  def collectionType: DataType = dataType
  def inline: Boolean
  def position: Boolean
}
----

.CollectionGenerator Contract
[cols="1,2",options="header",width="100%"]
|===
| Method
| Description

| [[collectionType]] `collectionType`
| The type of the returned collection object.

Used when...

| [[inline]] `inline`
| Flag whether to inline rows during whole-stage Java code generation.

Used when...

| [[position]] `position`
| Flag whether to include the positions of elements within the result collection.

Used when...
|===

[[implementations]]
.CollectionGenerators
[cols="1,2",options="header",width="100%"]
|===
| CollectionGenerator
| Description

| spark-sql-Expression-Inline.md[Inline]
|

| spark-sql-Expression-ExplodeBase.md[ExplodeBase]
|

| spark-sql-Expression-ExplodeBase.md#Explode[Explode]
|

| spark-sql-Expression-ExplodeBase.md#PosExplode[PosExplode]
|
|===

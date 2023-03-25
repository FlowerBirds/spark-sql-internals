# RelationProvider &mdash; Relations with Pre-Defined Schema

`RelationProvider` is an [abstraction](#contract) of [providers](#implementations) for [relational data sources with schema inference](#createRelation) (_schema discovery_). In other words, the schema of a relation is pre-defined (_fixed_), and a user-specified schema is not allowed.

!!! tip "SchemaRelationProvider"
    Use [SchemaRelationProvider](SchemaRelationProvider.md) for data source providers that require a user-defined schema.

## Contract

### <span id="createRelation"> Creating BaseRelation

```scala
createRelation(
  sqlContext: SQLContext,
  parameters: Map[String, String]): BaseRelation
```

Creates a [BaseRelation](BaseRelation.md)

Used when:

* `DataSource` is requested to [resolve a relation](DataSource.md#resolveRelation)

## Implementations

* [JdbcRelationProvider](jdbc/JdbcRelationProvider.md)
* [KafkaSourceProvider](kafka/KafkaSourceProvider.md)

## CreatableRelationProvider

It is a common pattern while developing a custom data source to use [RelationProvider.createRelation](#createRelation) with [CreatableRelationProvider](CreatableRelationProvider.md) when requested for a [relation](CreatableRelationProvider.md#createRelation) (after writing out a structured query).

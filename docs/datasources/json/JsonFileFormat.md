# JsonFileFormat -- Built-In Support for Files in JSON Format

[[shortName]]
`JsonFileFormat` is a [TextBasedFileFormat](../TextBasedFileFormat.md) for *json* format (i.e. [registers itself to handle files in json format](../../DataSourceRegister.md#shortName) and convert them to Spark SQL rows).

```text
spark.read.format("json").load("json-datasets")

// or the same as above using a shortcut
spark.read.json("json-datasets")
```

`JsonFileFormat` comes with <<options, options>> to further customize JSON parsing.

NOTE: `JsonFileFormat` uses https://github.com/apache/spark/commit/fb54a564d75aea835f57bc147b83a76d1da0a01f#diff-600376dffeb79835ede4a0b285078036[Jackson 2.6.7] as the JSON parser library and some <<options, options>> map directly to Jackson's internal options (as `JsonParser.Feature`).

[[options]]
[[JSONOptions]]
.JsonFileFormat's Options
[cols="1,1,2",options="header",width="100%"]
|===
| Option
| Default Value
| Description

| [[allowBackslashEscapingAnyCharacter]] `allowBackslashEscapingAnyCharacter`
| `false`
a|

NOTE: Internally, `allowBackslashEscapingAnyCharacter` becomes `JsonParser.Feature.ALLOW_BACKSLASH_ESCAPING_ANY_CHARACTER`.

| [[allowComments]] `allowComments`
| `false`
a|

NOTE: Internally, `allowComments` becomes `JsonParser.Feature.ALLOW_COMMENTS`.

| [[allowNonNumericNumbers]] `allowNonNumericNumbers`
| `true`
a|

NOTE: Internally, `allowNonNumericNumbers` becomes `JsonParser.Feature.ALLOW_NON_NUMERIC_NUMBERS`.

| [[allowNumericLeadingZeros]] `allowNumericLeadingZeros`
| `false`
a|

NOTE: Internally, `allowNumericLeadingZeros` becomes `JsonParser.Feature.ALLOW_NUMERIC_LEADING_ZEROS`.

| [[allowSingleQuotes]] `allowSingleQuotes`
| `true`
a|

NOTE: Internally, `allowSingleQuotes` becomes `JsonParser.Feature.ALLOW_SINGLE_QUOTES`.

| [[allowUnquotedControlChars]] `allowUnquotedControlChars`
| `false`
a|

NOTE: Internally, `allowUnquotedControlChars` becomes `JsonParser.Feature.ALLOW_UNQUOTED_CONTROL_CHARS`.

| [[allowUnquotedFieldNames]] `allowUnquotedFieldNames`
| `false`
a|

NOTE: Internally, `allowUnquotedFieldNames` becomes `JsonParser.Feature.ALLOW_UNQUOTED_FIELD_NAMES`.

| [[columnNameOfCorruptRecord]] `columnNameOfCorruptRecord`
|
|

| [[compression]] `compression`
|
a| Compression codec that can be either one of the [known aliases](../../CompressionCodecs.md#shortCompressionCodecNames) or a fully-qualified class name.

| [[dateFormat]] `dateFormat`
| `yyyy-MM-dd`
a| Date format

NOTE: Internally, `dateFormat` is converted to Apache Commons Lang's `FastDateFormat`.

| [[multiLine]] `multiLine`
| `false`
| Controls whether...FIXME

| [[mode]] `mode`
| `PERMISSIVE`
a| Case insensitive name of the parse mode

* `PERMISSIVE`
* `DROPMALFORMED`
* `FAILFAST`

| [[prefersDecimal]] `prefersDecimal`
| `false`
|

| [[primitivesAsString]] `primitivesAsString`
| `false`
|

| [[samplingRatio]] `samplingRatio`
| `1.0`
|

| [[timestampFormat]] `timestampFormat`
| `yyyy-MM-dd'T'HH:mm:ss.SSSXXX`
a| Timestamp format

NOTE: Internally, `timestampFormat` is converted to Apache Commons Lang's `FastDateFormat`.

| [[timeZone]] `timeZone`
|
| Java's `TimeZone`
|===

=== [[isSplitable]] `isSplitable` Method

[source, scala]
----
isSplitable(
  sparkSession: SparkSession,
  options: Map[String, String],
  path: Path): Boolean
----

`isSplitable`...FIXME

`isSplitable` is part of [FileFormat](../FileFormat.md#isSplitable) abstraction.

=== [[inferSchema]] `inferSchema` Method

[source, scala]
----
inferSchema(
  sparkSession: SparkSession,
  options: Map[String, String],
  files: Seq[FileStatus]): Option[StructType]
----

`inferSchema`...FIXME

`inferSchema` is part of [FileFormat](../FileFormat.md#inferSchema) abstraction.

=== [[buildReader]] Building Partitioned Data Reader -- `buildReader` Method

[source, scala]
----
buildReader(
  sparkSession: SparkSession,
  dataSchema: StructType,
  partitionSchema: StructType,
  requiredSchema: StructType,
  filters: Seq[Filter],
  options: Map[String, String],
  hadoopConf: Configuration): (PartitionedFile) => Iterator[InternalRow]
----

`buildReader`...FIXME

`buildReader` is part of the [FileFormat](../FileFormat.md#buildReader) abstraction.

=== [[prepareWrite]] Preparing Write Job -- `prepareWrite` Method

[source, scala]
----
prepareWrite(
  sparkSession: SparkSession,
  job: Job,
  options: Map[String, String],
  dataSchema: StructType): OutputWriterFactory
----

`prepareWrite`...FIXME

`prepareWrite` is part of the [FileFormat](../FileFormat.md#prepareWrite) abstraction.

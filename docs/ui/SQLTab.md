# SQLTab

<!---
## Review Me

*SQL* tab in spark-webui.md[web UI] shows spark-sql-SQLMetric.md[SQLMetrics] per SparkPlan.md[physical operator] in a structured query physical plan.

You can access the SQL tab under `/SQL` URL, e.g. http://localhost:4040/SQL/.

By default, it displays <<AllExecutionsPage, all SQL query executions>>. However, after a query has been selected, the SQL tab <<ExecutionPage, displays the details for the structured query execution>>.

=== [[AllExecutionsPage]] AllExecutionsPage

`AllExecutionsPage` displays all SQL query executions in a Spark application per state sorted by their submission time reversed.

.SQL Tab in web UI (AllExecutionsPage)
image::images/spark-webui-sql.png[align="center"]

Internally, the page requests [SQLListener](SQLListener.md) for query executions in running, completed, and failed states (the states correspond to the respective tables on the page).

=== [[ExecutionPage]] ExecutionPage -- Details for Query

`ExecutionPage` shows details for structured query execution by `id`.

NOTE: The `id` request parameter is mandatory.

`ExecutionPage` displays a summary with *Submitted Time*, *Duration*, the clickable identifiers of the *Running Jobs*, *Succeeded Jobs*, and *Failed Jobs*.

It also display a visualization (using [accumulator updates](SQLListener.md#getExecutionMetrics) and the `SparkPlanGraph` for the query) with the expandable *Details* section (that corresponds to `SQLExecutionUIData.physicalPlanDescription`).

.Details for Query in web UI
image::images/spark-webui-sql-execution-graph.png[align="center"]

If there is no information to display for a given query `id`, you should see the following page.

.No Details for SQL Query
image::images/spark-webui-sql-no-details-for-query.png[align="center"]

Internally, it uses [SQLListener](SQLListener.md) exclusively to get the SQL query execution metrics. It requests [`SQLListener` for SQL execution data](SQLListener.md#getExecution) to display for the `id` request parameter.

## Creating SQLTab Instance

`SQLTab` is created when SharedState.md[SharedState] is or at the first [SparkListenerSQLExecutionStart](SQLListener.md#SparkListenerSQLExecutionStart) event when Spark History Server is used.

.Creating SQLTab Instance
image::images/spark-SQLTab-creating-instance.png[align="center"]

NOTE: SharedState.md[SharedState] represents the shared state across `SparkSessions`.
-->

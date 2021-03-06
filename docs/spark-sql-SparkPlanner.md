title: SparkPlanner

# SparkPlanner -- Spark Query Planner

`SparkPlanner` is a concrete catalyst/QueryPlanner.md[Catalyst Query Planner] that converts a spark-sql-LogicalPlan.md[logical plan] to one or more SparkPlan.md[physical plans] using <<strategies, execution planning strategies>> with support for <<extraStrategies, extra strategies>> (by means of <<experimentalMethods, ExperimentalMethods>>) and <<extraPlanningStrategies, extraPlanningStrategies>>.

NOTE: `SparkPlanner` is expected to plan (aka _generate_) at least one SparkPlan.md[physical plan] per spark-sql-LogicalPlan.md[logical plan].

`SparkPlanner` is available as SessionState.md#planner[planner] of a `SessionState`.

[source, scala]
----
val spark: SparkSession = ...
scala> :type spark.sessionState.planner
org.apache.spark.sql.execution.SparkPlanner
----

[[strategies]]
.SparkPlanner's Execution Planning Strategies (in execution order)
[cols="1,2",options="header",width="100%"]
|===
| SparkStrategy
| Description

| [[extraStrategies]] ``ExperimentalMethods``'s spark-sql-ExperimentalMethods.md#extraStrategies[extraStrategies]
|

| <<extraPlanningStrategies, extraPlanningStrategies>>
| Extension point for extra spark-sql-SparkStrategy.md[planning strategies]

| [DataSourceV2Strategy](execution-planning-strategies/DataSourceV2Strategy.md)
|

| [FileSourceStrategy](execution-planning-strategies/FileSourceStrategy.md)
|

| [DataSourceStrategy](execution-planning-strategies/DataSourceStrategy.md)
|

| [SpecialLimits](execution-planning-strategies/SpecialLimits.md)
|

| [Aggregation](execution-planning-strategies/Aggregation.md)
|

| [JoinSelection](execution-planning-strategies/JoinSelection.md)
|

| [InMemoryScans](execution-planning-strategies/InMemoryScans.md)
|

| [BasicOperators](execution-planning-strategies/BasicOperators.md)
|
|===

NOTE: `SparkPlanner` extends spark-sql-SparkStrategies.md[SparkStrategies] abstract class.

=== [[creating-instance]] Creating SparkPlanner Instance

`SparkPlanner` takes the following when created:

* [[sparkContext]] spark-SparkContext.md[SparkContext]
* [[conf]] [SQLConf](SQLConf.md)
* [[experimentalMethods]] spark-sql-ExperimentalMethods.md[ExperimentalMethods]

[NOTE]
====
`SparkPlanner` is <<creating-instance, created>> in:

* BaseSessionStateBuilder.md[BaseSessionStateBuilder]
* hive/HiveSessionStateBuilder.md[HiveSessionStateBuilder]
* Structured Streaming's `IncrementalExecution`
====

=== [[extraPlanningStrategies]] Extension Point for Extra Planning Strategies -- `extraPlanningStrategies` Method

[source, scala]
----
extraPlanningStrategies: Seq[Strategy] = Nil
----

`extraPlanningStrategies` is an extension point to register extra spark-sql-SparkStrategy.md[planning strategies] with the query planner.

NOTE: `extraPlanningStrategies` are executed after <<extraStrategies, extraStrategies>>.

[NOTE]
====
`extraPlanningStrategies` is used when `SparkPlanner` <<strategies, is requested for planning strategies>>.

`extraPlanningStrategies` is overriden in the `SessionState` builders -- BaseSessionStateBuilder.md#planner[BaseSessionStateBuilder] and hive/HiveSessionStateBuilder.md#planner[HiveSessionStateBuilder].
====

=== [[collectPlaceholders]] Collecting PlanLater Physical Operators -- `collectPlaceholders` Method

[source, scala]
----
collectPlaceholders(plan: SparkPlan): Seq[(SparkPlan, LogicalPlan)]
----

`collectPlaceholders` collects all spark-sql-SparkStrategy.md#PlanLater[PlanLater] physical operators in the `plan` SparkPlan.md[physical plan].

NOTE: `collectPlaceholders` is part of catalyst/QueryPlanner.md#collectPlaceholders[QueryPlanner Contract].

=== [[prunePlans]] Pruning "Bad" Physical Plans -- `prunePlans` Method

[source, scala]
----
prunePlans(plans: Iterator[SparkPlan]): Iterator[SparkPlan]
----

`prunePlans` gives the input `plans` SparkPlan.md[physical plans] back (i.e. with no changes).

NOTE: `prunePlans` is part of catalyst/QueryPlanner.md#prunePlans[QueryPlanner Contract] to remove somehow "bad" plans.

=== [[pruneFilterProject]] Creating Physical Operator (Possibly Under FilterExec and ProjectExec Operators) -- `pruneFilterProject` Method

[source, scala]
----
pruneFilterProject(
  projectList: Seq[NamedExpression],
  filterPredicates: Seq[Expression],
  prunePushedDownFilters: Seq[Expression] => Seq[Expression],
  scanBuilder: Seq[Attribute] => SparkPlan): SparkPlan
----

!!! note
    `pruneFilterProject` is almost like [DataSourceStrategy.pruneFilterProjectRaw](execution-planning-strategies/DataSourceStrategy.md#pruneFilterProjectRaw).

`pruneFilterProject` branches off per whether it is possible to use a column pruning only (to get the right projection) and the input `projectList` columns of this projection are enough to evaluate all input `filterPredicates` filter conditions.

If so, `pruneFilterProject` does the following:

. Applies the input `scanBuilder` function to the input `projectList` columns that creates a new <<SparkPlan.md#, physical operator>>

. If there are Catalyst predicate expressions in the input `prunePushedDownFilters` that cannot be pushed down, `pruneFilterProject` creates a <<spark-sql-SparkPlan-FilterExec.md#creating-instance, FilterExec>> unary physical operator (with the unhandled predicate expressions)

. Otherwise, `pruneFilterProject` simply returns the physical operator

NOTE: In this case no extra <<spark-sql-SparkPlan-ProjectExec.md#, ProjectExec>> unary physical operator is created.

If not (i.e. it is neither possible to use a column pruning only nor evaluate filter conditions), `pruneFilterProject` does the following:

. Applies the input `scanBuilder` function to the projection and filtering columns that creates a new <<SparkPlan.md#, physical operator>>

. Creates a <<spark-sql-SparkPlan-FilterExec.md#creating-instance, FilterExec>> unary physical operator (with the unhandled predicate expressions if available)

. Creates a <<spark-sql-SparkPlan-ProjectExec.md#creating-instance, ProjectExec>> unary physical operator with the optional `FilterExec` operator (with the scan physical operator) or simply the scan physical operator alone

`pruneFilterProject` is used when [HiveTableScans](hive/HiveTableScans.md) and [InMemoryScans](execution-planning-strategies/InMemoryScans.md) execution planning strategies are executed.

title: InMemoryRelation

# InMemoryRelation Leaf Logical Operator -- Cached Representation of Dataset

`InMemoryRelation` is a spark-sql-LogicalPlan-LeafNode.md[leaf logical operator] that represents a cached `Dataset` by the <<child, physical query plan>>.

[[creating-instance]]
`InMemoryRelation` takes the following to be created:

* [[output]] Output schema spark-sql-Expression-Attribute.md[attributes]
* [[cacheBuilder]] <<spark-sql-CachedRDDBuilder.md#, CachedRDDBuilder>>
* [[statsOfPlanToCache]] spark-sql-Statistics.md[Statistics] of the <<child, child>> query plan
* [[outputOrdering]] Output <<spark-sql-Expression-SortOrder.md#, orderings>> (`Seq[SortOrder]`)

NOTE: `InMemoryRelation` is usually created using <<apply, apply>> factory methods.

`InMemoryRelation` is <<apply, created>> when:

* spark-sql-caching-and-persistence.md#persist[Dataset.persist] operator is used (that in turn requests `CacheManager` to spark-sql-CacheManager.md#cacheQuery[cache a structured query])

* `CatalogImpl` is requested to spark-sql-CatalogImpl.md#cacheTable[cache] or spark-sql-CatalogImpl.md#refreshTable[refresh] a table or view in-memory

* `InsertIntoDataSourceCommand` logical command is <<spark-sql-LogicalPlan-InsertIntoDataSourceCommand.md#run, executed>> (and in turn requests `CacheManager` to <<spark-sql-CacheManager.md#recacheByPlan, recacheByPlan>>)

* `CatalogImpl` is requested to spark-sql-CatalogImpl.md#refreshByPath[refreshByPath] (and in turn requests `CacheManager` to spark-sql-CacheManager.md#recacheByPath[recacheByPath])

* `QueryExecution` is requested for a spark-sql-QueryExecution.md#withCachedData[cached logical query plan] (and in turn requests `CacheManager` to spark-sql-CacheManager.md#useCachedData[replace logical query segments with cached query plans])

[source, scala]
----
// Cache sample table range5 using pure SQL
// That registers range5 to contain the output of range(5) function
spark.sql("CACHE TABLE range5 AS SELECT * FROM range(5)")
val q1 = spark.sql("SELECT * FROM range5")
scala> q1.explain
== Physical Plan ==
InMemoryTableScan [id#0L]
   +- InMemoryRelation [id#0L], true, 10000, StorageLevel(disk, memory, deserialized, 1 replicas), `range5`
         +- *Range (0, 5, step=1, splits=8)

// you could also use optimizedPlan to see InMemoryRelation
scala> println(q1.queryExecution.optimizedPlan.numberedTreeString)
00 InMemoryRelation [id#0L], true, 10000, StorageLevel(disk, memory, deserialized, 1 replicas), `range5`
01    +- *Range (0, 5, step=1, splits=8)

// Use Dataset's cache
val q2 = spark.range(10).groupBy('id % 5).count.cache
scala> println(q2.queryExecution.optimizedPlan.numberedTreeString)
00 InMemoryRelation [(id % 5)#84L, count#83L], true, 10000, StorageLevel(disk, memory, deserialized, 1 replicas)
01    +- *HashAggregate(keys=[(id#77L % 5)#88L], functions=[count(1)], output=[(id % 5)#84L, count#83L])
02       +- Exchange hashpartitioning((id#77L % 5)#88L, 200)
03          +- *HashAggregate(keys=[(id#77L % 5) AS (id#77L % 5)#88L], functions=[partial_count(1)], output=[(id#77L % 5)#88L, count#90L])
04             +- *Range (0, 10, step=1, splits=8)
----

`InMemoryRelation` is a <<spark-sql-MultiInstanceRelation.md#, MultiInstanceRelation>> so a <<newInstance, new instance will be created>> to appear multiple times in a physical query plan.

[source, scala]
----
// Cache a Dataset
val q = spark.range(10).cache

// Make sure that q Dataset is cached
val cache = spark.sharedState.cacheManager
scala> cache.lookupCachedData(q.queryExecution.logical).isDefined
res0: Boolean = true

scala> q.explain
== Physical Plan ==
InMemoryTableScan [id#122L]
   +- InMemoryRelation [id#122L], true, 10000, StorageLevel(disk, memory, deserialized, 1 replicas)
         +- *Range (0, 10, step=1, splits=8)

val qCrossJoined = q.crossJoin(q)
scala> println(qCrossJoined.queryExecution.optimizedPlan.numberedTreeString)
00 Join Cross
01 :- InMemoryRelation [id#122L], true, 10000, StorageLevel(disk, memory, deserialized, 1 replicas)
02 :     +- *Range (0, 10, step=1, splits=8)
03 +- InMemoryRelation [id#170L], true, 10000, StorageLevel(disk, memory, deserialized, 1 replicas)
04       +- *Range (0, 10, step=1, splits=8)

// Use sameResult for comparison
// since the plans use different output attributes
// and have to be canonicalized internally
import org.apache.spark.sql.execution.columnar.InMemoryRelation
val optimizedPlan = qCrossJoined.queryExecution.optimizedPlan
scala> optimizedPlan.children(0).sameResult(optimizedPlan.children(1))
res1: Boolean = true
----

[[simpleString]]
The catalyst/QueryPlan.md#simpleString[simple text representation] of a `InMemoryRelation` (aka `simpleString`) is *InMemoryRelation [output], [storageLevel]* (that uses the <<output, output>> and the <<cacheBuilder, CachedRDDBuilder>>).

[source, scala]
----
val q = spark.range(1).cache
val logicalPlan = q.queryExecution.withCachedData
scala> println(logicalPlan.simpleString)
InMemoryRelation [id#40L], StorageLevel(disk, memory, deserialized, 1 replicas)
----

`InMemoryRelation` is resolved to <<spark-sql-SparkPlan-InMemoryTableScanExec.md#, InMemoryTableScanExec>> leaf physical operator when [InMemoryScans](../execution-planning-strategies/InMemoryScans.md) execution planning strategy is executed.

[[internal-registries]]
.InMemoryRelation's Internal Properties (e.g. Registries, Counters and Flags)
[cols="1m,3",options="header",width="100%"]
|===
| Name
| Description

| partitionStatistics
| [[partitionStatistics]] <<PartitionStatistics, PartitionStatistics>> for the <<output, output>> schema

Used exclusively when `InMemoryTableScanExec` is <<creating-instance, created>> (and initializes spark-sql-SparkPlan-InMemoryTableScanExec.md#stats[stats] internal property).
|===

=== [[computeStats]] Computing Statistics -- `computeStats` Method

[source, scala]
----
computeStats(): Statistics
----

NOTE: `computeStats` is part of spark-sql-LogicalPlan-LeafNode.md#computeStats[LeafNode Contract] to compute statistics for spark-sql-cost-based-optimization.md[cost-based optimizer].

`computeStats`...FIXME

=== [[withOutput]] `withOutput` Method

[source, scala]
----
withOutput(newOutput: Seq[Attribute]): InMemoryRelation
----

`withOutput`...FIXME

NOTE: `withOutput` is used exclusively when `CacheManager` is requested to spark-sql-CacheManager.md#useCachedData[replace logical query segments with cached query plans].

=== [[newInstance]] `newInstance` Method

[source, scala]
----
newInstance(): this.type
----

NOTE: `newInstance` is part of spark-sql-MultiInstanceRelation.md#newInstance[MultiInstanceRelation Contract] to...FIXME.

`newInstance`...FIXME

=== [[cachedColumnBuffers]] `cachedColumnBuffers` Method

[source, scala]
----
cachedColumnBuffers: RDD[CachedBatch]
----

`cachedColumnBuffers`...FIXME

NOTE: `cachedColumnBuffers` is used when...FIXME

=== [[PartitionStatistics]] `PartitionStatistics`

[source, scala]
----
PartitionStatistics(tableSchema: Seq[Attribute])
----

NOTE: `PartitionStatistics` is a `private[columnar]` class.

`PartitionStatistics`...FIXME

NOTE: `PartitionStatistics` is used exclusively when `InMemoryRelation` is <<creating-instance, created>> (and initializes <<partitionStatistics, partitionStatistics>>).

=== [[apply]] Creating InMemoryRelation Instance -- `apply` Factory Methods

[source, scala]
----
apply(
  useCompression: Boolean,
  batchSize: Int,
  storageLevel: StorageLevel,
  child: SparkPlan,
  tableName: Option[String],
  logicalPlan: LogicalPlan): InMemoryRelation
apply(
  cacheBuilder: CachedRDDBuilder,
  logicalPlan: LogicalPlan): InMemoryRelation
----

`apply` creates an <<InMemoryRelation, InMemoryRelation>> logical operator.

NOTE: `apply` is used when `CacheManager` is requested to <<spark-sql-CacheManager.md#cacheQuery, cache>> and <<spark-sql-CacheManager.md#recacheByCondition, re-cache>> a structured query.

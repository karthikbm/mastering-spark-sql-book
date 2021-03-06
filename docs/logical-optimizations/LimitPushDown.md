# LimitPushDown Logical Optimization

`LimitPushDown` is a [base logical optimization](../Optimizer.md#batches) that <<apply, transforms>> the following logical plans:

* `LocalLimit` with `Union`
* `LocalLimit` with spark-sql-LogicalPlan-Join.md[Join]

`LimitPushDown` is part of the [Operator Optimization before Inferring Filters](../Optimizer.md#Operator_Optimization_before_Inferring_Filters) fixed-point batch in the standard batches of the [Logical Optimizer](../Optimizer.md).

`LimitPushDown` is simply a <<catalyst/Rule.md#, Catalyst rule>> for transforming <<spark-sql-LogicalPlan.md#, logical plans>>, i.e. `Rule[LogicalPlan]`.

[source, scala]
----
// test datasets
scala> val ds1 = spark.range(4)
ds1: org.apache.spark.sql.Dataset[Long] = [value: bigint]

scala> val ds2 = spark.range(2)
ds2: org.apache.spark.sql.Dataset[Long] = [value: bigint]

// Case 1. Rather than `LocalLimit` of `Union` do `Union` of `LocalLimit`
scala> ds1.union(ds2).limit(2).explain(true)
== Parsed Logical Plan ==
GlobalLimit 2
+- LocalLimit 2
   +- Union
      :- Range (0, 4, step=1, splits=Some(8))
      +- Range (0, 2, step=1, splits=Some(8))

== Analyzed Logical Plan ==
id: bigint
GlobalLimit 2
+- LocalLimit 2
   +- Union
      :- Range (0, 4, step=1, splits=Some(8))
      +- Range (0, 2, step=1, splits=Some(8))

== Optimized Logical Plan ==
GlobalLimit 2
+- LocalLimit 2
   +- Union
      :- LocalLimit 2
      :  +- Range (0, 4, step=1, splits=Some(8))
      +- LocalLimit 2
         +- Range (0, 2, step=1, splits=Some(8))

== Physical Plan ==
CollectLimit 2
+- Union
   :- *LocalLimit 2
   :  +- *Range (0, 4, step=1, splits=Some(8))
   +- *LocalLimit 2
      +- *Range (0, 2, step=1, splits=Some(8))
----

## <span id="apply"> Executing Rule

```scala
apply(plan: LogicalPlan): LogicalPlan
```

`apply`...FIXME

`apply` is part of the [Rule](../catalyst/Rule.md#apply) abstraction.

=== [[creating-instance]] Creating LimitPushDown Instance

`LimitPushDown` takes the following when created:

* [[conf]] spark-sql-CatalystConf.md[CatalystConf]

`LimitPushDown` initializes the <<internal-registries, internal registries and counters>>.

NOTE: `LimitPushDown` is created when

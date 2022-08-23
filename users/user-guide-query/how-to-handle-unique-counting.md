# Cardinality Estimation

Cardinality estimation is a classic problem. Pinot solves it with multiple ways each of which has a trade-off between accuracy and latency.

## Accurate Results

Functions:

* _**DistinctCount**(x) -> LONG_

Returns accurate count for all unique values in a column.

The underlying implementation is using a IntOpenHashSet in library: `it.unimi.dsi:fastutil:8.2.3` to hold all the unique values.

## Approximation Results

It usually takes a lot of resources and time to compute accurate results for unique counting on large datasets. In some circumstances, we can tolerate a certain error rate, in which case we can use approximation functions to tackle this problem.

### HyperLogLog

[HyperLogLog](https://en.wikipedia.org/wiki/HyperLogLog) is an approximation algorithm for unique counting. It uses fixed number of bits to estimate the cardinality of given data set.

Pinot leverages [HyperLogLog Class](https://github.com/addthis/stream-lib/blob/master/src/main/java/com/clearspring/analytics/stream/cardinality/HyperLogLog.java) in library `com.clearspring.analytics:stream:2.7.0`as the data structure to hold intermediate results.

Functions:

* **DistinctCountHLL**(x)\_ -> LONG\_

For column type **INT**/**LONG**/**FLOAT**/**DOUBLE**/**STRING** , Pinot treats each value as an individual entry to add into HyperLogLog Object, then compute the approximation by calling method **cardinality()**.

For column type **BYTES**, Pinot treats each value as a serialized HyperLogLog Object with pre-aggregated values inside. The bytes value is generated by `org.apache.pinot.core.common.ObjectSerDeUtils.HYPER_LOG_LOG_SER_DE.serialize(hyperLogLog)`.

All deserialized HyperLogLog object will be merged into one then calling method _\*\*cardinality() \*\*to get the approximated unique count._

### Theta Sketches

The [Theta Sketch](https://datasketches.apache.org/docs/Theta/ThetaSketchFramework.html) framework enables set operations over a stream of data, and can also be used for cardinality estimation. Pinot leverages the [Sketch Class](https://github.com/apache/incubator-datasketches-java/blob/master/src/main/java/org/apache/datasketches/theta/Sketch.java) and its extensions from the library `org.apache.datasketches:datasketches-java:1.2.0-incubating` to perform distinct counting as well as evaluating set operations.

Functions:

* **DistinctCountThetaSketch(**\<thetaSketchColumn>, \<thetaSketchParams>, predicate1, predicate2..., postAggregationExpressionToEvaluate\*\*) \*\*-> LONG
  * thetaSketchColumn (required): Name of the column to aggregate on.
  * thetaSketchParams (required): Parameters for constructing the intermediate theta-sketches. Currently, the only supported parameter is `nominalEntries`.
  * predicates (optional)\_: \_ These are individual predicates of form `lhs <op> rhs` which are applied on rows selected by the `where` clause. During intermediate sketch aggregation, sketches from the `thetaSketchColumn` that satisfies these predicates are unionized individually. For example, all filtered rows that match `country=USA` are unionized into a single sketch. Complex predicates that are created by combining (AND/OR) of individual predicates is supported.
  * postAggregationExpressionToEvaluate (required)_:_ The set operation to perform on the individual intermediate sketches for each of the predicates. Currently supported operations are `SET_DIFF, SET_UNION, SET_INTERSECT` , where DIFF requires two arguments and the UNION/INTERSECT allow more than two arguments.

In the example query below, the `where` clause is responsible for identifying the matching rows. Note, the where clause can be completely independent of the `postAggregationExpression`. Once matching rows are identified, each server unionizes all the sketches that match the individual predicates, i.e. `country='USA'` , `device='mobile'` in this case. Once the broker receives the intermediate sketches for each of these individual predicates from all servers, it performs the final aggregation by evaluating the `postAggregationExpression` and returns the final cardinality of the resulting sketch.

```sql
select distinctCountThetaSketch(
  sketchCol, 
  'nominalEntries=1024', 
  'country'=''USA'' AND 'state'=''CA'', 'device'=''mobile'', 'SET_INTERSECT($1, $2)'
) 
from table 
where country = 'USA' or device = 'mobile...' 
```

* **DistinctCountRawThetaSketch(**\<thetaSketchColumn>, \<thetaSketchParams>, predicate1, predicate2..., postAggregationExpressionToEvaluate\*\*)\*\* -> HexEncoded Serialized Sketch Bytes

This is the same as the previous function, except it returns the byte serialized sketch instead of the cardinality sketch. Since Pinot returns responses as JSON strings, bytes are returned as hex encoded strings. The hex encoded string can be deserialized into sketch by using the library `org.apache.commons.codec.binary`as `Hex.decodeHex(stringValue.toCharArray())`.
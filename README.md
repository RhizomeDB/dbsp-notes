# PomoFlow v0.1.0

## Editors

* [Quinn Wilton], [Fission Codes]
* [Brooklyn Zelenka], [Fission Codes]

## Authors

* [Quinn Wilton], [Fission Codes]
* [Brooklyn Zelenka], [Fission Codes]

# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119].

# Abstract

This document describes PomoFlow, a dataflow runtime for PomoDB. This design is especially suited for incrementalizing programs to efficiently compute over deltas to the input EDB.

# 1. Introduction

Most relational query runtimes implement a variant of semi-naive evaluation, where queries are run over the course of multiple iterations, with each iteration refining the result toward a fixed point. Such designs can efficiently compute views over a fixed database, but are insufficient for incrementalizing evaluation over a database that changes over time.

This document describes an alternative runtime, built on dataflow, which represents programs as circuits whose vertices and edges correspond to computations over streams of data. These circuits can be incrementalized to instead operate over deltas, with the results being combined back into a materialized view.

The design is based on ideas from Differential Dataflow, and is heavily inspired by the Database Stream Processor Framework (DBSP). Links to both can be found in the [research repo][Pomo Research].

# 2. Concepts

## 2.1 ZSet

A set of elements, each associated with a [weight] and [timestamp].

ZSets can be written as lists of triples:

```
[
    {element1, timestamp1, weight1},
    {element2, timestamp2, weight2},
    ...
]
```

Implementations should support efficient sequential access of a ZSet, first by element, and then by timestamp.

Some operations return ZSets without timestamp information, and implementations MAY specialize a variant of the data structure for those cases.

## 2.2 Indexed ZSet

A set of key-value pairs, each associated with a [weight] and [timestamp].

Indexed ZSets can be written as lists of triples:

```
[
    {{key1, value1}, timestamp1, weight1},
    {{key2, value2}, timestamp2, weight2},
    ...
]
```

Implementations should support efficient sequential access of an Indexed ZSet, first by key, and then by value, and lastly by timestamp.

Some operations return Indexed ZSets without timestamp information, and implementations MAY specialize a variant of the data structure for those cases.

## 2.3 Trace

A collection of deltas, combining multiple ZSets or Indexed ZSets.

Traces can be written as lists of 4-tuples:

```
[
    {key1, value1, timestamp1, weight1},
    {key2, value2, timestamp2, weight2},
    ...
]
```

Indexed ZSets should be merged into a trace using the key, value, timestamp, and weight associated with each element. ZSets should be merged into the trace by treating the element itself as a key, and associating no value with the delta.

Implementations MUST support efficient sequential access of traces, first by key, and then by value, and lastly by timestamp.

## 2.4 Weight

An integer associated with each element of a ZSet. Positive weights indicate the number of derivations of that element within the ZSet, and negative weights indicate the number of deletions of that element.

## 2.5 Time

PomoFlow represents [timestamp]s for the root circuit using a counter which denotes the current epoch.

Every recursive subcircuit refines its parent's timestamp using a pair which further associates that timestamp with the iteration count through the subcircuit. These pairs are ordered using product order.

Implementations MUST represent both epochs and iteration counts as an unsigned integer.

For example, a root circuit may progress through the following timestamps:

```
[0, 1, 2, 3, ...]
```

Whereas a subcircuit under that root may progress through these:

```
[
    (0, 0),
    (0, 1),
    (0, 2),
    (1, 0),
    (1, 1),
    (1, 2),
    ...
]
```

Under this example, using product order, `(0, 0) <= (0, 1) <= (1, 1)`, but neither `(0, 2) <= (1, 1)` or `(1, 1) <= (2, 0)`. In all of these cases, the epoch of each timestamp is the first component of the pair, and the iteration is given by its second.

## 2.6 Circuit

A circuit is an embedding of a PomoRA query plan into a directed graph whose vertices, called [nodes], represent computation against [streams], and whose edges describe those streams.

Circuits may contain subcircuits, representing recursive subcomputations that evaluate to a fixed point every iteration.

Each iteration, a circuit is evaluated by evaluating its nodes in topological order, until each has reached a fixed point.

## 2.7 Stream

A stream is an infinite sequence of values, each associated with subsequent timestamps. It is RECOMMENDED that streams not be reified directly, and they MAY instead be modeled as a cell containing the singular value in the stream at the current timestamp.

The edges between nodes in a circuit describe streams of values flowing from the output of one node to the input of another, and these edges SHOULD be defined in terms of the IDs of the nodes they connect.

## 2.8 Node

A node is a vertex of a circuit, and describes a computation over the circuit's streams.

Every node is associated with both a local ID and a global ID. Local IDs MUST be unique among all nodes belonging to the same circuit, and global IDs MUST be unique across all nodes.

It is RECOMMENDED that local IDs be represented by a node's index into its parents nodes, and that global IDs be represented as the path formed by the local IDs of a node's ancestors.

| Type       | Inputs | Outputs |
| ---------- | ------ | ------- |
| [Operator] | N-ary  | 1       |
| [Child]    | 0      | N-ary   |
| [Feedback] | 1      | 1       |
| [Import]   | 1      | 1       |
| [Sink]     | N-ary  | 0       |
| [Source]   | 0      | 1       |

## 2.8.1 Operator Node

Performs an [operation] against its input streams, outputting the result over a stream.

Operator nodes achieve a fixed point when their associated operator has done so.

## 2.8.2 Child Node

Introduces a subcircuit that can be used to perform recursive computations. Such circuits evaluate to a fixed point each iteration, using the same rules as the root circuit, then emit their result to downstream nodes.

## 2.8.3 Feedback Node

Introduces a temporal edge between nodes in subsequent iterations of a circuit. Such nodes support persisting data between iterations of a circuit, and are also used to implement recursive circuits by propagating partial results forward in time until a fixed point is reached.

These nodes introduce apparently cycles into a circuit, however implementations SHOULD NOT treat them as such, and SHOULD consider their output stream to be lazily evaluated as part of the subsequent iteration.

Intuitively, these nodes can be thought of as delaying a stream by one iteration.

## 2.8.4 Import Node

Imports a parent stream into a subcircuit.

## 2.8.5 Sink Node

Performs an operation against its input streams, without outputting anything.

## 2.8.6 Source Node

Emits data over a stream, without accepting any inputs from the circuit.

## 2.9 Operator

An operator specifies an operation against a stream, and is represented by a node. Operators MAY be stateful, and linear operators are those which can be computed using only the deltas at the current timestamp.

| Name                   | Linearity  | Input Types                | Output Type  |
| ---------------------- | ---------- | ---------------------------| ------------ |
| [Aggregate]            | Varies     | Indexed ZSet               | ZSet         |
| [Consolidate]          | Linear     | Trace                      | ZSet         |
| [Distinct]             | Non-Linear | ZSet                       | ZSet         |
| [Filter]               | Linear     | ZSet                       | ZSet         |
| [Index With]           | Linear     | ZSet                       | Indexed ZSet |
| [Inspect]              | Linear     | ZSet                       | ZSet         |
| [Map]                  | Linear     | ZSet                       | ZSet         |
| [Negate]               | Linear     | ZSet                       | ZSet         |
| [Z1 Trace]             | Linear     | Trace                      | Trace        |
| [Z1]                   | Linear     | ZSet                       | ZSet         |
| [Distinct Trace]       | Non-Linear | ZSet, Trace                | ZSet         |
| [Join Stream]          | Linear     | Indexed ZSet, Indexed ZSet | ZSet         |
| [Join Trace]           | Bilinear   | Indexed ZSet, Trace        | ZSet         |
| [Minus]                | Linear     | ZSet                       | ZSet         |
| [Plus]                 | Linear     | ZSet                       | ZSet         |
| [Trace Append]         | Non-Linear | ZSet, Trace                | Trace        |
| [Untimed Trace Append] | Non-Linear | ZSet, Trace                | Trace        |
| [Distinct Incremental] | Non-Linear | ZSet, Trace                | ZSet         |

## 2.9.1 Aggregate Operator

Applies an aggregate to all elements of the input stream.

This operator takes an [Indexed ZSet] as input, and applies an aggregate function over it, to return a [ZSet] that summarizes the values under each key, and associating a weight of `1` to each element. The resulting ZSet has no timestamps associated with any element.

Implementations MAY support user defined aggregates, but MUST support the aggregate functions described in the [specification for the query language][PomoLogic Aggregates].

If additional aggregates are supported, they MUST be pure functions, and implementations are RECOMMENDED to enforce this constraint.

For example:
```
aggregate(count, [
    {{1, "foo"}, 0, 1},
    {{1, "bar"}, 0, 1},
    {{2, "baz"}, 0, 1}
]) => [
    {{1, 2}, nil, 1},
    {{2, 1}, nil, 1}
]
```

## 2.9.2 Consolidate Operator

Merges all deltas in the input trace into a [ZSet]. The resulting ZSet has no timestamps associated with any element, and sums the weights for each key-value pair. Elements with weight equal to zero are discarded.

This operator is intended to combine deltas from across multiple timestamps into a single ZSet, and is used to export the results of a recursive subcircuit to the parent for further processing.

For example:
```
consolidate([
    {0, 0, 0, 1},
    {0, 0, 0, -1},
    {0, 1, 0, 1},
    {0, 1, 0, 1}.
    {1, 2, 0, 2},
    {1, 3, 0, 1},
    {1, 3, 0, -1},
    {1, 4, 0 -1},
    {2, 2, 0, 1},
    {2, 4, 0 1}
]) => [
    {{0, 1}, nil, 2},
    {{1, 2}, nil, 2},
    {{1, 4}, nil, -1},
    {{2, 2}, nil, 1},
    {{2, 4}, nil, 1}
]
```

## 2.9.3 Distinct Operator

Returns a ZSet containing elements in the input [ZSet] with positive weight, replacing their weight with `1`.

For example:
```
distinct([
    {0, 0, 1},
    {1, 0, 2},
    {2, 0, -1},
    {3, 0, 0}
]) => [
    {0, 0, 1},
    {1, 0, 1}
]
```

## 2.9.4 Filter Operator

Filters a [ZSet] by a predicate. The predicate MUST be a pure function, returning a boolean.

```
predicate = fn x -> x >= 1 end

filter(predicate, [
    {0, 0, 1},
    {1, 0, 2},
    {2, 1, -3}
]) => [
    {1, 0, 2},
    {2, 1, -3}
]
```

## 2.9.5 Index With Operator

Groups elements of a [ZSet] according to some key function, returning an [Indexed ZSet].

For example:
```
key_function = fn {src, dst, cost} -> src end

index_with(key_function, [
    {{0, 1, 1}, 0, 1},
    {{1, 2, 1}, 1, 1},
    {{1, 3, 2}, 1, -1}
]) => [
    {{0, {0, 1, 1}}, 0, 1},
    {{1, {1, 2, 1}}, 1, 1},
    {{1, {1, 3, 2}}, 1, -1}
]
```

## 2.9.6 Inspect Operator

Applies a callback to a [ZSet], returning the original ZSet.

This operator is primarily intended as a debugging aid, and can be used to output the contents of streams at runtime.

For example:
```
inspect_fun = fn x -> IO.inspect(x) end

inspect(inspect_fun, [
    {0, 1, 1},
    {1, 1, 1}
]) => [
    {0, 1, 1},
    {1, 1, 1}
] # Also printing out [{0, 1, 1}, {1, 1, 1}]
```

## 2.9.7 Map Operator

Transforms elements of a [ZSet] according to some function. The predicate MUST be a pure function.

For example:
```
predicate = fn x -> x >= 1 end

filter(predicate, [
    {0, 0, 1},
    {1, 0, 2},
    {2, 1, -3}
]) => [
    {1, 0, 2},
    {2, 1, -3}
]
```

## 2.9.8 Negate Operator

Negates the weights of each element in a [ZSet].

For example:
```
negate([
    {0, 0, 1},
    {1, 0, -1},
    {2, 0, -2}
]) => [
    {0, 0, -1},
    {1, 0, 1},
    {2, 0, 2}
]
```

## 2.9.9 Z1 Trace Operator

Returns the previous input [trace].

## 2.9.10 Z1 Operator

Returns the previous input [ZSet].

## 2.9.11 Distinct Trace Operator

A variant of [Distinct] that offers more performance for incremental computation and computes across multiple timestamps, with support for use in nested contexts, like recursive circuits. It computes the distinct elements of a [ZSet] in its first argument, with respect to a [Trace] in its second, returning them in a new ZSet. The resulting ZSet has no timestamps associated with any element.

Note that because operator computes the delta of [Distinct], it is possible for returned elements to have negative weights, if those elements are deleted between timestamps.

Distinct is not a linear operation, and requires access to the entire history of updates, however there's a few observations that can reduce the search space of timestamps to examine.

Consider evaluating the operator at some [timestamp] `t = (e, i)`, where `e` denotes the epoch, and `i` denotes the iteration.

Then there are two possible classes of elements which may be returned:
1) Elements in the current input ZSet
2) Elements that were returned in response to a previous input, at timestamp `(e, i0)`, such that `i0 < i`, and where the element was also returned at timestamp `(e0, i)`, such that `e0 < e`

For each element, with weight `w`, meeting at least one of the above requirements, if that element does not appear in the trace, it is returned in the ZSet with weight 1. Otherwise, the following routine is performed:

1) `w1` is computed, as the sum of all weights in which that element appears at times `(e0, i0)` for `e0 < e` and `i0 < i`
2) `w2` is computed, as the sum of all weights in which that element appears at times `(e0, i)` for `e0 < e`
3) `w3` is computed, as the sum of all weights in which that element appears at times `(e, i0)` for `i0 < i`
4) `d0` is computed, such that:
   1) `d0 = 1`, if `w1 <= = && w1 + w2 > 0`
   2) `d0 = -1`, if `w1 > 0 && w1 + w2 <= 0`
   3) `d0 = 0`, otherwise
5) `d1` is computed, such that:
   1) `d1 = 1`, if `w1 + w3 <= 0 && w1 + w2 + w3 + w > 0`
   2) `d1 = -1`, if `w1 + w3 > 0 && w1 + w3 + w4 <= 0`
6) If `d1 - d0 != 0`, then the element is returned in the ZSet, with weight `d1 - d0`

For example:

```
b00 = [
    {0, {0, 0}, 1},
    {2, {0, 0}, 1},
    {3, {0, 0}, -1}
]

b01 = [
    {5, {0, 1}, 1}
]

b10 = [
    {5, {1, 0}, 1}
]

b11 = [
    {0, {1, 1}, 1},
    {1, {1, 1}, 1},
    {2, {1, 1}, -1},
    {3, {1, 1}, 1},
    {4, {1, 1}, -1}
]

t00 = []
t01 = insert_trace(t00, b00)
t10 = insert_trace(t01, b01)
t11 = insert_trace(t10, b10)

# At time {0, 0}
distinct_trace(b00, t00) => [
    {0, nil, 1},
    {2, nil, 1}
]

# At time {0, 1}
distinct_trace(b01, t01) => [
    {5, nil, 1}
]

# At time {1, 0}
distinct_trace(b10, t10) => [
    {5, nil, 1}
]

# At time {1, 1}
distinct_trace(b11, t11) => [
    {1, nil, 1},
    {2, nil, -1},
    {5, nil, -1}
]
```

## 2.9.12 Join Stream Operator

Joins two [Indexed ZSets] together. It applies a join function to values with matching keys, returning a [ZSet] containing the resulting elements, with no timestamps associated with those elements, and each element's weight given by the product of the two elements' weights that were joined together.

For example:
```
join_fun =
    fn key, v1, v2 ->
        {key, {v1, v2}}
    end

a = [
    {{:a, 1}, 0, 1},
    {{:b, 2}, 0, 2},
    {{:c, 2}, 0, 1}
]

b = [
    {{:a, 1}, 0, 1},
    {{:b, 3}, 0, 1},
    {{:b, 4}, 0, -1}
]

join_stream(join_fun, a, b) => [
    {{:a, {1, 1}}, nil, 1},
    {{:b, {2, 3}}, nil, 2},
    {{:b, {2, 4}}, nil, -2}
]
```

## 2.9.13 Join Trace Operator

A variant of join that joins an [Indexed ZSet] with a [Trace]. This takes advantage of the bilinearity of relational joins in order to support incremental joins across timestamps.

It returns a ZSet containing the resulting elements, with no timestamps associated with each of those elements, and each element's weight given by the product of the two elements' weights that were joined together.

It behaves similarly to [Join Stream], and the first argument represents deltas for the current timestamp, with the second being a trace containing all updates observed thus far. In this way, an incremental join can be implemented as follows:

```
join_fun = ...

join_fun_flipped =
    fn k, v1, v2 ->
        join_fun(k, v2, v1)
    end

a = ... # some ZSet
b = ... # some ZSet

a_trace = z1_trace(a)
b_trace = z1_trace(b)

incremental_join(join_fun, a, b) =
    join_stream(join_fun, a, b)      +
    join_trace(join_fun, a, b_trace) +
    join_trace(join_fun_flipped, b, a_trace)
```

Where `z1_trace(x)` denotes an application of the [Z1 Trace] operator, `join_fun_flipped` flips the value arguments of `join_fun`, and `+` denotes the [Plus Operator].

For example:
```
join_fun =
    fn key, v1, v2 ->
        {key, {v1, v2}}
    end

zset = [
    {{:a, 0}, 0, 1},
    {{:a, 0}, 0, -1},
    {{:a, 1}, 0, 1},
    {{:b, 2}, 0, 2},
    {{:c, 2}, 0, 1}
]

trace = [
    {:a, 1, 0, 1},
    {:b, -3, 0, -1},
    {:b, 3, 0, 1},
    {:b, 4, 0, -1},
    {:c, 4, 0, 1}
]

join_trace(join_fun, zset, trace) => [
    {{:a, 1, 1}, nil, 1},
    {{:b, 2, -3}, nil, -2},
    {{:b, 2, 3}, nil, 2},
    {{:b, 2, 4}, nil, -2},
    {{:c, 2, 4}, nil, 1}
]
```

## 2.9.14 Minus Operator

Subtracts all weights for matching elements in two ZSets. Elements with combined weight equal to zero are discarded.

For example:
```
a = [
    {0, 0, 1},
    {1, 0, 1},
    {2, 0, 2},
    {3, 0, 1}
]

b = [
    {0, 0, 1},
    {1, 0, -1},
    {2, 0, 1}
]

minus(a, b) => [
    {1, 0, 2},
    {2, 0, 1},
    {3, 0, 1}
]
```

## 2.9.15 Plus Operator

Adds all weights for matching keys in two ZSets. Elements with combined weight equal to zero are discarded.

For example:
```
a = [
    {0, 0, 1},
    {1, 0, 1},
    {2, 0, 2},
    {3, 0, 1}
]

b = [
    {0, 0, 1},
    {1, 0, -1},
    {2, 0, 1}
]

plus(a, b) => [
    {0, 0, 2},
    {2, 0, 3},
    {3, 0, 1}
]
```

## 2.9.16 Trace Append Operator

Inserts the input ZSet into the input Trace, with the current timestamp.

## 2.9.17 Untimed Trace Append Operator

Inserts the input ZSet into the input Trace 

## 2.9.18 Distinct Incremental Operator

A variant of [Distinct] that offers more performance for incremental computation and computes across multiple timestamps. It computes the distinct elements of a [ZSet] in its first argument, with respect to a [Trace] in its second, returning them in a new ZSet. The resulting ZSet has no timestamps associated with any element.

Note that because operator computes the delta of [Distinct], it is possible for returned elements to have negative weights, if those elements are deleted between timestamps.

This computation can be performed by returning the elements in the ZSet whose weight has the opposite sign as the sum of all matching weights in the Trace.

For example:
```
batch = [
    {0, nil, 2},
    {2, nil, 1},
    {3, nil, -1}
]

distinct(batch, [
    {0, 0, 1},
]) => [
    {2, nil, 1}
]

distinct(batch, [
    {2, 1, 1},
    {3, 1, 1}
]) => [
    {0, nil, 1},
    {3, nil, -1}
]

distinct(batch, [
    {0, 2, -1},
]) => [
    {0, nil, 1},
    {2, nil, 1}
]
```

<!-- Links -->

[Aggregate]: #291-aggregate-operator
[Brooklyn Zelenka]: https://github.com/expede
[Child]: #282-child-node
[Consolidate]: #292-consolidate-operator
[Distinct Incremental]: #2918-distinctincremental-operator
[Distinct Trace]: #2911-distincttrace-operator
[Distinct]: #293-distinct-operator
[Feedback]: #283-feedback-node
[Filter]: #294-filter-operator
[Fission Codes]: https://fission.codes
[Import]: #284-import-node
[Index With]: #295-indexwith-operator
[Indexed ZSet]: #22-indexed-zset
[Inspect]: #296-inspect-operator
[Join Stream]: #2912-joinstream-operator
[Join Trace]: #2913-jointrace-operator
[Map]: #297-map-operator
[Minus]: #2914-minus-operator
[Negate]: #298-negate-operator
[Operator]: #281-operator-node
[Plus Operator]: #2915-plus-operator
[Plus]: #2915-minus-operator
[Pomo Research]: https://github.com/RhizomeDB/research
[PomoLogic Aggregates]: https://github.com/RhizomeDB/PomoLogic#253-aggregation
[Quinn Wilton]: https://github.com/QuinnWilton
[RFC 2119]: https://datatracker.ietf.org/doc/html/rfc2119
[Sink]: #285-sink-node
[Source]: #286-source-node
[Trace Append]: #2916-traceappend-operator
[Untimed Trace Append]: #2917-untimedtraceappend-operator
[Z1 Trace]: #299-z1trace-operator
[Z1]: #2910-z1-operator
[ZSet]: #21-zset
[nodes]: #28-node
[operation]: #29-operator
[streams]: #27-stream
[timestamp]: #25-time
[trace]: #23-trace
[weight]: #24-weight

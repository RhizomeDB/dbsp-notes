# Notes on DBSP
## Incrementalizing Queries

# 1. Introduction

Most datalog query runtimes implement a variant of [semi-naive evaluation], where queries are iteratively refined until they reach a fixed point. Such designs can efficiently compute views over a fixed database, but are insufficient for incrementalizing evaluation over a database that changes over time. Since semi-naive evaluation of nonmonotonic queries requires evaluation on changes to the input, it can be inefficient for many applications, especially when real time operation is desired.

This document describes an alternative runtime based in the [dataflow] model. This represents programs as circuits (graphs) whose vertices and edges correspond to computation over streams of data. These circuits MAY be incrementalized to instead operate over deltas, with the results combined into a final materialized view. Converting from relation algebra to dataflow is a fully mechanical, static transformation. 

[DBSP] and [Differential Dataflow] were not developed as part of the Rhizome DB project. The papers and documentation lacked some detail required to implement such a system. This document contains the basics that are required to put these ideas into practice.

# 2. Concepts

The following describes the basic concepts and data types:

## 2.1 $\Bbb{Z}$-Set

A set of triples, each associated with a [weight] and [timestamp]: `{element, timestamp, weight}`.

| Index | Name      | Description                                              |
|-------|-----------|----------------------------------------------------------|
| 0     | Element   | The actual data being processed                          |
| 1     | Timestamp | Information about processing order                       |
| 2     | Weight    | How many dependencies (or deletions) its responsible for |

$\Bbb{Z}$-Sets MAY be written as lists of triples:

```elixir
[
    {element1, timestamp1, weight1},
    {element2, timestamp2, weight2},
    ...
]
```

Implementations SHOULD support efficient sequential access of a $\Bbb{Z}$-Set, first by element, and then by timestamp.

Some operations return $\Bbb{Z}$-Sets without timestamp information, and implementations MAY specialize a variant of the data structure for those cases.

## 2.2 Indexed $\Bbb{Z}$-Set

A set of key-value pairs, each associated with a [weight] and [timestamp].

Indexed $\Bbb{Z}$-Sets MAY be written as lists of triples:

```elixir
[
    {{key1, value1}, timestamp1, weight1},
    {{key2, value2}, timestamp2, weight2},
    ...
]
```

Implementations SHOULD support efficient sequential access of an Indexed $\Bbb{Z}$-Set in the following order:

1. By key
2. By value
3. By timestamp.

Some operations return Indexed $\Bbb{Z}$-Sets without timestamp information, and implementations MAY specialize a variant of the data structure for those cases.

## 2.3 Trace

A collection of deltas, combining multiple $\Bbb{Z}$-Sets or Indexed $\Bbb{Z}$-Sets.

Traces MAY be written as lists of 4-tuples:

```elixir
[
    {key1, value1, timestamp1, weight1},
    {key2, value2, timestamp2, weight2},
    ...
]
```

Indexed $\Bbb{Z}$-Sets SHOULD be merged into a trace using the key, value, timestamp, and weight associated with each element.

$\Bbb{Z}$-Sets SHOULD be merged into the trace by treating the element itself as a key, and associating no value with the delta.

Implementations MUST support efficient sequential access of traces, first by key, and then by value, and lastly by timestamp.

## 2.4 Weight

Weights describe the number of downstream dependency count for a [𝕫-Set]. It MUST be represented by an integer, and MAY be either positive or negative.

Positive weights indicate the number of derivations of that element within the $\Bbb{Z}$-Set. Negative weights indicate the number of deletions of that element.

## 2.6 Time

Dataflow engines require a concept of ordering. Each node in the circuit MUST be labeled with an incrementing integer that represents the number of times that it has gone through that subcircuit. [DBSP] represents [timestamp]s for the root circuit using a counter which denotes the current epoch.

Every recursive subcircuit MUST refine its parent's timestamp. This is signalled via a pair which associates the timestamp with the iteration count through the subcircuit. The pairs MUST be given in the following order: `{epoch, iteration}`. Both values MUST be given as unsigned integers. They form a [partial order], and MUST use [product order] for comparison.

### 2.6.1 Product Order

Under this example, using product order, $\langle 0, 0 \rangle \le \langle 0, 1 \rangle \le \langle 1, 1 \rangle$, but neither $\langle 0, 2 \rangle \not\le \langle 1, 1 \rangle$ nor $\langle 1, 1 \rangle \not\le \langle 2, 0 \rangle$ are directly comparable.

### 2.6.2 Example

Below is an example of a path through a root circuit:

```elixir
[0, 1, 2, 3, ...]
```

Whereas a subcircuit under that root may progress through these:

```elixir
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

## 2.6 Circuit

A circuit is an embedding of a [PomoRA] query plan into a directed (potentially cyclic) graph whose vertices ([nodes]) represent computation over [streams], and whose edges describe those streams.

Circuits MAY contain subcircuits, which MUST represent recursive subcomputations that evaluate to a fixed point by the end of every iteration. For each iteration, a circuit is run by evaluating its nodes (subcircuits and single nodes) in some topologically sorted order, until a fixed point is reached.

## 2.7 Stream

A stream is an infinite sequence of values. A node processing a stream progresses one tuple at a time, and increments its counter after every tuple. It is RECOMMENDED that streams not be reified directly, and they MAY instead be modeled as cells containing the element field in the stream at the current timestamp.

The edges between nodes in a circuit describe streams of values flowing from the output of one node to the input of another. It is RECOMMENDED that these edges be defined in terms of the IDs of the nodes they connect.

## 2.8 Node

A node is a vertex of a circuit, and describes a computation over the circuit's streams.

Every node is associated with both a local ID and a global ID. Local IDs MUST be unique among all nodes belonging to the same circuit, and global IDs MUST be unique across all nodes.

It is RECOMMENDED that local IDs be represented by a node's index into its parents nodes, and that global IDs be represented as the path formed by the local IDs of a node's ancestors.

Nodes come in several types:

| Type       | Inputs | Outputs |
| ---------- | ------ | ------- |
| [Operator] | N-ary  | 1       |
| [Child]    | 0      | N-ary   |
| [Feedback] | 1      | 1       |
| [Import]   | 1      | 1       |
| [Sink]     | N-ary  | 0       |
| [Source]   | 0      | 1       |

## 2.8.1 Operator Node

An operator node performs an [operation] on its input stream(s), outputting the result over a stream.

Operator nodes achieve a fixed point when their associated operator has done so.

## 2.8.2 Child Node

A child node introduces a subcircuit that MUST used to perform recursive computations. Such circuits evaluate to a fixed point each iteration, using the same rules as the root circuit, then emit their result to downstream nodes.

## 2.8.3 Feedback Node

Feedback nodes introduce a temporal edge between nodes in subsequent iterations of a circuit. Such nodes support persisting data between iterations of a circuit, and are also used to implement recursive circuits by propagating partial results forward in time until a fixed point is reached.

These nodes introduce apparent cycles into a circuit, however implementations SHOULD NOT treat them as such. Instead, their output stream SHOULD be lazily evaluated as part of the subsequent iteration.

Intuitively, these nodes can be thought of as delaying a stream by one iteration.

## 2.8.4 Import Node

Import nodes attach a parent stream to a subcircuit.

## 2.8.5 Sink Node

Sink nodes perform an operation against its input streams. They MUST NOT produce circuit outputs.

## 2.8.6 Source Node

Source nodes emit data over a stream. They MUST NOT accept any inputs from the circuit.

## 2.9 Operator

An operator specifies an operation against a stream, and is represented by a node. Operators MAY be stateful, and linear operators are those which can be computed using only the deltas at the current timestamp.

| Name                   | Linearity  | Input Types                                  | Output Type           |
|------------------------|------------|----------------------------------------------|-----------------------|
| [Aggregate]            | Varies     | Indexed $\Bbb{Z}$-Set                        | $\Bbb{Z}$-Set         |
| [Consolidate]          | Linear     | Trace                                        | $\Bbb{Z}$-Set         |
| [Distinct]             | Non-Linear | $\Bbb{Z}$-Set                                | $\Bbb{Z}$-Set         |
| [Filter]               | Linear     | $\Bbb{Z}$-Set                                | $\Bbb{Z}$-Set         |
| [Index With]           | Linear     | $\Bbb{Z}$-Set                                | Indexed $\Bbb{Z}$-Set |
| [Inspect]              | Linear     | $\Bbb{Z}$-Set                                | $\Bbb{Z}$-Set         |
| [Map]                  | Linear     | $\Bbb{Z}$-Set                                | $\Bbb{Z}$-Set         |
| [Negate]               | Linear     | $\Bbb{Z}$-Set                                | $\Bbb{Z}$-Set         |
| [𝓏⁻¹ Trace]            | Linear     | Trace                                        | Trace                 |
| [𝓏⁻¹]                  | Linear     | $\Bbb{Z}$-Set                                | $\Bbb{Z}$-Set         |
| [Distinct Trace]       | Non-Linear | $\Bbb{Z}$-Set, Trace                         | $\Bbb{Z}$-Set         |
| [Join Stream]          | Linear     | Indexed $\Bbb{Z}$-Set, Indexed $\Bbb{Z}$-Set | $\Bbb{Z}$-Set         |
| [Join Trace]           | Bilinear   | Indexed $\Bbb{Z}$-Set, Trace                 | $\Bbb{Z}$-Set         |
| [Minus]                | Linear     | $\Bbb{Z}$-Set                                | $\Bbb{Z}$-Set         |
| [Plus]                 | Linear     | $\Bbb{Z}$-Set                                | $\Bbb{Z}$-Set         |
| [Trace Append]         | Non-Linear | $\Bbb{Z}$-Set, Trace                         | Trace                 |
| [Untimed Trace Append] | Non-Linear | $\Bbb{Z}$-Set, Trace                         | Trace                 |
| [Distinct Incremental] | Non-Linear | $\Bbb{Z}$-Set, Trace                         | $\Bbb{Z}$-Set         |

## 2.9.1 Aggregate Operator

The aggregate operator takes an [Indexed Z-Set] as input, and applies an aggregate function over it, to return a [𝕫-Set] that summarizes the values under each key, and associating a weight of `1` to each element. The resulting $\Bbb{Z}$-Set has no timestamps associated with any element.

Implementations MAY support user defined aggregates, but MUST support the aggregate functions described in the [specification for the query language][PomoLogic Aggregates].

If additional aggregates are supported, they MUST be pure functions. It is RECOMMENDED that implementations enforce this constraint.

For example:

```elixir
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

Consolidation operators merge all deltas in the input trace into a [𝕫-Set]. The resulting $\Bbb{Z}$-Set has no timestamps associated with any element, and sums the weights for each key-value pair. Elements with weight equal to zero are discarded.

This operator is intended to combine deltas from across multiple timestamps into a single $\Bbb{Z}$-Set, and is used to export the results of a recursive subcircuit to the parent for further processing.

For example:

```elixir
consolidate([
    {0, 0, 0, 1},
    {0, 0, 0, -1},
    {0, 1, 0, 1},
    {0, 1, 0, 1}.
    {1, 2, 0, 2},
    {1, 3, 0, 1},
    {1, 3, 0, -1},
    {1, 4, 0, -1},
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

Returns a $\Bbb{Z}$-Set containing elements in the input [𝕫-Set] with positive weight, replacing their weight with `1`.

For example:

```elixir
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

Filters a [𝕫-Set] by a predicate. The predicate MUST be a pure function, returning a boolean.

```elixir
is_positive = fn x -> x >= 1 end

filter(is_positive, [
    {0, 0, 1},
    {1, 0, 2},
    {2, 1, -3}
]) => [
    {1, 0, 2},
    {2, 1, -3}
]
```

## 2.9.5 Index With Operator

Indexing operators group elements of a [𝕫-Set] according to some key function, returning an [Indexed Z-Set].

For example:

```elixir
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

The inspect operator applies a callback to a [𝕫-Set], and returns the original $\Bbb{Z}$-Set.

This operator is primarily intended as a debugging aid, and can be used to output the contents of streams at runtime.

For example:

```elixir
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

The map operator transforms elements of a [𝕫-Set] according to some function. The predicate MUST be a pure function.

For example:

```elixir
is_positive = fn x -> x >= 1 end

filter(is_positive, [
    {0, 0, 1},
    {1, 0, 2},
    {2, 1, -3}
]) => [
    {1, 0, 2},
    {2, 1, -3}
]
```

## 2.9.8 Negate Operator

The negate operator flips the sign on the weight of each element in a [𝕫-Set].

For example:

```elixir
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

## 2.9.9 $z^{-1}$ Trace Operator

The trace operator returns the previous input [trace].

## 2.9.10 $z^{-1}$ Operator

The $z^{-1}$ operator returns the previous input [𝕫-Set].

## 2.9.11 Distinct Trace Operator

Distinct trace is a variant of [Distinct] that offers more performance for incremental computation and computes across multiple timestamps, with support for use in nested contexts, like recursive circuits. It computes the distinct elements of a [𝕫-Set] in its first argument, with respect to a [Trace] in its second, returning them in a new $\Bbb{Z}$-Set. The resulting $\Bbb{Z}$-Set has no timestamps associated with any element.

Note that because operator computes the delta of [Distinct], it is possible for returned elements to have negative weights, if those elements are deleted between timestamps.

Distinct is not a linear operation, and requires access to the entire history of updates, however there's a few observations that can reduce the search space of timestamps to examine.

Consider evaluating the operator at some [timestamp] $t = \langle e, i \rangle$, where $e$ denotes the epoch, and $i$ denotes the iteration.

Then there are two possible classes of elements which may be returned:
1) Elements in the current input $\Bbb{Z}$-Set
2) Elements that were returned in response to a previous input, at timestamp $\langle e, i_0 \rangle$, such that $i_0 < i$, and where the element was also returned at timestamp $\langle e_0, i \rangle$, such that $e_0 < e$.

For each element, with weight $w$, meeting at least one of the above requirements, if that element does not appear in the trace, it is returned in the $\Bbb{Z}$-Set with weight 1. Otherwise, the following routine is performed:

1) $w_1$ is computed, as the sum of all weights in which that element appears at times $\langle e_0, i_0 \rangle$ for $e_0 < e$ and $i_0 < i$
2) $w_2$ is computed, as the sum of all weights in which that element appears at times $\langle e_0, i \rangle$ for $e_0 < e$
3) $w_3$ is computed, as the sum of all weights in which that element appears at times $\langle e, i_0 \rangle$ for $i_0 < i$
4) $d_0$ is computed, such that:
   1) $d_0 = 1$, if $w_1 \le \land w_1 + w_2 > 0$
   2) $d_0 = -1$, if $w_1 > 0 \land w_1 + w_2 \le 0$
   3) $d_0 = 0$, otherwise
5) $d_1$ is computed, such that:
   1) $d_1 = 1$, if $w_1 + w_3 \le 0 \land w_1 + w_2 + w_3 + w > 0$
   2) $d_1 = -1$, if $w_1 + w_3 > 0 \land w_1 + w_3 + w_4 \le 0$
6) If $d_1 - d_0 \ne 0$, then the element is returned in the $\Bbb{Z}$-Set, with weight $d_1 - d_0$

For example:

```elixir
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

<details>
<summary>A more complete Elixir example</summary>

``` elixir
# Sum of weights where epoch(t) < epoch(time) and iteration(t) < iteration(time)
w1 = 0

# Sum of weights where epoch(t) < epoch(time) and iteration(t) == iteration(time)
w2 = 0

# Sum of weights where epoch(t) == epoch(time) and iteration(t) < iteration(time)
w3 = 0

{trace_cursor, times} = ZSetTraceCursor.times(trace_cursor)

{w1, w2, w3, next_time} =
  Enum.reduce(times, {w1, w2, w3, nil}, fn
    {{t_epoch, t_iteration} = t, w}, {w1, w2, w3, next_time} ->
      cond do
        t_epoch == epoch ->
          if Algebra.PartialOrder.less_than(t_iteration, iteration) do
            {w1, w2, w3 + w, next_time}
          else
            {w1, w2, w3, next_time}
          end

        Algebra.PartialOrder.less_than(t_iteration, iteration) ->
          {w1 + w, w2, w3, next_time}

        t_iteration == iteration ->
          {w1, w2 + w, w3, next_time}

        is_nil(next_time) or Algebra.PartialOrder.less_than(t, next_time) ->
          {w1, w2, w3, t}

        true ->
          {w1, w2, w3, next_time}
      end
  end)

w12 = w1 + w2
w13 = w1 + w3
w123 = w12 + w3
w1234 = w123 + weight

delta_old =
  cond do
    w1 <= 0 and w12 > 0 ->
      1

    w1 > 0 and w12 <= 0 ->
      -1

    true ->
      0
  end

delta_new =
  cond do
    w13 <= 0 and w1234 > 0 ->
      1

    w13 > 0 and w1234 <= 0 ->
      -1

    true ->
      0
  end

output =
  if delta_old == delta_new do
    output
  else
    w = delta_new - delta_old
    tuple = {{key, {}}, nil, w}

    [tuple | output]
  end
  ``````
</details>

## 2.9.12 Join Stream Operator

The join stream operator MUST merge two [Indexed $\Bbb{Z}$-Sets]. It MUST apply a join function to values with matching keys, returning a [𝕫-Set] containing the resulting elements. There MUST be no timestamps associated with the return elements, and each output element's weight MUST be given by the product of the two elements' weights that were joined.

For example:

```elixir
join_fun = fn key, v1, v2 -> {key, {v1, v2}} end

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

The join trace operator is a variant of an [Indexed Z-Set] join with a [Trace]. This takes advantage of the [bilinearity][bilinear] of relational joins in order to support incremental joins across timestamps.

This operator MUST return a $\Bbb{Z}$-Set containing the resulting elements. It MUST NOT include the timestamps associated with each of those elements. Each element's weight MUST be given by the product of the two elements' weights that were joined together.

Join trace behaves similarly to [Join Stream]. The first argument MUST represent deltas for the current timestamp. The second argument MUST be a trace containing all updates observed thus far. In this way, an incremental join MAY be implemented as follows:

```elixir
join_fun = ...

join_fun_flipped =
    fn k, v1, v2 ->
        join_fun(k, v2, v1)
    end

a = ... # some Z-Set
b = ... # some Z-Set

a_trace = z1_trace(a)
b_trace = z1_trace(b)

incremental_join(join_fun, a, b) =
    join_stream(join_fun, a, b)      +
    join_trace(join_fun, a, b_trace) +
    join_trace(join_fun_flipped, b, a_trace)
```

Where `z1_trace(x)` denotes an application of the [𝓏⁻¹ Trace] operator, `join_fun_flipped` flips the value arguments of `join_fun`, and `+` denotes the [Plus Operator].

For example:

```elixir
join_fun = fn key, v1, v2 -> {key, {v1, v2}} end

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

The minus operator MUST subtract all weights for matching elements in two $\Bbb{Z}$-Sets. Elements with combined weight equal to zero  MUST be discarded.

For example:

```elixir
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

Adds all weights for matching keys in two $\Bbb{Z}$-Sets. Elements with combined weight equal to zero are discarded.

For example:

```elixir
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

Inserts the input $\Bbb{Z}$-Set into the input Trace, with the current timestamp.

## 2.9.17 Untimed Trace Append Operator

Inserts the input $\Bbb{Z}$-Set into the input Trace 

## 2.9.18 Distinct Incremental Operator

A variant of [Distinct] that offers more performance for incremental computation and computes across multiple timestamps. It computes the distinct elements of a [𝕫-Set] in its first argument, with respect to a [Trace] in its second, returning them in a new $\Bbb{Z}$-Set. The resulting $\Bbb{Z}$-Set has no timestamps associated with any element.

Note that because operator computes the delta of [Distinct], it is possible for returned elements to have negative weights, if those elements are deleted between timestamps.

This computation can be performed by returning the elements in the $\Bbb{Z}$-Set whose weight has the opposite sign as the sum of all matching weights in the Trace.

For example:

```elixir
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
[DBSP]: https://arxiv.org/abs/2203.16684
[Differential Dataflow]: https://www.cidrdb.org/cidr2013/Papers/CIDR13_Paper111.pdf
[Distinct Incremental]: #2918-distinctincremental-operator
[Distinct Trace]: #2911-distincttrace-operator
[Distinct]: #293-distinct-operator
[Feedback]: #283-feedback-node
[Filter]: #294-filter-operator
[Fission Codes]: https://fission.codes
[Import]: #284-import-node
[Index With]: #295-indexwith-operator
[Indexed Z-Set]: #22-indexed-Bbbz-set
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
[PomoRA]: https://github.com/RhizomeDB/PomoRA
[Quinn Wilton]: https://github.com/QuinnWilton
[RFC 2119]: https://datatracker.ietf.org/doc/html/rfc2119
[Sink]: #285-sink-node
[Source]: #286-source-node
[Trace Append]: #2916-traceappend-operator
[Untimed Trace Append]: #2917-untimedtraceappend-operator
[bilinear]: https://en.wikipedia.org/wiki/Bilinear_form
[dataflow]: https://en.wikipedia.org/wiki/Dataflow
[nodes]: #28-node
[operation]: #29-operator
[partial order]: https://en.wikipedia.org/wiki/Partially_ordered_set#Partial_order
[product order]: #261-product-order
[semi-naive evaluation]: https://pages.cs.wisc.edu/~paris/cs784-s21/lectures/lecture9.pdf
[streams]: #27-stream
[timestamp]: #25-time
[trace]: #23-trace
[weight]: #24-weight
[𝓏⁻¹ Trace]: #299-z-1-trace-operator
[𝓏⁻¹]: #2910-z-1-operator
[𝕫-Set]: #21-bbbz-set

- Feature Name: N/A
- Start Date: 2017-03-22
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

A description of the new "on-demand" setup for rustc and how it would
integrate with incremental compilation.

# Motivation
[motivation]: #motivation

There are a number of goals:

- Ability to compile a subset of a crate for faster feedback (useful for e.g. the RLS).
- More comprehensive incremental compilation with better re-use.
- Potential for parallelization of tasks within compilation
  (eventually, that is not part of this design document).

# Detailed design
[design]: #detailed-design

## High-Level Overview

The goal of the compiler's new "on-demand" architecture is to refactor the bulk of the compiler's internals into a set of explicit, clearly delineated data transformations, each of which is equivalent to the application of a side effect free function to some input data, which in turn had been the result of a previous such function application. We call these function applications "queries", since they can be viewed as a query into a database of program information that the compiler has built up so far, or will build up "on-demand" as part of the query being made. Here is an example:

```

  HIR(foo) <--- Ty(foo) <--- TypeCheckTables(foo) <--- MIR(foo) <---+
     ^                         |     |                  |   ^       |
     |               +---------+     |                  |   |       |
     +-------------- | --------------+------------------+   +------ | --- Trans(CGU2) <---+
                     |                                              |                     |
                     v                                              |                     |
  HIR(bar) <--- Ty(bar) <--- TypeCheckTables(bar) <--- MIR(bar) <---+                   Trans(Crate)
     ^                         |     |                  |   ^       |                     |
     |               +---------+     |                  |   |       |                     |
     +-------------- | --------------+------------------+   +------ | --- Trans(CGU2) <---+
                     |                                      |       |                     |
                     v                                      v       |                     |
  HIR(baz) <--- Ty(baz) <--- TypeCheckTables(baz) <--- MIR(baz) <---+                     |
     ^                               |                  |           |                     |
     |                               |                  |           |                     |
     +-------------------------------+------------------+           |                     |
                                                                    |                     v
  ListOfHirItems <--------------------------------------- ListOfTransItems <---- ListOfCodegenUnits

```

Each of the nodes in the graph above is such a query. The nodes' label is the "query key" which globally identifies the query, i.e. making two queries with the same key will result in the same data being returned, just like when evaluating a pure function. The edges in the graph show which sub-queries each query has to make in order to compute its result. So, for example, computing the MIR for `foo` (represented by the `MIR(foo)` query) will need to access the HIR of foo (=`HIR(foo)`) and also the foo's type-check information (=`TypeCheckTables(foo)`).

Query functions have a few useful properties:

+ All query functions only take 2 parameters, the "query key" and the "context". The context only gives access to other query functions. Importantly, the context does not contain any writable state.

+ Since query functions only have access to their two (immutable) parameters, they cannot mutate anything, and are thus side-effect free.

+ Since the only way to access additional data is through the context object, we can easily and completely track all data a given query accesses. This is how the edges in the graph above are generated.

+ Because a query function has no access to mutable state, it is sound to cache its result.

In pseudo-Rust, a query function might thus be defined as follows:

```rust
fn mir_query(ctx, id) -> Mir {
    let hir = ctx.queries.hir(id);
    let ty_info = ctx.queries.type_check_tables(id);

    return construct_mir(hir, ty_info);
}
```

Caching and dependency tracking is handled transparently in the background.

### Why is this setup good for on-demand evaluation?

Because queries are like pure functions they are self-contained and their result is completely determined by their parameters. As a consequence, once the whole compiler is implemented in terms of query functions, one can just call, for example, `ctx.queries.mir(some_id)` and the query function will evaluate whatever sub-queries it needs to evaluate and not more. This has two advantages:

+ There is no need to bring the environment into a known state (like populating certain caches, etc), which is error prone and potentially inefficient (e.g. when one cannot easily just compute the needed value but has to run a whole pass that computes all values of the given kind).
+ This recursive, on-demand evaluation of queries allows for things that are not easily implemented in a compiler that strictly runs one pass after the other, like evaluating complex constants (?) during type-checking.

### Why is this setup good for incremental compilation?

There are several reasons why incremental compilation profits from the described architecture:

+ Since each query has a well defined boundary and all data accesses go through the context, we can automate data dependency tracking in a really safe way. Experience has shown that if data tracking is not enforced by an API it is really easily to accidentally create data leaks.
+ Because queries provide an opaque, uniform interface to data, we can bake cross-compilation-session data caching right into the query engine. Incremental and non-incremental compilation can run the same code paths with just a few switches flipped.
+ When data caching is implemented in a single place and in a uniform way, it is easy to experiment with different caching policies for different kinds of data and see which has the best performance characteristics.

## The Memoization Algorithm in Detail

Since queries behave just like pure functions, doing simple memoization on them is straight-forward: We can just have one map per query-kind and have this map cache the result for each query-key. However, for incremental compilation we want to do this memoization across multiple compiler runs and with (slightly) different inputs. This complicates things as we not only have to preserve cached result across process boundaries but also have to update the memoization cache in a correct and efficient manner.

First, let's take a look at how the situation changes when moving into an incremental compilation scenario. As described in the previous section, as queries are evaluated they build up a directed acyclic graph where each node is one query instance and the edges denote which other query values have been read by that instance. At the end of the session we have the complete graph that makes up the compilation of the given program. For incremental compilation, we are working under the assumption that only a small part of the input program changes in between compilation sessions and we can thus just re-use large parts of that query graph. More concretely, changing the input program means changing the values of some of the root nodes of the query graph, which in turn results in potential changes to any nodes that transitively depend on those root nodes. Looking at the example above, changing the definition of the function `bar` results in the HIR of `bar` having a different value, which in turn means that also `Ty(bar)`, `TypeCheckTables(bar)`, `MIR(bar)`, `Trans(CGU2)`, `ListOfTransItems`, `ListOfCodegenUnits`, and `Trans(Crate)` *might* have different values than in the previous query graph. Our goal is to find out which parts of the query graph from the previous session can be reloaded and which parts we have to reconstruct.

The algorithm that we came up with to fulfill this goal looks like the following:

- When a compilation session finishes we store the "dependency graph" of the queries executed into our on-disk incremental compilation cache. This dependency graph looks exactly like the query graph, except that for each node of the query graph we just store the query-key (=the node's unique identifier within the graph) and a fingerprint of the node's value. For example, the `Mir(bar)` node in the dependency graph would be `Mir(bar)=fingerprint(MIR of bar)`. This way the dependency graph allows us to know what depends on what (via the edges) and, when we have a freshly computed value for some node, we can also find out if it is the same as the value it had previously (via the fingerprint).
- Separately from the dependency graph we store the memoized values for each query node. These might be big and expensive to load, so we want to make sure we don't load them if we don't need to. This is purely a performance optimization. In theory we could store them in the dependency graph instead of the fingerprints.

With this data in place we can start a new compilation session that makes use of it:

- At the beginning of the session, load the dependency graph so we have it available.
- When a query with query-key `K` is performed, check if we already have a result cached in-memory. If so, we are done. Otherwise, we check in the dependency graph if `K`'s value memoized in the on-disk cache is still up-to-date and if so, load it and cache it in memory. If we can't use anything from any cache, we have to re-execute the query.

So far so good, but how do we know if something in the on-disk cache is still consistent with the new set of inputs? For that we use the dependency graph and a lazy change propagation algorithm which we call the red/green algorithm. It works as follows:

- When we load the dependency graph, we mark all its nodes with the color grey. Grey means that we don't know yet if the corresponding query value has changed compared to the previous compilation session.
- Once we have read the new version of the source code to be compiled, we know for each input node if it has the same value as before or if it has changed. In the example above, we would compare the fingerprint of each HIR node to the fingerprint it had in the previous version. If it has changed, we mark it as red, if not, we mark it as green.
- Now, when a query is made for some query-key `K`, and we don't have anything in the in-memory cache yet, we can look up the corresponding node in the dependency graph.
  - Since the node must still be grey, we need to find out it's current state: is the on-disk value still up-to-date, and if not what is the new value and is that different from the previous one. And here we exploit the fact that queries behave like pure functions: In order to find out if the on-disk value is still up-to-date, we take a look at all the input values to that query (which we know via the node's edges) and see if all of them are green. If they are, that means the current node cannot have changed its value because all the inputs are the same. We can load the corresponding value from the on-disk cache and mark the node as green as well. If one of the input nodes is red, we have to recompute its value. Once we have the new value, we can check if it has actually changed compared to the cached value. If it has changed, we mark the node as red, otherwise as green.

It might not be immediately obvious why we go to the trouble of comparing re-computed values to their previous version: We already have put in the effort to recompute them, why spend even more time comparing? The reason for this is that it prevents "false positives", i.e. values that *might* have changed but are actually unchanged, to cause transitive cache invalidation. Consider the following example: Let's say we have a function `foo` and its corresponding HIR representation `HIR(foo)`. We also have the type of the function `Ty(foo)` which describes its signature and is all that is needed in order to generate a call to that function. Now, if we change some expression within `foo` then `HIR(foo)` changes while `Ty(foo)` is not affected. Yet, since `Ty(foo)` is computed from `foo`'s HIR, there's an edge from `HIR(foo)` to `Ty(foo)`. Let's play this scenario through with and without the red/green algorithm:

```
                          +--- MIR(caller_1)
                          |
HIR(foo) <--- Ty(foo) <---+--- MIR(caller_2)
                          |
                         ...
                          |
                          +--- MIR(caller_N)
```

This is our dependency graph. Without the red/green algorithm, since `HIR(foo)` has changed, we have to assume that `Ty(foo)` has also changed and transitively also the MIR of each and any caller of `foo`. It's a real shame.

With the red/green algorithm things are different. Let's say we start by make a query for `MIR(caller_1)`. That node is still grey, we don't know yet if it is up-to-date, so we walk its inputs to see if all of them are green. When we encounter `Ty(foo)` we have to recompute it since `HIR(foo)` has changed, but in contrast to before we can switch it to green and consequently do not have to recompute `MIR(caller_1)` and later can also re-use the other MIR instances.


<!--  - what happens when a query is made -->
<!-- - red/green algorithm -->

  - pseudocode
  - anonymous nodes (why and how)
    - pseudocode
    - discussion of alternatives
  - handling diagnostics/"side effects"

## Cached Values and Persistence
  - caching policies
  - caching for anonymous nodes
  - "garbage collection"

# How We Teach This
[how-we-teach-this]: #how-we-teach-this

N/A -- this design document is how we teach it. =)

# Drawbacks
[drawbacks]: #drawbacks

N/A -- various tradeoffs are discussed in the detailed design.


# Alternatives
[alternatives]: #alternatives

Mention Adapton model and why we don't use it verbatim.


# Unresolved questions
[unresolved]: #unresolved-questions

Well, in rustc development, all things are uncertain. However, there are some
areas that we expect to tune over time:

- What is the best strategy for caching things like trait selection?
  This design document describes **anonymous nodes**, but there are a
  number of alternatives that were also described which trade off on
  precision and performance.



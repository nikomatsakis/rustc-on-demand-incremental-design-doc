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
  - what happens when a query is made
  - red/green algorithm
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

N/A -- various alternatives are discussed in the detailed design.

# Unresolved questions
[unresolved]: #unresolved-questions

Well, in rustc development, all things are uncertain. However, there are some
areas that we expect to tune over time:

- What is the best strategy for caching things like trait selection?
  This design document describes **anonymous nodes**, but there are a
  number of alternatives that were also described which trade off on
  precision and performance.



mw's preliminary notes for on-demand/incr. comp. document:
==============================================

Some thoughts on document contents:

+ Description of basic setup:
  - the compiler as a set of memoized, pure functions
  - no side effects (?)
  - good for incr. comp. because clear dependency structure
  - good for on-demand because evaluation is lazy
  - Roslyn works the same way, I think
  - example

+ Description of memoization algorithm
  - what happens when a query is made

+ Caching policy (defined per query):
  - Don't cache at all
  - Cache in memory
  - Persist just a fingerprint (for red/green checking)
  - Persist the complete value

Questions:

+ How do dynamic/anonymous nodes fit into the picture?
  - How does memoization work for those?
    (one node corresponds to multiple cached values, do we need to store this mapping)
  - What is to be gained by supporting them?
    We save dep nodes but do we need to store some kind of additional info for them?
  - My assumption:
      - Cache result of each single query in memory
      - DepNode for that query is shared
      - So there is a 1:N relationship between DepNode and cached value
      - We can not persist any of these values to disk
      - During evaluation of e.g. X -> Anon -> Y,
        before re-computing Y, we check that Anon is green, which consists of
        just checking whether X is green
      - If Anon comes back green, we might be able keep Y, if not, we re-eval
        Y, which probably will cause Y to depend on a different anon-node.
      - Is this correct?
      - Is there any flaw in this approach?
  - Resolution:
      - Exactly what mw described above
      - Integrate anonymous node handling into regular query infrastructure, so it looks the same from the outside
  - Example of two trait selection queries, how would that work?
      - `ty::queries::TraitSelect::get(tcx, TraitRef)` as the interface
          - this looks in some cache `Map<TraitRef, (Result, DepNode)>`
          - if it finds a hit, return the result and indicate a "read" of `DepNode`
      - if no hit:
          - start an "anonymous task", which is basically just a set<DepNode>
          - execute procedure
          - for each thing we read, collect in the task
          - end of task, we create name of anonymous node by hashing the dep-nodes (or whatever)
          - record in map (result, anon-node)
      - Advantages:
          - We could use a purely nominal system, but we expect there to be many trait-selection queries (lots of types)
              - many of which share the same dependencies (e.g., Rc<T>: clone for various values of T)
              - not obvious that we can structure things to avoid this (or that we would want to)
          - Using anonymous nodes for things we do not persist should result in smaller dep-graph overall
          - If this proves not to be true, since we have the same external interface, easy enough to change strategy
          - Other options:
              - everything has a name
                  - con: lots of nodes
              - "imprecise" names (e.g., `DepNode::TraitSelect(DefId, DefId)`)
                  - con: imprecise dependencies in the graph
                  - con: set of dependencies for a given row can grow over time, if multiple queries happen to share same line
              - if hashing is too expensive, threading may help:
                  - hash progressively (in bg thread)
                      - but we lose precision, order dependent
                  - unification approach
                      - if hashing is too expensive, give a fresh id at start
                      - in the "background", merge ids that wind up having the same set of dependencies
                      - the cache key can just keep the original id
                      - only makes sense if we can parallelize it

+ What about diagnostics?
  - Are they handled as values?
  - As sideeffects?
    -> might be possible if things containing messages are not persisted, not sure...
  - One possibility:
      - side-effects go through a central API (e.g., report-diagnostic, abort-compilation)
      - result of a node is always `SideEffects<T>` where `T` is the value
      - side-effects take effect "immediately" (or upon completion of the query, something like that)
      - when you replay a persisted query, also replay side-effects (e.g., re-issue messages)
          - i.e., when node is grey

+ What to do about the DepGraph?
  - Merge it completely into the query/map infrastructure?
  - Can we still manage it in a background thread with the red/green algorithm?
  - More or less answered in the first question:
      - `enum DepNode<'tcx> { Anonymous(hash), Named(Query<'tcx>) }` <-- much simpler than before
  - Background thread:
      - will need to access values more frequently (specifically, when transitioning a grey persisted value)
      - unclear if background thread still worth the effort

+ How to handle persistence?
  - Maybe every query/map should be able to decide where and how cached values
    are stored.
  - Can we asynchronously start writing things to disk?
    -> once a value is red or green, it won't change anymore
       => save to persist it, but might be wasted effort if compilation aborts
  - Have one file per query/map?
    -> would make it easier to persist and load stuff
  - Keep strategy of only persisting things if there is no error?
  - Thought:
      - background thread would be nice, but we have to deal with the lack of `Send` on certain types
      - e.g. interned strings
      - "just work"

+ Do we need some kind of garbage collection?
  - Probably just don't persist anything that is neither red nor green?
  - But wait, a red node might cause a downstream node to not depend on it
    anymore -> though that should be OK because of ordering of deps (except maybe for anonymous nodes)
    => would be a good example of a more detailed description of red/green algorithm
    
==============================================

Things to do:
    
    A. Convert writes and non-memoization patterns into queries https://github.com/rust-lang/rust/issues/40614
    B. Convert tasks we will want to persist into queries 
    C. Introduce anonymous nodes infrastructure
        - Use e.g. to replace trait selection
    D. Convert remaining tasks to use anonymous nodes (Depends on C)
    E. Make values serializable and hashable:
        - local node
        - ???
    F. Serialize values we want to keep (kind of depends on E, maybe some types are readily doable)
    G. Figure out diagnostics
    H. Red/green algorithm (kind of depends on D, F, and G)

Other stuff:
    
    A. LLVM / trans split https://github.com/rust-lang/rust/issues/39280
       - can we come up with a more precise plan?

==============================================


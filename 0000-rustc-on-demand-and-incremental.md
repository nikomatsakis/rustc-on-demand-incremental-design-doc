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



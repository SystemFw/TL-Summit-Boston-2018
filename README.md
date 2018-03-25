# Declarative Program flow with fs2 Stream

Slides for my talk at [Typelevel Summit](https://typelevel.org/event/2018-03-summit-boston/) 2018, in Boston.

## Description

fs2 is a purely functional streaming library, with support for concurrent and nondeterministic merging of arbitrary streams. Concurrency support means that we can use Stream not only to process data in constant memory, but also as a very general abstraction for program flow: whilst IO gives us an excellent model for a single effectful action, assembling behaviour with it often has a very imperative flavour (pure, but still imperative). This talk will introduce fs2 combinators by example, and will hopefully show how we can model program flow in a declarative, high level, composable fashion. In particular, we will focus on concurrent combinators.

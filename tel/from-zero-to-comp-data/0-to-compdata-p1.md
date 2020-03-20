# From Zero to Comp Data, Part 1: Introduction

One of the first joys I experienced learning Haskell was the wonderful,
terse way it allowed you to express sophisticated data types.

```haskell
-- A linked list
data List a = Nil | Cons a (List a)

-- A n-ary tree
data Tree a = Tree a [Tree a]
```

Once you learn to appreciate the (wildly) recursive nature of such
definitions they're liable to permanently replace your mental technology
for and conception of (immutable) data structures. As is the tradition
in publications on functional programming, I'd say recursive data types
like these are truly the "essence" of certain kinds of data and their
associated algorithms.

"What are recursive data types, *really*?" You might, if you're like me,
program happily for a long while without ever diving down that rabbit
hole; however, for the curious, this article series is designed to be a
guide to modern answers to that question.

The answer involves F-algebras, fixed points, bananas, lenses, babed
wire, functor composition, profunctors, category theory, applicatives,
monads, natural transformations, and, if we're lucky, it'll even finally
lead to a description of the elusive zygohistoprepromorphism. If any of
those words are gibberish, then hopefully by the end of this series they
will be more clear.

Simultaneously, this series will explore related problems in DSL design
as it is a popular problem area for working with complex recursive data
structures, abstract syntax trees specifically, by varying them,
tranforming them, or annotating them with other information.

* * *

As the title claims, the culmination of this series is the Haskell
[`CompData`](http://hackage.haskell.org/package/compdata) package by
Patrick Bahr and Tom Hvitved, based on Wouter Swierstra's well-known
paper *Data Types à la carte* (2008). Famous for having a longer module
list than many meaningful *programs*, `CompData` is a culminating
(ab)use of the nature of recursive data types providing great power.
While the Haddock documentation is terse, the authors have written a
number of beautiful papers (see Bahr and Hvitved 2012; Bahr and Hvitved
2011) explaining their work. This series will leave off having explored
some of the concepts of that paper, hopefully enough to successfully use
the library itself.

### References

Bahr, Patrick, and Tom Hvitved. 2011. “Compositional Data Types.”
*Proceedings of the seventh ACM SIGPLAN workshop on Generic
programming*: 83–94.

———. 2012. “Parametric Compositional Data Types.” *ArXiv* (feb).

Swierstra, W. 2008. “Functional Pearl: Data types a la carte.” *Journal
of functional programming* 18 (4): 423.

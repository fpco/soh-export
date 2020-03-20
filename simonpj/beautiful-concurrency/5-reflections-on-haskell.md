# 5. Reflections on Haskell

Haskell is, first and foremost, a _functional_ language. Nevertheless, I think that
it is also the world’s most beautiful _imperative_ language. Considered as an
imperative language, Haskell’s unusual features are that

- Actions (which have effects) are rigorously distinguished from pure values
by the type system.
- Actions are first-class values. They can be passed to functions, returned
as results, formed into lists, and so on, all without causing any side effects.

Using actions as first-class values, the programmer can define _application-specific
control structures_, rather than make do with the ones provided by the language
designer. For example, `nTimes` is a simple `for` loop, and `choose` implements a
sort of guarded command. We also saw other applications of actions as values.
In the main program we used Haskell’s rich expression language (in this case list
comprehensions) to generate a list of actions, which we then performed in order,
using `sequence_`. Earlier, when defining `helper1`, we improved modularity by
abstracting out an action from a chunk of code. To illustrate these points I
have perhaps over-used Haskell’s abstraction power in the Santa code, which is
a very small program. For large programs, though, it is hard to overstate the
importance of actions as values.

On the other hand, I have under-played other aspects of Haskell — higher
order functions, lazy evaluation, data types, polymorphism, type classes, and
so on — because of the focus on concurrency. Not many Haskell programs
are as imperative as this one! You can find a great deal of information about
Haskell at [http://haskell.org](http://haskell.org), including books, tutorials, Haskell compilers and interpreters, Haskell libraries, mailing lists and so on.  Plus, of course, all the tutorials here at the [School of Haskell](http://fpcomplete.com)!

» [Next: Conclusion](6-conclusion)
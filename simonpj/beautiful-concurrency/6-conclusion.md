# 6. Conclusion

My main goal is to persuade you that you can write programs in a fundamentally
more modular way using STM than you can with locks and condition variables.
First, though, note that transactional memory allows us to completely avoid
many of the standard problems that plague lock-based concurrent programs
(Section 2.2). _None of these problems arise in STM Haskell_. The type system
prevents you reading or writing a `TVar` outside an atomic block, and since there
_are_ no programmer-visible locks, the questions of which locks to take, and in
which order, simply do not arise. Other benefits of STM, which I lack the
space to describe here, include freedom from lost wake-ups and the treatment
of exceptions and error recovery.

However, as we also discussed in Section 2.2, the worst problem with lock-based
programming is that _locks do not compose_. In contrast, any function with an
`STM` type in Haskell can be composed, using sequencing or choice, with any other
function with an `STM` type to make a new function of `STM` type. Furthermore,
the compound function will guarantee all the same atomicity properties that the
individual functions did. In particular, blocking (`retry`) and choice (`orElse`),
which are fundamentally non-modular when expressed using locks, are fully
modular in STM. For example, consider this transaction, which uses functions
we defined in Section 3.4.

``` haskell
atomically (do limitedWithdraw  a1 10
               limitedWithdraw2 a2 a3 20)
```

This transaction blocks until `a1` contains at least 10 units, and either `a2` or
`a3` has 20 units. However, that complicated blocking condition is not written
explicitly by the programmer, and indeed if the `limitedWithdraw` functions are
implemented in a sophisticated library the programmer might have no idea what
their blocking conditions are. STM is modular: small programs can be glued
together to make larger programs _without exposing their implementations_.

There are many aspects of transactional memory that I have not covered in this
brief overview, including important topics such as nested transactions, exceptions, progress, starvation, and invariants. You can find many of them discussed
in papers about STM Haskell [4, 5, 3].

Transactional memory is a particularly good “fit” for Haskell. In STM, the implementation potentially must track every memory load and store, but a Haskell
STM need only track `TVar` operations, and these form only a tiny fraction of
all the memory loads and stores executed by a Haskell program. Furthermore,
the treatment of actions as first-class values, and the rich type system, allow
us to offer strong static guarantees without extending the language in any way.

However, there is nothing to stop the adoption of transactional memory in mainstream imperative languages, although it may be less elegant and require more
language support. Indeed doing so is a hot research topic: Larus and Rajwar
give a comprehensive summary [6].

Using STM is like using a high-level language instead of assembly code – you
can still write buggy programs, but many tricky bugs simply cannot occur, and
it is much easier to focus attention on the higher-level aspects of the program.
There is, alas, no silver bullet that will make concurrent programs easy to write.
But STM looks like a promising step forward, and one that will help you write
beautiful code.

## Acknowledgements

I would like to thank those who helped me to improve the chapter with their
feedback: Bo Adler, Justin Bailey, Matthew Brecknell, Paul Brown, Conal Elliot, Tony Finch, Kathleen Fisher, Greg Fitzgerald, Benjamin Franksen, Jeremy
Gibbons, Tim Harris, Robert Helgesson, Dean Herington, David House, Brian
Hulley, Dale Jordan, Marnix Klooster, Chris Kuklewicz, Evan Martin, Greg
Meredith, Neil Mitchell, Jun Mukai, Michal Palka, Zhang Ruochen, Sebastian Sylvan, Johan Tibell, Aruthur van Leeuwen, Wim Vanderbauwhede, David
Wakeling, DanWang, EricWilligers PeterWasilko, Gaal Yahas, and Brian Zimmer. My special thanks go to Kirsten Chevalier, Andy Oram, and Greg Wilson,
for their particularly detailed reviews.

## References

[1] Mordechai Ben-Ari. How to solve the Santa Claus problem. _Concurrency:
Practice and Experience_, 10(6):485–496, 1998.

[2] Nick Benton. Jingle bells: Solving the Santa Claus problem in Polyphonic
C#. Technical report, Microsoft Research, 2003.

[3] Anthony Discolo, Tim Harris, Simon Marlow, Simon Peyton Jones, and
Satnam Singh. Lock-free data structures using STMs in Haskell. In
_Eighth International Symposium on Functional and Logic Programming
(FLOPS’06)_, April 2006.

[4] Tim Harris, Simon Marlow, Simon Peyton Jones, and Maurice Herlihy.
Composable memory transactions. In _ACM Symposium on Principles and
Practice of Parallel Programming (PPoPP’05)_, June 2005.

[5] Tim Harris and Simon Peyton Jones. Transactional memory with data
invariants. In _First ACM SIGPLAN Workshop on Languages, Compilers,
and Hardware Support for Transactional Computing (TRANSACT’06)_, Ottowa, June 2006. ACM.

[6] James Larus and Ravi Rajwar. _Transactional memory_. Morgan & Claypool,
2006.

[7] Edward A. Lee. The problem with threads. _IEEE Computer_, 39(5):33–42,
May 2006.

[8] J. K. Ousterhout. Why threads are a bad idea (for most purposes), January 1996. Invited Talk, USENIX Technical Conference.

[9] Simon Peyton Jones. Tackling the awkward squad: monadic input/output,
concurrency, exceptions, and foreign-language calls in Haskell. In CAR
Hoare, M Broy, and R Steinbrueggen, editors, _Engineering Theories of
Software Construction, Marktoberdorf Summer School 2000_, NATO ASI
Series, pages 47–96. IOS Press, 2001.

[10] Simon Peyton Jones, Mark Jones, and Erik Meijer. Type classes: an exploration of the design space. In J Launchbury, editor, _Haskell workshop_,
Amsterdam, 1997.

[11] Herb Sutter. The free lunch is over: a fundamental turn toward concurrency
in software. _Dr. Dobb’s Journal_, March 2005.

[12] Herb Sutter and James Larus. Sofware and the concurrency revolution.
_ACM Queue_, 3, September 2005.

[13] SJ Thompson. _Haskell: the craft of functional programming_. Addison
Wesley, 1999.

[14] JA Trono. A new exercise in concurrency. _SIGCSE Bulletin_, 26:8–10, 1994.

[15] PL Wadler. The essence of functional programming. In _20th ACM Symposium on Principles of Programming Languages (POPL’92)_, pages 1–14.
ACM, Albuquerque, January 1992.

[16] PL Wadler and S Blott. How to make ad-hoc polymorphism less ad hoc.
In _Proc 16th ACM Symposium on Principles of Programming Languages,
Austin, Texas_. ACM, January 1989.
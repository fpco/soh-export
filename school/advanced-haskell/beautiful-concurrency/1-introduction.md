# 1. Introduction

The free lunch is over [11]. We have grown used to the idea that our programs
will go faster when we buy a next-generation processor, but that time has passed.
While that next-generation chip will have more CPUs, each individual CPU will
be no faster than the previous year’s model. If we want our programs to run
faster, we must learn to write parallel programs [12].

Parallel programs execute in a non-deterministic way, so they are hard to test
and bugs can be almost impossible to reproduce. For me, a beautiful program
is one that is so simple and elegant that it obviously has no mistakes, rather
than merely having no obvious mistakes<sup>1</sup>. If we want to write parallel programs
that work reliably, we must pay particular attention to beauty. Sadly, parallel
programs are often less beautiful than their sequential cousins; in particular
they are, as we shall see, less modular.

In this tutorial I’ll describe Software Transactional Memory (STM), a promising
new approach to programming shared-memory parallel processors, that seems
to support modular programs in a way that current technology does not. By
the time we are done, I hope you will be as enthusiastic as I am about STM. It
is not a solution to every problem, but it is a beautiful and inspiring attack on
the daunting ramparts of concurrency.

But only *one* of the ramparts!  It is worth distinguishing **parallelism** from **concurrency**:

* **Parallelism** employs multiple processors to make programs run faster.  Performance is the only goal. Ideally the semantics of the program is unchanged from the semantics of a sequential program: it always gives the same, deterministic results, and if it works on a uni-processor it'll work on a multi-processor.  Using `par` is a good example of parallelism in Haskell.  Parallelism is therefore to do with *implementation*.

* **Concurrency** describes a situation where non-determinism behaviour, and having many things going on at once, is part of the *specification* of the program.  In a web server you want a thread to deal with each client; in a telephone switch you have many concurrent phone calls going on at once.  This concurrency must happen *even on a uni-processor*.  Using `forkIO` to fork an explicit thread is a signal that a program uses concurreny.

This tutorial is exclusively about concurrency, and not at all about parallelism.  You can find more tutorials on the broader aspects of parallelism and concurrency in Haskell on the [Parallel Haskell page](http://haskell.org/haskellwiki/parallel). Don't miss the [Parallel Haskell reading](http://www.haskell.org/haskellwiki/Parallel/Reading) link at the bottom.

The tutorial is taken, with O'Reilly's kind agreement, directly from my chapter in "[Beautiful Code](http://shop.oreilly.com/product/9780596510046.do)", edited by Greg Wilson, O'Reilly 2007.  The difference is that in this version all the examples are executable, and you can modify them yourself.  Do read the book though!  It has 32 other chapters, written by amazing people like Jon Bentley, Jack Dongarra, Brian Kernighan, and Kent Dybvig.

» [Next: A simple example: bank accounts](2-a-simple-example-bank-accounts).

---
<sup>1</sup> This turn of phrase is due to Tony Hoare

---

[11] Herb Sutter. The free lunch is over: a fundamental turn toward concurrency
in software. _Dr. Dobb’s Journal_, March 2005.

[12] Herb Sutter and James Larus. Sofware and the concurrency revolution.
_ACM Queue_, 3, September 2005.

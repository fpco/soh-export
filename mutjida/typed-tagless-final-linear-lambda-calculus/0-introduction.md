# 0 Introduction

## Introduction

This tutorial is inspired by Oleg Kiselyov's typed tagless-final
[linear lambda calculus
interpreter](http://http://okmij.org/ftp/tagless-final/course/LinearLC.hs). In
particular, I will explain how to extend Oleg's interpreter, which
uses a first-order ([deBruijn
index](http://en.wikipedia.org/wiki/De_Bruijn_index)) representation of LLC,
to an interpreter which uses a higher-order ([higher-order abstract syntax](http://en.wikipedia.org/wiki/Higher-order_abstract_syntax)) representation of LLC. 
In order to accomplish this, I will make use of several advanced Haskell
features including _higher-rank polymorphism_, _type families_, and
_promoted data kinds_.  After showing the representation for a minimal
linear lambda calculus (LLC), just lambdas and applications, I will
extend the interpreter to a full LLC (including additive and unit
types); and finally, I will add recursive types to allow standard
programming examples to be written.

## Linear Lambda Calculus

Before diving in, I will attempt to give a brief summary of LLC; a
more thorough summary can be found in Frank Pfenning's [online
notes](http://www.cs.cmu.edu/~fp/courses/15816-f01/handouts/linlam.pdf).

A linear lambda is a function which must use its argument exactly
once. I use `-<>` as concrete syntax for a linear function type, `\x
-<> e` as concrete syntax for a linear lambda with argument `x` and
body `e`, and `f ^ x` as concrete syntax for linear application.
```haskell
-- show Grammar for LLC
m ::= x | \x -<> m | m ^ m
```
Thus the following is a valid linear lambda:

``` haskell
\f -<> \x -<> f ^ x
```
with type
``` haskell
(a -<> b) -<> a -<> b
```
while the following are not:
``` haskell
\f -<> \x -<> f ^ x ^ x  -- uses linear variable x twice

\f -<> \x -<> x  -- doesn't use linear variable f
```

### Some Type Theory

The linearity of linear lambdas is syntactically expressed in their
typing rules, which are built on the following judgement

```
D  |-  m :: A
```

denoting that term `m` has type `A` in linear context `D`, where
`D` is a set of typed variables-- so we assume all variables have
unique names, which can always be achieved via alpha conversion.

The typing rules for (minimal) LLC are as follows:
```
-------------- (lvar)     
x::A  |-  x :: A            


   D, x::A  |-  m :: B
-------------------------- (-<>I)
D  |-  \x -<> m :: A -<> B


D  |-  m :: A -<> B     D'  |-  n :: A
-------------------------------------- (-<>E)
      D union D'  |-  m ^ n :: B
```

Linearity is the consequence of the `lvar` rule requiring that only
one variable be in the linear context (thus no variables can be
ignored); and of the `-<>E` rule splitting the linear context beween
it's two subtrees (thus no variable can be used more than once). For the curious: the `I` and `E` stand for introduction and elimination since when read from premise to conclusion, `I` rules introduce (show how to construct) a type, while `E` rules eliminate (show how to use) a type.

Here is a complete typing derivation for `\f -<> \x -<> f ^ x`:

```
---------------------------- (lvar)      ---------------- (lvar)
f::A -<> B  |-  f :: A -<> B             x::A  |-  x :: A
--------------------------------------------------------- (-<>E)
f::A -<> B, x::A  |-  f ^ x :: B
--------------------------------------- (-<>I)
f::A -<> B  |-  \x -<> f ^ x :: A -<> B
-------------------------------------------------- (-<>I)
  |-  \f -<> \x -<> f ^ x :: (A -<> B) -<> A -<> B
```

Type checkers generally follow the structure of the term they are checking and thus, in the context of type checking, typing rules are usually read from conclusion to premise.
Therefore, the typing rules above are not suitable for implementing a type
checking (or inference) algorithm for LLC due to the need to split the
linear context in the `-<>E` rule-- it would not be feasible
to try every possible splitting until one succeeds. 

### Algorithmic type checking

LLC typing rules can be recast in a more
algorithmic fashion by lazily splitting the linear context,
i.e. passing everything to the first branch and then giving the
leftovers to the second branch; this technique comes from [linear logic programming](http://www.lfcs.inf.ed.ac.uk/reports/91/ECS-LFCS-91-158/index.html). To this end, we change our typing judgement to have the form:

```
Di \ Do  |-  m :: A
```
where `Di` is the input linear context and `Do` is the output linear
context and `Di - Do` is the actual linear context used in the
derivation; the derivations must maintain the constraint that `Di` is
a superset of `Do`. Here are the typing rules:

```
--------------------- (lvar)     
D, x::A \ D  |-  x :: A            


Di, x::A \ Do  |-  m :: B   (x not in Do)
----------------------------------------- (-<>I)
Di \ Do  |-  \x -<> m :: A -<> B


Di \ D  |-  m :: A -<> B     D \ Do  |-  n :: A
----------------------------------------------- (-<>E)
            Di \ Do  |-  m ^ n :: B
```
These rules can be easily implemented. 

## Typed Tagless-Final Interpreters

Writing a typed tagless-final interpreter for a language can be
thought of as an exercise in transcribing the typing rules of the
language into Haskell types. A much more comprehensive (and perhaps
comprehensible) presentation of the typed tagless-final approach can be
found in [Oleg's
notes](http://okmij.org/ftp/tagless-final/course/lecture.pdf).

The preceding algorithmic typing rules are so easy to implement that
they can be directly encoded as Haskell types. I will present two such
encodings. The first encoding will show how a presentation of the
preceding rules, modified to work with a deBruijn index based calculus
(rather than explicit named variables), can be directly, and simply,
encoded in Haskell types. The second encoding will keep the explicitly
named variables (and use HOAS) and directly model the rules above at
the cost of slightly more complicated Haskell types.



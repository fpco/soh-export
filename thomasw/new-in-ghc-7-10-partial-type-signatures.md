# New in GHC 7.10: Partial Type Signatures

In this tutorial we introduce you to the `PartialTypeSignatures` extension
available in GHC starting from version 7.10. The goal is to teach you what you
can do with this extension and how it can improve your development workflow.

# Introduction

Partial type signatures were introduced in GHC 7.10 to give the
programmer fine-grained control over type annotations. Prior to GHC
7.10, the programmer essentially faced a binary choice when it came to
writing a type signature: do I write a **complete** type signature, or
**none** at all?

Let's say you're writing a simple program tracking some preferences of
your co-workers. Currently, we're only interested in the editor they're
using. Let's define some data types to model this information.

``` haskell
data CoWorker = CoWorker String
              deriving (Eq, Ord)

data Editor = Emacs | Vim
            deriving (Eq)
```

Now let's define a function to remember a co-worker's editor preference.
We know we need at least two arguments, the co-worker and his/her editor
preference. However, we're not sure yet about the rest of the type,
maybe we'll go with a simple association list, i.e.
`[(CoWorker, Editor)]` or maybe with a map, i.e. `Map CoWorker Editor`, we don't know yet. With the limited information we currently have, our function will probably look like this:

``` haskell
rememberEditor coWorker editor = undefined
```

As we're not sure about what the function will actually do, we don't
know its return type yet, which prevents us from annotating its type.
Wouldn't it be helpful to already annotate the types of the arguments?

Imagine we're using this function later on. Our memory is a bit foggy
and we accidentally swap the argument order:

``` haskell
test = rememberEditor Emacs (CoWorker "John")
```

Unfortunately, GHC's type checker will happily accept this code! As
`rememberEditor` has no type signature yet, and its body is missing, GHC
has no clue about the types of the arguments.

Even though we knew the types of the arguments, we couldn't pass on this
information to GHC's type checker as we couldn't annotate this (pattern
signatures could help, but you have to sprinkle them throughout your
program).

This is exactly the problem *partial type signatures* solves, by giving
the programmer fine-grained control over type annotations. Let's add a
partial type signature to our example so GHC's type checker can help us
find our type errors again.

``` haskell
rememberEditor :: CoWorker -> Editor -> _
rememberEditor coWorker editor = undefined
```

A partial type signature looks just like a regular type signature except
for the fact that it can contain *wildcards* (written as underscores),
which take the place of the unknown parts of your type signature. The
wildcard in the (partial) type signature above thus indicates that the
return type (which can be another function) of `rememberEditor` is unknown.

Now GHC's type checker can use these type annotations to detect the
swapped argument order:

```
Couldn't match expected type `CoWorker' with actual type `Editor'
In the first argument of `rememberEditor', namely `Emacs'

Couldn't match expected type `Editor' with actual type `CoWorker'
In the second argument of `rememberEditor', namely `(CoWorker "John")'
```

After giving it some thought, we decide to go for a `Map` and implement `rememberEditor` as follows.

``` haskell
import           Data.Map (Map)
import qualified Data.Map as Map

rememberEditor :: CoWorker -> Editor -> _
rememberEditor coWorker editor = Map.insert coWorker editor
```

Now GHC's type checker will even inform us of the type to replace the wildcard
with!

```
Found hole `_'
  with type: Map CoWorker Editor -> Map CoWorker Editor
  ...
```

<br/>

> A partial type signature combines *type checking* (type
> annotations) with *type inference* (wildcards) in one type signature.

# Syntax & Options

There are a number of different kinds of wildcards you can use, as well
compiler flags to control the warnings and/or errors.

## Type Wildcards

The simplest wildcard is the *type wildcard*. For instance the wildcard
used in the example above was a type wildcard. A type wildcard is a
wildcard that occurs in the *type* part of a type signature, or put in
other words, a wildcard *not occurring in the constraints* of a type
signature.

A wildcard can be filled in with a closed type like `Int`, `[String]`,
`Editor`; a function like `Int -> Bool`; a type containing a type
variable like `a -> Bool`; a higher-kinded type like `Maybe`, `Either`,
`Either a`, `m`; pretty much every type.

When a wildcard is unconstrained, i.e. it is not filled in with any type
after type inference, it is *generalized* over: it is replaced with a
new type variable.

Take for example this standard `filter` function:

``` haskell
filter :: (a -> Bool) -> [a] -> [a]
filter p xs = [x | x <- xs, p x]
```

The following partial type signatures are all valid. The type checker will each
time infer exactly the same type for `filter`: `(a -> Bool) -> [a] -> [a]`.

``` haskell
(a -> _)    -> [a] -> [a]
(_ -> Bool) -> [a] -> [a]
({-hi-}_ _ _{-/hi-})     -> [a] -> [a]
_           -> [a] -> [a]
_           -> [a] -> [_]
_           -> [a] -> {-hi-}_{-/hi-} a
_           -> [a] -> _
_           -> _   -> _
_           -> {-hi-}_{-/hi-}
{-hi-}_{-/hi-}
```

Highlighted wildcards:

1.  `a -> Bool` can also be written in prefix-notation as `(->) a Bool`.
    The first wildcard will be filled in with `(->)`, the second with
    `a`, and the third with `Bool`.

2.  The wildcard will unify with `[]`, the type constructor for lists.

3.  As a wildcard can unify with a function type, a single wildcard can
    be filled in with a function with any arity.

4.  This partial type signature can be left out, as it is the same as
    writing no type signature at all and fully rely on type inference.

## Named Wildcards

Wildcards can also be named, to clarify their meaning, but more
important, to refer to them later on. All wildcards with the same name
within one type signature will be filled in with the same type. You can
name a wildcard by writing an identifier after the underscore of the
wildcard, e.g. `_foo`.

> Haskell 2010 and previous versions of GHC interpret named
> wildcards like `_foo` as type variables. To remain backwards compatible,
> named wildcards are hidden behind the `NamedWildCards` extension flag.
> Remember to add the following line to the top of your file when you want
> to `_foo` to be interpreted as a named wildcard instead of a type
> variable.

``` haskell
{-# LANGUAGE NamedWildCards #-}
```

Let's say we wrote the following partial type signature for `filter`:

``` haskell
-- Partial type signature with named wildcards
filter :: (_x   -> _x)   -> [_x]   -> [_x]
-- Inferred type
filter :: (Bool -> Bool) -> [Bool] -> [Bool]
-- Most general type of filter
filter :: (a    -> Bool) -> [a]    -> [a]
```

The second occurrence of `_x` is filled in with `Bool` after type
inference, all other instances of `_x` were not filled in with any type.
If they hadn't been named wildcards, they would have been replaced with
a type variable, e.g. `a`. But because they have the same name as the
wildcard filled in with `Bool`, they are all filled in with `Bool`.

When a named wildcard is not filled in with any type after type
inference, it is generalised over, just like an unnamed wildcard. There
is one small difference: the new type variable will have `w_NAME` as
name where `NAME` is the name of the named wildcard.

``` haskell
identity :: _foo -> _foo
identity x = x
-- Inferred type:
identity :: w_foo -> w_foo
```

Named wildcards must not be confused with type variables because they
can still be filled in with a closed type like `Int`, e.g.

``` haskell
not :: _foo -> _foo
not True = False
not False = True
-- Inferred type:
not :: Bool -> Bool
```

## Constraint Wildcards

As you might have guessed, wildcards occurring in the constraints part
are called *constraint wildcards*. Unfortunately, constraint wildcards
aren't all that useful. Take the following example.

``` haskell
showPlusOne :: _ a => a -> String
showPlusOne x = show (1 + x)
```

Because we use `show` and `(+)`, `x` must implement both the `Show` and
the `Num` type classes. Should the wildcard be filled in with `Show` or
`Num`? We certainly don't want the type checker to guess! Unlike types,
which are unified to fill in the wildcards, constraints are solved,
which isn't amenable to filling in wildcards. Therefore, wildcards in
constraints are not supported.

However, some wildcards do make sense in constraints, but only when they
comply with certain restrictions to keep the type checker from guessing.
The rules are as follows.

-   Only named wildcards are allowed in constraints ...
-   ... when they're also present in the rest of the type

Of the following type signatures only the last one satisfies both rules
and will be allowed.

``` haskell
Eq _  => a  -> a  -> Bool -- No
Eq _x => a  -> a  -> Bool -- No
Eq _x => _x -> _x -> Bool -- Yes
```

When these rules are followed, regular type inference will take care of
filling in the wildcards, and the type checker won't have to guess.

## Extra-constraints Wildcard

There is one more kind of wildcard, the extra-constraints wildcard, not
to be confused with constraint wildcards. Whereas constraint wildcards
occur *within* a constraint, an extra-constraints wildcard occurs *as*
a constraint. When a partial type signature contains an
extra-constraints wildcard, it means that the type checker may infer any
number (0..n) of extra constraints, which will then be integrated in the
final type of the binding. For example:

``` haskell
showPlusOne :: _ => a -> String -- Inferred for `_': (Num a, Show a)
showPlusOne x = show (1 + x)
```

The type checker will infer the extra constraints `Num a` and `Show a`,
just as when you would have left out the type signature. The two
constraints will be used to fill in the extra-constraints wildcard.

You can combine annotated constraints with an extra-constraints
wildcard. There are however some restrictions: there can't be more than
one extra-constraints wildcard in a type signature and it must come as
the last constraint.

Some more examples:

``` haskell
showNum :: (Num a, _) => a -> String
showNum x = show x
-- Found hole `_' with inferred constraints: (Show a)
-- ..
```

``` haskell
forM :: _ => _
forM x f = mapM f x -- mapM :: Monad m => (a -> m b) -> t a -> m (t b)
-- Found hole `_' with inferred constraints: (Monad m, Traversable t)
-- ..
--
-- Found hole `_' with type: t -> a -> (a -> m b) -> m (t b)
-- ..
```

``` haskell
false :: _ => Bool
false = False
-- Found hole `_' with inferred constraints: ()
-- ..
```

In the `filter` example, we said that the partial type signature
`filter :: _` is the same as writing no type signature at all. This
wasn't entirely true, because extra constraints are not allowed without
the extra-constraints wildcard. But now we have all the ingredients to
define the most general partial type signature: `_ => _`, which you can
leave out just as well, as it is equivalent to writing no type signature
at all!

## Flags

In past version of GHC, wildcards (underscores) in type signatures
produced parse errors. Starting from GHC 7.10 they no longer produce
parse errors but type errors.

Take the following simple program:

``` haskell
x :: _
x = True
```

Compiling this with GHC 7.8 produces:

```
Tutorial.hs:1:6: parse error on input `_'
```

Whereas compiling it with GHC 7.10 produces:

```
Tutorial.hs:1:6:
    Found hole `_' with type: Bool
    To use the inferred type, enable PartialTypeSignatures
    In the type signature for `x': _
```

The error message now informs you of the inferred type of the wildcard
or hole, just like `TypedHoles` tells you the type of a hole. However,
unlike `TypedHoles` which requires you to choose the right expression of
that type, knowing the type of a wildcard or hole is enough to fill in
the hole!

When you want to compile your program without manually filling in the holes,
just enable the `PartialTypeSignatures` extension flag, which will demote the
errors caused by holes in type signatures to warnings and use the inferred
types to fill in the holes. When you don't want these warnings either, pass
`-fno-warn-partial-type-signatures` to GHC and it will suppress the warnings
too.

Remember from the section on [Named Wildcards](#named-wildcards) that you need the `NamedWildCards`
extension flag to use named wildcards, otherwise they will be interpreted as
type variables.

# Use Cases

Partial type signatures can come in handy in the following use cases:

-   During development, like in the introductory example. By annotating the
    types you already know, even though you don't know the whole type
    yet, you give the type checker information to help catch your type
    errors. Furthermore, a (partial) type signature is a form of
    machine-checked documentation that you can quickly glance at to
    remind you of the type of the function, or at least the parts you
    already know.

-   Interactive hole-driven development in combination with
    `TypedHoles`. The type checker just tells you what types to fill in.

-   The type signature is too verbose and complicated. By replacing the
    verbose or complicated parts with underscores you can direct the
    user's focus to the parts of the type signature that really matter.
    Or when a bunch of related functions have similar but verbose types,
    the distracting common boilerplate can be hidden with underscores,
    thereby stressing the differences.

-   Not all programs can be written without type annotations as some
    types cannot be inferred, e.g. higher-rank types. With a partial
    type signature, you can annotate the parts required for type
    inference to succeed, but leave out the boilerplate. In the example
    below, the argument of `foo` cannot be inferred and needs a type
    annotation. However, the return type can easily be inferred, but why
    should we have to annotate it as well?

    ``` haskell
    foo :: (forall a. [a] -> [a]) -> {-hi-}([Bool], [Char]){-/hi-}
    foo x = (x [True, False], x ['a', 'b'])
    test = foo reverse  -- reverse :: forall a. [a] -> [a]
    ```

    With a partial type signature:

    ``` haskell
    foo :: (forall a. [a] -> [a]) -> {-hi-}_{-/hi-}
    foo x = (x [True, False], x ['a', 'b'])
    test = foo reverse  -- reverse :: forall a. [a] -> [a]
    ```


Further reading:

* [Paper](https://lirias.kuleuven.be/handle/123456789/423475)

* [GHC wiki page](https://ghc.haskell.org/trac/ghc/wiki/PartialTypeSignatures)

# List and Comprehension Extensions

These extensions enhance the abilities of Haskell’s list and comprehension syntaxes.

# `ParallelListComp`

**Available in:  All recent GHC versions**

The `ParallelListComp` extension allows you to `zip` multiple sub-comprehensions together.  For example:

``` haskell
[ w + x + y + z | ((w, x), (y, z)) <- zip [ (w, x) | w <- ws, x <- xs ]
                                          [ (y, z) | y <- ys, z <- zs ] ]
```

can be reduced to 

``` haskell
[ w + x + y + z | w <- ws, x <- xs | y <- ys, z <- zs ]
```

Try it out!

``` active haskell
{-# LANGUAGE ParallelListComp #-}

main = print [ (w, x, y, z) | w <- [1 .. 3],
                              x <- [2 .. 4]
                            | y <- [3 .. 5],
                              z <- [4 .. 6] ]
```

# `TransformListComp`

**Available in:  All recent GHC versions**

The `TransformListComp` extension allows you to use SQL-like statements in list comprehensions.

## `then` clauses

You can use `then` clauses to alter the list at that point in the comprehension.  For example:

``` haskell
[ (x, y) | x <- xs,
           y <- ys,
           then reverse ]
```

A `then` clause takes the form:

``` haskell
then f
```

where:

``` haskell
f :: forall a. [a] -> [a]
```

## `then by` clauses

You can use `then by` clauses to alter the list at that point in the comprehension in a way that uses some combination of the prior bindings in that comprehension.  For example:

``` haskell
[ (x, y) | x <- xs,
           y <- ys,
           then sortWith by (x + y) ]
```

A `then by` clause takes the form:

``` haskell
then f by e
```

where, for some type `t`:

``` haskell
f :: forall a. (a -> t) -> [a] -> [a]
e :: t
```

## `then group using` clauses

You can use `then group using` clauses to group up the list at that point in the comprehension.  For example:

``` haskell
[ (x, y) | x <- xs,
           y <- ys,
           then group using permutations ]
```

A `then group using` clause takes the form:

``` haskell
then group using f
```

where:

``` haskell
f :: forall a. [a] -> [[a]]
```

Be aware that after the point of the `then group using` clause, all bindings before the clause now have a different type.  Specifically, if a binding `x` had type `t` before the clause, then it now has type `[t]` after the clause.

## `then group by using` clauses

You can use `then group by using` clauses to group up the list at that point in the comprehension in a way that uses some combination of the prior bindings in that comprehension.  For example:

``` haskell
[ (x, y) | x <- xs,
           y <- ys,
           then group by (x + y) using groupWith ]
```

A `then group by using` clause takes the form:

``` haskell
then group by e using f
```

where, for some type `t`,

``` haskell
f :: forall a. (a -> t) -> [a] -> [[a]]
e :: t
```

A `then group by using` clause has the same effect on the types of existing bindings as a `then group using` clause does.

## The `the` function

The function `the` (found in the `GHC.Exts` module in the `base` package) is sometimes useful with `TransformListComp`.  It takes a list, checks it to make sure that all values in the list are equal to each other, then returns the single unique value that the list holds.  This is useful when grouping, where you might want to include a variable in the output but, since you’ve used a grouping clause, it’s now a list that contains just one unique value.

For example:

``` haskell
1 = the [1, 1, 1]
```

## Example

Try it out!

``` active haskell
{-# LANGUAGE TransformListComp #-}
import GHC.Exts (groupWith, the)
import Data.List (permutations)

main = print $ [ (x, y, map the v) | x <- [1 .. 10],
                                     y <- [1 .. 10],
                                     let v = x + y,
                                     then group by v using groupWith,
                                     then take 10,
                                     then group using permutations,
                                     t <- concat v,
                                     then takeWhile by t < 3]
```

# `MonadComprehensions`

**Available in:  GHC 7.2 or later**

The `MonadComprehensions` extension generalizes list comprehensions, including `ParallelListComp` and `TransformListComp`, to work for any `Monad` (or `MonadPlus`, or, in the case of `ParallelListComp`, `MonadZip`).  This removes all limitations from list comprehensions relative to `do` notation.  For example:

``` haskell
[ (x, y) | x <- lookup e1 l,
           y <- lookup e2 l,
           then (\f -> maybe Nothing
                             (\x -> if f x == 2
                                       then Just x
                                       else Nothing))
                by (x * y) ]
```

`MonadComprehensions` automatically implies both `ParallelListComp` and `TransformListComp`, so you don’t have to activate them seperately.

The generalization of `ParallelListComp` uses the `MonadZip` type class, which is defined in the `Control.Monad.Zip` module in the `base` package; the use of guards requires the `MonadPlus` type class, which is defined in the `Control.Monad` module in the `base` package.

Try it out!

``` active haskell
{-# LANGUAGE TransformListComp, MonadComprehensions #-}

l :: [(String, Int)]
l = [("a", 1), ("b", 2), ("c", 3)]

main = print $ [ (x, y) | x <- lookup "a" l,
                          y <- lookup "b" l,
                          then (\f ->
                                 maybe Nothing
                                       (\x -> if f x == 2
                                                 then Just x
                                                 else Nothing))
                               by (x * y) ]
```

# `OverloadedLists`

**Available in:  GHC 7.8 or later**

**TODO**
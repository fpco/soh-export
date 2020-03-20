# Pattern and Guard Extensions

These extensions enhance Haskell’s patterns and guards.

# `NPlusKPatterns`

**Available in:  GHC 6.12 and later**

The deprecated extension `NPlusKPatterns` was originally part of Haskell 98, but has since been removed in Haskell 2010.  It allows a very specific extension of pattern syntax, such that you can write, e.g.:

``` haskell
myFunc (x+3) = f x
```

and have it mean exactly:

``` haskell
myFunc x' | x' >= 3 = let x = x'-3 in f x
```

In place of `3`, you can have any fixed positive integer literal (the “k” in the term “n-plus-k pattern”).

Because of the extremely limited utility of this feature, and its nature as an explicit special case, n-plus-k patterns were eventually deemed unworthy of inclusion in the language and were dropped in the Haskell 2010 Report.

**WARNING:** *Do not use `NPlusKPatterns` in new code!*  I only include it here because it may be important when trying to understand legacy Haskell codebases.  If `NPlusKPatterns` seems like it would have been a perfect fit for your use case, try using [`ViewPatterns`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/pattern-and-guard-extensions#viewpatterns-as-a-replacement-for-npluskpatterns) instead.

# `BangPatterns`

**Available in:  All recent GHC versions**

The `BangPatterns` extension allows a small extension to pattern syntax.  You can now write patterns of the form `!p`, where p is any other pattern, and have it mean “match as though by `p`, but first evaluate the value being matched to weak head normal form.”  In essence, just as standard Haskell allows you to force a pattern match to be lazy (or “irrefutable”) by using `~`, the `BangPatterns` extension allows you to force a pattern match to be strict (or “immediate”) by using `!`.

Try it out!

``` active haskell
{-# LANGUAGE BangPatterns #-}

lazyFunc, strictFunc :: () -> ()

-- lazy, even though it would normally be strict
lazyFunc ~() = ()

-- strict, even though it would normally be lazy
strictFunc !v = ()

-- replace 'lazyFunc' with 'strictFunc' and see what happens
main = print $ lazyFunc undefined
```

# `ViewPatterns`

**Available in:  All recent GHC versions**

The `ViewPatterns` extension adds a new form of pattern that can be used anywhere any other pattern can, and that applies an arbitrary expression (of appropriate function type), called a *view*, to the argument before matching.  A view pattern `e -> p` applies the view `e` to the argument that would be matched by the whole view pattern, and then matches the pattern `p` against `e`’s result.  The whole view pattern matches against the original value exactly when `p` matches against the derived value.  A consequence of these semantics is that `e -> p` is irrefutable exactly when `p` is irrefutable, as long as `e` is sufficiently lazy.

Try it out!

``` active haskell
{-# LANGUAGE ViewPatterns #-}

eitherEndIsZero :: [Int] -> Bool
eitherEndIsZero (head -> 0) = True
eitherEndIsZero (last -> 0) = True
eitherEndIsZero          _  = False

main = print $ eitherEndIsZero [0,1,8,9]
```

## [`ViewPatterns`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/pattern-and-guard-extensions#viewpatterns) as a Replacement for [`NPlusKPatterns`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/pattern-and-guard-extensions#npluskpatterns)

You can use `ViewPatterns` to replace the deprecated `NPlusKPatterns` by simply using `subtract `*`k`* as the view.  Although it is somewhat longer to type, it affords greater future-proofing and less fragility than `NPlusKPatterns` does:

``` haskell
myFunc x'@(subtract 3 -> x) | x' >= 3 = f x
```

# `PatternGuards`

**Available in:  All recent GHC versions**

The `PatternGuards` extension, now officially incorporated into the Haskell 2010 language, expands guards to allow arbitrary pattern matching and condition chaining.  The existing syntax for guards then becomes a special case of the new, much more general form.

You start a guard in the same way as always, with a `|`.  You then continue with either a *boolean guardlet* or a *pattern guardlet*.  A *boolean guardlet* is any expression whose type is `Bool`, and they function as before.  A *pattern guardlet* is of the form `p <- e`, where `p` is an arbitrary pattern and `e` is an arbitrary expression, and which is fullfilled exactly when `e` matches against `p`.  You may then add additional boolean or pattern guardlets, seperated from each other by commas.  Finally, you finish off the complete guard with `=`, as usual.

`PatternGuards` syntax is deliberately designed to be reminicent of list comprehension syntax, but be aware that, in a pattern guardlet, *`p` matches against the same type as `e` has*, unlike in a list comprehension generator.  All the provided guardlets are tried in order, and only if all of them succeed is that guard’s clause chosen.  Otherwise, evaluation moves to the next clause down, as usual (or unceremoniously falls off the bottom if there is no next clause, also as usual).

Try it out!

``` active haskell
{-# LANGUAGE PatternGuards #-}
import Data.List (nub)

strangeOperation :: [Int] -> Ordering
strangeOperation xs | 7  <- sum xs
                    , n  <- length xs
                    , n  >= 5
                    , n  <= 20
                      = EQ
                    | 1  <- sum xs
                    , 18 <- length xs
                    , r  <- nub xs `compare` [1,2,3]
                    , r /= EQ
                      = r
                    | otherwise
                      = [3,1,2] `compare` xs

main = print $ strangeOperation ([5,7..21] ++ [20,19..4])
```

# `PatternSignatures`

**Available in:  All recent GHC versions**

The deprecated extension `PatternSignatures` allowed type signatures to be put on patterns; it has been wholly subsumed by the much cleaner and more powerful [`ScopedTypeVariables`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/explicit-forall#scopedtypevariables) extension, and should not be used in new code.

# `PatternSynonyms`

**Available in:  GHC 7.8 and later**

In standard Haskell, you can abstract over values and functions, as well as over types and type constructors, but you *can’t* abstract over patterns.  The `PatternSynonyms` extension fixes this, and gives you a very powerful mechanism for implementation hiding while still providing your module’s users with the streamlined interface that they would get from a fully open implementation.  In short, `PatternSynonyms` lets you present even the most complicated multiply-nested and hyperoptimized data representation with whatever simple constructors-and-pattern-matching-based interface you feel is appropriate.

In order to acheive this, `PatternSynonyms` enables a new form for top-level declarations in a Haskell module:  *pattern synonym declarations*, which always begin with the new keyword `pattern`.  Pattern synonym declarations produce new patterns that are valid anywhere you can do pattern matching.

There are three styles of pattern synonym declaration; which one you should use depends on exactly how you want the resulting pattern synonym to behave.  A *monodirectional* pattern synonym declaration produces a pattern synonym that is only valid in patterns.  For example:

``` haskell
-- This pattern synonym will match the first and third elements of a 3-tuple
-- and bind them to 'x' and 'y' respectively.
pattern OneAndThree x y <- (x, _, y)

-- Here is how the above pattern synonym might be used:
OneAndThree one three = (4, 5, 6)
-- Now 'one' is bound to '4' and 'three' is bound to '6'.
```

Although the `OneAndThree` pattern is [irrefutable](https://www.haskell.org/onlinereport/haskell2010/haskellch3.html#dx8-60022), pattern synonyms in general don’t have to be; for example:

``` haskell
-- This pattern will fail if the list it matches against
-- isn't exactly of the form it's expecting.
pattern VerySpecificList a b c <- [(0, a), (b, "hello"), (1, [_]), c]

-- Here is how the above pattern synonym might be used:
extractMyData :: [(Int, String)] -> Maybe (String, Int, Int, String)
extractMyData (VerySpecificList str0 n0 (n1, str1)) = Just (str0, n0, n1, str1)
extractMyData _ = Nothing
```

The syntax of a monodirectional pattern synonym declaration is pretty straightforward.  You start with the keyword `pattern`, then give the name of the pattern synonym you’re declaring.  Pattern synonyms share a namespace and naming rules with data constructors; that is, you cannot have a pattern synonym and a data constructor with the same name in the same module, and pattern synonyms must either begin with an uppercase letter and continue alphanumerically or begin with a colon and continue symbolically, just like data constructors do.

After the pattern synonym’s name, you give a series of variable names (which might be empty, as in `pattern SingleOne <- [1]`).  You cannot duplicate any of the names in this list; these are your *pattern variables*.

You follow the pattern variables with the symbol `<-`, and then finish the declaration with the *underlying pattern*.  This is what your pattern synonym is actually a synonym *for*; in other words, the underlying pattern is what your pattern synonym will “expand into,” just like type synonyms “expand into” their definitions.  In order to guarantee that this makes sense at all, though, there are some restrictions on the form of the underlying pattern:  you must use all of your pattern variables *exactly* once, and any additional variables present must be used *at most* once.  (That last part means we could rewrite our `OneAndThree` example as `pattern OneAndThree x y <- (x, z, y)` and still have it behave the same way.)  Otherwise, anything that’s a valid pattern elsewhere in the language is a valid underlying pattern for a pattern synonym; this includes pattern extensions like `ViewPatterns`, `BangPatterns`, and `NPlusKPatterns`, as well as previously defined pattern synonyms.

As previously mentioned, monodirectional pattern synonyms can *only* be used in pattern matching; if you try to use one as part of an expression, GHC will produce an error (at compile time).  This is in contrast to *bidirectional* pattern synonyms, which can be used in both contexts.  For example:

``` haskell
pattern NestedTriple x y z = (x, (y, z))

-- Using the above pattern synonym in a pattern context:
NestedTriple x y z = (1, (2, 3))

-- Using the same pattern synonym in an expression context:
t = [NestedTriple 'a' 'b' 'c', ('d', ('e', 'f'))]
```

Bidirectional pattern synonym declarations take advantage of the fact that Haskell’s pattern and expression syntaxes are (deliberately) very similar to each other.  Notionally, a bidirectional pattern synonym “expands into the same thing,” regardless of whether that “thing” is required to be a pattern or an expression; the resulting abstract syntax is then automatically interpreted appropriately for the context it’s found in.  This behaviour is a lot like that of data constructors, which can themselves be viewed as bidirectional pattern synonyms that the language just so happens to provide “primitively.”

The only difference in appearance between monodirectional and bidirectional pattern synonym declarations is that the latter uses the symbol `=` where the former uses `<-`; the only additional restrictions on bidirectional pattern synonyms compared to monodirectional pattern synonyms are that the underlying pattern must also be valid as an expression, and that you cannot use any “extra“ variables besides the pattern variables.

However, sometimes those additional restrictions are unacceptable; you might have an underlying pattern that is conceptually usable in a bidirectional fashion, but that you cannot express in a way that is syntactically uniform between pattern and expression contexts.  In such cases, as well as when you simply what more fine-grained control over the exact behavior of a bidirectional pattern synonym, you can use an *explicitly bidirectional* pattern synonym declaration.  For example:

``` haskell
pattern DoubleHead x <- (x:_):_ where
    DoubleHead x = [[x]]
```

This pattern will successfully match any non-empty list of lists whose first element is also non-empty, binding its first element; it’s also usable in an expression context, where it will produce a singleton list of a singeton list of its sole parameter.

To declare a bidirectional pattern synonym explicitly, you declare the behavior of the pattern synonym in a pattern context as though you were declaring a monodirectional pattern synonym (*i.e.*, using `<-` rather than `=`), but continue after the underlying pattern with the keyword `where` and an appropriately indented value or function declaration, except that the “value” or “function” that you’re declaring is the pattern synonym as it behaves in an expression context.

It should be noted that the sensibility of explicitly bidirectional pattern synonym declarations is very much in the hands of the programmer; all GHC cares about is that the syntax and types all check successfully.  If you want to produce a pattern synonym whose behaviour in an expression context doesn’t “match” its behaviour in a pattern context, you are free to do so!  Just make sure your users know what they’re getting into, or else you’ll severely break their expectations about how patterns are “supposed to work.”

## Pattern Synonym Type Signatures

In GHC 7.10 and later, pattern synonyms can be given type signatures.

**TODO**

## Pattern Synonyms in Export Lists

**TODO**

## Use Case: Simple API, Complex Implementation

**TODO**

## Use Case: Regex-By-Pattern

**TODO**
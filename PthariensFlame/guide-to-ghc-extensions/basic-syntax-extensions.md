# Basic Syntax Extensions

These extensions simply enhance Haskell’s core syntax without providing any actually new semantic features.

# `PostfixOperators`

**Available in:  All recent GHC versions**

The `PostfixOperators` extension allows you some slight extra leeway with Haskell’s operator section syntax.  Normally, when you write, for example:

``` haskell
(4 !)
```

it expands into:

``` haskell
\x -> 4 ! x
```

or, equivalently:

``` haskell
\x -> (!) 4 x
```

`PostfixOperators` instead expands this left section into:

``` haskell
(!) 4
```

which may look the same to you initially, and it behaves the same way where they both compile, but the new form allows GHC to be somewhat more lenient about the type of `(!)`.

For example, `(!)` can now be the factorial function and have the type:

``` haskell
(!) :: Integer -> Integer
```

Unfortunately, `PostfixOperators` does not allow you to *define* operators in postfix fashion, it just allows you to *use* them that way.

Try it out!

``` active haskell
{-# LANGUAGE PostfixOperators #-}

(!) :: Integer -> Integer
(!) n | n == 0    = 1
      | n >  0    = n * ((n - 1) !)
      | otherwise = error "factorial of a negative number"

main = print (4 !)
```

# `TupleSections`

**Available in:  GHC 6.12 and later**

The `TupleSections` extension allows you to omit values from the tuple syntax, unifying the standard tuple sugar with the tuple constructor syntax to form one generalized syntax for tuples.

Normally, tuples are constructed with the standard tuple sugar, which looks like this:

``` haskell
(1, "hello", 6.5, Just (), [5, 5, 6, 7])
```

This could be considered shorthand for the following explicit tuple constructor use:

``` haskell
(,,,,) 1 "hello" 6.5 (Just ()) [5, 5, 6, 7]
```

However, the explicit tuple constructor `(,,,,)` could just as easily be considered section sugar for tuples, expanding to:

``` haskell
\v w x y z -> (v, w, x, y, z)
```

Looking at it this way allows us to ask, “Why can’t we partially section a tuple?  After all, `(+)` is valid, `(,)` is valid, and `(1 +)` is valid, but `(1,)` is not valid.  The `TupleSections` extension fixes this oversight.

With `TupleSections` you can now write, for example:

``` haskell
(1, "hello",, Just (),)
```

and have it mean the same as

``` haskell
\x y -> (1, "hello", x, Just (), y)
```

Try it out!

``` active haskell
{-# LANGUAGE TupleSections #-}

main = print $ map (1, "hello", 6.5,, [5, 5, 6, 7]) [Just (), Nothing]
```

# `PackageImports`

**Available in:  All recent GHC versions**

Let’s say you want to import module `Data.Module.X` from package `package-one`, but `package-two` is also installed and also contains a module named `Data.Module.X`.  You could try to mess with package hiding, either manually or through cabal, but sometimes you might want some other module from `package-two`, so hiding it is not an option.

Enter the `PackageImports` extension.  Rather than writing:

``` haskell
import Data.Module.X
```

and hoping that GHC gets the one from the right package, `PackageImports` lets you write:

``` haskell
import "package-one" Data.Module.X
```

and explicitly specify the package you want to import that module from.  You can even import from a specific package version:

``` haskell
import "package-one-0.1.0.1" Data.Module.X
```

You can use `PackageImports` in combination with any other variant of the `import` syntax, and you can use both package-qualified imports and regular imports in the same file.

Try it out!

``` active haskell
{-# LANGUAGE PackageImports #-}
import Data.Monoid (Sum(..))
import "base" Data.Foldable (foldMap)
import qualified "containers" Data.Map as Map

main = print . getSum . foldMap Sum $ Map.fromList [(1, 2), (3, 4)]
```

# `OverloadedStrings`

**Available in:  All recent GHC versions**

By default, Haskell’s numeric literals are polymorphic over `Num` (in the case of integer literals) or `Fractional` (in the case of decimal literals).  That is, you can write:

``` haskell
a :: Int
a = 1

b :: Double
b = 1

c :: Float
c = 3.5

d :: Rational
d = 3.5
```

and it just works as expected.

String literals, on the other hand, are always of type `String`, and are not polymorphic at all.  The `OverloadedStrings` extension corrects this, making string literals polymorphic over the `IsString` type class, which is found in the `Data.String` module in the `base` package.  That is, you can write:

``` haskell
a :: String
a = "hello"

b :: Text
b = "hello"
```

`OverloadedStrings` also adds `IsString` to the list of defaultable type classes, so you can use types like `String`, `Text`, and `Bytestring` in a `default` declaration.

Try it out!

``` active haskell
{-# LANGUAGE OverloadedStrings #-}
import qualified Data.Text.IO as T

main = do putStrLn   "Hello as String!"
          T.putStrLn "Hello as Text!"
```

# `UnicodeSyntax`

**Available in:  All recent GHC versions**

With the `UnicodeSyntax` extension (along with the [`base-unicode-symbols`](https://hackage.haskell.org/package/base-unicode-symbols) and [`containers-unicode-symbols`](https://hackage.haskell.org/package/containers-unicode-symbols) packages), you can use Unicode alternatives to many of the standard operators.  The `UnicodeSyntax` extension itself handles just the operators and symbols that are built into the Haskell language, whereas the `base-unicode-symbols` package handles the operators and functions provided by the [`base`](https://hackage.haskell.org/package/base) package and the `containers-unicode-symbols` package handles the operators and functions provided by the [`containers`](https://hackage.haskell.org/package/containers) package.

For the package-based Unicode symbols, you need to import the appropriate syntax module.  For example, if you wanted to use Unicode symbols when working with `Data.Map`, you would import `Data.Map.Unicode`.

The various aliased ASCII syntax pieces, values, and types, along with their `UnicodeSyntax` equivalents, are as follows:

- From the `UnicodeSyntax` extension
    - `::` = `∷`
    - `=>` = `⇒`
    - `forall` = `∀`
    - `->` = `→`
    - `<-` = `←`
    - `-<` = `⤙`
    - `>-` = `⤚`
    - `-<<` = `⤛`
    - `>>-` = `⤜`
    - `*` = `★`
- From the [`base-unicode-symbols`](https://hackage.haskell.org/package/base-unicode-symbols) package
    - From the [`Prelude.Unicode`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html) module
        - Values
            - [`not`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:not) = [`(¬)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-172-)
            - [`(&&)`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:-38--38-) = [`(∧)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-8744-)
            - [`(||)`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:-124--124-) = [`(∨)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-8743-)
            - [`(==)`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:-47--61-) = [`(≡)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-8801-)
            - [`(/=)`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:-47--61-) = [`(≠)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-8800-) = [`(≢)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-8802-)
            - [`<=`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:-60--61-) = [`≤`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-8804-) = [`≯`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-8815-)
            - [`>=`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:-62--61-) = [`≥`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-8805-) = [`≮`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-8814-)
            - [`pi`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:pi) = [`π`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-960-)
            - [`(/)`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:-47-) = [`(÷)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-247-)
            - [`(*)`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:-42-) = [`(⋅)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-8901-)
            - [`(.)`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:.) = [`(∘)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-8728-)
            - [`(++)`](https://hackage.haskell.org/package/base/docs/Data-List.html#v:-43--43-) = [`(⧺)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-10746-)
            - [`elem`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:elem) = [`(∈)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-8712-)
            - [`notElem`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:notElem) = [`(∉)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-8713-)
            - [`undefined`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:undefined) = [`(⊥)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#v:-8869-)
        - Types
            - [`Integer`](https://hackage.haskell.org/package/base/docs/Prelude.html#t:Integer) = [`ℤ`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#t:-8484-)
            - [`Rational`](https://hackage.haskell.org/package/base/docs/Prelude.html#t:Rational) = [`ℚ`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Prelude-Unicode.html#t:-8474-)
    - From the [`Control.Applicative.Unicode`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Control-Applicative-Unicode.html) module
        - [`(<*>)`](https://hackage.haskell.org/package/base/docs/Control-Applicative.html#v:-60--42--62-) = [`(⊛)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Control-Applicative-Unicode.html#v:-8859-)
        - [`empty`](https://hackage.haskell.org/package/base/docs/Control-Applicative.html#v:empty) = [`(∅)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Control-Applicative-Unicode.html#v:-8709-)
    - From the [`Control.Arrow.Unicode`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Control-Arrow-Unicode.html) module
        - [`(>>>)`](https://hackage.haskell.org/package/base/docs/Control-Arrow.html#v:-62--62--62-) = [`(⋙)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Control-Arrow-Unicode.html#v:-8921-)
        - [`(<<<)`](https://hackage.haskell.org/package/base/docs/Control-Arrow.html#v:-60--60--60-) = [`(⋘)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Control-Arrow-Unicode.html#v:-8920-)
        - [`(***)`](https://hackage.haskell.org/package/base/docs/Control-Arrow.html#v:-42--42--42-) = [`(⁂)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Control-Arrow-Unicode.html#v:-8258-)
        - [`(+++)`](https://hackage.haskell.org/package/base/docs/Control-Arrow.html#v:-43--43--43-) = [`(⧻)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Control-Arrow-Unicode.html#v:-10747-)
        - [`(|||)`](https://hackage.haskell.org/package/base/docs/Control-Arrow.html#v:-124--124--124-) = [`(⫴)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Control-Arrow-Unicode.html#v:-10996-)
    - From the [`Control.Category.Unicode`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Control-Category-Unicode.html) module
        - [`(.)`](https://hackage.haskell.org/package/base/docs/Control-Category.html#v:.) = [`(∘)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Control-Category-Unicode.html#v:-8728-)
        - [`(>>>)`](https://hackage.haskell.org/package/base/docs/Control-Category.html#v:-62--62--62-) = [`(⋙)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Control-Category-Unicode.html#v:-8921-)
        - [`(<<<)`](https://hackage.haskell.org/package/base/docs/Control-Category.html#v:-60--60--60-) = [`(⋘)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Control-Category-Unicode.html#v:-8920-)
    - From the [`Control.Monad.Unicode`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Control-Monad-Unicode.html) module
        - [`(>>=)`](https://hackage.haskell.org/package/base/docs/Control-Monad.html#v:-62--62--61-) = [`(≫=)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Control-Monad-Unicode.html#v:-8811--61-)
        - [`(>>)`](https://hackage.haskell.org/package/base/docs/Control-Monad.html#v:-62--62-) = [`(≫)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Control-Monad-Unicode.html#v:-8811-)
        - [`(=<<)`](https://hackage.haskell.org/package/base/docs/Control-Monad.html#v:-61--60--60-) = [`(=≪)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Control-Monad-Unicode.html#v:-61--8810-)
    - From the [`Data.Bool.Unicode`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Bool-Unicode.html) module
        - [`(&&)`](https://hackage.haskell.org/package/base/docs/Data-Bool.html#v:-38--38-) = [`(∧)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Bool-Unicode.html#v:-8743-)
        - [`(||)`](https://hackage.haskell.org/package/base/docs/Data-Bool.html#v:-124--124-) = [`(∨)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Bool-Unicode.html#v:-8744-)
        - [`not`](https://hackage.haskell.org/package/base/docs/Data-Bool.html#v:not) = [`(¬)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Bool-Unicode.html#v:-172-)
    - From the [`Data.Eq.Unicode`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Eq-Unicode.html) module
        - [`(==)`](https://hackage.haskell.org/package/base/docs/Data-Eq.html#v:-61--61-) = [`(≡)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Eq-Unicode.html#v:-8801-)
        - [`(/=)`](https://hackage.haskell.org/package/base/docs/Data-Eq.html#v:-47--61-) = [`(≠)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Eq-Unicode.html#v:-8800-) = [`(≢)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Eq-Unicode.html#v:-8802-)
    - From the [`Data.Foldable.Unicode`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Foldable-Unicode.html) module
        - [`elem`](https://hackage.haskell.org/package/base/docs/Data-Foldable.html#v:elem) = [`(∈)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Foldable-Unicode.html#v:-8712-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`elem`](https://hackage.haskell.org/package/base/docs/Data-Foldable.html#v:elem) = [`(∋)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Foldable-Unicode.html#v:-8715-)
        - [`notElem`](https://hackage.haskell.org/package/base/docs/Data-Foldable.html#v:notElem) = [`(∉)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Foldable-Unicode.html#v:-8713-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`notElem`](https://hackage.haskell.org/package/base/docs/Data-Foldable.html#v:notElem) = [`(∌)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Foldable-Unicode.html#v:-8716-)
    - From the [`Data.Function.Unicode`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Function-Unicode.html) module
        - [`(.)`](https://hackage.haskell.org/package/base/docs/Data-Function.html#v:.) = [`(∘)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Function-Unicode.html#v:-8728-)
    - From the [`Data.List.Unicode`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-List-Unicode.html) module
        - [`(++)`](https://hackage.haskell.org/package/base/docs/Data-List.html#v:-43--43-) = [`(⧺)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-List-Unicode.html#v:-10746-)
        - [`elem`](https://hackage.haskell.org/package/base/docs/Data-List.html#v:elem) = [`(∈)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-List-Unicode.html#v:-8712-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`elem`](https://hackage.haskell.org/package/base/docs/Data-List.html#v:elem) = [`(∋)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-List-Unicode.html#v:-8715-)
        - [`notElem`](https://hackage.haskell.org/package/base/docs/Data-List.html#v:notElem) = [`(∉)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-List-Unicode.html#v:-8713-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`notElem`](https://hackage.haskell.org/package/base/docs/Data-List.html#v:notElem) = [`(∌)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-List-Unicode.html#v:-8716-)
        - [`union`](https://hackage.haskell.org/package/base/docs/Data-List.html#v:union) = [`(∪)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-List-Unicode.html#v:-8746-)
        - [`(\\)`](https://hackage.haskell.org/package/base/docs/Data-List.html#v:-92--92-) = [`(∖)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-List-Unicode.html#v:-8726-)
        - `\x y -> `[`union`](https://hackage.haskell.org/package/base/docs/Data-List.html#v:union)` (x `[`\\`](https://hackage.haskell.org/package/base/docs/Data-List.html#v:-92--92-)` y) (y `[`\\`](https://hackage.haskell.org/package/base/docs/Data-List.html#v:-92--92-)` x)` = [`(∆)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-List-Unicode.html#v:-8710-)
        - [`intersect`](https://hackage.haskell.org/package/base/docs/Data-List.html#v:intersect) = [`(∩)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-List-Unicode.html#v:-8745-)
    - From the [`Data.Monoid.Unicode`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Monoid-Unicode.html) module
        - [`mempty`](https://hackage.haskell.org/package/base/docs/Data-Monoid.html#v:mempty) = [`(∅)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Monoid-Unicode.html#v:-8709-)
        - [`mappend`](https://hackage.haskell.org/package/base/docs/Data-Monoid.html#v:mappend) = [`(⊕)`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Monoid-Unicode.html#v:-8853-)
    - From the [`Data.Ord.Unicode`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Ord-Unicode.html) module
        - [`<=`](https://hackage.haskell.org/package/base/docs/Data-Ord.html#v:-60--61-) = [`≤`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Ord-Unicode.html#v:-8804-) = [`≯`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Ord-Unicode.html#v:-8815-)
        - [`>=`](https://hackage.haskell.org/package/base/docs/Data-Ord.html#v:-62--61-) = [`≥`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Ord-Unicode.html#v:-8805-) = [`≮`](https://hackage.haskell.org/package/base-unicode-symbols/docs/Data-Ord-Unicode.html#v:-8814-)
- From the [`containers-unicode-symbols`](https://hackage.haskell.org/package/containers-unicode-symbols) package
    - From the [`Data.Sequence.Unicode`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Sequence-Unicode.html) module
        - [`empty`](https://hackage.haskell.org/package/containers/docs/Data-Sequence.html#v:empty) = [`(∅)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Sequence-Unicode.html#v:-8709-)
        - [`(<|)`](https://hackage.haskell.org/package/containers/docs/Data-Sequence.html#v:-60--124-) = [`(⊲)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Sequence-Unicode.html#v:-8882-)
        - [`(|>)`](https://hackage.haskell.org/package/containers/docs/Data-Sequence.html#v:-124--62-) = [`(⊳)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Sequence-Unicode.html#v:-8883-)
        - [`(><)`](https://hackage.haskell.org/package/containers/docs/Data-Sequence.html#v:-62--60-) = [`(⋈)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Sequence-Unicode.html#v:-8904-)
    - From the [`Data.Set.Unicode`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Set-Unicode.html) module
        - [`member`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:member) = [`(∈)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Set-Unicode.html#v:-8712-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`member`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:member) = [`(∋)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Set-Unicode.html#v:-8715-)
        - [`notMember`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:notMember) = [`(∉)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Set-Unicode.html#v:-8713-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`notMember`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:notMember) = [`(∌)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Set-Unicode.html#v:-8716-)
        - [`empty`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:empty) = [`(∅)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Set-Unicode.html#v:-8709-)
        - [`union`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:union) = [`(∪)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Set-Unicode.html#v:-8746-)
        - [`difference`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:difference) = [`(∖)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Set-Unicode.html#v:-8726-)
        - `\x y -> `[`union`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:union)` (`[`difference`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:difference)` x y) (`[`difference`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:difference)` y x)` = [`(∆)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Set-Unicode.html#v:-8710-)
        - [`intersection`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:intersection) = [`(∩)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Set-Unicode.html#v:-8745-)
        - [`isSubsetOf`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:isSubsetOf) = [`(⊆)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Set-Unicode.html#v:-8838-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`isSubsetOf`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:isSubsetOf) = [`(⊇)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Set-Unicode.html#v:-8839-)
        - `\x y -> (x `[`/=`](http://hackage.haskell.org/package/base/docs/Prelude.html#v:-47--61-)` y) `[`&&`](http://hackage.haskell.org/package/base/docs/Prelude.html#v:-38--38-)` `[`not`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:not)` (`[`isProperSubsetOf`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:isProperSubsetOf)` x y)` = [`(⊈)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Set-Unicode.html#v:-8840-)
        - `\x y -> (x `[`/=`](http://hackage.haskell.org/package/base/docs/Prelude.html#v:-47--61-)` y) `[`&&`](http://hackage.haskell.org/package/base/docs/Prelude.html#v:-38--38-)` `[`not`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:not)` (`[`isProperSubsetOf`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:isProperSubsetOf)` y x)` = [`(⊉)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Set-Unicode.html#v:-8841-)
        - [`isProperSubsetOf`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:isProperSubsetOf) = [`(⊂)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Set-Unicode.html#v:-8834-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`isProperSubsetOf`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:isProperSubsetOf) = [`(⊃)`](https://hackage.haskell.org/package/containers-unicode-symbol/docs/Data-Set-Unicode.html#v:-8835-)
        - `\x y -> `[`not`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:not)` (`[`isProperSubsetOf`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:isProperSubsetOf)` x y)` = [`(⊄)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Set-Unicode.html#v:-8836-)
        - `\x y -> `[`not`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:not)` (`[`isProperSubsetOf`](https://hackage.haskell.org/package/containers/docs/Data-Set.html#v:isProperSubsetOf)` y x)` = [`(⊅)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Set-Unicode.html#v:-8837-)
    - From the [`Data.Map.Lazy.Unicode`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Lazy-Unicode.html) module (reexported by the [`Data.Map.Unicode`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Unicode.html) module)
        - [`member`](https://hackage.haskell.org/package/containers/docs/Data-Map-Lazy.html#v:member) = [`(∈)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Lazy-Unicode.html#v:-8712-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`member`](https://hackage.haskell.org/package/containers/docs/Data-Map-Lazy.html#v:member) = [`(∋)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Lazy-Unicode.html#v:-8715-)
        - [`notMember`](https://hackage.haskell.org/package/containers/docs/Data-Map-Lazy.html#v:notMember) = [`(∉)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Lazy-Unicode.html#v:-8713-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`notMember`](https://hackage.haskell.org/package/containers/docs/Data-Map-Lazy.html#v:notMember) = [`(∌)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Lazy-Unicode.html#v:-8716-)
        - [`empty`](https://hackage.haskell.org/package/containers/docs/Data-Map-Lazy.html#v:empty) = [`(∅)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Lazy-Unicode.html#v:-8709-)
        - [`union`](https://hackage.haskell.org/package/containers/docs/Data-Map-Lazy.html#v:union) = [`(∪)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Lazy-Unicode.html#v:-8746-)
        - [`difference`](https://hackage.haskell.org/package/containers/docs/Data-Map-Lazy.html#v:difference) = [`(∖)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Lazy-Unicode.html#v:-8726-)
        - `\x y -> `[`union`](https://hackage.haskell.org/package/containers/docs/Data-Map-Lazy.html#v:union)` (`[`difference`](https://hackage.haskell.org/package/containers/docs/Data-Map-Lazy.html#v:difference)` x y) (`[`difference`](https://hackage.haskell.org/package/containers/docs/Data-Map-Lazy.html#v:difference)` y x)` = [`(∆)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Lazy-Unicode.html#v:-8710-)
        - [`intersection`](https://hackage.haskell.org/package/containers/docs/Data-Map-Lazy.html#v:intersection) = [`(∩)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Lazy-Unicode.html#v:-8745-)
    - From the [`Data.Map.Strict.Unicode`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Strict-Unicode.html) module
        - [`member`](https://hackage.haskell.org/package/containers/docs/Data-Map-Strict.html#v:member) = [`(∈)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Strict-Unicode.html#v:-8712-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`member`](https://hackage.haskell.org/package/containers/docs/Data-Map-Strict.html#v:member) = [`(∋)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Strict-Unicode.html#v:-8715-)
        - [`notMember`](https://hackage.haskell.org/package/containers/docs/Data-Map-Strict.html#v:notMember) = [`(∉)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Strict-Unicode.html#v:-8713-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`notMember`](https://hackage.haskell.org/package/containers/docs/Data-Map-Strict.html#v:notMember) = [`(∌)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Strict-Unicode.html#v:-8716-)
        - [`empty`](https://hackage.haskell.org/package/containers/docs/Data-Map-Strict.html#v:empty) = [`(∅)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Strict-Unicode.html#v:-8709-)
        - [`union`](https://hackage.haskell.org/package/containers/docs/Data-Map-Strict.html#v:union) = [`(∪)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Strict-Unicode.html#v:-8746-)
        - [`difference`](https://hackage.haskell.org/package/containers/docs/Data-Map-Strict.html#v:difference) = [`(∖)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Strict-Unicode.html#v:-8726-)
        - `\x y -> `[`union`](https://hackage.haskell.org/package/containers/docs/Data-Map-Strict.html#v:union)` (`[`difference`](https://hackage.haskell.org/package/containers/docs/Data-Map-Strict.html#v:difference)` x y) (`[`difference`](https://hackage.haskell.org/package/containers/docs/Data-Map-Strict.html#v:difference)` y x)` = [`(∆)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Strict-Unicode.html#v:-8710-)
        - [`intersection`](https://hackage.haskell.org/package/containers/docs/Data-Map-Strict.html#v:intersection) = [`(∩)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-Map-Strict-Unicode.html#v:-8745-)
    - From the [`Data.IntSet.Unicode`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntSet-Unicode.html) module
        - [`member`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:member) = [`(∈)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntSet-Unicode.html#v:-8712-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`member`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:member) = [`(∋)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntSet-Unicode.html#v:-8715-)
        - [`notMember`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:notMember) = [`(∉)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntSet-Unicode.html#v:-8713-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`notMember`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:notMember) = [`(∌)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntSet-Unicode.html#v:-8716-)
        - [`empty`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:empty) = [`(∅)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntSet-Unicode.html#v:-8709-)
        - [`union`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:union) = [`(∪)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntSet-Unicode.html#v:-8746-)
        - [`difference`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:difference) = [`(∖)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntSet-Unicode.html#v:-8726-)
        - `\x y -> `[`union`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:union)` (`[`difference`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:difference)` x y) (`[`difference`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:difference)` y x)` = [`(∆)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntSet-Unicode.html#v:-8710-)
        - [`intersection`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:intersection) = [`(∩)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntSet-Unicode.html#v:-8745-)
        - [`isSubsetOf`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:isSubsetOf) = [`(⊆)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntSet-Unicode.html#v:-8838-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`isSubsetOf`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:isSubsetOf) = [`(⊇)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntSet-Unicode.html#v:-8839-)
        - `\x y -> (x `[`/=`](http://hackage.haskell.org/package/base/docs/Prelude.html#v:-47--61-)` y) `[`&&`](http://hackage.haskell.org/package/base/docs/Prelude.html#v:-38--38-)` `[`not`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:not)` (`[`isProperSubsetOf`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:isProperSubsetOf)` x y)` = [`(⊈)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntSet-Unicode.html#v:-8840-)
        - `\x y -> (x `[`/=`](http://hackage.haskell.org/package/base/docs/Prelude.html#v:-47--61-)` y) `[`&&`](http://hackage.haskell.org/package/base/docs/Prelude.html#v:-38--38-)` `[`not`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:not)` (`[`isProperSubsetOf`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:isProperSubsetOf)` y x)` = [`(⊉)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntSet-Unicode.html#v:-8841-)
        - [`isProperSubsetOf`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:isProperSubsetOf) = [`(⊂)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntSet-Unicode.html#v:-8834-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`isProperSubsetOf`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:isProperSubsetOf) = [`(⊃)`](https://hackage.haskell.org/package/containers-unicode-symbol/docs/Data-IntSet-Unicode.html#v:-8835-)
        - `\x y -> `[`not`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:not)` (`[`isProperSubsetOf`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:isProperSubsetOf)` x y)` = [`(⊄)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntSet-Unicode.html#v:-8836-)
        - `\x y -> `[`not`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:not)` (`[`isProperSubsetOf`](https://hackage.haskell.org/package/containers/docs/Data-IntSet.html#v:isProperSubsetOf)` y x)` = [`(⊅)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntSet-Unicode.html#v:-8837-)
    - From the [`Data.IntMap.Lazy.Unicode`]() module (reexported by the [`Data.IntMap.Unicode`]() module)
        - [`member`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Lazy.html#v:member) = [`(∈)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Lazy-Unicode.html#v:-8712-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`member`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Lazy.html#v:member) = [`(∋)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Lazy-Unicode.html#v:-8715-)
        - [`notMember`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Lazy.html#v:notMember) = [`(∉)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Lazy-Unicode.html#v:-8713-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`notMember`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Lazy.html#v:notMember) = [`(∌)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Lazy-Unicode.html#v:-8716-)
        - [`empty`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Lazy.html#v:empty) = [`(∅)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Lazy-Unicode.html#v:-8709-)
        - [`union`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Lazy.html#v:union) = [`(∪)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Lazy-Unicode.html#v:-8746-)
        - [`difference`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Lazy.html#v:difference) = [`(∖)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Lazy-Unicode.html#v:-8726-)
        - `\x y -> `[`union`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Lazy.html#v:union)` (`[`difference`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Lazy.html#v:difference)` x y) (`[`difference`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Lazy.html#v:difference)` y x)` = [`(∆)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Lazy-Unicode.html#v:-8710-)
        - [`intersection`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Lazy.html#v:intersection) = [`(∩)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Lazy-Unicode.html#v:-8745-)
    - From the [`Data.IntMap.Strict.Unicode`]() module
        - [`member`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Strict.html#v:member) = [`(∈)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Strict-Unicode.html#v:-8712-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`member`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Strict.html#v:member) = [`(∋)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Strict-Unicode.html#v:-8715-)
        - [`notMember`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Strict.html#v:notMember) = [`(∉)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Strict-Unicode.html#v:-8713-)
        - [`flip`](https://hackage.haskell.org/package/base/docs/Prelude.html#v:flip)` `[`notMember`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Strict.html#v:notMember) = [`(∌)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Strict-Unicode.html#v:-8716-)
        - [`empty`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Strict.html#v:empty) = [`(∅)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Strict-Unicode.html#v:-8709-)
        - [`union`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Strict.html#v:union) = [`(∪)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Strict-Unicode.html#v:-8746-)
        - [`difference`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Strict.html#v:difference) = [`(∖)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Strict-Unicode.html#v:-8726-)
        - `\x y -> `[`union`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Strict.html#v:union)` (`[`difference`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Strict.html#v:difference)` x y) (`[`difference`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Strict.html#v:difference)` y x)` = [`(∆)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Strict-Unicode.html#v:-8710-)
        - [`intersection`](https://hackage.haskell.org/package/containers/docs/Data-IntMap-Strict.html#v:intersection) = [`(∩)`](https://hackage.haskell.org/package/containers-unicode-symbols/docs/Data-IntMap-Strict-Unicode.html#v:-8745-)

Try it out!

``` active haskell
{-# LANGUAGE UnicodeSyntax #-}
import Data.List.Unicode ((∪))
import qualified Data.Map as M
import Data.Map.Unicode ((∆))

main ∷ IO ()
main = do print $ [1, 2, 3] ∪ [1, 3, 5]
          print $ M.fromList [(1, 2), (3, 4)] ∆ M.fromList [(3, 4), (5, 6)]
```

# `RecursiveDo` and `DoRec`

**Available in:  All recent GHC versions**

The `RecursiveDo` extension (as well as its deprecated synonym `DoRec`) enables syntactic sugar for value recursion in a monadic context.  “What on Earth does that mean?” you might ask.  To explain, let’s take a look at how `let` behaves in Haskell.

`let` in Haskell allows lazy recursion; that is, you can write:

``` active haskell
main = print $
-- show
    let x = fst y
        y = (3, x)
    in  snd y
-- /show
```

However, `do` in Haskell does not allow lazy recursion; in fact, it doesn’t allow recursion at all.  If you try to write a recursive binding in do notation, it will fail; for example, the following code will cause an error that complains about `y` not being in scope:

``` active haskell
{-# LANGUAGE StandaloneDeriving #-}
import Control.Monad.Identity

deriving instance (Show a) => Show (Identity a)

main = print ((
-- show
    do x <- return $ fst y
       y <- return (3, x)
       return $ snd y
-- /show
    ) :: Identity Integer)
```

However, sometimes we want to be able to use value recursion but still need to be within a monad.  The `MonadFix` type class, from the `Control.Monad.Fix` module in the `base` package, provides an `mfix` function that helps us do exactly that, but the results, while they work, are not very pretty:

``` active haskell
{-# LANGUAGE StandaloneDeriving #-}
import Control.Monad.Identity

deriving instance (Show a) => Show (Identity a)

main = print ((
-- show
    do y <- mfix $ \y0 -> do x <- return $ fst y0
                             y1 <- return (3, x)
                             return y1
       return $ snd y
-- /show
    ) :: Identity Integer)
```

The `RecursiveDo` extension provides sugar for using `mfix` this way, so that the previous example can be equivalently rewritten as:

``` active haskell
{-# LANGUAGE RecursiveDo, StandaloneDeriving #-}
import Control.Monad.Identity

main = print ((
-- show
    mdo x <- return $ fst y
        y <- return (3, x)
        return $ snd y
-- /show
    ) :: Identity Integer)
```

`RecursiveDo` also provides a second type of syntactic sugar for `mfix` that uses the `rec` keyword instead of the `mdo` keyword.  The `rec`-based sugar is somewhat more direct and “low-level” than the `mdo`-based sugar.  In terms of the `rec` sugar, our running example is expressed as:

``` active haskell
{-# LANGUAGE RecursiveDo, StandaloneDeriving #-}
import Control.Monad.Identity

main = print ((
-- show
    do rec x <- return $ fst y
           y <- return (3, x)
       return $ snd y
-- /show
    ) :: Identity Integer)
```

The two types of sugar are subtly different in meaning, and the difference has to do with something called *segmentation*.

When GHC encounters a `let` binding, rather than naïvely binding all of the variables at once, it will divide (or *segment*) them into minimal mutually-dependent groups.  For example, take this expression:

``` haskell
let x = 1
    y = (x, z)
    z = fst y
    v = snd w
    w = (v, y)
in  (snd y, fst w)
```

Instead of just binding everything in a single group, GHC improves the code’s efficiency somewhat by treating it as though you’d actually written something like:

``` haskell
let x = 1
in  let y = (x, z)
        z = fst y
    in  let v = snd w
            w = (v, y)
        in  (snd y, fst w)
```

In a pure `let` binding, the only way this might matter is performance; the semantics of the code is guaranteed to not change. However, segmenting monadic code might produce unexpected results, because `mfix` has to deal with the monadic context somehow during the value recursion, and segmenting a set of bindings into minimal groups could potentially change the meaning of the code.

Only `mdo` segments its bindings.  `rec` does no segmentation at all, instead translating to calls to `mfix` exactly where you put `rec`s in the original code.  This means that, in the following example, the first two of the following three expressions are equivalent to each other, but the third one is not equivalent to either of the first two:

``` haskell
-- | expression 1 (equivalent to expression 2)
mdo x <- return 1
    y <- return $ (x, z)
    z <- return $ fst y
    v <- return $ snd w
    w <- return (v, y)
    return (snd y, fst w)

-- | expression 2 (equivalent to expression 1)
do x <- return 1
   rec y <- return $ (x, z)
       z <- return $ fst y
   rec v <- return $ snd w
       w <- return (v, y)
   return (snd y, fst w)

-- | expression 3 (not equivalent to expression 1 or expression 2)
do rec x <- return 1
       y <- return $ (x, z)
       z <- return $ fst y
       v <- return $ snd w
       w <- return (v, y)
   return (snd y, fst w)
```

Both *expression 1* and *expression 2* translate roughly to:

``` haskell
do x <- return 1
   (y, z) <- mfix $ \(y0, z0) -> do y1 <- return $ (x, z0)
                                    z1 <- return $ fst y0
                                    return (y1, z1)
   (v, w) <- mfix $ \(v0, w0) -> do v1 <- return $ snd w0
                                    w1 <- return (v0, y)
                                    return (v1, w1)
   return (snd y, fst w)
```

On the other hand, *expression 3* translates roughly to:

``` haskell
do (x, y, z, v, w) <- mfix $ \(x0, y0, z0, v0, w0) -> do x1 <- return 1
                                                         y1 <- return $ (x0, z0)
                                                         z1 <- return $ fst y0
                                                         v1 <- return $ snd w0
                                                         w1 <- return (v0, y0)
                                                         return (x1, y1, z1, v1, w1)
   return (snd y, fst w)
```

Try it out!

``` active haskell
{-# LANGUAGE RecursiveDo #-}
import Control.Monad.State.Lazy

comp = do x0 <- get
          modify (+1)
          x1 <- get
          rec y <- return $ (x0, fst z)
              z <- return $ (x1, fst y)
          put 3
          return (y, z)

main = print $ runState comp 1
```

**WARNING:** In GHC versions before 7.6, there was a lot of churn in the meanings of the `RecursiveDo` and `DoRec` extensions and their relationship to each other.  For such older GHC versions, the above discussion may be partially or wholly inaccurate; consult your GHC version’s User’s Guide for more detailed information.

# `LambdaCase`

**Available in:  GHC 7.6 and later**

The `LambdaCase` extension is very simple.  Any time you would otherwise have written:

``` haskell
\x -> case x of ...
```

you can instead simply write

``` haskell
\case ...
```

which is both shorter and doesn’t bind `x` as a name.  The Layout Rule works as usual with `LambdaCase`, so, for example:

``` haskell
[Just 1, Just 2, Nothing, Just 3] `forM_` \x -> case x of
    Just v  -> putStrLn ("just a single" ++ show v)
    Nothing -> putStrLn "no numbers at all"
```

can be shortened to:

``` haskell
[Just 1, Just 2, Nothing, Just 3] `forM_` \case
    Just v  -> putStrLn ("just a single" ++ show v)
    Nothing -> putStrLn "no numbers at all"
```

Try it out!

``` active haskell
{-# LANGUAGE LambdaCase #-}
import Control.Monad (forM_)

-- | should print:
--   @["just a single 1","just a single 2","no numbers at all","just a single 3"]@
main = [Just 1, Just 2, Nothing, Just 3] `forM_` \case
    Just v  -> putStrLn ("just a single " ++ show v)
    Nothing -> putStrLn "no numbers at all"
```

# `EmptyCase`

**Available in:  GHC 7.8 and later**

The `EmptyCase` extension allows you to write a `case` statement that has no clauses; the syntax is `case `*`e`*` of {}` (where *`e`* is any expression).  If you also have [`LambdaCase`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/basic-syntax-extensions#lambdacase) enabled, you can abbreviate `\x -> case x of {}` to `\case {}`

This is most useful when you have a type that you *know for sure* has no values, but Haskell‘s syntax and type system force you to do something with a hypothetical such value anyway.  Without `EmptyCase`, you could just use `error` or `undefined`, or otherwise [diverge](https://en.wikipedia.org/wiki/Divergence_%28computer_science%29), and such an action is still possible; however, using an empty `case` statement for such things is more indicative of intent, and holds some promise of being [better supported by the exhaustivity checker in the future.](http://research.microsoft.com/en-us/um/people/simonpj/papers/pattern-matching/gadtpm.pdf)

# `MultiWayIf`

**Available in:  GHC 7.6 and later**

The `MultiWayIf` extension allows you to use the full power of Haskell’s guard syntax in an `if` expression.  For example, this code:

``` haskell
if x == 1
   then "a"
   else if y < 2
           then "b"
           else "c"
```

can be rewritten as:

``` haskell
if | x == 1    -> "a"
   | y <  2    -> "b"
   | otherwise -> "d"
```
which is much nicer.

Try it out!

``` active haskell
{-# LANGUAGE MultiWayIf #-}

fn :: Int -> Int -> String
fn x y = if | x == 1    -> "a"
            | y <  2    -> "b"
            | otherwise -> "c"

-- | should print:
--   @c@
main = putStrLn $ fn 3 4
```

**WARNING:** In GHC 7.6, the use of `MultiWayIf` doesn’t affect [layout](https://www.haskell.org/onlinereport/haskell2010/haskellch2.html#x7-210002.7), instead allowing the previous layout (prior to the `if` keyword) to remain unchanged.  This was changed shortly afterwards; in GHC 7.8 and later, `MultiWayIf` affects layout, just like ordinary function guards do.

# `BinaryLiterals`

**Available in:  GHC 7.10 and later**

Standard Haskell allows you to write integer literals in decimal (without any prefix), hexadecimal (preceded by `0x` or `0X`), and octal (preceded by `0o` or `0O`).  The `BinaryLiterals` extension adds binary (preceded by `0b` or `0B`) to the list of acceptable integer literal styles.

Try it out!

``` active haskell
{-# LANGUAGE BinaryLiterals #-}

-- | should print:
--   @(1458,1458,1458,1458)@
main = print (1458, 0x5B2, 0o2662, 0b10110110010)
```

# `NegativeLiterals`

**Available in:  GHC 7.8 and later**

Standard Haskell desugars negative numeric literals (of either integer or fractional form) by applying the `negate` function from the `Num` type class to the corresponding positive numeric literal (which is then expanded again using either `fromInteger` or `fromRational`, as appropriate).  That is, the standard full desugaring of the literal `-1458` is `negate (fromInteger 1458)`.  The `NegativeLiterals` extension changes this, making negative numeric literals instead desugar as `fromInteger` or `fromRational` applied directly to a negative `Integer` or `Rational` value; that is, `-1458` is desugared as `fromInteger (-1458)`.  In a sense, `NegativeLiterals` swaps the positions of negation and conversion in the desugaring of numeric literals.

This doesn’t make a difference for the common cases, but certain edge cases can behave differently (and usually better) under `NegativeLiterals` than otherwise.  The example that the [GHC User’s guide](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/syntax-extns.html#negative-literals) gives is 8-bit signed arithmetic, in which `128` is not representable but `-128` *is* representable.  The naïve desugaring of `-128` to `negate (fromInteger 128)` results in an overflow from `128` to `-128` followed by a negation to `128` followed by *another* overflow back to `-128`; meanwhile, the `NegativeLiterals` desugaring to `fromInteger (-128)` doesn’t waste cycles (or risk trapping on some architectures), but instead produces the appropriate value from the start.  Other examples might actually change behavior rather than simply be less efficient; you should make sure that you understand a piece of numeric Haskell code fairly well before enabling or disabling `NegativeLiterals` for it.

Try it out!

``` active haskell
{-# LANGUAGE NegativeLiterals #-}

main = do print (-1 :: ExplicitNegation Integer)
          print (negate 1 :: ExplicitNegation Integer)
          print (-1.5 :: ExplicitNegation Rational)
          print (negate 1 :: ExplicitNegation Rational)

-- /show
-- this type exists solely to explicitly mark where negation happens
data ExplicitNegation n = Value n | Negate (ExplicitNegation n) deriving Show

collapseNegation :: Num n => ExplicitNegation n -> n
collapseNegation (Value x) = x
collapseNegation (Negate v) = negate $ collapseNegation v

instance (Eq n, Num n) => Eq (ExplicitNegation n) where
    v == w = collapseNegation v == collapseNegation w
instance (Ord n, Num n) => Ord (ExplicitNegation n) where
    v `compare` w = collapseNegation v `compare` collapseNegation w
instance Num n => Num (ExplicitNegation n) where
    v + w = Value $ collapseNegation v + collapseNegation w
    v * w = Value $ collapseNegation v * collapseNegation w
    negate = Negate
    abs = Value . abs . collapseNegation
    signum = Value . signum . collapseNegation
    fromInteger = Value . fromInteger
instance Fractional n => Fractional (ExplicitNegation n) where
    recip = Value . recip . collapseNegation
    fromRational = Value . fromRational
```

# `NumDecimals`

**Available in:  GHC 7.8 and later**

Standard Haskell gives the polymorphic type `(Fractional a) => a` to otherwise-unconstrained fractional numeric literals; however, some such literals are guaranteed to actually be integers, because they have an exponent (whether implicit or explicit) that is larger than the distance from the decimal point at which their last non-zero digit occurs (for example, `4.65690e4` is “the same number” as `46569`, which is clearly an integer).  The `NumDecimals` extension exploits this fact by giving fractional literals which are “really just integers” the more general type `(Num a) => a` instead.


Try it out!

``` active haskell
{-# LANGUAGE NumDecimals #-}

-- notice that this code will not compile if
-- '1.6e1' isn't allowed to be an 'Integer'
main = print (1.6e1 `div` 5 :: Integer)
```

# `DoAndIfThenElse`

**Available in:  GHC 7.0 and later**

**TODO**

# `NondecreasingIndentation`

**Available in:  GHC 7.2 and later**

**TODO**
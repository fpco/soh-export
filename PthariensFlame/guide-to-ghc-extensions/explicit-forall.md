# Explicit `forall`

These extensions allow finer-grained control over polymorphism.

# `ExplicitForAll`

**Available in:  GHC 6.12 and later**

The `ExplicitForAll` extension allows you to explicitly specify the universal quantification in polymorphic type signatures.  For example, this:

``` haskell
Just :: a -> Maybe a
Nothing :: Maybe a
reverse :: [a] -> [a]
map :: (a -> b) -> [a] -> [b]
show :: (Show a) => a -> String
```

becomes this:

``` haskell
Just :: forall a. a -> Maybe a
Nothing :: forall a. Maybe a
reverse :: forall a. [a] -> [a]
map :: forall a b. (a -> b) -> [a] -> [b]
show :: forall a. (Show a) => a -> String
```

GHC does this for you anyway, but while `ExplicitForAll` is not very useful on its own, the other extensions in this section depend on the ability to write out your `forall`s manually, to allow you to write them in places GHC normally wouldn't.

Although `ExplicitForAll` is important as a prerequisite for all of the other extensions in this section, it does not do anything on its own that could not be done in standard Haskell.

# `ScopedTypeVariables`

**Available in:  All recent GHC versions**

The `ScopedTypeVariables` extension is perhaps the simplest of the actually useful `forall`-based extensions.  It allows you to specify that the type variables in a definition's signature should be *scoped* over the body of that definition.

To give some motivation, consider the following function:

``` active haskell
import Data.List

main = putStrLn "No errors."

-- show
myFunction :: Ord a => [a] -> [(a, a)]
myFunction inputList = zip sortedList nubbedList
    where sortedList = sort inputList
          nubbedList = nub inputList
-- /show
```

What happens if we try to give a type to `sortedList` or `nubbedList`?  Let's see:

``` active haskell
import Data.List

main = putStrLn "No errors."

-- show
myFunction :: Ord a => [a] -> [(a, a)]
myFunction inputList = zip sortedList nubbedList
    where sortedList :: [a]
          sortedList = sort inputList
          nubbedList :: [a]
          nubbedList = nub inputList
-- /show
```

We get some ugly errors telling us the the `a`'s in our inner definitions are not the same as the `a` in our outer definition (the two inner `a`'s aren't the same as each other, either).  How can we fix this?  We need a way to tell GHC that, inside the `where` clause, `[a]` does not mean `forall a. [a]`, but instead *closes over the `a` from the outer definition*.

This is what `ScopedTypeVariables` is for.   By enabling it, and by placing a `forall` in the outer definition's signature, that `a` becomes scoped over the whole body of the outer definition, including the `where` clause:

``` active haskell
{-# LANGUAGE ScopedTypeVariables #-}
import Data.List

main = putStrLn "No errors."

-- show
myFunction :: forall a. Ord a => [a] -> [(a, a)]
myFunction inputList = zip sortedList nubbedList
    where sortedList :: [a]
          sortedList = sort inputList
          nubbedList :: [a]
          nubbedList = nub inputList
-- /show
```

Try it out!


``` active haskell
{-# LANGUAGE ScopedTypeVariables #-}
import Data.List

main = print $ myFunction [1, 1, 3, 2]

myFunction :: forall a. Ord a => [a] -> [(Char, a, a)]
myFunction inputList = zip3 ['a'..'z'] sortedList nubbedList
    where sortedList :: [a]
          sortedList = sort inputList
          nubbedList :: [a]
          nubbedList = nub inputList
```

# `LiberalTypeSynonyms`

**Available in:  All recent GHC versions**

In standard, unextended Haskell, type synonyms are fairly restricted.  They must always be fully applied and they cannot contain type variables other than their parameters.  The `LiberalTypeSynonyms` extension relaxes both of these limitations, as well as some others that only come into play when certain other extensions are used (the use of `LiberalTypeSynonyms` in combination with each other relevant extension is described in the section for that particular other extension).

When `LiberalTypeSynonyms` is enabled, GHC only type-checks a signature *after* all type synonyms have been expanded, outermost first.  You can now partially apply a type synonym, as long as it's surrounded by another type synonym such that the obvious outermost-first expansion will cause the partially-applied synonym to become fully applied.

This means that the following partial application of type synonyms is now legal:

``` active haskell
{-# LANGUAGE LiberalTypeSynonyms #-}
import Data.Char

main = putStrLn "No errors."

-- show
type Const a b = a

type Id a = a

type NatApp f g i = f i -> g i

myFunc :: NatApp Id (Const Int) Char
--     ~  Id Char -> Const Int Char
--     ~  Char    -> Int
myFunc = ord
-- /show
```

It's important to note that GHC will still *kind-check* type synonyms and their (possibly partial) applications, just not *type-check* them (until after full expansion).  It will also still forbid any expansion that would be illegal for some other reason; i.e., `LiberalTypeSynonyms` alone will not allow type safety to be broken.  Anything that would not type-check even after all type synonyms are expanded will still remain illegal with `LiberalTypeSynonyms` enabled.

While `LiberalTypeSynonyms` does not by itself use explicit `forall`s, it is often important  when working with the rest of the extensions in this section, so its guide is placed here for convenience.

# `RankNTypes`, `Rank2Types`, and `PolymorphicComponents`

**Available in:  All recent GHC versions**

## Basic Usage

The `RankNTypes` extension (as well as its deprecated synonyms `Rank2Types` and `PolymorphicComponents`) allows you to nest explicit `forall`s within function types and data definitions.  For example, the following program requires `RankNTypes`:

``` active haskell
{-# LANGUAGE RankNTypes #-}

-- show
main = print $ rankN (+1)

rankN :: (forall n. Num n => n -> n) -> (Int, Double)
rankN f = (f 1, f 1.0)
-- /show
```

The difference between the signature of `rankN` and the similar signature

``` haskell
forall n. Num n => (n -> n) -> (Int, Double)
```

is that, in the latter, `n` is chosen by the *caller*, but in the former, `n` is chosen by the *callee*; in both cases *`n` may be chosen more than once!*  In other words, you could pass a function of type `Int -> Int` or `Double -> Double` as the first parameter of the latter signature, and the type system would be fine with it.  However, the former signature forces its user to only pass in truly polymorphic functions:  functions precisely of type `forall n. Num n => n -> n` or more general.  `(+1)` is one such function, as are `(6*)`, `abs`, and `id`; however, you could not pass in `(/5)` because that requires `Fractional`, even if some types with an instance of `Num` also have an instance of `Fractional`.  The latter signature requires a function from `n` to `n` *for some* `Num` `n`; the former signature requires a function from `n` to `n` *for every* `Num` `n`.

## Interaction with [`LiberalTypeSynonyms`](https://www.fpcomplete.com/user/PthariensFlame/guide-to-ghc-extensions/explicit-forall#liberaltypesynonyms)

With both `RankNTypes` and `LiberalTypeSynonyms` enabled, it becomes possible to:

- use `forall` and/or constraints in type synonyms:

``` active haskell
{-# LANGUAGE RankNTypes, LiberalTypeSynonyms #-}

main = putStrLn "No errors."

-- show
type Const a b = a

type Id a = a

type Indexed f g = forall i. f i -> g i

type ShowIndexed f g = forall i. (Show i) => f i -> g i

type ShowConstrained f a = (Show a) => f a

type FunctionTo a b = b -> a

myFunc1 :: Indexed Id (Const Int)
--      ~  forall i. Id i -> Const Int i
--      ~  forall i. i    -> Int
myFunc1 _ = 2

myFunc2 :: ShowIndexed Id (Const Int)
--      ~  forall i. (Show i) => Id i -> Const Int i
--      ~  forall i. (Show i) => i    -> Int
myFunc2 = length . show

myFunc3 :: forall a. ShowConstrained (FunctionTo Char) a
--      ~  forall a. (Show a) =>      FunctionTo Char  a
--      ~  forall a. (Show a) =>      a -> Char
myFunc3 = head . show
-- /show
```

- apply (or even partially apply) a type synonym to a type containing `forall` and/or constraints:

``` active haskell
{-# LANGUAGE RankNTypes, LiberalTypeSynonyms #-}

main = putStrLn "No errors."

-- show
type ShowConstrained f = forall a. (Show a) => f a

type EnumFunctionTo b a = (Enum a) => a -> b

myFunc :: ShowConstrained              (EnumFunctionTo Char)
--     ~  forall a. (Show a) =>         EnumFunctionTo Char a
--     ~  forall a. (Show a, Enum a) => a -> Char
myFunc = head . show . succ
-- /show
```

## Use Case: The `ST` Monad

There is a place in the standard libraries where `RankNTypes` is used to great effect.  That place is the `Control.Monad.ST` module in the `base` package (as well as its submodules), and `RankNTypes` is used there to allow local mutability while maintaining a pure interface.

Let's say we want to implement an algorithm in Haskell, but the algorithm we've chosen depends internally upon mutability.  Nonetheless, we know that the algorithm is effectively pure.  Take the following semi-contrived example (in C-like pseudocode), trying to demonstrate the [Collatz conjecture](http://en.wikipedia.org/wiki/Collatz_conjecture):

```
Integer collatz(Integer n) {
    
    assert(n > 0, "n must be positive");
    
    Integer x = n;
    Integer count = 0;
    
    while (x != 1) {
        
        count = count+1;
        
        if (x % 2 == 0) {
            x = x/2;
        } else {
            x = 3*x+1;
        }
        
    }
    
    return count;
    
}
```

Let's say we want to implement this algorithm in Haskell, and we want it to be a pure function (because it effectively is), but we don't want to give up the efficiency that mutability affords us.  Fortunately, the `ST` monad comes to our rescue.  Here's how this example looks in Haskell:

``` active haskell
import Control.Exception (evaluate)
import Control.Monad.ST
import Data.STRef
import System.IO

-- copied verbatim from:
-- monad-loops-0.3.3.0:Control.Monad.Loops.whileM_
whileM_ :: (Monad m) => m Bool -> m a -> m ()
whileM_ p f = go
    where go = do x <- p
                  if x
                     then f >> go
                     else return ()

main = do putStrLn "Enter a positive integer:"
          ln <- getLine
          if null ln then hFlush stdout else
            do let x = collatz $ read ln
               evaluate x
               putStr $ "collatz " ++ ln ++ " = "
               print x
               main

-- show
collatz :: Integer -> Integer
collatz n | n > 0 = runST $ do xRef <- newSTRef n
                               countRef <- newSTRef (0 :: Integer)
                               whileM_ (do x <- readSTRef xRef
                                           return $ x /= 1)
                                       (do modifySTRef countRef (+1)
                                           modifySTRef xRef (\x -> if even x
                                                                      then x `div` 2
                                                                      else 3*x+1))
                               readSTRef countRef
          | otherwise = error "n must be positive"
-- /show
```

What does this have to do with `RankNTypes`?  Well, the signatures of the various `ST` functions we've used are as follows:

``` haskell
newSTRef :: forall s a. a -> ST s (STRef s a)
readSTRef :: forall s a. STRef s a -> ST s a
writeSTRef :: forall s a. STRef s a -> a -> ST s () -- not used in this example
modifySTRef :: forall s a. STRef s a -> (a -> a) -> ST s ()
runST :: forall a. (forall s. ST s a) -> a
```

Notice that the `ST` monad (as well as `STRef`s) carries around an extra type parameter `s` that `runST` forces to be completely polymorphic *and not present in its result*.  If you try to pass a mutable variable out of the `ST` monad (*e.g.*, if you call `runST` on a value of type `ST s (STRef s a)`), it will be a compile-time error.  See what happens when we try to break referential transparency:

``` active haskell
import Control.Monad.ST
import Data.STRef

main = putStrLn "No errors."

-- show
illegal = runST $ do xRef <- newSTRef (0 :: Integer)
                     modifySTRef xRef (+1)
                     return xRef
-- /show
```

We get a compile-time error that tells us that there is no possible valid type for `illegal`.  Because of the use of `RankNTypes` in the signature of `runST`, it is statically guaranteed that the result of `runST` is pure, even if the computation uses mutability internally.

## Use Case: `lens`

The [`lens` library](http://hackage.haskell.org/package/lens) makes use of `RankNTypes` (and `LiberalTypeSynonyms`) to implement the very powerful notion of lenses, sometimes called “functional references”.  The idea behind lenses is that we need a thing like a property accessor:  something that, given a value of its source type, can produce a value of its target type, and that, given a value of its source type and a value of its target type, can produce a modified value of its source type:

``` haskell
type Lens' s a
view :: Lens' s a -> s -> a
set :: Lens' s a -> a -> s -> s
```

(Note that the actual types are a bit more general than this; see the `lens` package's documentation, linked above, for the exact types used).

For example, the `lens` package (in the `Data.Complex.Lens` module) conveniently provides a lens to get the real part of a complex number:

``` haskell
_realPart :: Lens' (Complex a) a
```

We can use `view` to feed this lens a complex number, and we will get back its real part; we can also use `set` to feed this lens a complex number and a new real part, and we will get back a complex number with the new real part and the original imaginary part:

``` active haskell
import Data.Complex
import Control.Lens
import Data.Complex.Lens

main = do putStr "c  = "
          print   c
          putStr "r  = "
          print   r
          putStr "r' = "
          print   r'
          putStr "c' = "
          print   c'


-- show
c  :: Complex Double
c  =  1.0 :+ 2.5

r  :: Double
r  =  view _realPart c

r' :: Double
r' =  2.0

c' :: Complex Double
c' =  set _realPart r' c
-- /show
```

How is `RankNTypes` involved here?  Well, it has to do with the implementation of the `Lens'` type alias (see the `lens` package documentation for the actual code involved):

``` haskell
type Lens' s a = forall f. (Functor f) => (a -> f a) -> (s -> f s)
```

While this is not the place for a full introduction to the van Laarhoven lens representation and its many and varied uses and extensions, suffice to say that `RankNTypes` plays a critical role in the core functionality of the `lens` package, and that much of that functionality is impossible to reproduce in any language without either higher-rank universal polymorphism or some equivalent capability.
# A more general interface to collections with mono-kinded Abstract Data Types

## Extending the span of typeclasses with mono-traversable Abstract Data Types.

The standard Haskell typeclass library is based on classes for types parameterized on the element type, e.g. `[a], Seq a`. This excludes from its use the collection types with a fixed element type, eg. `ByteString, Text, ...`.


The [mono-traversable](https://hackage.haskell.org/package/mono-traversable) library uses class associated types and type families to bring type classes with a mono-kinded type index ([kind](https://wiki.haskell.org/Kind): *) suited to all sort of collection types, whether parameterized or not.

```haskell
-- see https://hackage.haskell.org/package/mono-traversable/docs/Data-Sequences.html#t:SemiSequence

class SemiSequence t     -- Semigroup based sequence operations
    type Index t         -- the type of the index as a class associated one
    -- type Element t    -- possible Element associated type

-- detacching the Element type from the classes avoids having Foldable as a derived class dependent on the structure

type family Element t

-- Element for sequences
type instance Element [a] = a
type instance Element ByteString = Word8
type instance Element Text = Char

-- Element for sets
type instance Element IntSet = Int
type instance Element (Set a) = a

-- Element for dictionaries
type instance Element (Map k v) = v
type instance Element (IntMap v) = v

-- a MonoFoldable class for all sort of structures, since Element is not class dependent
class MonoFoldable t where
    ofoldr :: (Element t -> b -> b) -> b -> t -> b 
    ofoldl' :: (a -> Element t -> a) -> a -> t -> a
    ofoldMap :: Monoid m => (Element t -> m) -> t -> m
    ...
    
```

The function names which collide with names predefined in Prelude, like `null`, add an 'o' prefix, as `onull` to avoid continuos desambiguation.

Example of use:

```haskell
{-| chunks.hs 
It implements the function chunks of bounded length of a sequence,
returning the sequence of sequences, abstracting over any sequence type
-}
{-# LANGUAGE PackageImports, TypeFamilies #-}
import "mono-traversable" Data.Sequences as S
import "mono-traversable" Data.MonoTraversable as M
import Data.Function ((&))        -- (&): backwards application
import Control.Monad (mfilter)

-- IsSequence implies Monoid and MonoFoldable to which belongs 'onull'

chunks :: (IsSequence t, IsSequence t', Element t' ~ t) => Index t -> t -> t'
chunks n seq = unfoldr (splitAndValidate n) seq

splitAndValidate :: IsSequence t => Index t -> t -> Maybe (t, t)
splitAndValidate n seq = seq & S.splitAt n 
                             & Just
                             & mfilter (not . M.onull . fst)

unfoldr :: (IsSequence t, Element t ~ a) => (b -> Maybe (a, b)) -> b -> t
unfoldr f state = case f state of
                Just (x, state') -> S.cons x $ unfoldr f state'
                Nothing -> mempty    
```
Using it:

```haskell
$ ghci
GHCi, version 7.10.3: http://www.haskell.org/ghc/  :? for help
Prelude> :load chunks
[1 of 1] Compiling Main             ( chunks.hs, interpreted )
Ok, modules loaded: Main.

*Main> chunks 2 "abcdef" :: [String]
["ab","cd","ef"]

*Main> import Data.Vector as Vec
*Main Vec> chunks 2 (Vec.fromList [1..4]) :: Vector (Vector Int)
[[1,2],[3,4]]
```

### Sequences ADTs

* [SemiSequence](https://hackage.haskell.org/package/mono-traversable/docs/Data-Sequences.html#t:SemiSequence): Semigroup based sequence operations.
* IsSequence: extends SemiSequence with Monoid based operations.
* Textual: A sequence splittable in lines and words with letter case operations.
* LazySequence: A generic class (multiparameter) parameterized with a strict sequence companion.
* Utf8: A generic class for Textual data which can be encoded to and decoded from UTF8, parameterized with the binary counterpart.

### Container ADTs

* [SetContainer](https://hackage.haskell.org/package/mono-traversable/docs/Data-Containers.html#t:SetContainer): (key, value) pairs containers as monoids (union and empty ops.), foldable on its elements (MonoFoldable)
* IsMap: A traversable SetContainer
* IsSet: A SetContainer whose elements are ContainerKeys

### ADTs for all

* [MonoFunctor](https://hackage.haskell.org/package/mono-traversable/docs/Data-MonoTraversable.html#t:MonoFunctor)
* [MonoFoldable](https://hackage.haskell.org/package/mono-traversable/docs/Data-MonoTraversable.html#t:MonoFoldable)
* [MonoTraversable](https://hackage.haskell.org/package/mono-traversable/docs/Data-MonoTraversable.html#t:MonoTraversable)

## The NonNull type wrapper grants a use of MonoFoldable without exceptions.

See [NonNull](https://hackage.haskell.org/package/mono-traversable/docs/Data-NonNull.html#t:NonNull).

```haskell
-- Safely convert from an unsafe container to a safe non-empty container.

fromNullable :: MonoFoldable mono => mono -> Maybe (NonNull mono)

-- Safe foldings over ordered domains
maximum, minimum :: (MonoFoldable mono, Ord (Element mono)) => NonNull mono -> Element mono 

```




# Monoids Tour

Like the term "monad", "monoid" can be a bit intimidating:  Why would I want to think about math while I'm doing real world programming?!?

One answer is that thinking in terms of monoids can be very beneficial to parallelism and efficient data structures.  Using the `Monoid` interface in Haskell allows you to leverage the many convenient functions that work with them.


What's in a Monoid?
-------------------

A monoid has two things: a 'unit' value, which we call <hoogle>mempty</hoogle>, and an append operation that combines values, called <hoogle>mappend</hoogle>.  The typeclass for <hoogle>Monoid</hoogle> looks like:

```haskell
-- Defined in Data.Monoid

class Monoid a where
    mempty  :: a
    mappend :: a -> a -> a

    -- This is here because some types might have a more efficient definition for it.
    mconcat :: [a] -> a
    mconcat = foldr mappend mempty
```

The `(<>)` operator has become the conventional way to refer to `mappend`:

```haskell
infixr 6 (<>)

(<>) :: Monoid a => a -> a -> a
(<>) = mappend
```

So what's so special about this `(<>)` thing?  Mainly, that it's _associative_ -- this means using `(<>)` to merge together a sequence of items results in the same value regardless of the evaluation order. The following laws must hold:

```haskell
a <> (b <> c) = (a <> b) <> c
mempty <> a = a
a <> mempty = a
```

These last two laws say that `mempty` is the identity for `(<>)`, and it doesn't matter which side it's used on.  Without `mempty` and those laws, associativity is still an interesting property.  Types that have this property can be an instance of the "[Semigroup](http://hackage.haskell.org/packages/archive/semigroups/latest/doc/html/Data-Semigroup.html)" typeclass.  However, since `Monoid` is in the base libraries, it's much more commonly used.

The main thing to understand about a series of values merged by `(<>)` is that the order that they are evaluated doesn't matter.  However, often the order of the arguments does matter -- `a <> b` is not necessarily the same thing as `b <> a`.


Lists
-----

List concatenation is an associative operation, and adding an empty list to either side just results in the same list.  It's a monoid!

```haskell
instance Monoid [a] where
    mempty  = []
    mappend = (++)
```

Let's test that this makes sense as a monoid:

```active haskell
main = do
    let a = [1,1,2]
        b = [3,5]
        c = [8,13]

    putStrLn "These are equal:"
    print $  a ++ (b  ++ c)
    print $ (a ++  b) ++ c
    putStrLn "\nThese leave 'a' alone:"
    print $ a ++ []
    print $ [] ++ a
```

Yup!  It satisfies the laws we want.  Let's try using the monoid instance on `String` (which uses the list monoid):

```active haskell
import Data.Monoid

main = do 
    putStrLn "These are equal:"
    print $ "Hello there!" <>  mempty <> " Monoids are"  <> " neat!" <> mempty
    print $ "Hello there!" <> (mempty <> " Monoids are") <> " neat!" <> mempty
```

Many other collection-like structures have `Monoid` instances corresponding to concatenation or union.  These include many of the datatypes in the [containers package](http://hackage.haskell.org/package/containers) - <hoogle>Sequence</hoogle>, <hoogle>Map</hoogle>, <hoogle>Set</hoogle>, <hoogle>IntMap</hoogle>, and <hoogle>IntSet</hoogle>.


Dual Wrapper
------------

If an operation satisfies these laws no matter the input values, then it is by definition a monoid - there may be multiple valid monoids for a given datatype.  For example, this is another monoid for lists:

```haskell
instance Monoid [a] where
    mempty = []
    mappend = flip (++)
```

_Note: <hoogle>flip</hoogle> just takes a function of two arguments, and switches them._

It's not a very useful instance!  It still concatenate the two lists, but with the second one before the first.  In general, if you're going to define an instance of monoid for your datatype, then the most straightforward and useful definition should be chosen.

This particular variation of the list monoid can be generalized - there is a <hoogle>Dual</hoogle> newtype in "Data.Monoid", which flips the `(<>)` of whatever monoid it wraps.

```active haskell
import Data.Monoid

main = print $ Dual "world" <> Dual "hello " 
```

`Dual` is not used very frequently - but its a useful way of showing that there are often several valid instances of `Monoid` for a given datatype.


Maybe
-----

The <hoogle>Maybe</hoogle> monoid follows a common pattern found when writing typeclass instances for containers: the type of the element is also constrained to being an instance of the class.

```haskell
instance Monoid a => Monoid (Maybe a) where
  mempty = Nothing

  Nothing `mappend` x = x
  x `mappend` Nothing = x
  Just x `mappend` Just y = Just (x `mappend` y)
```

The last line of this definition is saying that `Just x <> Just y` is merged by using the `Monoid a` to merge the two.  Let's look at an example:

```active haskell
import Data.Monoid
import Safe (readMay)

main = print numbers

numbers :: Maybe [Int]
numbers = readMay "[1,2,3]" <> readMay "[90" <> readMay "[4,5,6]"
```

What's happening here is that `readMay "[90"` returns `Nothing`, because the parse fails.  In the `Maybe` monoid, `Nothing` is `mempty`, so due to the monoid laws, we know we can just leave it out.  So it becomes just the other two parses merged with `<>`.

```haskell
numbers = readMay "[1,2,3]" <> readMay "[4,5,6]"

--      = Just [1,2,3] <> Just [4,5,6]

--      = Just [1,2,3,4,5,6]
```

This isn't the only monoid for `Maybe` that's useful!  After all, `Maybe` is often used to represent failure.  What if we just wanted the first thing that worked?


First and Last
--------------

The <hoogle>First</hoogle> / <hoogle>Last</hoogle> wrappers in `Data.Monoid` wrap `Maybe` values to provide a different monoid instance.  These instances don't require the type of the inner value to also be a `Monoid`, because they don't need to do any operations on them.  Here are there definitions:

```haskell
-- | Maybe monoid returning the leftmost non-Nothing value.
newtype First a = First { getFirst :: Maybe a }
        deriving (Eq, Ord, Read, Show)

instance Monoid (First a) where
        mempty = First Nothing
        r@(First (Just _)) `mappend` _ = r
        First Nothing `mappend` r = r

-- | Maybe monoid returning the rightmost non-Nothing value.
newtype Last a = Last { getLast :: Maybe a }
        deriving (Eq, Ord, Read, Show)

instance Monoid (Last a) where
        mempty = Last Nothing
        _ `mappend` r@(Last (Just _)) = r
        r `mappend` Last Nothing = r
```

Let's see an example!  This is a convenient time to use `mconcat :: Monoid a => [a] -> a`, because then we can just use `map` to wrap all of our values:

```active haskell
import Data.Monoid
import Safe (readMay)

vals :: [Maybe Int]
vals = [readMay "ignored", readMay "1", readMay "5", readMay "10"]

main = do
    print . mconcat $ map First vals
    print . mconcat $ map Last vals
```


Other Wrappers
--------------

Along with <hoogle>Dual</hoogle>, <hoogle>First</hoogle>, and <hoogle>Last</hoogle>, <hoogle>Data.Monoid</hoogle> defines a few more newtype wrappers.  All of them encode the observation that existing, commonly used operations and values are monoids.  For example, the <hoogle>Sum</hoogle> monoid encodes that `(+)` and `0` form a monoid:

```haskell
-- | Monoid under addition.
newtype Sum a = Sum { getSum :: a }
        deriving (Eq, Ord, Read, Show, Bounded)

instance Num a => Monoid (Sum a) where
        mempty = Sum 0
        Sum x `mappend` Sum y = Sum (x + y)
```

Here's an example:

```active haskell
import Data.Monoid

vals :: [Int]
vals = [1,2,3,4,5]

main = print . mconcat $ map Sum vals
```

The rest of the newtype wrappers follow this pattern -- the `mappend` operation just unwraps the value, and re-wraps them after combining with some operation.


* <hoogel>Sum</hoogle> is `(+)` and `0` (it wraps any <hoogle>Num</hoogle> instance)
* <hoogle>Product</hoogle> is `(*)` and `1` (it wraps any <hoogle>Num</hoogle> instance).
* <hoogle>All</hoogle> is `(&&)` and `True` (it wraps <hoogle>Bool</hoogle>).
* <hoogle>Any</hoogle> is `(||)` and `False`
* <hoogle>Endo</hoogle> is `(.)` and `id`

This last one, `Endo`, is pretty interesting.  Its name comes from category theory -- "Endomorphism" -- but its meaning here isn't so complicated.  It just means that functions with the type `a -> a` can be combined by composition, and the order of the application of composition doesn't matter much:

```active haskell
main = do
    let f =  (+1) . ((*2)  . negate)
        g = ((+1) .  (*2)) . negate
    
    putStrLn "These are equal:"
    print $ f 5
    print $ g 5
```

Using `Endo`, this looks like:

```active haskell
import Data.Monoid

f :: Endo Int
f = mconcat $ map Endo [(+1), (*2), negate]

main = print $ f `appEndo` 5
```

Here, `appEndo` extracts the `Int -> Int` function from the `Endo Int`.  Even though it's a field accessor, it can be used infix, because when supplied with an `Endo a`, it returns a function `a -> a`.

This property is one of the main things that makes using function composition attractive.  If you see "f . g . h" used to define a function, then you know that you could refactor out "f . g" or "g . h" as separate definitions.

```active haskell
v1 :: String -> Int
v1 = length . filter (not . null) . lines

v2 :: String -> Int
v2 = fg . lines
  where
    fg = length . filter (not . null)

v3 :: String -> Int
v3 = length . gh
  where
    gh = filter (not . null) . lines

main = do
    print $ v1 "a\n\nb\nc"
    print $ v2 "a\n\nb\nc"
    print $ v3 "a\n\nb\nc"
```

Applications
============

Naturally, practical applications of monoids take advantage of the fact that the operation is associative.  This can allow for efficient data structures and parallel computation.

Foldable
--------

The `Foldable` typeclass contains quite a few functions, but the important one is `foldMap`:

```haskell
class Foldable t where
    -- | Combine the elements of a structure using a monoid.
    fold :: Monoid m => t m -> m
    fold = foldMap id

    -- | Map each element of the structure to a monoid, and combine the results.
    foldMap :: Monoid m => (a -> m) -> t a -> m
    foldMap f = foldr (mappend . f) mempty
    
    -- | Right-associative fold of a structure.
    foldr :: (a -> b -> b) -> b -> t a -> b
    foldr f z t = appEndo (foldMap (Endo . f) t) z
    
    -- etc.. foldr', foldl, foldl', foldr1, foldl1
```

`foldMap` allows you to execute a query on some datatype, by providing a function from the "elements" of the datatype to some monoidal value.  Let's take `LeafyTree`, below, as an example datatype for the `t` in `Foldable t` :

```haskell
data LeafyTree a
    = Branch [LeafyTree a]
    | Leaf a
```

Let's say we decide that our `Foldable` instance should provide a left-to-right traversal of the leaves.  Then, the fact that it takes a `Monoid` means that the actual structure of the `Branch` parts of the structure don't matter!  This means that tree-like data structures can be used to represent efficient sequences, while supporting efficient query  (more on this in the next section).

```active haskell
import Data.Foldable
import Data.Monoid

data LeafyTree a
    = Branch [LeafyTree a]
    | Leaf a
-- show
instance Foldable LeafyTree where
    foldMap f (Branch xs) = mconcat $ map (foldMap f) xs
    foldMap f (Leaf a) = f a

tree1 = Branch [Leaf "abr", Branch [Branch [Leaf "a", Leaf "ca"], Leaf "dabra"]]

tree2 = Branch [Leaf "abr", Leaf "a", Branch [Leaf "ca", Leaf "dabra"]]

main = do
    putStrLn "These are equal:"
    print $ foldMap (Sum . length) tree1
    print $ foldMap (Sum . length) tree2
    
    -- Since the leaves are Monoids, we can just directly merge them:
    putStrLn "Also equal:"
    print $ foldMap id tree1
    print $ foldMap id tree2
```

If the implementation of this datatype is hidden, and only functions similar to `foldMap` are exported, then the user can't inspect the structure of the tree if they fold it with valid monoids.  This allows the tree's structure to be an implementation detail.


Measured Monoids
----------------

In [quite an excellent article](http://apfelmus.nfshost.com/articles/monoid-fingertree.html), Heinrich Apfelmus describes the monoidal annotations supported by fingertrees.  Such annotations really can be supported by any tree data structure.

Here's the typeclass used for "measured monoids" in the [fingertree package](http://hackage.haskell.org/packages/archive/fingertree/0.0.1.1/doc/html/Data-FingerTree.html):

```haskell
data FingerTree v a   -- = ...

class Monoid v => Measured a v | a -> v where
    measure :: a -> v
```

What this means is that for a given type `a` we have a way of creating a monoidal summary of its value.  In other words, this `measure` function is quite similar to the first parameter of `foldMap`.  The difference is that since it's in a typeclass, the types themselves specify the function rather than requiring a function to be passed in.

This means that when we create a `FingerTree v a`, where the leaves have type `a` and the inner nodes always cache the monoidal summary (`v`) of that subtree.  As Apfelmus describes, this lets us efficiently search and seek in the tree, by navigating by the cached monoid.

I'll leave my summary of the material at that -- no reason to re-do his excellent article!  However, I would like to throw in a few comments:

1. While the `FingerTree` only supports one monoidal summary, you can easily use the datatype with multiple monoids by relying on the tuple instances of `Monoid`.  For example:

```haskell
-- TODO
```

2. For many reasonable uses of these monoidal annotations, the `measure` above is a ["monoid homomorphism"](http://en.wikibooks.org/wiki/Haskell/Monoids#Homomorphisms).  This is a very fancy way of saying that if the leaf value `a` is also a monoid, then if you `mconcat`-ed the whole sequence together, and applied `measure` to get `v`, you'd get the same result as if you just looked at the `v` annotation at the root.


Diagrams
--------

In his [Monoid Pearl](http://www.cis.upenn.edu/~byorgey/pub/monoid-pearl.pdf), Brent Yorgey describes the usage of monoids in the [diagrams](http://projects.haskell.org/diagrams/) libraries.  A lot of the primitives in the system are monoids:

* Diagrams.  The core representation of diagrams is a monoid where `above <> below` means that `above` is drawn on top of `below`. 
* Envelopes - bounding area of a diagram
* Traces - line intersections with a diagram
* Styles - color / line width / stroke style / etc
* Transformations - stuff like scaling / rotation / shift
* Bounding boxes

One of the core `Diagram` type is really a [DUAL tree](http://hackage.haskell.org/package/dual-tree), a tree that supports both upwards and downwards monoidal annotations.  This means that attributes like styles and transformations can be accumulated downwards, while 

Beyond discussing a very practical yet theory-motivated usage of Monoids, this article is also worth a read for going into the details of "Monoid Actions", "Deep Embeddings", and "Difference Lists".

Here's an example of using the `Monoid` instance for `Diagram`:

![](http://www.mgsloan.com/diagrams_example.png)

```haskell
example = mconcat [ circle 0.1 # fc green
                  , eqTriangle 1 # scale 0.4 # fc yellow
                  , square 1 # fc blue
                  , circle 1 # fc red
                  ]
```

Writer
------

The 'Writer' type, from the ["mtl"](http://hackage.haskell.org/package/mtl) or the preferred ["transformers"](http://hackage.haskell.org/package/transformers) package, allows you to write monadic computations that write down data without reading it.  The usage of "transformers" is the same, but the definition in "mtl" is simpler, so I'll use that for explanation:

```haskell
class (Monoid w, Monad m) => MonadWriter w m | m -> w where
    tell   :: w -> m ()
    listen :: m a -> m (a, w)
    pass   :: m (a, w -> w) -> m a
```

The main reason to use a `Writer` is that you can't arbitrarily set or modify the value of `w`, the accumulator.  You can only use `(<>)` to merge in more information, by using `tell`.  You can still inspect the information (using `listen`), and even apply transformations to it (using `pass`), but these operations must happen as transformations _outside_ the actual monad.  In other words, even with these methods, you can't get an action `foo :: MonadWriter w m => m a` that uses something other than `(<>)` to mutate `w`.

[This](http://blog.sigfpe.com/2009/01/haskell-monoids-and-their-uses.html) excellent monoids tutorial (which covers some similar content) has a good section about using the Writer monad.


Parallelism
-----------

Monoids are a very useful abstraction for parallel programming.  If you are dealing with a monoidal reduction (many "reduces" in map-reduce fit this pattern), then you are allowed to make many different decisions about the division of labor.

```haskell
--TODO
```


http://byorgey.wordpress.com/2011/04/18/monoids-for-maybe/
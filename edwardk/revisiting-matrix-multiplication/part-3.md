# Part III: Extending Vector

As before, if you haven't already, I'd highly recommend reading parts [1](https://www.fpcomplete.com/user/edwardk/revisiting-matrix-multiplication-part-1) and [2](https://www.fpcomplete.com/user/edwardk/revisiting-matrix-multiplication-part-2) before proceeding. That said, if you ignore the motivation for the sorting, this post largely stands alone as a practicum on how to extend the `vector` package with a custom `Vector` type and custom stream fusion combinators.

**All** of the No-Prizes from the previous post are still available! That said, [ssyvlan](http://www.reddit.com/user/ssylvan) on reddit was quite helpful in discussions about how to do the leading zero count more efficiently.

There is only one No-Prize opportunity in this post simply because it otherwise is a fairly simple engineering exercise, and other than that one point I can't think of open questions I have about this code.

Vector
======

Now that we have Morton keys, I finally want to at least be able to _represent_ a sparse matrix.

When I think speed in Haskell, I think `vector`, `repa`, or `accelerate`. Today I'm going to focus on `vector`, because I think it is the easiest to adapt to suit my purposes.

Roman Leshchinskiy, author of `vector` is probably the single most dedicated disciple of speed I can think of in the Haskell community today. I pay lip service to speed, but he'll sit there and spend weeks fixing any single thing that he can't optimize away into a tight c-like loop. If we're going to try to be competitive with a more traditonal language implementation in the end, we'll want to piggyback on his efforts. Any loss of speed due to stupid misuses of his tools, on the other hand, will entirely be my fault.

One of the main things that `vector` gives us is the power of [stream fusion](http://citeseer.ist.psu.edu/viewdoc/summary?doi=10.1.1.104.7401). That is to say that if you build a `Vector` and then immediately consume it, the `vector` library is often smart enough to never bother to construct the intermediate `Vector` at all! As we multiply we'll be merging and concatenating streams of values. It is nice to know we can avoid building huge arrays in the interim.

Again, as I merely pay lip service to the speed God and don't attend services regularly, I'll be going through the motions with a couple of custom stream fusion combinators later on, but I've done rather poor software engineering in that I haven't bothered to check that I needed them. I'd be truly surprised if that turned out not to be the case though.

Sadly, the vocabulary of `Vector` here is somewhat overloaded, so from here out when I talk about a `Vector`, unless I explicitly state otherwise, I mean the concept of `Vector` from `Data.Vector`. If it isn't clear from context then, I'll try to pretend the following imports are in scope:

```active haskell
-- show
import Data.Vector as Boxed
import Data.Vector.Generic as Generic
import Data.Vector.Unboxed as Unboxed
import Data.Vector.Primitive as Primitive
import Data.Vector.Storable as Storable
-- /show

-- show These will also occur in code samples
import qualified Data.Vector.Generic.Mutable as GM
import qualified Data.Vector.Generic as G
import Data.Vector.Fusion.Stream as Stream

-- /show
main = putStrLn "Those modules still exist!"
```

... that way if I talk about an `Unboxed.Vector` it should make some sense.

Sorting out Vectors
===================

The reason I want to use `vector` is that I want my storage to be arranged contiguously in memory whenever possible. Moreover, it enables me to use Dan Doel's excellent `vector-algorithms` to manipulate the the Vector of `(Key,Value)` pairs in our sparse matrices. 

This comes up right away when we start to try to build a sparse matrix, as I want to sort my sparse matrices by `Key` in Morton order, and `vector-algorithms` provides me with a large number of sorting algorithms to experiment with, including the fastest [intro sort](http://hackage.haskell.org/packages/archive/vector-algorithms/0.5.4.2/doc/html/Data-Vector-Algorithms-Intro.html) available in Haskell today. Moreover, it also includes an [american flag](http://hackage.haskell.org/packages/archive/vector-algorithms/0.5.4.2/doc/html/Data-Vector-Algorithms-AmericanFlag.html) sort that could be used to gain asymptotically on the initial insertion by using the fact that we can not only compare but tell you the bit position at which our keys differ, even if we use variable length keys. This means we're not limited to the asymptotics of a `comparison sort` when constructing our initial matrices.

Dan's sorts rely very heavily on aggressive inlining to get worker-wrapper transforms to fire and to ensure that things like the comparisons done inside the sorting routine are being passed unboxed values. 

Using `vector` and `vector-algorithms` leads to our first stumbling block.

What I want to do is store a `Vector (Key, a)` in such a way that I can have unboxed keys. However, I don't want to lose the flexibility to have unboxed values!

To that end, when I started this project about a week ago, I pushed out a [`hybrid-vectors`](http://hackage.haskell.org/package/hybrid-vectors) package to [hackage](http://hackage.haskell.org/packages/hackage.html).

As it'll be the glue that holds together our matrix representation until I find something better, I felt it made sense to say a few words motivating its construction.

Unboxed Vectors
===============

`vector` is already quite smart about managing storage. In particular an unboxed `Vector` of pairs is managed as a pair of vectors of the individual parts. Anybody who has done much GPU work will recognize this as the *structure of arrays* (**SoA**) approach that is often used there rather than *array of structures* (**AoS**) approach used more traditionally throughout the rest of the industry.

In case you've never thought about it before, it is worth mentioning that it has many benefits. One of those benefits is that you don't pay cache storage for parts of the structure you don't look at. You also don't incur unnecessary time/space trade-offs for alignment issues, etc. The cache dominates most concerns about slightly more complicated addressing logic, and if you look at things like the x86 instruction set, the addressing logic usually simplifies as well! We can calculate the `\*{1,2,4,8}` multiplier in the index _en passant_, but the odd sizes you get in "array of structures" are usually not so easy to use.

All of this is well and good, but I can't just use [`Data.Vector.Unboxed`](http://hackage.haskell.org/packages/archive/vector/0.10.0.1/doc/html/Data-Vector-Unboxed.html).

My particular motivation for working on this code in the first place requires me to be able to do matrix multiplication over certain fairly complicated ring-like structures that I can't [`Unbox`](http://hackage.haskell.org/packages/archive/vector/0.10.0.1/doc/html/Data-Vector-Unboxed.html#t:Unbox). I say "ring-like" because right seminearrings without [zero-divisors](http://en.wikipedia.org/wiki/Zero_divisor) arise in various forms of [chart parsing](http://en.wikipedia.org/wiki/Chart_parser), and [some](http://www.cse.chalmers.se/~bernardy/PP.pdf) are even [non-associative](http://en.wikipedia.org/wiki/Nonassociative_ring)!

Hybrid Vectors
==============

Fortunately, in a great feat of engineering, Roman left the entire `Vector` framework open to extension with new `Vector` types, by providing us with [`Data.Vector.Generic`](http://hackage.haskell.org/packages/archive/vector/0.10.0.1/doc/html/Data-Vector-Generic.html) and moreover, he made it so that the entire stream fusion framework he uses will just magically work with any instance of [`Generic.Vector`](http://hackage.haskell.org/packages/archive/vector/0.10.0.1/doc/html/Data-Vector-Generic.html#t:Vector).

So all we need to do is define our own `Vector` (and `MVector`) type for the kinds of "hybrid" vectors we need.

Then Dan's fast sorting algorithms can be used out of the box and we can steal and extend Roman's fusion framework.

`Data.Vector.Unboxed` has already shown us the way to build an SoA-style vector. We just need it to be more permissive about what kind of `Generic.Vector` vectors are allowed to comprise each side.

To define a `Generic.Vector`, first we must define `Generic.MVector` that'll be used for most operations involved in building or manipulating it behind the scenes.

```active haskell
{-# LANGUAGE CPP #-}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE KindSignatures #-}
{-# LANGUAGE GADTs #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE UndecidableInstances #-}
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE DeriveDataTypeable #-}
{-# LANGUAGE ScopedTypeVariables #-}

import Control.Monad
import Data.Monoid
import qualified Data.Vector.Generic.Mutable as GM
import qualified Data.Vector.Generic as G
import Data.Vector.Fusion.Stream as Stream
import Data.Data
import Prelude hiding ( length, null, replicate, reverse, map, read, take, drop, init, tail )
import Text.Read
-- show
data MVector :: (* -> * -> *) -> (* -> * -> *) -> * -> * -> * where
  MV :: !(u s a) -> !(v s b) -> MVector u v s (a, b)

instance (GM.MVector u a, GM.MVector v b) => GM.MVector (MVector u v) (a, b) where
  basicLength (MV ks _) = GM.basicLength ks
  basicUnsafeSlice s e (MV ks vs) = MV (GM.basicUnsafeSlice s e ks) (GM.basicUnsafeSlice s e vs)
  basicOverlaps (MV ks vs) (MV ks' vs') = GM.basicOverlaps ks ks' || GM.basicOverlaps vs vs'
  basicUnsafeNew n = liftM2 MV (GM.basicUnsafeNew n) (GM.basicUnsafeNew n)
  basicUnsafeReplicate n (k,v) = liftM2 MV (GM.basicUnsafeReplicate n k) (GM.basicUnsafeReplicate n v)
  basicUnsafeRead (MV ks vs) n = liftM2 (,) (GM.basicUnsafeRead ks n) (GM.basicUnsafeRead vs n)
  basicUnsafeWrite (MV ks vs) n (k,v) = GM.basicUnsafeWrite ks n k >> GM.basicUnsafeWrite vs n v
  basicClear (MV ks vs) = GM.basicClear ks >> GM.basicClear vs
  basicSet (MV ks vs) (k,v) = GM.basicSet ks k >> GM.basicSet vs v
  basicUnsafeCopy (MV ks vs) (MV ks' vs') = GM.basicUnsafeCopy ks ks' >> GM.basicUnsafeCopy vs vs'
  basicUnsafeMove (MV ks vs) (MV ks' vs') = GM.basicUnsafeMove ks ks' >> GM.basicUnsafeMove vs vs'
  basicUnsafeGrow (MV ks vs) n = liftM2 MV (GM.basicUnsafeGrow ks n) (GM.basicUnsafeGrow vs n)
-- /show
main = putStrLn "It typechecks, so it must be correct."
```

All we've done is say that our `Hybrid.MVector u v s (a,b)` is going to be comprised (strictly!) of two other vectors `u s a` and `v s b` so long as we have appropriate `Generic.MVector` instances to rely upon, and so long as they can both be manipulated in the same [`PrimMonad`](http://hackage.haskell.org/packages/archive/primitive/0.5.0.1/doc/html/Control-Monad-Primitive.html#t:PrimMonad).

All we've done is borrow from similar operations from the vectors that comprise our constituent parts.

In the real code we then proceed to `INLINE` everything in sight, so that any inlined combinators from `Data.Vector.Generic` can "see through" our `instance` and `INLINE` the bodies of our methods.

We can construct a custom `Generic.Vector` similarly.

```active haskell
{-# START_FILE Hybrid.hs #-}
{-# LANGUAGE CPP #-}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE KindSignatures #-}
{-# LANGUAGE GADTs #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE UndecidableInstances #-}
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE DeriveDataTypeable #-}
{-# LANGUAGE ScopedTypeVariables #-}
module Hybrid where

import Control.Monad
import Data.Monoid
import qualified Data.Vector.Generic.Mutable as GM
import qualified Data.Vector.Generic as G
import Data.Vector.Fusion.Stream as Stream
import Data.Data
import Prelude hiding ( length, null, replicate, reverse, map, read, take, drop, init, tail )
import Text.Read

data MVector :: (* -> * -> *) -> (* -> * -> *) -> * -> * -> * where
  MV :: !(u s a) -> !(v s b) -> MVector u v s (a, b)

instance (GM.MVector u a, GM.MVector v b) => GM.MVector (MVector u v) (a, b) where
  basicLength (MV ks _) = GM.basicLength ks
  basicUnsafeSlice s e (MV ks vs) = MV (GM.basicUnsafeSlice s e ks) (GM.basicUnsafeSlice s e vs)
  basicOverlaps (MV ks vs) (MV ks' vs') = GM.basicOverlaps ks ks' || GM.basicOverlaps vs vs'
  basicUnsafeNew n = liftM2 MV (GM.basicUnsafeNew n) (GM.basicUnsafeNew n)
  basicUnsafeReplicate n (k,v) = liftM2 MV (GM.basicUnsafeReplicate n k) (GM.basicUnsafeReplicate n v)
  basicUnsafeRead (MV ks vs) n = liftM2 (,) (GM.basicUnsafeRead ks n) (GM.basicUnsafeRead vs n)
  basicUnsafeWrite (MV ks vs) n (k,v) = do
    GM.basicUnsafeWrite ks n k
    GM.basicUnsafeWrite vs n v
  basicClear (MV ks vs) = do
    GM.basicClear ks
    GM.basicClear vs
  basicSet (MV ks vs) (k,v) = do
    GM.basicSet ks k
    GM.basicSet vs v
  basicUnsafeCopy (MV ks vs) (MV ks' vs') = do
    GM.basicUnsafeCopy ks ks'
    GM.basicUnsafeCopy vs vs'
  basicUnsafeMove (MV ks vs) (MV ks' vs') = do
    GM.basicUnsafeMove ks ks'
    GM.basicUnsafeMove vs vs'
  basicUnsafeGrow (MV ks vs) n = liftM2 MV (GM.basicUnsafeGrow ks n) (GM.basicUnsafeGrow vs n)

  {-# INLINE basicLength #-}
  {-# INLINE basicUnsafeSlice #-}
  {-# INLINE basicOverlaps #-}
  {-# INLINE basicUnsafeNew #-}
  {-# INLINE basicUnsafeReplicate #-}
  {-# INLINE basicUnsafeRead #-}
  {-# INLINE basicUnsafeWrite #-}
  {-# INLINE basicClear #-}
  {-# INLINE basicSet #-}
  {-# INLINE basicUnsafeCopy #-}
  {-# INLINE basicUnsafeMove #-}
  {-# INLINE basicUnsafeGrow #-}

-- show
data Vector :: (* -> *) -> (* -> *) -> * -> * where
  V :: !(u a) -> !(v b) -> Vector u v (a, b)

type instance G.Mutable (Vector u v) = MVector (G.Mutable u) (G.Mutable v)

instance (G.Vector u a, G.Vector v b) => G.Vector (Vector u v) (a, b) where
  basicUnsafeFreeze (MV ks vs) = liftM2 V (G.basicUnsafeFreeze ks) (G.basicUnsafeFreeze vs)
  basicUnsafeThaw (V ks vs) = liftM2 MV (G.basicUnsafeThaw ks) (G.basicUnsafeThaw vs)
  basicLength (V ks _) = G.basicLength ks
  basicUnsafeSlice i j (V ks vs) = V (G.basicUnsafeSlice i j ks) (G.basicUnsafeSlice i j vs)
  basicUnsafeIndexM (V ks vs) n = liftM2 (,) (G.basicUnsafeIndexM ks n) (G.basicUnsafeIndexM vs n)
  basicUnsafeCopy (MV ks vs) (V ks' vs') = do
    G.basicUnsafeCopy ks ks'
    G.basicUnsafeCopy vs vs'
  elemseq (V ks vs) (k,v) b = G.elemseq ks k (G.elemseq vs v b)
-- /show
  {-# INLINE basicUnsafeFreeze #-}
  {-# INLINE basicUnsafeThaw #-}
  {-# INLINE basicLength #-}
  {-# INLINE basicUnsafeSlice #-}
  {-# INLINE basicUnsafeIndexM #-}
  {-# INLINE basicUnsafeCopy #-}
  {-# INLINE elemseq #-}

instance (G.Vector u a, G.Vector v b, c ~ (a, b)) => Monoid (Vector u v c) where
  mappend = (G.++)
  {-# INLINE mappend #-}
  mempty = G.empty
  {-# INLINE mempty #-}
  mconcat = G.concat
  {-# INLINE mconcat #-}

instance (G.Vector u a, G.Vector v b, Show a, Show b, c ~ (a, b)) => Show (Vector u v c) where
  showsPrec = G.showsPrec

instance (G.Vector u a, G.Vector v b, Read a, Read b, c ~ (a, b)) => Read (Vector u v c) where
  readPrec = G.readPrec
  readListPrec = readListPrecDefault
  
instance (G.Vector u a, G.Vector v b, Eq a, Eq b, c ~ (a, b)) => Eq (Vector u v c) where
  xs == ys = Stream.eq (G.stream xs) (G.stream ys)
  xs /= ys = not (Stream.eq (G.stream xs) (G.stream ys))
  {-# INLINE (==) #-}
  {-# INLINE (/=) #-}

instance (G.Vector u a, G.Vector v b, Ord a, Ord b, c ~ (a, b)) => Ord (Vector u v c) where
  compare xs ys = Stream.cmp (G.stream xs) (G.stream ys)
  {-# INLINE compare #-}

{-# START_FILE Main.hs #-}

module Main where
import Hybrid
import Data.Vector.Unboxed as Unboxed
import Data.Vector
import Data.Vector as Boxed
import Data.Vector.Generic as Generic
import Data.Vector.Unboxed as Unboxed
import Data.Vector.Primitive as Primitive
import Data.Vector.Storable as Storable
import qualified Data.Vector.Generic.Mutable as GM
import qualified Data.Vector.Generic as G
import Data.Vector.Fusion.Stream as Stream

-- show
type V = Hybrid.Vector Unboxed.Vector Boxed.Vector (Int,Integer)
main = do
  print (G.fromList [(1,2),(3,4)] :: V) 
  print $ G.slice 2 3 (G.fromList (Prelude.zip [1..10] [11..20]) :: V)
-- /show
```

It is worth taking a couple of minutes to talk about the instances, though.

`Generic.Vector` types typically provide the obvious `Monoid`, `Show`, `Read`, `Eq`, and `Ord` instances we've come to expect.

But if we were not careful and wrote

```haskell
instance (G.Vector u a, G.Vector v b) => Monoid (Vector u v (a,b))
```

then the compiler would be unduly reticent to use the instance. That is, it won't realize that the moment it can see that we have a `Hybrid.Vector`, that the argument must be a pair as, after all, our GADT only has one constructor!

So instead we need to write the remaining instances to abuse the power of [system Fc](http://research.microsoft.com/en-us/um/people/simonpj/papers/ext-f/) and modern Haskell to say that once the compiler figures out it needs one of these instances for a `Hybrid.Vector u v c` it can infer `c` must be of the form `(a,b)` by writing these instances as

```haskell
instance (G.Vector u a, G.Vector v b, c ~ (a, b)) => Monoid (Vector u v c) where
  mappend = (G.++)
  mempty = G.empty
  mconcat = G.concat
```

```haskell
instance (G.Vector u a, G.Vector v b, Show a, Show b, c ~ (a, b)) => Show (Vector u v c) where
  showsPrec = G.showsPrec
```

```haskell
instance (G.Vector u a, G.Vector v b, Read a, Read b, c ~ (a, b)) => Read (Vector u v c) where
  readPrec = G.readPrec
  readListPrec = readListPrecDefault
```

With `Eq` and `Ord` we need to lean a little bit on the fusion machinery, just because no such defaults are provided for us.

```haskell
instance (G.Vector u a, G.Vector v b, Eq a, Eq b, c ~ (a, b)) => Eq (Vector u v c) where
  xs == ys = Stream.eq (G.stream xs) (G.stream ys)
  xs /= ys = not (Stream.eq (G.stream xs) (G.stream ys))
```

```haskell
instance (G.Vector u a, G.Vector v b, Ord a, Ord b, c ~ (a, b)) => Ord (Vector u v c) where
  compare xs ys = Stream.cmp (G.stream xs) (G.stream ys)
```

Now, we finally have everything we need for our `Hybrid.Vector` type to feel like a real `Vector`.

OK, not quite *everything*, for reasons that I'm sure make sense to Roman, he goes and duplicates the entire `vector` API individually customized to each vector subtype within their own modules. Admittedly, it does help with inference in the presence of such an all encompassing API that can both produce and consume values of the same type. To that end, `hybrid-vectors` includes a couple thousand lines of boilerplate as well to make a `vector` programmer feel at home. It also contains another, related notion, that of a [`Mixed.Vector`](http://hackage.haskell.org/packages/archive/hybrid-vectors/0.1/doc/html/Data-Vector-Mixed.html) that permits more operations to cooperate across vector types, as it was something that was easy to implement while I had all of the `vector` internals paged in mentally.

However, the details of that implementation is entirely mechanical.

Similarly, we need to go through a similar boilerplate exercise making an instance of `Unbox` for `Key`, but nothing new is learned.

Custom Stream Fusion
====================

Before we move on, we are going to need at least one custom stream fusion combinator for adding two matrices together. 

I don't want to pre-judge that all addition will fall under the purview of the `Num` typeclass, as for instance since we're sparse we can use `Mat Unboxed.Vector ()` as a Boolean matrix and pay surprisingly little overhead as an `Unboxed.Vector` of `()`s is represented simply by its `size`!

Consequently, let's define a generalized merge operation that can permit values to cancel.

```haskell
mergeStreamsWith 
  :: (Monad m, Ord i) 
  => (a -> a -> Maybe a) 
  -> Stream m (i, a)
  -> Stream m (i, a)
  -> Stream m (i, a)
```

In `Vector`'s current version of monadic stream fusion, a `Stream` consists of a step function, a state, and any knowledge we have about the `Size` of the `Stream`.

```haskell
data Stream m a = forall s . Stream (s -> m (Step s a)) s Size	 
```

Therefore with an appropriate new `step` function, we can write:

```haskell
mergeStreamsWith f (Stream stepa sa0 na) (Stream stepb sb0 nb)
  = Stream step (MergeStart sa0 sb0) (toMax na + toMax nb)
```

Here our starting state `MergeStart` is described below, and we convert the bounds we know on the input streams into a conservative upper bound on our new stream size.

Stream fusion works by ensuring that each `Step` you take does not recurse, so the compiler is able to move all of the iteration to one outer-most loop.

At each `Step`, we can `Yield` a new answer and switch to a new state, simply switch to a new state or terminate our `Stream`. 

```haskell
data Step s a = Yield a s | Skip s | Done	
```

In our case we need to consume from two other streams, but we can only afford to ask for a bounded amount of work from each at each `Step`. So let's build a custom state type for our `Stream`:

```haskell
data MergeState sa sb i a
  = MergeL sa sb i a
  | MergeR sa sb i a
  | MergeLeftEnded sb
  | MergeRightEnded sa
  | MergeStart sa sb
```

We'll want to reason through our `step` function by cases as it is rather tedious.

`MergeStart` is our initial state, consisting of the initial states of both of our input streams.

```haskell
  step (MergeStart sa sb) = do
    r <- stepa sa
    return $ case r of
      Yield (i, a) sa' -> Skip (MergeL sa' sb i a)
      Skip sa'         -> Skip (MergeStart sa' sb)
      Done             -> Skip (MergeLeftEnded sb)
```

We try to read from the left stream. If it succeeds we know the value from the left stream (and are in state `MergeL`). If it skips we have to try again. If it says it has no more content, we should fast forward through the right hand stream with `MergeLeftEnded`.

The rest of the logic follows similarly.

Once we have a value from our left `Stream`, we should try to read from our right, merging values if their indices are equal, and otherwise putting them in order. Depending on which candidate was merged, we proceed to `MergeR` or `MergeL`. If during the `Merge`, we determine that our elements cancelled out, e.g. `5 + (-5)`, thn rather than recurse into start to ensure a bounded amount of work is done, we `Skip` back there.

```haskell
  step (MergeL sa sb i a) = do
    r <- stepb sb
    return $ case r of
      Yield (j, b) sb' -> case compare i j of
        LT -> Yield (i, a)     (MergeR sa sb' j b)
        EQ -> case f a b of
           Just c  -> Yield (i, c) (MergeStart sa sb')
           Nothing -> Skip (MergeStart sa sb')
        GT -> Yield (j, b)     (MergeL sa sb' i a)
      Skip sb' -> Skip (MergeL sa sb' i a)
      Done     -> Yield (i, a) (MergeRightEnded sa)
```

`MergeR` is entirely symmetric to `MergeL`:

```haskell
  step (MergeR sa sb j b) = do
    r <- stepa sa
    return $ case r of
      Yield (i, a) sa' -> case compare i j of
        LT -> Yield (i, a)     (MergeR sa' sb j b)
        EQ -> case f a b of
          Just c  -> Yield (i, c) (MergeStart sa' sb)
          Nothing -> Skip (MergeStart sa' sb)
        GT -> Yield (j, b)     (MergeL sa' sb i a)
      Skip sa' -> Skip (MergeR sa' sb j b)
      Done     -> Yield (j, b) (MergeLeftEnded sb)
```

and then we just have to deal with fast forwarding:

```haskell
  step (MergeLeftEnded sb) = do
    r <- stepb sb
    return $ case r of
      Yield (j, b) sb' -> Yield (j, b) (MergeLeftEnded sb')
      Skip sb'         -> Skip (MergeLeftEnded sb')
      Done             -> Done
  step (MergeRightEnded sa) = do
    r <- stepa sa
    return $ case r of
      Yield (i, a) sa' -> Yield (i, a) (MergeRightEnded sa')
      Skip sa'         -> Skip (MergeRightEnded sa')
      Done             -> Done
```

Putting it all together we get:

```active haskell
-- show
import Data.Vector.Fusion.Stream.Monadic (Step(..), Stream(..))
import Data.Vector.Fusion.Stream.Size

data MergeState sa sb i a
  = MergeL sa sb i a
  | MergeR sa sb i a
  | MergeLeftEnded sb
  | MergeRightEnded sa
  | MergeStart sa sb

mergeStreamsWith
  :: (Monad m, Ord i) 
  => (a -> a -> Maybe a) 
  -> Stream m (i, a) 
  -> Stream m (i, a) 
  -> Stream m (i, a)
mergeStreamsWith f (Stream stepa sa0 na) (Stream stepb sb0 nb)
  = Stream step (MergeStart sa0 sb0) (toMax na + toMax nb) where
  step (MergeStart sa sb) = do
    r <- stepa sa
    return $ case r of
      Yield (i, a) sa' -> Skip (MergeL sa' sb i a)
      Skip sa'         -> Skip (MergeStart sa' sb)
      Done             -> Skip (MergeLeftEnded sb)
  step (MergeL sa sb i a) = do
    r <- stepb sb
    return $ case r of
      Yield (j, b) sb' -> case compare i j of
        LT -> Yield (i, a)     (MergeR sa sb' j b)
        EQ -> case f a b of
           Just c  -> Yield (i, c) (MergeStart sa sb')
           Nothing -> Skip (MergeStart sa sb')
        GT -> Yield (j, b)     (MergeL sa sb' i a)
      Skip sb' -> Skip (MergeL sa sb' i a)
      Done     -> Yield (i, a) (MergeRightEnded sa)
  step (MergeR sa sb j b) = do
    r <- stepa sa
    return $ case r of
      Yield (i, a) sa' -> case compare i j of
        LT -> Yield (i, a)     (MergeR sa' sb j b)
        EQ -> case f a b of
          Just c  -> Yield (i, c) (MergeStart sa' sb)
          Nothing -> Skip (MergeStart sa' sb)
        GT -> Yield (j, b)     (MergeL sa' sb i a)
      Skip sa' -> Skip (MergeR sa' sb j b)
      Done     -> Yield (j, b) (MergeLeftEnded sb)
  step (MergeLeftEnded sb) = do
    r <- stepb sb
    return $ case r of
      Yield (j, b) sb' -> Yield (j, b) (MergeLeftEnded sb')
      Skip sb'         -> Skip (MergeLeftEnded sb')
      Done             -> Done
  step (MergeRightEnded sa) = do
    r <- stepa sa
    return $ case r of
      Yield (i, a) sa' -> Yield (i, a) (MergeRightEnded sa')
      Skip sa'         -> Skip (MergeRightEnded sa')
      Done             -> Done
  {-# INLINE [0] step #-}
{-# INLINE [1] mergeStreamsWith #-}
-- /show
main = putStrLn "That compiles, too."
```

It is worth noting the phase-controlled `INLINE` pragmas in the final definition as well. Without them you really won't see much benefit from stream fusion! They ensure that the compiler is careful to hold onto the steps uninlined until the right moment.

> **No-Prize #6** 
> 
> It'd probably be much more efficient for us to us an efficient `k-merge` or, even better, a cache-oblivious
> variant that is careful to make no assumptions about our caches. The only two operations we'll be using
> `Stream` fusion for are concatenation and `mergeStreamsWith` for now. Is there a more efficient fusion form we should be using that can exploit this structure?

Matrices, not Vectors
=====================

Now that we have `hybrid-vectors`, we can _finally_ peek at what our matrix representation can look like.

```haskell
newtype Mat v a = Mat { runMat :: Hybrid.Vector Unboxed.Vector v (Key, a) }
```

and how we could build one:

```haskell
fromList :: Generic.Vector v a => [(Key, a)] -> Mat v a
fromList xs 
  = Mat 
  $ Generic.modify (Intro.sortBy (compare `on` fst)) 
  $ Generic.fromList xs
```

As we'll see next time don't need type parameters for the number of dimensions and we don't need to store information on the dimensionality of the contents as well as we'll be able to turn back to the notions of "most significant most significant difference" and 2-fattest numbers a second time. 

And you'd hoped the bit-twiddling was behind us. Hah!

-[Edward Kmett](mailto:ekmett@gmail.com)

August 18, 2013
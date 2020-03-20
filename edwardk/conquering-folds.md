# Conquering Folds

I've been posting a lot about comonads lately, mostly for working with folds of various flavors over lists.

This time I want to take a different tack and use the tools we've been building to perform certain computations over arbitrary polynomial data types in parallel with arbitrary cuts of the data potentially distributed on different hosts.

Close Encounters of the Third Kind
==================================

The techniques I want to use here derive from an old "folklore" tool in the constructive algorithmics community known as the ["Third Homomorphism Theorem"](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.45.2247&rep=rep1&type=pdf), which was written up back in 1995 by [Jeremy Gibbons](http://www.cs.ox.ac.uk/people/jeremy.gibbons/), but which dates back further to a conjecture by [Richard Bird](http://web.comlab.ox.ac.uk/oucl/people/richard.bird.html) that was proven by a bored [Lambert Meertens](http://www.kestrel.edu/home/people/meertens/) on a train ride through the Netherlands in 1989.

<img src="http://ekmett.github.io/images/closeencounters1.jpg">

The Third Homomorphism Theorem basically states that if you can compute something as both a left fold and a right fold then you can compute it monoidally.

It is wrapped up in slightly different vocabulary, and various treatments often forget the unit for the monoid or work with what would be non-empty lists. The terminology in the papers that talk about it focus on list homomorphisms, which are just monoid homomorphisms from the list monoid.

The Third Homomorphism Theorem also provides a not-so-efficient way to derive such a `Monoid`, that you can then hopefully improve by reasoning about your special case. In Gibbons' paper on the topic, he derives _O(n log n)_ `mergesort` from the _O(n^2)_ `insertsort` via the sort of equational reasoning that is the hallmark of the constructive algorithmics style.

Unfortunately, you _do_ have to reason about the particular list homomorphism to make it useful, as the result is generally not terribly efficient without some effort.

Sadly, the Third Homomorphism Theorem itself isn't terribly useful as an automated parallelization technique. There has actually been some progress in that area in terms of parallel skeletons, but it has been a pretty slow couple of decades.

If we ignore the particulars of how the theorem itself is used, and just focus on what it means, then it does help indicate that something like `foldMap` is a potentially powerful tool, but we've pretty much accepted that by now.

In addition, the original Gibbons' paper linked above also supplied most of the examples in Conal Elliott's 2011 series on ["Deriving List Scans"](http://conal.net/blog/posts/deriving-list-scans) about 16 years earlier, and is generally good reading, so it is a good way to get your head around that style of equational reasoning.

Beyond Lists
============

But the Third Homomorphism Theorem is of course all _about_ lists, so how can we use it to manipulate other things?

This topic was broached by Morihata et al. in  ["The Third Homomorphism Theorem on Trees"](http://www.prg.nii.ac.jp/publications/2009/popl09.pdf), which noted that [Gérard Huet's notion of a zipper](http://www.st.cs.uni-saarland.de/edu/seminare/2005/advanced-fp/docs/huet-zipper.pdf) is just a _list_ of steps, and so we can apply the third _list_ homomorphism theorem to it.

So if you can compute something that can push information down the tree, and which can push information up the tree, then you can derive a list homomorphism that you can use on the zipper itself, and then you can subdivide your tree however you want, and compute partial answers that you can stitch together.

So, let's do that!

[Clowns And Jokers](http://www.youtube.com/watch?v=8StG4fFWHqg)
==================

Conor McBride wrote the first section of the wonderfully-titled ["Clowns to the left of me, jokers to the right"](http://strictlypositive.org/Dissect.pdf) on how one can compute the derivatives of a polynomial functor and use it as a one hole context. 

*Edit:* It was pointed out to me on reddit that perhaps I should have cited the more directly relevant [The Derivative of a Regular Type is its Type of One-Hole Contexts](http://strictlypositive.org/diff.pdf) instead.

My first attempt at writing this up actually used GHC.Generics to derive the zippers automatically, based on Conor's formulae. Unfortunately, the automatically generated zippers aren't really fit for human consumption.

Consequently, I'm going to be asking the user to describe their derivatives manually, and I'll try to keep the set of steps minimal.

In particular, I'm going to be working with some kind of recursive data type `t`, for which I'll assume it can be presented as the fixed point of some base functor `f`. I'll never actually need to use `f` directly, but when I talk about the "derivative" `D t` of `t`, I'll actually mean the derivative of the base functor for `t`.

The derivative of a polynomial functor is also polynomial, so we can expect `D t` to be `Traversable`.

```haskell
class Recursive t where
  data D t :: * -> *
  walk   :: Applicative f => (a -> f b) -> D t a -> f (D t b)
  ...
  
instance Recursive t => Functor (D t) where
  fmap = fmapDefault

instance Recursive t => Foldable (D t) where
  foldMap = foldMapDefault

instance Recursive t => Traversable (D t) where
  traverse = walk
```

But before we continue too far we'll need to add a couple of additional methods to `Recursive`.

If we have a derivative of `t`, then it is a 'one hole context'. It is one level of the base functor of `t`, wrapped around `t`s, with one of the `t`s ripped out. We want some way to `plug` that hole:

```haskell
class Recursive t where
  ...
  plug   :: D t t -> t -> t
```

We also need some way to pick a path through our `t`.

For simplicity and to make the tools I want to build later easier to work with using standard Haskell classes I'm going to assume that you can either find a hole and name its contents or we've reached a trivial leaf. 

```haskell
class Recursive t where
  ...
  path :: t -> Maybe (D t t, t)
```

For now I'm just going to assume you can pick a path. Perhaps it'd be more interesting later on to try to choose a better path when we talk about parallelism. e.g. using [heavy path decomposition](http://en.wikipedia.org/wiki/Heavy_path_decomposition) or [long path decomposition](http://courses.csail.mit.edu/6.897/spring05/lec/lec19.pdf), but I'll leave that to you to play with.

Putting it all together we have:

```haskell
class Recursive t where
  data D t :: * -> *
  plug :: D t t -> t -> t
  walk :: Applicative f => (a -> f b) -> D t a -> f (D t b)
  path :: t -> Maybe (D t t, t)
```

This is enough that we can peel apart our data type into a zipper. Remember we asserted that all the leaves were boring, there isn't any leaf value type to go with the zipper.

```haskell
type Z t = [D t t]

divide :: Recursive t => t -> Z t
divide = unfoldr path
```

If on the other hand we do have something interesting that we want to use to fill in a hole, we can plug it in by recursively folding our way up the tree.

```haskell
zipped :: Recursive t => Z t -> t -> t
zipped dt t = Prelude.foldr plug t dt
```

Example: Nats
======================

If we take a look at the basic Peano-style natural number type in Haskell:

```haskell
data Nat = Z | S Nat
```

We can talk about the type of one-hole contexts there are in its base functor, and it is easy to plug it back up, since there is only one case. Once we take the only hole for our one-hole context, there aren't any uses of the type parameter left.

Similarly the `path` only has one option for us at every turn.

```haskell
instance Recursive Nat where
  data D Nat a = Succ
  plug Succ = S
  walk _ Succ = pure Succ
  path (S n) = Just (Succ, n)
  path Z     = Nothing
```

I'll throw a few more instances up later as we get to where we can use them.

Needless to say I'm not going to get a lot of parallelism out of a `Nat`.


Conquering Reflection
=====================

So now we're ready to actually build our divide-and-conquer `Comonad`.

```haskell
data Conquer t a = forall b. Monoid b => Conquer (b -> a) (D t b -> b)
```

What we're saying is, if you can reduce one level of the zipper's path to a monoidal result, where you've already reduced all of the other parts to the monoid, we can smash together all the answers and use the final calculation to give you an 'a' for your overall tree.


If you were paying attention last time at the end of my post on parallelizing CRC calculations, this may look somewhat familiar, except for the fact that I've given in to peer pressure and decided to use an actual `Monoid`, and the input type now references the same `Monoid`.

We can still use `reflection` to manufacture the `Monoid` if needed, without the user ever being any the wiser:

```haskell
newtype M a p = M { runM :: a }

instance Reifies p (a -> a -> a, a)  => Monoid (M a p) where
  mempty = M $ snd $ reflect (Proxy :: Proxy p)
  mappend (M a) (M b) = M $ fst (reflect (Proxy :: Proxy p)) a b

conquer :: forall t a b. Recursive t => 
  (b -> a) -> (D t b -> b) -> (b -> b -> b) -> b -> Conquer t a
conquer k h m z = reify (m,z) $ \(Proxy :: Proxy p) -> Conquer
   (k . runM :: M b p -> a) (M . h . fmap runM)
```

On the Run
==========

`Conquer` describes a calculation that we expect to be able to work with both up or down the paths of the tree. Morihata et al would probably call it a path-based decomposition. However, this presentation diverges a bit from theirs to let me vary the result type covariantly and to match up with the other foldings I've been talking about lately.

Now that we can build one, we should figure out how to apply it:

```haskell
run :: Recursive t => Conquer t a -> t -> a
run (Conquer k h) = k . go where
  go = foldMap (h . fmap go) . divide
```

To run our decomposition, we first take the input and divide it by pulling down the zipper. Then we run down the zipper recursively transforming each of the other entries in the context using the same approach, then applying (h :: D t b -> b) to the intermediate results. When we're all done, we polish up with whatever function we wanted to use to make the result presentable to the end-user.


[Comonad and Conquer](http://www.commandandconquer.com/en)
===================

`Conquer` admits many of the same instances as `M` and `L` from the `folds` package. One of the main reasons is that we assumed that the leaf was trivial. We lose a lot of structure if we don't.

```haskell
data Pair a b = Pair { pi1 :: !a, pi2 :: !b }

instance (Monoid a, Monoid b) => Monoid (Pair a b) where
  mempty = Pair mempty mempty
  mappend (Pair a b) (Pair c d) = Pair (mappend a c) (mappend b d)

instance Functor (Conquer t) where
  fmap f (Conquer k h) = Conquer (f.k) h
```

The `Comonad` is basically identical to the one for `M`.

```haskell
instance Comonad (Conquer t) where
  extract (Conquer k h) = k mempty
  duplicate (Conquer k h) = Conquer (\b -> Conquer (k . mappend b) h) h
```

And we get a nice `Applicative` that can be used to fuse together multiple passes:

```haskell
instance Recursive t => Applicative (Conquer t) where
  pure a = Conquer (const a) (const ())
  Conquer kf hf <*> Conquer ka ha = Conquer
    (\(Pair bf ba) -> kf bf (ka ba))
    (\dab -> Pair (hf (pi1 <$> dab)) (ha (pi2 <$> dab)))

instance Recursive t => ComonadApply (Conquer t) where
  (<@>) = (<*>)
```

We can even get a `Monad` for rather inefficient multipass folding.

Now, it isn't immediately obvious which way the `Monoid` is used when you `duplicate`, so lets define a couple of helper functions:

```haskell
above :: Recursive t => Z t -> Conquer t a -> Conquer t a
above t0 (Conquer k h) = let x = go t0 in Conquer (k . mappend x) h where
  go = foldMap (h . fmap (go . divide))

below :: Recursive t => Conquer t a -> Z t -> Conquer t a
below (Conquer k h) t0 = let y = go t0 in Conquer (\x -> k (mappend x y)) h where
  go = foldMap (h . fmap (go . divide))
```

These can be used to "pre-apply" our divide-and-conquer comonad to a prefix of our zipper `above` or `below` the rest of what we want to feed the fold without leaking space or holding onto anything unnecessarily.

Example: Folding Lists
==============

```haskell
instance Recursive [a] where
  data D [a] b = Cons { getCons :: a} deriving (Show,Read)
  plug (Cons a) as = a : as
  walk _ = pure . Cons . getCons
  path (x:xs)   = Just (Cons x, xs)
  path []       = Nothing

sumList :: Conquer [Int] Int
sumList = Conquer getSum (Sum . getCons)
```

With that we can run the summation over a list.

```haskell
>>> run sumList [1,2,3]
6
```

and we can also partially apply it 

```haskell
>>> run (run (duplicate sumList) [1,2,3]) [4,5]
15
```

If we have a zipper lying around with the rest of what we want cut out, we can pre-apply it:

```
>>> run (above (divide [1,2,3]) sumList) [4,5]
15
```

Putting it to Work
==================

It isn't immediately obvious that there are many algorithms that you can resolve this way.

Most of the time when we're talking about tree calculations we're talking about some kind of synthesized or inherited attribute. So what kinds of things can be implemented both ways?

Let's move to another example, so we can finally get some interesting structure:

Example: Folding Trees
=============

<img src="http://ekmett.github.io/images/folding-trees.jpg">

```haskell
data Tree a
  = Bin (Tree a) a (Tree a)
  | Tip deriving (Show,Read)

instance Recursive (Tree a) where
  data D (Tree a) b = L a b | R b a deriving (Show,Read)

  plug (L a r) l = Bin l a r
  plug (R l a) r = Bin l a r

  walk f (L a b) = L a <$> f b
  walk f (R b a) = (`R` a) <$> f b

  path (Bin l a r) = Just (L a r, l)
  path Tip = Nothing
```

`path` has to pick something, so here we just lean to the left.


Analogous to our `sumList`, we can define

```haskell
sumTree :: Num a => Conquer (Tree a) a
sumTree = Conquer getSum $ \ case
  L a (Sum b) -> Sum (a + b)
  R (Sum b) a -> Sum (a + b)
```

```haskell
>>> run sumTree (Bin (Bin Tip 1 Tip) 2 Tip)
3
```

To give ourselves some non-trivial examples, lets define

```haskell
unfoldTree :: (s -> Maybe (s, a, s)) -> s -> Tree a
unfoldTree f s = case f s of
  Nothing -> Tip
  Just (l, a, r) -> Bin (unfoldTree f l) a (unfoldTree f r)
```

To help cut them off:

```haskell
takeTree :: Int -> Tree a -> Tree a
takeTree 0 _   = Tip
takeTree n Tip = Tip
takeTree n (Bin l a r) = Bin (takeTree (n-1) l) a (takeTree (n-1) r)
```

Now we can define a nice infinite tree that contains all of the (positive) rationals.

```haskell
sternBrocot :: Tree Rational
sternBrocot = unfoldTree 
  (\(Pair a b) -> Just (Pair (a+1) b, a % b, Pair a (b+1))) 
  (Pair 1 1)
```

<img src="http://upload.wikimedia.org/wikipedia/commons/3/37/SternBrocotTree.svg">

```haskell
>>> run sumTree $ takeTree 12 sternBrocot
26859127 % 5544
```

It is a fun exercise to `Conquer` a search through the [Stern-Brocot tree](http://en.wikipedia.org/wiki/Stern%E2%80%93Brocot_tree) for a given positive `Rational`. They are all in there, so you'll find it eventually, but the tree goes on forever in all directions.

So what can we compute that isn't just the usual `Foldable` fare? After all we could just run through a tree left to right in-order, and answer `sum`.

How about the maximum path weight?

We can compute the maximum path weight top to bottom and bottom to top, so a `Monoid` for it should exist, by the third homomorphism theorem... and indeed it does!

```haskell
mpw :: (Num a, Ord a) => Conquer (Tree a) a
mpw = conquer k h m (Pair 0 0) where
  k (Pair a _) = a
  h (L n m) = Pair (n + k m) n
  h (R m n) = Pair (n + k m) n
  m (Pair m1 w1) (Pair m2 w2) = Pair (max m1 (w1 + m2)) (w1 + w2)
```

```haskell
>>> run mpw (Bin (Bin Tip 1 Tip) 2 (Bin (Bin Tip 2 Tip) 3 Tip))
7
```

Similarly, we can compute the height of the tree. (Note: this version fixes a bug in the paper by Morihata et al.)

```haskell
height :: Conquer (Tree a) Int
height = conquer k h m (Pair 0 0) where
  k (Pair a _) = a + 1
  h (L n m) = Pair (k m) 1
  h (R m n) = Pair (k m) 1
  m (Pair m1 w1) (Pair m2 w2) = Pair (max m1 (w1 + m2)) (w1 + w2)
```

Putting It All Together
=======================

```active haskell
{-# LANGUAGE ExistentialQuantification #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE UndecidableInstances #-}

import Control.Applicative
import Control.Comonad
import Control.Parallel.Strategies
import Data.Foldable
import Data.List (unfoldr)
import Data.Monoid
import Data.Proxy
import Data.Ratio
import Data.Reflection (reify, Reifies(..))
import Data.Traversable

class Recursive t where
  data D t :: * -> *
  plug   :: D t t -> t -> t
  walk   :: Applicative f => (a -> f b) -> D t a -> f (D t b)
  path :: t -> Maybe (D t t, t)

instance Recursive t => Functor (D t) where
  fmap = fmapDefault

instance Recursive t => Foldable (D t) where
  foldMap = foldMapDefault

instance Recursive t => Traversable (D t) where
  traverse = walk

type Z t = [D t t]

divide :: Recursive t => t -> Z t
divide = unfoldr path

zipped :: Recursive t => Z t -> t -> t
zipped dt t = Prelude.foldr plug t dt

run :: Recursive t => Conquer t a -> t -> a
run (Conquer k h) = k . go where
  go = foldMap (h . fmap go) . divide
{-# INLINE run #-}

runPar :: Recursive t => Conquer t a -> t -> a
runPar (Conquer k h) = k . go where
  go = foldMap (h . (`using` parTraversable rseq) . fmap go) . divide
{-# INLINE runPar #-}

above :: Recursive t => Z t -> Conquer t a -> Conquer t a
above t0 (Conquer k h) = let x = go t0 in Conquer (k . mappend x) h where
  go = foldMap (h . fmap (go . divide))
{-# INLINE above #-}

below :: Recursive t => Conquer t a -> Z t -> Conquer t a
below (Conquer k h) t0 = let y = go t0 in Conquer (\x -> k (mappend x y)) h where
  go = foldMap (h . fmap (go . divide))
{-# INLINE below #-}

newtype M a p = M { runM :: a }

instance Reifies p (a -> a -> a, a)  => Monoid (M a p) where
  mempty = M $ snd $ reflect (Proxy :: Proxy p)
  mappend (M a) (M b) = M $ fst (reflect (Proxy :: Proxy p)) a b

conquer :: forall t a b. Recursive t => (b -> a) -> (D t b -> b) -> (b -> b -> b) -> b -> Conquer t a
conquer k h m z = reify (m,z) $ \(Proxy :: Proxy p) -> Conquer
   (k . runM :: M b p -> a) (M . h . fmap runM)

-- * The Main Attraction

data Conquer t a = forall b. Monoid b => Conquer (b -> a) (D t b -> b)

-- * Instances

data Pair a b = Pair { pi1 :: !a, pi2 :: !b }

instance (Monoid a, Monoid b) => Monoid (Pair a b) where
  mempty = Pair mempty mempty
  mappend (Pair a b) (Pair c d) = Pair (mappend a c) (mappend b d)

instance Functor (Conquer t) where
  fmap f (Conquer k h) = Conquer (f.k) h

instance Comonad (Conquer t) where
  extract (Conquer k h) = k mempty
  duplicate (Conquer k h) = Conquer (\b -> Conquer (k . mappend b) h) h

instance Recursive t => ComonadApply (Conquer t) where
  (<@>) = (<*>)

instance Recursive t => Applicative (Conquer t) where
  pure a = Conquer (const a) (const ())
  Conquer kf hf <*> Conquer ka ha = Conquer
    (\(Pair bf ba) -> kf bf (ka ba))
    (\dab -> Pair (hf (pi1 <$> dab)) (ha (pi2 <$> dab)))

-- * Naturals

data Nat = Z | S Nat

instance Recursive Nat where
  data D Nat a = Succ
  plug Succ = S
  walk _ Succ = pure Succ
  path (S n) = Just (Succ, n)
  path Z     = Nothing

nat :: Conquer Nat Int
nat = Conquer getSum (\Succ -> Sum 1)

-- * Lists

instance Recursive [a] where
  data D [a] b = Cons { getCons :: a} deriving (Show,Read)
  plug (Cons a) as = a : as
  walk _ = pure . Cons . getCons
  path (x:xs)   = Just (Cons x, xs)
  path []       = Nothing

sumList :: Conquer [Int] Int
sumList = Conquer getSum (Sum . getCons)

-- * Trees

data Tree a
  = Bin (Tree a) a (Tree a)
  | Tip deriving (Show,Read)

instance Recursive (Tree a) where
  data D (Tree a) b = L a b | R b a deriving (Show,Read)

  plug (L a r) l = Bin l a r
  plug (R l a) r = Bin l a r

  walk f (L a b) = L a <$> f b
  walk f (R b a) = (`R` a) <$> f b

  path (Bin l a r) = Just (L a r, l)
  path Tip = Nothing

unfoldTree :: (s -> Maybe (s, a, s)) -> s -> Tree a
unfoldTree f s = case f s of
  Nothing -> Tip
  Just (l, a, r) -> Bin (unfoldTree f l) a (unfoldTree f r)

sternBrocot :: Tree Rational
sternBrocot = unfoldTree (\(Pair a b) -> Just (Pair (a+1) b, a % b, Pair a (b+1))) (Pair 1 1)

takeTree :: Int -> Tree a -> Tree a
takeTree 0 _   = Tip
takeTree n Tip = Tip
takeTree n (Bin l a r) = Bin (takeTree (n-1) l) a (takeTree (n-1) r)

sumTree :: Num a => Conquer (Tree a) a
sumTree = Conquer getSum h where
  h (L a (Sum b)) = Sum (a + b)
  h (R (Sum b) a) = Sum (a + b)

mpw :: (Num a, Ord a) => Conquer (Tree a) a
mpw = conquer k h m (Pair 0 0) where
  k (Pair a _) = a
  h (L n m) = Pair (n + k m) n
  h (R m n) = Pair (n + k m) n
  m (Pair m1 w1) (Pair m2 w2) = Pair (max m1 (w1 + m2)) (w1 + w2)
{-# INLINE mpw #-}

height :: Conquer (Tree a) Int
height = conquer k h m (Pair 0 0) where
  k (Pair a _) = a + 1
  h (L n m) = Pair (k m) 1
  h (R m n) = Pair (k m) 1
  m (Pair m1 w1) (Pair m2 w2) = Pair (max m1 (w1 + m2)) (w1 + w2)
{-# INLINE height #-}

main = do
  print $ run height Tip
  print $ run height (Bin (Bin Tip 1 Tip) 2 (Bin Tip 3 Tip))
  print $ run ((/) <$> mpw <*> sumTree) (takeTree 10 sternBrocot) 
```

Go And Make It Fast
===================

Now, my challenge to you is to find a nice way to execute this in parallel.

The obvious approach sparks way too much.

```haskell
runPar :: Recursive t => Conquer t a -> t -> a
runPar (Conquer k h) = k . go where
  go = foldMap (h . (`using` parTraversable rseq) . fmap go) . divide
```

Perhaps Morihata and Matsuzaki's 2008 [A Parallel Tree Contraction Algorithm
on Non-Binary Trees](http://www.keisu.t.u-tokyo.ac.jp/research/techrep/data/2008/METR08-27.pdf) may serve as a guide for how to speed this up. 

Thoughts?

-[Edward Kmett](mailto:ekmett@gmail.com)

September 13, 2013

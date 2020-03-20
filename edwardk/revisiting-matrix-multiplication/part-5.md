# Part V: Heaps of Performance

Back in [part 3](https://www.fpcomplete.com/user/edwardk/revisiting-matrix-multiplication-part-3) I naïvely plodded ahead with implementing a nice fusion combinator to try to use inside of my matrix multiplication routine.

It was slow.

It was slow because I built nested trees of concatenations and merges, and then when pumping it stream fusion was never getting anything that it could unroll into a bigger loop. Moreover, when faced with a tree of merges, the `Stream` fusion framework was being forced to keep the associativity by which it was originally constructed, but there was no balance ensuring that tree was any good.

If you're just joining, you may want to go back and read parts [1](https://www.fpcomplete.com/user/edwardk/revisiting-matrix-multiplication-part-1),[2](https://www.fpcomplete.com/user/edwardk/revisiting-matrix-multiplication-part-2),[3](https://www.fpcomplete.com/user/edwardk/revisiting-matrix-multiplication-part-3), and [4](https://www.fpcomplete.com/user/edwardk/revisiting-matrix-multiplication-part-4), but like the last two parts, this one mostly stands alone if you ignore the motivation for the trick as a short practicum on how to bootstrap a datastructure and an introduction to pairing heaps and ephemeral steques.

Work Smarter
============

Let's work smarter, not harder, by finding a better algorithm, rather than trying to run a dumb one fast.

To that end, I need to be able to deal with merging together streams and concatenating streams with a minimum of impact from concatenation on the cost of getting the next element.

We could of course merge streams by representing them as a [heap](http://en.wikipedia.org/wiki/Heap_%28data_structure%29).

Pick your favorite heap. Mine today for ease of exposition is a [pairing heap](http://en.wikipedia.org/wiki/Pairing_Heap). It is pretty awful when you want to actually analyze the runtime performance of your algorithm, but it has excellent [amortized](http://en.wikipedia.org/wiki/Amortized_analysis) performance in practice and fits the form of what I'm going to put around it.

I'll deal with only non-empty heaps, as we can move all the reasoning for the empty case into `Maybe (Heap a)` and it admits a nicer recursive definition.

For simplicity, I'll just be using `Int` keys rather than [Morton-ordered](http://en.wikipedia.org/wiki/Z-order_curve) keys below.

A pairing heap is a heap built on a [rose tree](http://en.wikipedia.org/wiki/Rose_Tree). That is to say any node in your heap can have any number of children, so long as you satisfy the [heap property](http://xlinux.nist.gov/dads/HTML/heapproperty.html).

```haskell
data Heap a = Heap !Int a [Heap a] deriving Show
```

A pairing heap provides us with a dead simple union for two heaps. Compare them, and then shove the one with the larger starting key one underneath the smaller. That it is.

```haskell
mix :: Heap a -> Heap a -> Heap a
mix x@(Heap i a as) y@(Heap j b bs) 
  | i <= j    = Heap i a (y:as)
  | otherwise = Heap j b (x:bs)
```

We can obviously grab the `top` element

```haskell
top :: Heap a -> (Int, a)
top (Heap i a _) = (i, a)
```

and we can `pop` that element off the `Heap`:

```haskell
pop :: Heap a -> Maybe (Heap a)
pop (Heap _ _ [])    = Nothing
pop (Heap _ _ (x:xs) = Just (merge x xs)

merge :: Heap a -> [Heap a] -> Heap a
merge x (y:ys) = case ys of
  (z:zs) -> mix x y `mix` merge z zs
  []     -> mix x y
merge x [] = x
```

The complexity in analyzing the asymptotics of a pairing heap comes from the fact that `merge` might have to do a lot of work, but `merge` pairs up the heaps before doing it's work to get better balancing. Again, nothing in what I'm writing here cares about the pairing heap parts, other than it is easy to write and similar to the surrounding code we're adding, and it is the most obviously amenable heap to the transformation I'll be applying later.

`mix` now handles the behavior I need for merging streams/heaps, but I'm also going to need efficient concatenation. While `mix` is "technically correct" for concatenating two heaps that do not overlap in key space, it also isn't terribly good at it.

Steques
=======

A "stack-ended queue", or steque, is what [Tarjan](http://en.wikipedia.org/wiki/Robert_Tarjan) calls a structure that permits `cons` and `snoc` on both ends, but only efficient `head`/`tail`/`uncons` from one side.

This distinguishes it from a [deque](http://en.wikipedia.org/wiki/Double-ended_queue), which permits efficient `init`/`tail`/`unsnoc` as well.

The simplest implementation of a steque is

```haskell
data Steque a = Steque [a] [a] deriving (Show,Read)
```

The first list is in order from left to right, and the second list is reversed.

Note: This isn't the most robust steque or deque in Okasaki's book. Its asymptotics are amortized, not worst case, and they only hold for ephemeral rather than persistent use. However, I only care about that ephemeral use case and so the reduction in book-keeping overhead is more valuable than slower execution to support operations I don't use!

```haskell
instance Functor Steque where
  fmap f (Steque fs rs) = Steque (fmap f fs) (fmap f rs)

instance Foldable Steque where
  foldMap f (Steque fs rs) = foldMap f fs `mappend` getDual (foldMap (Dual . f) rs)
```

We can use a little bit of `lens` to make the `Traversable` instance easier, given the
combinator `backwards` that can turn a `Traversal` around.

```haskell
instance Traversable Steque where
  traverse f (Steque fs rs) = Steque <$> traverse f fs <*> backwards traverse f rs
```

and then we could quotient out the irrelevancies of how our elements are distributed between the two lists:

```haskell
instance Eq a => Eq (Steque a) where
  (==) = (==) `on` toList

instance Ord a => Ord (Steque a) where
  compare = compare `on` toList
```

We could even get ambitious and define a bunch of other instances on `Steque`, just because we can:

```haskell
instance Applicative Steque where
  pure a = Steque [a] []
  (<*>) = ap

instance Monad Steque where
  return a = Steque [a] []
  Steque fs bs >>= f = Steque (fs >>= toList . f) (bs >>= toListOf (backwards folded) . f)
```

... but we're getting distracted, we really just want to be able to `cons`/`head`/`tail`/`uncons`, and `snoc`.

To make this interesting, let us show how to implement these using some lesser understood parts of `lens`.

The `lens` package provides a common [`Cons`](http://hackage.haskell.org/packages/archive/lens/3.9.0.2/doc/html/Control-Lens-Cons.html) class for dealing with `cons`-like behavior. It is tricky to instantiate correctly, because it is overloaded to permit usecases where you can merely `cons` and not `uncons` and vice-versa, but we can define an instance:

```haskell
instance (Choice p, Applicative f) => Cons p f (Steque a) (Steque b) a b where
  _Cons = prism (\(x,Steque fs bs) -> Steque (x:fs) bs) $ \ (Steque fs bs) -> case fs of
     x:xs -> Right (x,Steque xs bs)
     []   -> case reverse bs of
       x:xs -> Right (x,Steque xs [])
       [] -> Left (Steque [] [])
```

This gives us the following [`Prism`](http://hackage.haskell.org/packages/archive/lens/3.9.0.2/doc/html/Control-Lens-Prism.html):

```haskell
_Cons :: Prism (Steque a) (Steque b) (a, Steque a) (b, Steque b)
```

We can now use it to `cons` and `uncons`, using the definitions from `Control.Lens.Cons`:

```haskell
cons a as = _Cons # (a,as)
uncons as = as^?_Cons
```

That module also provides combinators that makes either a `Getter`, `Setter`, `Lens` or `Traversal` for each of `_head` and `_tail` depending on the restricted form of `_Cons` you chose.

```haskell
_head = _Cons._1
_tail = _Cons._2
```

On the other side we only want to be able to `snoc`, not use `tail`, `init` or `unsnoc`, so let's define:

```haskell
instance (p ~ Reviewed, f ~ Identity, a ~ b) => Snoc p f (Steque a) (Steque b) a b where
  _Snoc = unto $ \(Steque f b,x) -> Steque f (x:b)
```

Note: we _could_ also support `_init` and `_tail` and `unsnoc`, but they will be _O(n)_, even for ephemeral use, because we do not try to preserve any balance between the front and back lists.

`unto` is used to define a [`Review`](http://hackage.haskell.org/packages/archive/lens/3.9.0.2/doc/html/Control-Lens-Review.html#g:1), which is to a `Prism` what a `Getter` is to a `Lens`. In practice it makes something that is like a `Prism` in that you can apply `#` to it, but nothing else.

This is enough to permit us to use the stock definition of `snoc`:

```haskell
snoc as a = _Snoc # (as,a)
```

With all of that we can finally play with our `Steque`:

```active haskell
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE UndecidableInstances #-}
import Control.Applicative
import Control.Monad
import Control.Lens
import Control.Lens.Internal.Review
import Data.Foldable
import Data.Functor.Identity
import Data.Monoid
import Data.Function (on)

data Steque a = Steque [a] [a] deriving (Show,Read)

instance Eq a => Eq (Steque a) where
  (==) = (==) `on` toList

instance Ord a => Ord (Steque a) where
  compare = compare `on` toList

instance Functor Steque where
  fmap f (Steque fs rs) = Steque (fmap f fs) (fmap f rs)

instance Foldable Steque where
  foldMap f (Steque fs rs) = foldMap f fs `mappend` getDual (foldMap (Dual . f) rs)

instance Traversable Steque where
  traverse f (Steque fs rs) = Steque <$> traverse f fs <*> backwards traverse f rs

instance Applicative Steque where
  pure a = Steque [a] []
  (<*>) = ap
  
instance Alternative Steque where
  Steque fs rs <|> Steque fs' rs' = Steque (fs ++ reverse rs ++ fs') rs'
  empty = Steque [] []

instance Monad Steque where
  return a = Steque [a] []
  Steque fs bs >>= f = Steque (fs >>= toList . f) (bs >>= toListOf (backwards folded) . f)

instance MonadPlus Steque where
  mplus = (<|>)
  mzero = Steque [] []

instance Monoid (Steque a) where
  mappend = (<|>)
  mempty = Steque [] []

instance (Choice p, Applicative f) => Cons p f (Steque a) (Steque b) a b where
  _Cons = prism (\(x,Steque fs bs) -> Steque (x:fs) bs) $ \ (Steque fs bs) -> case fs of
     x:xs -> Right (x,Steque xs bs)
     []   -> case reverse bs of
       x:xs -> Right (x,Steque xs [])
       [] -> Left (Steque [] [])

instance (p ~ Reviewed, f ~ Identity, a ~ b) => Snoc p f (Steque a) (Steque b) a b where
  _Snoc = unto $ \(Steque f b,x) -> Steque f (x:b)
  
-- show
main = print $ uncons (snoc (cons 1 (cons 2 (snoc (empty :: Steque Int) 4))) 5)
-- /show
```

Now, our definition of a `Steque` works passably well.

We can snoc, cons, uncons, head, or tail all in _O(1)_ amortized time.

but the `Monoid`, `Alternative` and `MonadPlus` instances perform poorly, doing _O(n)_ work because it has to glue together potentially rather long lists:

```haskell
Steque fs rs <|> Steque fs' rs' = Steque (fs ++ reverse rs ++ fs') rs'
```

It is that `Monoid` performance that makes us sad, so let's do something about it.

Bootstrapping
=============

To solve that we turn to [Chris Okasaki](http://www.usma.edu/eecs/SitePages/Chris%20Okasaki.aspx)'s [Purely Functional Data Structures](http://www.cs.cmu.edu/~rwh/theses/okasaki.pdf) once again, this time to find bootstrapping.

Bootstrapping is a tool for defining a data structure with cheap concatenation by recursively storing it in itself.

```haskell
data Catenable f a = Empty | Cons a (f (Catenable f a))
```

Here we could build a `Catenable Steque` out of our basic `Steque` following Okasaki's recipe. Here the vocabulary of Tarjan is more accurate and extensible. Tarjan calls a "catenable steque" a c-steque, while Okasaki uses the less informative name "catenable list."

As with the pairing `Heap`, recursion is simpler if we don't allow `Empty` catenable substructures, and I'm only going to need non-empty catenable structures, though, so `Catenable` simplifies to:

```haskell
data Catenable f a = Cons a (f (Catenable f a))
```

which advanced Haskell programmers in the audience may recognize as my old friend the [cofree comonad](http://hackage.haskell.org/packages/archive/free/3.4.2/doc/html/Control-Comonad-Cofree.html)! In the interest of retaining some semblance of accessibility, I'm not going to work through this exercise in its full generality.

Now it is obvious how to concatenate a pair of catenable steques, we simply insert one into the other. 

```haskell
data CSteque a = Cons a (Steque (CSteque a))

instance Semigroup (CSteque a) where
  CSteque a as <> bs = CSteque a (snoc as bs)
```

Hrmm. That sounds suspiciously like the operation that drove our pairing heap.

You can derive `cons` and `snoc` by just concatenating singleton catenable steques, and `uncons` simply has to put back the leftovers if any into the underlying `Steque`, which it can do because our `Steque` supports _O(1)_ `cons`!

If we inline the definition of `Steque` into `CSteque` we get:

```haskell
data CSteque a = Cons a [CSteque a] [CSteque a]
```

and that looks a _lot_ like our pairing heap, just with different invariants.

Hrmm.

Catenable Heaps
===============

Finally, what we want looks kind of like a steque of heaps, but where the heaps can also act like steques of heaps, _ad infinitum_. We can get there by mashing all the parts we've described together to get:

```haskell
data Heap a = Heap {-# UNPACK #-} !Int a [Heap a] [Heap a] [Heap a]
```

This type represents a form of catenable non-empty pairing heap. The three lists of heaps in turn are:

1.) the jumbled mess of heaps we haven't sorted relative to one another except partially via the heap property, followed by
2.) a list of heaps that do not overlap one another in key space in ascending order, followed by
3.) another list of heaps that do not overlap one another in key space in descending order.

Now we can concatenate in _O(1)_!

```haskell
fby :: Heap a -> Heap a -> Heap a
fby (Heap i a as ls rs) r = Heap i a as ls (r:rs)
```

A number of operations remain trivial:

```haskell
top :: Heap a -> (Key, a)
top (Heap i a _ _ _) = (i, a)

singleton :: Key -> a -> Heap a
singleton k v = Heap k v [] [] []

fromList :: [(Key,a)] -> Heap a
fromList ((k0,v0):xs) = Prelude.foldr (\(k,v) r -> mix (singleton k v) r) (singleton k0 v0) xs
fromList [] = error "empty Heap"

fromAscList :: [(Key,a)] -> Heap a
fromAscList ((k0,v0):xs) = Prelude.foldr (\(k,v) r -> fby (singleton k v) r) (singleton k0 v0) xs
fromAscList [] = error "empty Heap"
```

`pop` got a bit more complicated. We'll just pass it the 3 lists from our heap, as it doesn't use the key/value pair out front and we don't want to require the compiler to figure out to specialize on the constructor.

```haskell
pop :: [Heap a] -> [Heap a] -> [Heap a] -> Maybe (Heap a)
pop (x:xs) ls     rs = Just $ fbys (merge x xs) ls rs
pop []     (l:ls) rs = Just $ fbys l ls rs
pop []     []     rs = case reverse rs of
  f:fs -> Just (fbys f fs [])
  []   -> Nothing
  
fbys :: Heap a -> [Heap a] -> [Heap a] -> Heap a
fbys (Heap i a as [] []) ls' rs' = Heap i a as ls' rs'
fbys (Heap i a as ls []) ls' rs' = Heap i a as ls $ rs' <> reverse ls'
fbys (Heap i a as ls rs) ls' rs' = Heap i a as ls $ rs' <> reverse ls' <> rs
```

> **No-Prize #7**
>
> Computing with `fbys` in `pop` is somewhat unpleasant, but I don't have a cleaner way.
> 
> Is there something where I can preserve the correctness of this but not have to move content
> into the right hand reversed list and re-reverse it? Keep in mind it isn't sound to move 
> it into the left list, because we don't know the length of the left list and can only 
> move content left when the left list is empty.
>
> We could probably insert a pair of other heaps into the reversed list by building up a
> chain of `fby`s popping something off of the `ls'` and smashing the other `ls'` and `rs'` 
> underneath it by using this function recursively. What does that look like?

and `mix` also becomes more complex as it now has to deal with the extra concatenated components, pushing what it can into the nested pairing heap.

```haskell
mix :: Heap a -> Heap a -> Heap a
mix x@(Heap i a as al ar) y@(Heap j b bs bl br)
  | i <= j    = Heap i a (y:pops as al ar) [] []
  | otherwise = Heap j b (x:pops bs bl br) [] []
  
  
merge :: Heap a -> [Heap a] -> Heap a
merge x (y:ys) = case ys of
  (z:zs) -> mix x y `mix` merge z zs
  []     -> mix x y
merge x [] = x
  
pops :: [Heap a] -> [Heap a] -> [Heap a] -> [Heap a]
pops xs     []     [] = xs
pops (x:xs) ls     rs = [fbys (merge x xs) ls rs]
pops []     (l:ls) rs = [fbys l ls rs]
pops []     []     rs = case reverse rs of
  f:fs -> [fbys f fs []]
  _    -> [] -- caught above
```

`pops` is a slight optimization of `pop` that uses the fact that the result is winding up in a list subject to the heap property and need not be reduced all the way down to a `Maybe (Heap a)` just yet.

Finally, we can define a _useful_ conversion to a `Stream` that can get some benefit out of stream fusion.

```haskell
data HeapState a
  = Start !(Heap a)
  | Ready {-# UNPACK #-} !Key a !(Heap a)
  | Final {-# UNPACK #-} !Key a
  | Finished

streamHeapWith :: Monad m => (a -> a -> a) -> Maybe (Heap a) -> Stream m (Key, a)
streamHeapWith f h0 = Stream step (maybe Finished Start h0) Unknown where
  step (Start (Heap i a xs ls rs))     = return $ Skip $ maybe (Final i a) (Ready i a) $ pop xs ls rs
  step (Ready i a (Heap j b xs ls rs)) = return $ case compare i j of
    LT -> Yield (i, a)      $ maybe (Final j b) (Ready j b) $ pop xs ls rs
    EQ | c <- f a b -> Skip $ maybe (Final i c) (Ready i c) $ pop xs ls rs
    GT -> Yield (j, b)      $ maybe (Final i a) (Ready i a) $ pop xs ls rs
  step (Final i a) = return $ Yield (i,a) Finished
  step Finished    = return Done
  {-# INLINE [1] step #-}
{-# INLINE [0] streamHeapWith #-}
```

This doesn't quite hit the [stream fusion](http://citeseer.ist.psu.edu/viewdoc/summary?doi=10.1.1.104.7401) goal of being made of a single loop that can fuse into other loops, due to the recursive calls to `pop`. I leave it as an exercise for the reader to find a form that implements `pop` via iterative `Skip` steps and a more complex state -- In other words, I haven't bothered yet. ;)

Putting it all together yields:

```active haskell
-- show
{-# LANGUAGE BangPatterns #-}
{-# LANGUAGE PatternGuards #-}
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE TypeFamilies #-}

import Control.Applicative
import Control.Lens
import Data.Bits
import Data.Foldable
import Data.Monoid
import Data.Vector.Fusion.Stream.Monadic hiding (singleton, fromList)
import Data.Vector.Fusion.Stream.Size
import Data.Vector.Fusion.Util
import Data.Word
import Prelude

type Key = Int

-- | Bootstrapped _catenable_ non-empty pairing heaps
data Heap a = Heap {-# UNPACK #-} !Key a [Heap a] [Heap a] [Heap a]
  deriving (Show,Read)

-- | Append two heaps where we know every key in the first occurs before every key in the second
fby :: Heap a -> Heap a -> Heap a
fby (Heap i a as ls rs) r = Heap i a as ls (r:rs)

-- | Interleave two heaps making a new 'Heap'
mix :: Heap a -> Heap a -> Heap a
mix x@(Heap i a as al ar) y@(Heap j b bs bl br)
  | i <= j    = Heap i a (y:pops as al ar) [] []
  | otherwise = Heap j b (x:pops bs bl br) [] []


merge :: Heap a -> [Heap a] -> Heap a
merge x (y:ys) = case ys of
  (z:zs) -> mix x y `mix` merge z zs
  []     -> mix x y
merge x [] = x

top :: Heap a -> (Key, a)
top (Heap i a _ _ _) = (i, a)

pop :: [Heap a] -> [Heap a] -> [Heap a] -> Maybe (Heap a)
pop (x:xs) ls     rs = Just $ fbys (merge x xs) ls rs
pop []     (l:ls) rs = Just $ fbys l ls rs
pop []     []     rs = case reverse rs of
  f:fs -> Just (fbys f fs [])
  []   -> Nothing

singleton :: Key -> a -> Heap a
singleton k v = Heap k v [] [] []

fromList :: [(Key,a)] -> Heap a
fromList ((k0,v0):xs) = Prelude.foldr (\(k,v) r -> mix (singleton k v) r) (singleton k0 v0) xs
fromList [] = error "empty Heap"

fromAscList :: [(Key,a)] -> Heap a
fromAscList ((k0,v0):xs) = Prelude.foldr (\(k,v) r -> fby (singleton k v) r) (singleton k0 v0) xs
fromAscList [] = error "empty Heap"

-- * Internals

fbys :: Heap a -> [Heap a] -> [Heap a] -> Heap a
fbys (Heap i a as [] []) ls' rs' = Heap i a as ls' rs'
fbys (Heap i a as ls []) ls' rs' = Heap i a as ls $ rs' <> reverse ls'
fbys (Heap i a as ls rs) ls' rs' = Heap i a as ls $ rs' <> reverse ls' <> rs

pops :: [Heap a] -> [Heap a] -> [Heap a] -> [Heap a]
pops xs     []     [] = xs
pops (x:xs) ls     rs = [fbys (merge x xs) ls rs]
pops []     (l:ls) rs = [fbys l ls rs]
pops []     []     rs = case reverse rs of
  f:fs -> [fbys f fs []]
  _    -> [] -- caught above by the 'go as [] []' case

-- * Instances

instance Functor Heap where
  fmap f (Heap k a xs ls rs) = Heap k (f a) (fmap f <$> xs) (fmap f <$> ls) (fmap f <$> rs)

instance FunctorWithIndex Key Heap where
  imap f (Heap k a xs ls rs) = Heap k (f k a) (imap f <$> xs) (imap f <$> ls) (imap f <$> rs)

instance Foldable Heap where
  foldMap f = go where
    go (Heap _ a xs ls rs) = case pop xs ls rs of
      Nothing -> f a
      Just h  -> f a `mappend` go h
  {-# INLINE foldMap #-}

instance FoldableWithIndex Key Heap where
  ifoldMap f = go where
    go (Heap i a xs ls rs) = case pop xs ls rs of
      Nothing -> f i a
      Just h  -> f i a `mappend` go h
  {-# INLINE ifoldMap #-}

-- this linearizes the heap
instance Traversable Heap where
  traverse f xs = fromAscList <$> traverse (traverse f) (itoList xs)
  {-# INLINE traverse #-}

instance TraversableWithIndex Key Heap where
  itraverse f xs = fromAscList <$> traverse (\(k,v) -> (,) k <$> f k v) (itoList xs)
  {-# INLINE itraverse #-}

data HeapState a
  = Start !(Heap a)
  | Ready {-# UNPACK #-} !Key a !(Heap a)
  | Final {-# UNPACK #-} !Key a
  | Finished

streamHeapWith :: Monad m => (a -> a -> a) -> Maybe (Heap a) -> Stream m (Key, a)
streamHeapWith f h0 = Stream step (maybe Finished Start h0) Unknown where
  step (Start (Heap i a xs ls rs))     = return $ Skip $ maybe (Final i a) (Ready i a) $ pop xs ls rs
  step (Ready i a (Heap j b xs ls rs)) = return $ case compare i j of
    LT -> Yield (i, a)      $ maybe (Final j b) (Ready j b) $ pop xs ls rs
    EQ | c <- f a b -> Skip $ maybe (Final i c) (Ready i c) $ pop xs ls rs
    GT -> Yield (j, b)      $ maybe (Final i a) (Ready i a) $ pop xs ls rs
  step (Final i a) = return $ Yield (i,a) Finished
  step Finished    = return Done
  {-# INLINE [1] step #-}
{-# INLINE [0] streamHeapWith #-}

streamHeapWith0 :: Monad m => (a -> a -> Maybe a) -> Maybe (Heap a) -> Stream m (Key, a)
streamHeapWith0 f h0 = Stream step (maybe Finished Start h0) Unknown where
  step (Start (Heap i a xs ls rs))     = return $ Skip $ maybe (Final i a) (Ready i a) $ pop xs ls rs
  step (Ready i a (Heap j b xs ls rs)) = return $ case compare i j of
    LT -> Yield (i, a) $ maybe (Final j b) (Ready j b) $ pop xs ls rs
    EQ -> case f a b of
      Nothing -> Skip  $ maybe Finished Start $ pop xs ls rs
      Just c  -> Skip  $ maybe (Final i c) (Ready i c) $ pop xs ls rs
    GT -> Yield (j, b) $ maybe (Final i a) (Ready i a) $ pop xs ls rs
  step (Final i a) = return $ Yield (i,a) Finished
  step Finished = return Done
  {-# INLINE [1] step #-}
{-# INLINE [0] streamHeapWith0 #-}

-- /show
-- show
main = print $ (fromList [(10,"hi"),(20,"there")] `mix` singleton 100 "hello") `fby` singleton 200 "goodbye"
-- show
```

I've left figuring out how to write `_Cons` and `_Snoc` as an exercise for the reader.

With that we can finally concatenate heaps and mix them together as needed, and the stream fusion combinators can be used after the fact to spit out a stream of values that have been merged by our desired concept of addition. What has happened is that we for the most part are able to punt considering anything later in the chain of concatenations until after we've fully explored the stuff nearer to hand.

In practice this made a couple of orders of magnitude difference in the performance of the resulting code, so all this theoretical nonsense paid off.

I'd like a version of this that let's me reach the asymptotic bounds on set union set and reached by Erik Demaine et al.'s tour de force [Adaptive set intersections, unions, and differences](http://dl.acm.org/citation.cfm?id=338634), but I don't _think_ we can get there given the extra constraints that we don't fully know the contents of the streams we're skipping over in advance.

-[Edward Kmett](mailto:ekmett@gmail.com)

August 23 2013

*Edited September 14th, 2013*: Mihály Bárász pointed out to me that I had over-simplified the pairing heap implementation. I've rectified that by replacing `foldl mix` with `merge` uniformly throughout the code above. This makes a noticeable improvement in performance.


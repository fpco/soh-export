# Part IV: IntMap!?

Back in [part 2](https://www.fpcomplete.com/user/edwardk/revisiting-matrix-multiplication-part-2) we showed how we can compare two keys in Morton order without having to actually do the interleaving.

I'm going to take some time today to try to help folks build intuition for what that means by taking a look at an old standby in the Haskell ecosystem, `Data.IntMap`, and use the techniques we developed in part 2 to generate a version of some of the core routines that uses the same `xor` trick rather than store the prefix and mask it stores today.

Nothing in here has to do with matrix multiplication, but it is a powerful application of the notion of a "most significant difference" and `xor` based comparison by it. 

If you're just getting here, you might want to start with parts [1](https://www.fpcomplete.com/user/edwardk/revisiting-matrix-multiplication-part-1), [2](https://www.fpcomplete.com/user/edwardk/revisiting-matrix-multiplication-part-2) and [3](https://www.fpcomplete.com/user/edwardk/revisiting-matrix-multiplication-part-3), but there is no pressure. Like [part 3](https://www.fpcomplete.com/user/edwardk/revisiting-matrix-multiplication-part-3), this post can largely stand alone.

IntMap?!
========

Changing to my "difference tree" approach permits a number of operations to terminate earlier, and may well turn out to be a viable way to improve the venerable `IntMap` in the `containers` package, but I'm using it here mostly to help us develop familiarity with the 'most significant most significant difference'. 

In many ways this is a degenerate case, but it at least helps us develop some facility for using the tool!

In `Data.IntMap.Base`, based on some decade old code from [Daan Leijen](http://research.microsoft.com/en-us/people/daan/), the `containers` library defines:

```haskell
type Prefix = Int
type Mask   = Int

data IntMap a
  = Nil
  | Tip {-# UNPACK #-} !Int a
  | Bin {-# UNPACK #-} !Prefix {-# UNPACK #-} !Mask !(IntMap a) !(IntMap a)
```

The `Prefix` and `Mask` contain information about the known common prefix of the PATRICIA trie up to that point, and the `Mask` of the position where they diverge.

Using what we now know, we can change this to 

```haskell
data IntMap a
  = Nil
  | Tip {-# UNPACK #-} !Int a
  | Bin {-# UNPACK #-} !Int {-# UNPACK #-} !Int !(IntMap a) !(IntMap a)
```

where the values we store in the `Bin` constructor are just the minimum and maximum `Int` key in the tree below.

## Classifying Keys

To do so we need to be able to distinguish between roughly 6 cases for how a key can interact with the map, as if we had the `Prefix` and `Mask` in hand. From left to right:

```haskell
data Class
  = FarLeft   -- differs on a higher msb, outside left branch
  | NearLeft  -- differs on the same msb, but outside current left branch
  | InLeft    -- within the left branch
  | InRight   -- within the right branch
  | NearRight -- differs on the same msb, but outside current right branch
  | FarRight  -- differs on a higher msb, outside right branch
```

The `xor` trick I mentioned at the end of part 2 can be bundled into a slightly unwieldy combinator, `significant` such that `significant a b c d` implies that the position of the most significant difference between `c` and `d` dominates the position of the most significant difference between `a` and `b`.

```haskell
significant :: Int -> Int -> Int -> Int -> Bool
significant a b c d = ab < cd && ab < xor ab cd where
  cd = xor c d
  ab = xor a b
```

With that we can proceed to use trickery and slight of hand to classify our keys with regards to the bounds of our `IntMap`:

```active haskell
import Data.Bits

data Class
  = FarLeft   -- differs on a higher msb, outside left branch
  | NearLeft  -- differs on the same msb, but outside current left branch
  | InLeft    -- within the left branch
  | InRight   -- within the right branch
  | NearRight -- differs on the same msb, but outside current right branch
  | FarRight  -- differs on a higher msb, outside right branch
  deriving (Eq,Ord,Show,Read)

significant :: Int -> Int -> Int -> Int -> Bool
significant a b c d = ab < cd && ab < xor ab cd where
  cd = xor c d
  ab = xor a b

-- show
classify :: Int -> Int -> Int -> Class
classify k x y
  | k < x = if significant x y k y then FarLeft else NearLeft
  | k > y = if significant x y x k then FarRight else NearRight
  | significant k y x y = InRight
  | otherwise = InLeft
  
main = print $ classify 1 2 4
-- /show
```

We don't need to use the full power of classify, as often some subset of those 6 cases will be the same, so lets define a couple of additional combinators:

```haskell
outside :: Int -> Int -> Int -> Bool
outside k x y = k < x || k > y

insideR :: Int -> Int -> Int -> Bool
insideR k x y = significant k y x y
```

`outside` serves as a more accurate version of `nomatch` from the `Data.IntMap` internals, and `insideR` assumes we're inside the range `[x..y]` and notes that if there is an extra bit of difference between `x` and `y` than between `k` and `y`, then we're in the right branch.

We simply use integer comparisons and 3 `xor`s to classify how our key relates to the range of our `IntMap`.

We won't actually be using `classify` explicitly but you can play with it to see if you agree with its results! You'll be able to see it conceptually at work in the code below though.

## Stock Definitions

Some of the stock combinators don't change at all:

```haskell
null :: IntMap a -> Bool
null Nil = True
null _   = False

empty :: IntMap a
empty = Nil
```

Similarly the instances don't change:

```haskell
instance Traversable IntMap where
  traverse f m0 = go m0 where
    go (Bin x y l r) = Bin x y <$> go l <*> go r
    go (Tip x a) = Tip x <$> f a
    go Nil = pure Nil
  {-# INLINE traverse #-}

instance Foldable IntMap where
  foldMap f m0 = go m0 where
    go Nil = mempty
    go (Tip _ a) = f a
    go (Bin _ _ l r) = mappend (go l) (go r)
  {-# INLINE foldMap #-}

instance Functor IntMap where
  fmap f m0 = go m0 where
    go Nil = Nil
    go (Tip x a) = Tip x (f a)
    go (Bin x y l r) = Bin x y (go l) (go r)
  {-# INLINE fmap #-}
```

but fast new friends become possible. 

```haskell
range :: IntMap a -> Maybe (Int,Int)
range Nil           = Nothing
range (Tip i a)     = Just (i,i)
range (Bin i j _ _) = Just (i,j)
```

Given the common usecase of finding the maximum key in an `IntMap` and inserting a new entry, that is a pretty nice side-effect!


## Lookup

The next combinator to benefit from this change is `lookup`.

```haskell
lookup :: Int -> IntMap a -> Maybe a
lookup k m0 = go m0 where
  go (Tip i a)
    | k == i    = Just a
    | otherwise = Nothing
  go (Bin x y  l r)
    | outside k x y = Nothing
    | insideR r x y = go r
    | otherwise     = go l
  go Nil = Nothing
{-# INLINE lookup #-}
```

`lookup` can now use the smarter `outside` check to fail faster than it can in stock `containers`.

## Insert

Defining `insert` showcases the need for all 6 cases from `classify`. You can identify them in the reasoning below for how to handle the `Bin` case.

```haskell
insert :: Int -> a -> IntMap a -> IntMap a
insert k a m0 = go m0 where
  go Nil = Tip k a
  go (Tip j b) = case compare k j of
    LT -> Bin k j (Tip k a) (Tip j b)
    EQ -> Tip k a
    GT -> Bin j k (Tip j b) (Tip k a)
  go n@(Bin x y l r)
    | k < x = if significant x y k y then Bin k y (Tip k a) n
                                     else Bin k y (go l) r
    | k > y = if significant x y x k then Bin x k n (Tip k a)
                                     else Bin x k l (go r)
    | significant k y x y = Bin x y l (go r)
    | otherwise           = Bin x y (go l) r
```

## Delete

We can also define `delete`, benefiting similarly from the earlier exit in the unnecessary deletion case:

```haskell
newx :: Int -> IntMap a -> IntMap a -> IntMap a
newx _ Nil r = r
newx y l@(Tip x _) r = Bin x y l r
newx y l@(Bin x _ _ _) r = Bin x y l r
{-# INLINE newx #-}

newy :: Int -> IntMap a -> IntMap a -> IntMap a
newy _ l Nil = l
newy x l r@(Tip y _)     = Bin x y l r
newy x l r@(Bin _ y _ _) = Bin x y l r
{-# INLINE newy #-}

delete :: Int -> IntMap a -> IntMap a
delete k m0 = go m0 where
  go n@(Bin x y l r)
    | outside k x y = n
    | insideR k x y = newy x l (go r)
    | otherwise     = newx y (go l) r
  go n@(Tip x a)
    | k == x    = Nil
    | otherwise = n
  go Nil = Nil
{-# INLINE delete #-}
```

Here we suffer ever so slightly. The Prefix and Mask are fixed when we call bin in the old code, but now we need to inspect the values we're given in newx and newy to find their bounds.

## At

Finally, no post of mine would be complete without at least one reference to `lens`.

We can define the new `alterF` Lens that is being backported to `containers` for our modified `IntMap` directly. Here I'll call it `at`, due to its similarity to the `lens` combinator of the same name.

```haskell
at :: Functor f => Int -> (Maybe a -> f (Maybe a)) -> IntMap a -> f (IntMap a)
at k f m0 = go m0 where
  go Nil       = maybe Nil (Tip k) <$> f Nothing
  go n@(Tip x a) = case compare k x of
    LT -> maybe n (\b -> Bin k x (Tip k b) n) <$> f Nothing
    EQ -> maybe Nil (Tip k) <$> f (Just a)
    GT -> maybe n (\b -> Bin x k n (Tip k b)) <$> f Nothing
  go n@(Bin x y l r)
    | k < x = if significant x y k y then maybe n (\b -> Bin k y (Tip k b) n) <$> f Nothing
                                     else maybe n (\b -> Bin (min k x) y (insert k b l) r) <$> f Nothing
    | k > y = if significant x y x k then maybe n (\b -> Bin x k n (Tip k b)) <$> f Nothing
                                     else maybe n (\b -> Bin x (max k y) l (insert k b r)) <$> f Nothing
    | insideR k x y = newy x l <$> go r
    | otherwise     = (\l' -> newx y l' r) <$> go l
{-# INLINE at #-}
```

We can view the scarier, but Haskell 98 type for at in the definition above as 

```haskell
at :: Int -> Lens' (IntMap a) (Maybe a)
```

This combinator is a bit hideous, but it _should_ work! Feel free to test it. =)

## Run It!

Putting it all together we get:

```active haskell
-- show
import Control.Applicative hiding (empty)
import Control.Lens hiding (at,outside)
import Data.Bits
import Data.Foldable
import Data.Traversable
import Data.Monoid
import Prelude hiding (lookup, null)

data IntMap a
  = Nil
  | Tip {-# UNPACK #-} !Int a
  | Bin {-# UNPACK #-} !Int {-# UNPACK #-} !Int !(IntMap a) !(IntMap a)
  deriving (Eq,Ord,Show,Read)

null :: IntMap a -> Bool
null Nil = True
null _   = False
{-# INLINE null #-}

empty :: IntMap a
empty = Nil
{-# INLINE empty #-}

range :: IntMap a -> Maybe (Int,Int)
range Nil           = Nothing
range (Tip i a)     = Just (i,i)
range (Bin i j _ _) = Just (i,j)
{-# INLINE range #-}

instance Traversable IntMap where
  traverse f m0 = go m0 where
    go (Bin x y l r) = Bin x y <$> go l <*> go r
    go (Tip x a) = Tip x <$> f a
    go Nil = pure Nil
  {-# INLINE traverse #-}

instance Foldable IntMap where
  foldMap f m0 = go m0 where
    go Nil = mempty
    go (Tip _ a) = f a
    go (Bin _ _ l r) = mappend (go l) (go r)
  {-# INLINE foldMap #-}

instance Functor IntMap where
  fmap f m0 = go m0 where
    go Nil = Nil
    go (Tip x a) = Tip x (f a)
    go (Bin x y l r) = Bin x y (go l) (go r)
  {-# INLINE fmap #-}

-- @significant a b c d@ implies that the position of the most significant difference between
-- @c@ and @d@ dominates the position of the difference between @a and b@.
significant :: Int -> Int -> Int -> Int -> Bool
significant a b c d = ab < cd && ab < xor ab cd where
  cd = xor c d
  ab = xor a b
{-# INLINE significant #-}

-- | for expository purposes only
data Class
  = FarLeft   -- differs on a higher msb, outside left branch
  | NearLeft  -- differs on the same msb, but outside current left branch
  | InLeft    -- within the left branch
  | InRight   -- within the right branch
  | NearRight -- differs on the same msb, but outside current right branch
  | FarRight  -- differs on a higher msb, outside right branch
  deriving (Eq,Ord,Show,Read)

-- | classify a key @k@ with regards to a binary tree split on the 2-fattest number within @(x..y]@
classify :: Int -> Int -> Int -> Class
classify k x y
  | k < x = if significant x y k y then FarLeft else NearLeft
  | k > y = if significant x y x k then FarRight else NearRight
  | significant k y x y = InRight
  | otherwise = InLeft
{-# INLINE classify #-}

outside :: Int -> Int -> Int -> Bool
outside k x y = k < x || k > y
{-# INLINE outside #-}

insideR :: Int -> Int -> Int -> Bool
insideR k x y = significant k y x y
{-# INLINE insideR #-}

lookup :: Int -> IntMap a -> Maybe a
lookup k m0 = go m0 where
  go (Tip i a)
    | k == i    = Just a
    | otherwise = Nothing
  go (Bin x y  l r)
    | outside k x y = Nothing -- short-circuit
    | insideR k x y = go r
    | otherwise     = go l
  go Nil = Nothing
{-# INLINE lookup #-}

insert :: Int -> a -> IntMap a -> IntMap a
insert k a m0 = go m0 where
  go Nil = Tip k a
  go (Tip j b) = case compare k j of
    LT -> Bin k j (Tip k a) (Tip j b)
    EQ -> Tip k a
    GT -> Bin j k (Tip j b) (Tip k a)
  go n@(Bin x y l r)
    | k < x = if significant x y k y then Bin k y (Tip k a) n
                                     else Bin k y (go l) r
    | k > y = if significant x y x k then Bin x k n (Tip k a)
                                     else Bin x k l (go r)
    | significant k y x y = Bin x y l (go r)
    | otherwise           = Bin x y (go l) r
{-# INLINE insert #-}

newx :: Int -> IntMap a -> IntMap a -> IntMap a
newx _ Nil r = r
newx y l@(Tip x _) r = Bin x y l r
newx y l@(Bin x _ _ _) r = Bin x y l r
{-# INLINE newx #-}

newy :: Int -> IntMap a -> IntMap a -> IntMap a
newy _ l Nil = l
newy x l r@(Tip y _)     = Bin x y l r
newy x l r@(Bin _ y _ _) = Bin x y l r
{-# INLINE newy #-}

delete :: Int -> IntMap a -> IntMap a
delete k m0 = go m0 where
  go n@(Bin x y l r)
    | outside k x y = n
    | insideR k x y = newy x l (go r)
    | otherwise     = newx y (go l) r
  go n@(Tip x a)
    | k == x    = Nil
    | otherwise = n
  go Nil = Nil
{-# INLINE delete #-}

at :: Functor f => Int -> (Maybe a -> f (Maybe a)) -> IntMap a -> f (IntMap a)
at k f m0 = go m0 where
  go Nil       = maybe Nil (Tip k) <$> f Nothing
  go n@(Tip x a) = case compare k x of    
    LT -> maybe n (\b -> Bin k x (Tip k b) n) <$> f Nothing
    EQ -> maybe Nil (Tip k) <$> f (Just a)
    GT -> maybe n (\b -> Bin x k n (Tip k b)) <$> f Nothing
  go n@(Bin x y l r)
    | k > y = if significant x y x k then maybe n (\b -> Bin x k n (Tip k b)) <$> f Nothing               -- far right
                                     else maybe n (\b -> Bin x (max k y) l (insert k b r)) <$> f Nothing  -- near right
    | k < x = if significant x y k y then maybe n (\b -> Bin k y (Tip k b) n) <$> f Nothing               -- far left
                                     else maybe n (\b -> Bin (min k x) y (insert k b l) r) <$> f Nothing  -- near left
    | significant k y x y = newy x l <$> go r -- in right
    | otherwise           = (\l' -> newx y l' r) <$> go l -- in left
{-# INLINE at #-}

bin :: IntMap a -> IntMap a -> IntMap a
bin l Nil = l
bin Nil r = r
bin l@(Tip x _)     r@(Tip y _)     = Bin x y l r
bin l@(Tip x _)     r@(Bin _ y _ _) = Bin x y l r
bin l@(Bin x _ _ _) r@(Bin _ y _ _) = Bin x y l r
bin l@(Bin x _ _ _) r@(Tip y _)     = Bin x y l r
{-# INLINE bin #-}
-- /show
-- show Run it!
main = print $ (empty & at 1 ?~ "hello" & at 2 ?~ "world") ^. at 2
-- /show
```

I have no idea if this is faster than the approach taken by `Data.IntMap` in practice on real data, but `xor` is your friend.

A great opportunity for participation would be to prove whether this code is faster or slower than the code in `Data.IntMap` in practice and if it proves to be faster, flesh it out!

I have one last diversion I need to post about before I can finally get to talking about the algorithm that started this discussion.

-[Edward Kmett](mailto:ekmett@gmail.com)

August 23 2013
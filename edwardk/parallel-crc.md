# Parallel and Incremental CRCs

Back in the [second part](https://www.fpcomplete.com/user/edwardk/cellular-automata/part-2) of my [series on cellular automata](https://www.fpcomplete.com/user/edwardk/cellular-automata/), I implemented a [Moore machine](http://en.wikipedia.org/wiki/Moore_machine) for computing a [CRC-32](http://en.wikipedia.org/wiki/Cyclic_redundancy_check). It was used to generate the CRC-32 for the end of a [PNG](http://www.libpng.org/pub/png/) block header.

At the time I was somewhat saddened by the fact that I was forced to turn to a left fold, because there didn't appear to be an algorithm for computing a CRC in parallel or incrementally, beyond simply working a few more bits at a time through a lookup table.

It finally occurred to me how we can implement a parallel/incremental CRC calculator today during the walk home from work, and I figured I should share it. This has a number of practical applications, as it permits us to regenerate CRCs in response to small edits in very large datasets in logarithmic time!

I've never seen this algorithm before applied to CRCs, so it is at least new to me.

This may also let me play some games with fast composable hashing for data structures in a later post.

CRC-32
======

Calculating a CRC is just an exercise in long division. 

The particular number system we're working in is a bit peculiar.

But other than that, it really is just what you were taught in grade-school!

Probably the best explanation for how a CRC works comes from Ross William's rather unassuming and seemingly immortal 1996 ["A Painless Guide to CRC Error Detection Algorithms"](http://www.repairfaq.org/filipg/LINK/F_crc_v31.html), which goes through from soup to nuts how a CRC is defined from an operational perspective. It can be found on Filip G's [site on repairfaq.org](http://www.repairfaq.org/filipg/), which will provide you a fresh reminder of what the internet was like for us in the mid 90s.

The content, however, is excellent.

The View from the Left
======================

Recall my definition of CRC-32 from last time along with enough of an implementation of left folding to run it:

```active haskell
{-# LANGUAGE ExistentialQuantification, OverloadedStrings #-}
import Data.Bits
import Data.ByteString.Lazy as Lazy
import Data.ByteString.Lazy.Char8 ()
import Data.Vector.Unboxed as Unboxed
import Data.Word

data L b a = forall x. L (x -> b -> x) x (x -> a)

runL :: L Word8 a -> Lazy.ByteString -> a
runL (L xbx x xa) bs = xa (Lazy.foldl' xbx x bs)

crc32 :: L Word8 Word32
crc32 = L step 0xffffffff complement where
  step r b = unsafeShiftR r 8 
     `xor` crcs Unboxed.! fromIntegral (xor r (fromIntegral b) .&. 0xff)

crcs :: Unboxed.Vector Word32
crcs = Unboxed.generate 256 (go.go.go.go.go.go.go.go.fromIntegral) where
  go c = unsafeShiftR c 1 `xor` if c .&. 1 /= 0 then 0xedb88320 else 0

main = print $ runL crc32 "12345689"
```

This is already what is somewhat embarassingly considered the "parallel" algorithm for computing a CRC. It computes 8 bits of the CRC at a time by using a look-up table, but it doesn't let me put extra cores to work and it doesn't let me split up my workload.

The algorithm I used above as the left fold can be seen as an implementation of the reflected table-driven algorithm Williams defines in Chapter 12, but it is pretty standard, modulo the trappings I've put on it to treat it as a fold, and algorithms just like it can be found everywhere.

I was skimming [Reversing CRC, Theory and Practice](http://sar.informatik.hu-berlin.de/research/publications/SAR-PR-2006-05/SAR-PR-2006-05_.pdf) when it clicked for me what the properties are of a CRC that make this possible, so through the rest of the post, I'll try to follow their vocabulary when it comes to a CRC.

Tearing Apart a CRC
===================

Let's define `crc(a) = r` to be the core calculation that performs the polynomial division in _GF(2^32)_ that drives a CRC. We'll abuse notation, and permit us to have partially applied this long division to have obtained some remainder `r` so far, and take `crc(r,a) = r'` to be the calculation that picks up with remainder `r` and accepts more input `a`, obtaining the new remainder `r'`.

Using juxtaposition in the input argument to denote concatenation, this gives us the first property we'll need:

```
crc(r,ab) = crc(crc(r,a),b)
```

If we try to adapt it directly to get a `Monoid` we get stuck. This seems to be inherently sequential.

So, let's define the Galois field used for CRC-32.

When working with numbers in _GF(2^32)_ addition is given by `xor`, and multiplication is limited to like powers of `x`. If we fix the CRC polynomial and work bit reversed, we get the following number type.

```active haskell
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
import Data.Bits
import Data.Word

-- show
newtype GF = GF { runGF :: Word32 } deriving (Eq,Show,Read,Bits)

poly :: GF
poly = GF 0xedb88320 -- -- x^32+x^26+x^23+x^22+x^16+x^12+x^11+x^10+x^8+x^7+x^5+x^4+x^2+x+1

-- | compute x * p(x)
xtimes :: GF -> GF
xtimes c = unsafeShiftR c 1 + if testBit c 0 then poly else 0

instance Num GF where
  (+) = xor
  (-) = xor
  _ * 0 = 0
  a * b = xtimes a * unsafeShiftL b 1 + if testBit b 31 then a else 0
  negate = id
  abs = id
  signum = fromIntegral . signum . runGF
  fromInteger i
    | odd i     = GF 0x80000000 -- x^0
    | otherwise = GF 0          -- 0
-- /show

main = putStrLn "It compiles, so it is obviously correct."
```

Since we're working bit reversed, the coefficient of x^31 is the least significant bit, and on the other extreme the coefficient of `1 = x^0` is our most significant bit. 

The next most significant bit gives us `x` itself.

```haskell
x :: GF
x = GF 0x40000000
```

`xtimes` above was the more humorously named `go` calculation used in `crcs` in the original left fold. It computes `x * p(x)` for a polynomial `p`. In the multiplication algorithm above, we're using it in a peasant multiplier. Since we have a working `Num`, we can now use the built-in `(^)` from the Prelude to compute `x^n`. 

If you care about efficiency you could write the peasant exponentiator by hand:

```haskell
-- | peasant exponentiation
xpow :: Int -> GF
xpow 0 = 1
xpow 1 = x
xpow n = case divMod n 2 of
  (q,0) ->          square (xpow q)
  (q,1) -> xtimes $ square (xpow q)
  _     -> error "unimplemented: extended Euclidean algorithm"
  where square a = a * a
```

This has the benefit that it gets to use `xtimes` rather than `(* x)` internally, shaving a few cycles, but clutters exposition. That said, if you care about efficiency, perhaps you can find me a nicer multiplication routine. The peasant exponentiator we're using isn't the fastest form of multiplication in _GF(2^n)_. We might consider adapting something like [Montgomery multiplication and exponentiation](http://www.csd.uwo.ca/~eschost/Exam/Koc.pdf), for very large choices of `n`, there are other multiplication routines as well.

My implementation of `GF` is structured along the lines of a quickly dashed off sketch by Adrian Keet during a discussion on `#haskell-lens`. Any errors introduced in the above code, however, are all mine.

Properties of all CRCs
======================

[Tanenbaum](http://en.wikipedia.org/wiki/Andrew_S._Tanenbaum)'s [Computer Networks](http://www.amazon.com/Computer-Networks-Edition-Andrew-Tanenbaum/dp/0132126958) describes a few of the conditions necessary to define a good polynomial, including resistance to single and two bit errors, odd numbers of bits, bursts of all ones or zeroes, etc. But even with a well chosen standard polynomial in hand, there are a couple of classes of errors that standard CRC algorithms are tweaked to handle better.

Rather than reason about this through properties of _GF(2^32)_ directly, let's talk about two properties that are commonly seen as weaknesses of CRC and use them to derive an algorithm.

Zero Blindness
==============

An actual CRC looks like <code>CRC(a) = crc(INIT,a) `xor` FINAL</code>, rather than being a just pure long division. Usually `INIT` and `FINAL` will be something like all 1s to complement the initial state and the final state. The reason why `INIT` is chosen to be non-zero is to combat a phenomenon known as "zero blindness".

If you consider a CRC as a polynomial long division, then prefixing a number of zeroes onto the number we're dividing wouldn't change the CRC at all! So what is done in practice is that we start with a current "remainder" that is `0xffffffff` rather than `0`. 

There is another kind of zero blindness that affects the message followed by the CRC itself, any additional trailing 0's wouldn't affect the checksum, so most real world CRCs `complement` all of the bits of the remainder before emitting it into the output as well. 

Here we'll be using the same mask for both `INIT` and `FINAL`:

```haskell
-- | @x^31 + x^30 + ... + x + 1@
ones :: GF
ones = GF 0xffffffff
```

We'll be reasoning about our CRC by conceptually abusing this property of a CRC alongside a second, much more powerful property. 'forcing' our CRC into a blind state by exploiting:


Additive Homomorphism
=====================

This second property is one that is much more often used from a cracking perspective than as a desirable property:

```
crc(r1 `xor` r2, a1 `xor` a2) = crc(r1,a1) `xor` crc(r2,a2)
```

Exploiting Weakness
===================

If we have input fragments `a` and `b` with lengths `m` and `n` respectively:

```
CRC(ab) =                               -- definition of CRC
crc(INIT,ab) + FINAL =                  -- linearity
crc(INIT,a0^n + 0^m b) + FINAL =        -- additive homomorphism
crc(INIT,a0^n) + crc(0,0^nb) + FINAL =  -- zero blindness 
crc(INIT,a0^n) + crc(0,b) + FINAL       -- definition of crc
crc(crc(INIT,a),0^n) + crc(0,b) + FINAL -- additive homomorphism
crc(crc(INIT,0^m)+crc(0,a),0^n) + crc(0,b) + FINAL
```

That final expression requires us to be able to calculate `crc(r,0^m)` and to know `crc(0,a)` for each half of the input, but nothing more.

As James Deikun pointed out to me, `crc(r,0^m)` is particularly well behaved. When we look at what it does to our division is it appends just a number of 0's. When we work out the effect on the remainder we just need to multiply through our remainder by `x^m`!

Instead of storing the string `0^m` or the length `m`, I'll just store `x^m` directly.

Monoidal composition proceeds through a simplification of the definition above, where `INIT = FINAL = 0`:

```
crc(0,ab) =                    -- linearity
crc(0,a 0^n + 0^mb)            -- additive homomorphism
crc(0,a 0^n) + crc(0,0^m b) =  -- zero blindness
crc(0,a 0^n) + crc(0,b) =      -- definition of crc
crc(crc(0,a),0^n) + crc(0,b) = -- crc(r,0^m) = r*x^m 
crc(0,a)*x^n + crc(0,b)
```

So if we just store the length (mangled as `x^n`) and the CRC remainder from a starting remainder of `0`, we can compose these answers associatively!

```haskell
data CRC32 = CRC32 
  {-# UNPACK #-} !GF
  {-# UNPACK #-} !GF 
  deriving (Eq,Read,Show)

instance Monoid CRC32 where
  CRC32 p m `mappend` CRC32 q n = CRC32 (p*n+q) (m*n)
  mempty = CRC32 0 1
```

Here `CRC32 p m` stores `p = crc(0,a)` for our input string alongside `m = x^|a|`.

Our `mempty` comes from the fact that `crc(0,"") = 0`, and note that the 1 here is actually `GF 0x80000000` and represents `x^0` in our Galois field, so we're denoting an input of length 0.

If we build the standard byte-wise CRC table:

```haskell
crcs :: UArray Word8 GF
crcs = listArray (0,255) $ map (xtimes.xtimes.xtimes.xtimes.xtimes.xtimes.xtimes.xtimes.GF) [0..255]
```

We can then define the injection into our monoid for a byte worth of data at a time.

```haskell
byte :: Word8 -> CRC32
byte a = CRC32 (crcs ! a) (x^8) -- x^8 = GF 0x800000
```

The last thing we need to do is apply the `INIT` and `FINAL` masks. 

Going back to our original equations, `crc(INIT,0^m)` can be computed by multiplying the mask by `x^m`, and we can just add in the `FINAL` mask when we're done. Here it is also `0xffffffff`.

Using those results we can finally find our fold:

```haskell
runCRC :: CRC32 -> Word32
runCRC (CRC32 p m) = runGF (ones * m + p + ones)
```

The Story So Far
================

```active haskell
{-# LANGUAGE GADTs, GeneralizedNewtypeDeriving #-}
{-# OPTIONS_GHC -fno-warn-type-defaults #-}

import Control.Comonad
import Data.Bits
import Data.Foldable as Foldable
import Data.Monoid
import Data.Profunctor
import Data.Array.Unboxed
import Data.Word

newtype GF = GF { runGF :: Word32 } deriving (Eq,Show,Read,Bits,IArray UArray)

poly :: GF
poly = GF 0xedb88320 -- x^32+x^26+x^23+x^22+x^16+x^12+x^11+x^10+x^8+x^7+x^5+x^4+x^2+x+1

-- | compute x * p(x)
xtimes :: GF -> GF
xtimes c = unsafeShiftR c 1 + if testBit c 0 then poly else 0

instance Num GF where
  (+) = xor
  (-) = xor
  _ * 0 = 0
  a * b = xtimes a * unsafeShiftL b 1 + if testBit b 31 then a else 0
  negate = id
  abs = id
  signum = fromIntegral . signum . runGF
  fromInteger i
    | odd i     = GF 0x80000000 -- x^0
    | otherwise = GF 0          -- 0

x :: GF
x = GF 0x40000000 -- x^1

ones :: GF
ones = GF 0xffffffff -- | x^31+x^30+...+x+1

data CRC32 = CRC32 {-# UNPACK #-} !GF {-# UNPACK #-} !GF deriving (Eq,Read,Show)

instance Monoid CRC32 where
  CRC32 p m `mappend` CRC32 q n = CRC32 (p*n+q) (m*n)
  mempty = CRC32 0 1

crcs :: UArray Word8 GF
crcs = listArray (0,255) $ map (xtimes.xtimes.xtimes.xtimes.xtimes.xtimes.xtimes.xtimes.GF) [0..255]

runCRC :: CRC32 -> Word32
runCRC (CRC32 p m) = runGF (ones * m + p + ones)

byte :: Word8 -> CRC32
byte a = CRC32 (crcs ! a) (x^8)

main = print $ runCRC $ foldMap (byte.fromIntegral.fromEnum) "123456789"
```

Deeper Down the Rabbit Hole
===========================

This also means if you keep the blinded CRCs around you can calculate a modified aggregate CRC without starting over from the left of your document or ADT.

Combining this with a [finger tree](http://www.soi.city.ac.uk/~ross/papers/FingerTree.html) can enable you to calculate a CRC for an entire document in parallel and then to update it incrementally in mere logarithmic time in response to edits. It also means that you do not need to invalidate an entire CRC. 

Combining this with an ADT makes for cheap rehashing as portions of a structure changes.

Nothing here was specific to the modulus used for CRC-32. It works equally well for any other standard "Rocksoft"-style CRC. This works particularly well for the variant of CRC-8 used in [Rijndael/AES](http://en.wikipedia.org/wiki/Advanced_Encryption_Standard), because the lookup table can be used directly for multiplication.

A CRC isn't a cryptographically strong hash, however, due to their importance in network protocols, telecommunications, document and image formats,  and general ubiquity, having a way to compute them both in parallel and incrementally opens up new opportunities.

Now what we've come all the way through to the conclusion there are obvious analogies to how one can execute a Rabin-Karp or a simple addition modulo prime hash in parallel.

Monoidal Folding
================
*(if you aren't interested in folding, you can probably skip the remainder, but it does polish up the API nicely)*

I haven't mentioned a single `Comonad` all post. Time to fix that.

We can build up a new kind of folding, based capturing the arguments to a `foldMap` in amber, just like [Max Rabkin originally did](http://squing.blogspot.com/2008/11/beautiful-folding.html) with the arguments to `foldl'`.

This yields the following type:

```haskell
data M a b = forall r. M (r -> b) (a -> r) (r -> r -> r) r
```

We can map over the inputs and outputs:

```haskell
instance Profunctor M where
  dimap f g (M k h m e) = M (g.k) (h.f) m e
```

There is also an `Applicative` structured just like the `Applicative` for `L`. 

There is even a `Monad` that permits you to write (inefficient) multipass algorithms!

Pause for Reflection
====================

The problem then becomes, how to run it. `foldMap` doesn't let us explicitly pass arguments for the `mappend` and `mempty` of the Monoid in question.

We could run `M` using `foldr`:

```haskell
runM :: Foldable f => M a b -> f a -> b
runM (M k h m z) xs = k (Foldable.foldr (m.h) z xs)
```

But that isn't very satisfying, the whole point of introducing a 'Monoid' was to be able to associate however we pleased.

We could make a `Monoid`:

```haskell
newtype Slow a = Slow { runSlow :: (a -> a -> a) -> a -> a }

instance Monoid (Slow a) where
  mempty = Slow (\_ e -> e)
  mappend (Slow f) (Slow g) = Slow $ \p e = runSlow f p e `p` runShow g p e
```

But if you have any sharing of monoidal values, then working with `Slow` will actually only share the _functions_ it contains, not their results.

The better tool for evaluating `M`, unfortunately requires some heavy lifting. We can turn to my `reflection` package to allow us to manufacture a `Monoid` out of an arbitrary function and unit value.

[Austin Seipp](https://github.com/thoughtpolice) recently wrote [a nice tutorial on how to do this](https://www.fpcomplete.com/user/thoughtpolice/using-reflection), so I'll just skip to the solution:

```haskell
newtype N a s = N { runN :: a }

instance Reifies s (a -> a -> a, a) => Monoid (N a s) where
  mempty = N $ snd $ reflect (Proxy :: Proxy s)
  mappend (N a) (N b) = N $ fst (reflect (Proxy :: Proxy s)) a b

runM :: Foldable f => M a b -> f a -> b
runM (M k h m (z :: m)) s = reify (m, z) $
    \ (_ :: Proxy s) -> k $ runN (foldMap (N #. h) s :: N m s)
```

Here we make up a `Monoid` that reflects how to do what it needs to do out of the environment, then make up an instance with the `mappend` and `mempty` we want. 

Finally `(#.)` in `runM` is drawn from `Data.Profunctor.Unsafe` to work around the fact that `SomeNewType . f` doesn't erase down to `f`, but rather to the eta-expansion of `f`, so it has ever so slightly different semantics.

Now that we can run an monoidal folding, it'd be nice to be able to do the same things we could with left foldings.

Not The Comonads You Are Looking For
====================================

But if we go to copy the `Comonad` from `L` we get gobbledygook!

```haskell
instance Comonad (L a) where
  extract (L k _ z) = k z
  duplicate (L k h z) = L (L k h) h z
```

When we were working with `L`, this comonad was able to sneak in before the final tweak is applied at the end and snatches up the seed we've computed so far and uses at as the seed for the nested `L` in

```haskell
duplicate :: L a b -> L a (L a b)
```

But if we do that with `M`, we'd get something that replaced the `mempty` of our `Monoid` with the result of what we've tallied up so far!

Last time, we used the fact that `(L a)` was isomorphic to an `EnvT`'d `Store` `Comonad`, where we'd quantified away the state parameter.

This time, we'll have to turn to a different isomorphism.

There is a comonad for `(->) m`, whenever `m` is a `Monoid`. This `Comonad` is what I called the `Traced` comonad in `comonad-transformers`. If you squint hard enough, you can see that `M a` is isomorphic to `forall m. Monoid m => EnvT (a -> m) ((->) m) a`.

When we borrow this structure we get

```haskell
instance Comonad (M b) where
  extract (M k _ _ z) = k z
  duplicate (M k h m z) = M (\n -> M (k . m n) h m z) h m z
```

Isomorphism Does Not Imply Equal Efficiency
===========================================

I had a performance issue with the monoid homomorphism described in the last post, that this `Comonad` helps address.

Last time I noted that with `L`:

```haskell
extract = more [] 
more xs =<= more ys = more (xs ++ ys)
```

But if we expand the second law:

```haskell
more (xs ++ ys) = more xs =<= more ys = more xs . extend (more ys)
```

there we're partially applying a _suffix_ of our input to a left fold!

This means we're capturing it in the environment:

```haskell
more xs . extend (more ys) = more xs . fmap (more ys) . duplicate
```

This leads to the observation that we don't have to do that, our comonad was set up precisely so we could partially evaluate out to a new seed. 

By naturality or good old fashioned equational reasoning, we can get to an alternate statement of the second law:

```haskell
more (xs ++ ys) = more ys . more xs . duplicate
```

This gives us an efficient algorithm for composing left foldings.

When we work with the monoidal folding above, we have the flexibility to get the more efficient non-leaky version for both the left side and the right side of the structure.

Putting It Back Together Again
==============================

```active haskell
{-# LANGUAGE GADTs, GeneralizedNewtypeDeriving, ScopedTypeVariables, FlexibleContexts, UndecidableInstances #-}

import Control.Comonad
import Data.Bits
import Data.Foldable as Foldable
import Data.Monoid
import Data.Profunctor.Unsafe
import Data.Proxy
import Data.Reflection
import Data.Array.Unboxed
import Data.Word

newtype GF = GF { runGF :: Word32 } deriving (Eq,Show,Read,Bits,IArray UArray)

-- | @x^32 + x^26 + x^23 + x^22 + x^16 + x^12 + x^11 + x^10 + x^8 + x^7 + x^5 + x^4 + x^2 + x + 1@
poly :: GF
poly = GF 0xedb88320 -- the polynomial we're working modulo in GF(2^32) for CRC32.

-- | compute x * p(x)
xtimes :: GF -> GF
xtimes c = unsafeShiftR c 1 + if testBit c 0 then poly else 0

instance Num GF where
  (+) = xor
  (-) = xor
  _ * 0 = 0
  a * b = xtimes a * unsafeShiftL b 1 + if testBit b 31 then a else 0
  negate = id
  abs = id
  signum = fromIntegral . signum . runGF
  fromInteger i
    | odd i     = GF 0x80000000 -- x^0
    | otherwise = GF 0          -- 0

-- | @x^1@
x :: GF
x = GF 0x40000000

-- | @x^31 + x^30 + ... + x + 1@
ones :: GF
ones = GF 0xffffffff

data CRC32 = CRC32 {-# UNPACK #-} !GF {-# UNPACK #-} !GF deriving (Eq,Read,Show)

instance Monoid CRC32 where
  CRC32 p m `mappend` CRC32 q n = CRC32 (p*n+q) (m*n)
  mempty = CRC32 0 1

crcs :: UArray Word8 GF
crcs = listArray (0,255) $ map (xtimes.xtimes.xtimes.xtimes.xtimes.xtimes.xtimes.xtimes.GF) [0..255]

runCRC :: CRC32 -> Word32
runCRC (CRC32 p m) = runGF (ones * m + p + ones)

byte :: Word8 -> CRC32
byte a = CRC32 (crcs ! a) (x^8)

data M a b where
  M :: (r -> b) -> (a -> r) -> (r -> r -> r) -> r -> M a b

instance Functor (M a) where
  fmap f (M k h m z) = M (f.k) h m z

instance Profunctor M where
  dimap f g (M k h m z) = M (g.k) (h.f) m z

instance Comonad (M a) where
  extract (M k _ _ z) = k z
  duplicate (M k h m z) = M (\n -> M (k . m n) h m z) h m z

newtype N a s = N { runN :: a }

instance Reifies s (a -> a -> a, a) => Monoid (N a s) where
  mempty = N $ snd $ reflect (Proxy :: Proxy s)
  mappend (N a) (N b) = N $ fst (reflect (Proxy :: Proxy s)) a b

-- show
runM :: Foldable f => M a b -> f a -> b
runM (M k h m (z :: m)) s = reify (m, z) $
    \ (_ :: Proxy s) -> k $ runN (foldMap (N #. h) s :: N m s)

runR :: Foldable f => M a b -> f a -> b
runR (M k h m z) xs = k (Foldable.foldr (m.h) z xs)

runL :: Foldable f => M a b -> f a -> b
runL (M k h m z) xs = k (Foldable.foldl' (\r -> m r . h) z xs)

crc32 :: M Word8 Word32
crc32 = M runCRC byte mappend mempty

main = print $ runM crc32 [0x12,0x34]
-- /show
```

Now the only part of the CRC API that the user has to concern themselves with is the actual `crc32` combinator. Everything else is generic and can be reused for other folds. If you want to keep folding you can keep the fold alive by using `duplicate`!

We could actually abstract over the choice of polynomial for our Galois field, by calculating our lut locally in a `crc` combinator that was parameterized on the values of `INIT`, `FINAL` and `poly`, or even by using `reflection` to move the modulus and/or lookup table into a type parameter. By switching to `Bits` and `Integral` constraints rather than fixing ourselves to `Word32`, we can implement `CRC-8`, `CRC-16` and `CRC-64` variants very easily as well.


@@@
```active haskell
{-# LANGUAGE GADTs, GeneralizedNewtypeDeriving, RankNTypes, ScopedTypeVariables, StandaloneDeriving, MultiParamTypeClasses, FlexibleContexts #-}
{-# OPTIONS_GHC -fno-warn-type-defaults #-}

import Control.Comonad
import Data.Bits
import Data.Foldable as Foldable
import Data.Monoid
import Data.Profunctor
import Data.Proxy
import Data.Reflection
import Data.Array.Unboxed
import Data.Word

newtype GF a s = GF { runGF :: a } deriving (Eq,Show,Read,Bits)

deriving instance IArray UArray a => IArray UArray (GF a s)

-- | compute x * p(x)
xtimes :: (Integral a, Bits a, Reifies s a) => GF a s -> GF a s
xtimes c = unsafeShiftR c 1 + if testBit c 0 then GF (reflect c) else 0
{-# INLINE xtimes #-}

instance (Integral a, Bits a, Reifies s a) => Num (GF a s) where
  (+) = xor
  (-) = xor
  _ * 0 = 0
  a * b = xtimes a * unsafeShiftL b 1 + if testBit b (bitSize b - 1) then a else 0
  negate = id
  abs = id
  signum = fromIntegral . signum . runGF
  fromInteger i
    | odd i     = GF (bit (bitSize (undefined :: a) - 1)) -- x^0
    | otherwise = GF 0                     -- 0

-- | @x^1@
x :: forall a s. Bits a => GF a s
x = GF (bit (bitSize (undefined :: a) - 2))
{-# INLINE x #-}

data CRC a = CRC !a !a deriving (Eq,Read,Show)

instance Num a => Monoid (CRC a) where
  CRC p m `mappend` CRC q n = CRC (p*n+q) (m*n)
  mempty = CRC 0 1

crc :: forall a. (Integral a, Bits a, IArray UArray a) => a -> a -> a -> M Word8 a
crc _INIT _FINAL poly = reify poly $ \(_ :: Proxy s) ->
  let crcs :: UArray Word8 (GF a s)
      crcs = listArray (0,255) $ map (xtimes.xtimes.xtimes.xtimes.xtimes.xtimes.xtimes.xtimes.GF) [0..255]
      k (CRC p m) = runGF (GF _INIT * m + p + GF _FINAL)
      h a = CRC (crcs ! a) (x^8)
  in M k h mappend mempty
{-# INLINE crc #-}

-- | @x^32+x^26+x^23+x^22+x^16+x^12+x^11+x^10+x^8+x^7+x^5+x^4+x^2+x+1@
crc32 :: M Word8 Word32
crc32 = crc 0xffffffff 0xffffffff 0xedb88320

crc64_ecma182 :: M Word8 Word64
crc64_ecma182 = crc (-1) (-1) 0xC96C5795D7870F42

crc8 :: M Word8 Word8
crc8 = crc 0xff 0xff 0xab

crc8_sae :: M Word8 Word8
crc8_sae = crc 0xff 0xff 0xb8

data M a b where
  M :: (r -> b) -> (a -> r) -> (r -> r -> r) -> r -> M a b

instance Functor (M a) where
  fmap f (M k h m z) = M (f.k) h m z

instance Profunctor M where
  dimap f g (M k h m z) = M (g.k) (h.f) m z

instance Comonad (M a) where
  extract (M k _ _ z) = k z
  duplicate (M k h m z) = M (\n -> M (k . m n) h m z) h m z

runM :: Foldable f => M a b -> f a -> b
runM (M k h m z) xs = k (Foldable.foldr (m.h) z xs)

main = print $ runM crc32 [0x12,0x34]
```
@@@

Now that the algorithm is fully assembled, I can recognize aspects of it. 

I've seen a similar trick applied to turn a Rabin-Karp hash into a rolling hash. 

There the `Monoid` was the same, except we worked with a different ring `Z/nZ`, with `n` odd, and instead of `x^k`, we track `2^k mod n` and exploit the fact that 2 is coprime with n.

Spotting that connection means that this also provides a way to make an efficient rolling-hash function out of any standard CRC!

-[Edward Kmett](mailto:ekmett@gmail.com)

September 10th, 2013

P.S. I packaged `M` up over the weekend in the [`folds` package on hackage](http://hackage.haskell.org/package/folds), but I reserve the right to change the API and welcome feedback.

[Edit: It appears either [Kadatch or Jenkins](https://crcutil.googlecode.com/hg/doc/crc.pdf) figured out the same basic trick, which isn't surprising given how basic it is once you peel apart the math. They also identified the opportunity for a CRC-based rolling hash.]

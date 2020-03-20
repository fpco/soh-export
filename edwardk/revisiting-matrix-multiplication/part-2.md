# Part II: The Zen of Z-Ordering

If you haven't already, you'll want to read [Revisiting Matrix Multiplication, Part I](https://www.fpcomplete.com/user/edwardk/revisiting-matrix-multiplication-part-1) before proceeding.

I've posted the result from the first 2 No-Prizes inline in the original post, with the spoilers being hidden.

A Most Significant Difference
=============================

As anyone who read through [the wikipedia article on Z-order curves](http://en.wikipedia.org/wiki/Z-order_curve) for spoilers can tell you, it turns out we don't have to actually interleave the bits if all we want is the ability to compare two keys _as if_ they had been interleaved.

What we need to know is where the most significant difference between them occurs.

Given two halves of a key we can exclusive or them together to find the positions at which they differ.

I'll call the mask with only the most significant bit (`msb`) of this mask set their "most significant difference" (`msd`) and that position as indicating the "critical bit" if it exists.

One way to compute that most significant difference is to first "smear" all of the bits that are set in that mask to the right to get a new mask.  

```active haskell
import Data.Bits
import Data.Word
import Control.Lens
import Numeric.Lens
-- show
-- | @smear x@ returns the smallest @2^n-1 >= x@
smear :: Word64 -> Word64
smear k0 = k6 where
  k1 = k0 .|. unsafeShiftR k0 1
  k2 = k1 .|. unsafeShiftR k1 2
  k3 = k2 .|. unsafeShiftR k2 4
  k4 = k3 .|. unsafeShiftR k3 8
  k5 = k4 .|. unsafeShiftR k4 16
  k6 = k5 .|. unsafeShiftR k5 32
-- /show
-- | @msb x@ returns the largest @2^n <= x@
msb :: Word64 -> Word64
msb x = y `xor` unsafeShiftR y 1 
  where y = smear x

-- | @fat x y@ for @x < y@ returns the unique selection of @z = i*2^n@ 
-- in @x < z <= y@ that maximizes n. 
fat :: Word64 -> Word64 -> Word64
fat x y = complement (unsafeShiftR (smear (xor x y)) 1) .&. y

-- show
main = print $ base 16 # smear 0xf001030900
-- /show
```

And then we could shift them mask right one place and xor that with the original mask to get the most significant bit as a mask.

```active haskell
import Data.Bits
import Data.Word
import Control.Lens
import Numeric.Lens
-- | @smear x@ returns the smallest @2^n-1 >= x@
smear :: Word64 -> Word64
smear k0 = k6 where
  k1 = k0 .|. unsafeShiftR k0 1
  k2 = k1 .|. unsafeShiftR k1 2
  k3 = k2 .|. unsafeShiftR k2 4
  k4 = k3 .|. unsafeShiftR k3 8
  k5 = k4 .|. unsafeShiftR k4 16
  k6 = k5 .|. unsafeShiftR k5 32
-- | @msb x@ returns the largest @2^n <= x@
-- show
msb :: Word64 -> Word64
msb x = y `xor` unsafeShiftR y 1 
  where y = smear x
-- /show

-- | @fat x y@ for @x < y@ returns the unique selection of @z = i*2^n@ 
-- in @x < z <= y@ that maximizes n. 
fat :: Word64 -> Word64 -> Word64
fat x y = complement (unsafeShiftR (smear (xor x y)) 1) .&. y

-- show
main = do
  print $ base 16 # msb 0xf001030900
  print $ base 16 # msb (xor 0xff0 0xc00)
-- /show
```

> **No-Prize opportunity #3**
> 
> `__builtin_clz` can be used to count leading zeros and we can shift that many places. How big of a
> performance difference is there in doing it this way vs. doing this with the fiddly masks? I need benchmarks
> and performance numbers -- preferably as patches to that shiny new [sparse](https://github.com/ekmett/sparse)
> repository I mentioned last time as a patch to [bits](https://github.com/ekmett/bits) which provides `nlz`
> for the number of leading zeroes as part of a class in `Data.Bits.Extras`. Please feel free to send me pull requests if you find faster ways to do things! We'll need all the speed we can get soon.

The 2-Fattest Number
====================

(You can skip this on first-reading, but it'll be relevant later and we _just_ introduced all the bits needed to define it.)

There is another related quantity that we'll need soon when I turn to matrix multiplication that I'd like to point out. It is known as the 2-fattest number in an interval. The 2-fattest number in an interval is the number in that interval that has the largest number of trailing 0s in its binary representation. 

The first time I saw this term was in Djamal Belazzougui et al.'s excellent treatise on [Monotone Minimal Perfect Hashing](http://www.itu.dk/people/pagh/papers/sparse.pdf).

```active haskell
import Data.Bits
import Data.Word
import Control.Lens
import Numeric.Lens
-- | @smear x@ returns the smallest @2^n-1 >= x@
smear :: Word64 -> Word64
smear k0 = k6 where
  k1 = k0 .|. unsafeShiftR k0 1
  k2 = k1 .|. unsafeShiftR k1 2
  k3 = k2 .|. unsafeShiftR k2 4
  k4 = k3 .|. unsafeShiftR k3 8
  k5 = k4 .|. unsafeShiftR k4 16
  k6 = k5 .|. unsafeShiftR k5 32

-- | @msb x@ returns the largest @2^n <= x@
msb :: Word64 -> Word64
msb x = y `xor` unsafeShiftR y 1 
  where y = smear x

-- show
-- | @fat x y@ for @x < y@ returns the unique selection of @z = b*2^i@ 
-- in @x < z <= y@ that maximizes @i@. 
fat :: Word64 -> Word64 -> Word64
fat x y = complement (unsafeShiftR (smear (xor x y)) 1) .&. y
-- /show

-- show
main = print $ base 16 # fat 0x1 0xf
-- /show
```

The 2-fattest number for an interval has a number of interesting properties. 

1.) The 2-fattest number `z = b*2^i`in an interval `[x..y]` is uniquely determined by `x` and `y`. *Proof Sketch* If it were not, then since it the two candidates are odd, there'd exists an even number between them and hence also in the interval, but then that number would be of the form `b*2^(i+1)`, violating our claim that either of them was 2-fattest. Note the shift to talking about a closed interval rather than one open on the left.
 
2.) It is the least value in the interval for which their most significant differing bit goes from low to high. 

3.) If `y - x < 2^i` then there exists at most one `b` value for which `b*2^i` falls within the interval `[x..y]`.

4.) If `i` is such that `[x..y]` does not contain any value of the form `b*2^i` then `y - x + 1 <= 2^i - 1` and the interval may contain at most one single value of the form `b*2^(i-1)`.

Most if not all of these properties will be useful to us later on.

> **No-Prize opportunity #4**
> 
> Cleanly rewrite those laws in terms of `(x..y]`, so I can clean up the exposition above and be the first to send them to me.

The Only Winning Move
=====================

Now that we are so equipped with new bit twiddling tools, we can finally tackle comparing keys for relative ordering by their Morton order without interleaving their bits at all!

```haskell
data Key = Key {-# UNPACK #-} !Word32 {-# UNPACK #-} !Word32 deriving (Eq,Show,Read)
```

The lenses to access the fields become trivial.

```haskell
instance (a ~ Word32, b ~ Word32) => Field2 Key Key a b where
  _2 f (Key i j) = Key i <$> indexed f (1 :: Int) j
```
We now know how to compute the most significant difference between two key _components_. So to compare keys what we want to do is determine which key component has the most significant "most significant difference".

We could do that directly by computing the `msd` of each key component independently, and then comparing it, and then following up by comparing just the key component that won the toss. This is a fun exercise to do. It is also slightly more general than the construction I'm about to use, as I don't have to have keys with the same word size, and can support variable length keys so long as they have the [prefix property](http://en.wikipedia.org/wiki/Prefix_code) enabling me to work with compressed indices, and provides other benefits, but we can perform an equivalent operation with a lot less bit twiddling!

```haskell
instance Ord Key where
  compare (Key a b) (Key c d)
    | ac < bd && ac < xor ac bd = compare b d
    | otherwise                 = compare a c
    where
      ac = xor a c
      bd = xor b d
```

> **No-Prize opportunity #5**
> 
> Be the first to email me a proof of why this works and see your name immortalized as the proud owner of the 5th No-Prize!

This isn't a pure win as they are a number of operations that can still be performed on the shuffled representation somewhat faster, such as calculating `succ` in Morton order. 

Fun Exercise: What does a nice version of `succ` or `pred` that moves to the next address in Morton order look like in this representation?

However, this does provide a reference for how we can play around with identifying and using the "most significant most significant difference" in a useful way.

Putting it all together:

```active haskell
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE UndecidableInstances #-}
{-# LANGUAGE MultiParamTypeClasses #-}
import Control.Applicative
import Control.Lens
import Data.Bits
import Data.Word

data Key = Key {-# UNPACK #-} !Word32 {-# UNPACK #-} !Word32 deriving (Eq,Show,Read)

instance Ord Key where
  compare (Key a b) (Key c d)
    | ac < bd && ac < xor ac bd = compare b d
    | otherwise                 = compare a c
    where
      ac = xor a c
      bd = xor b d

instance (a ~ Word32, b ~ Word32) => Field1 Key Key a b where
  _1 f (Key a b) = (`Key` b) <$> indexed f (0 :: Int) a

instance (a ~ Word32, b ~ Word32) => Field2 Key Key a b where
  _2 f (Key a b) = Key a <$> indexed f (1 :: Int) b


main = putStrLn "it compiles!"
```

As an alternative to the comparison above, we could have checked the if `ac .&. msb (ac .|. bd) == 0`. But that relies on `msb` being particularly fast to calculate relative to a few `xor`s and inequalities. I'm dubious but available to be swayed by benchmarks. Also, I'll need to know if the msb is in an even or odd position later on, so any calculation of it that doesn't let me calculate the msb's position parity won't be very useful.

Omake: Dilated Arithmetic
=========================

We could continue to explore alternative alternatives.

When we extracted `_1` and `_2` from `Key` in the previous article, we bothered to shuffle them into and out of position.

It turns out that we can support a form of dilated arithmetic directly on them in their shuffled form. 

We could make newtype wrappers around a `Word64` for two different dilations.

```haskell
newtype DilatedEven = DilatedEven Word64
newtype DilatedOdd  = DilatedOdd Word64
```

Then there is a nice paper by Wise, Frens and Gu on [Language Support for Morton-order Matrices](http://www.cs.indiana.edu/~dswise/ppopp01.pdf) that describes how to directly perform dilated arithmetic where the bits you are interested are only in the even or odd bits of a larger number. They further talk about a number of compiler optimizations you may want to perform on these. Some of those optimizations could be implemented with `RULES` pragmas.

I'd be curious to see what anyone comes up with down this path of inquiry.

In practice we'll see a couple of numbers left in dilated form later, but I haven't (yet!) bothered to write newtypes for them or duplicate Wise et al's arithmetic operators for working on them directly.

Where Do We Go Now?
===================

We'll be abusing `smear`, this terminology, the notion of a "most significant most significant difference" and the 2-fattest number in a range when we start to talk about how we might want to decompose matrices. However, next time I need to take a slight detour into talking about how to make a custom `Vector` type!

-[Edward Kmett](mailto:ekmett@gmail.com)
August 16, 2013
# Part II: PNGs and Moore

[Last time](https://www.fpcomplete.com/user/edwardk/cellular-automata/part-1) I showed how we can render an automaton in your browser using existing tools.

This time we're going to roll a few of our own, so we can render fancier things. The SVG 
we generated last time was just too slow for many users and some folks complained that they couldn't see it at all on an iPad, or that it crashed Firefox. 

To rectify those concerns, we'll start off by writing a PNG generator!

Folds
=====

... but I'll take a bit of a circuitous path to get there.

A couple of weeks back, Gabriel Gonzales posted about his `foldl` library. In that he used the following type to capture the essence of a left fold:

```haskell
data Fold a b = forall x . Fold (x -> a -> x) x (x -> b)
```

I want to take a bit of a digression to note a few things about this type, and then show that it is just a presentation of something we already know pretty well in computer science!

Gabriel proceeded to supply an `Applicative` for his `Fold` type that looked something like:

```haskell
instance Functor (Fold a) where
  fmap f (Fold rar r rb) = Fold rar r (f.rb)

data Pair a b = Pair !a !b

instance Applicative (Fold a) where
  pure b = Fold (\() _ -> ()) () (\() -> b)
  {-# INLINABLE pure #-}
  Fold sas s0 s2f <*> Fold rar r0 r2x = Fold 
    (\(Pair s r) a -> Pair (sas s a) (rar r a)) 
    (Pair s0 r0) 
    (\(Pair s r) -> s2f s (r2x r))
  {-# INLINABLE (<*>) #-}
```

But there is actually a fair bit more we can say about this type!

Being `Applicative`, we can lift numeric operations directly into it:

```haskell
instance Num b => Num (Fold a b) where
  (+) = liftA2 (+)
  (-) = liftA2 (-)
  (*) = liftA2 (*)
  abs = fmap abs
  signum = fmap signum
  fromInteger = pure . fromInteger

instance Fractional b => Fractional (Fold a b) where
  recip = fmap recip
  (/) = liftA2 (/)
  fromRational = pure . fromRational
```

But we can also note that it is contravariant in its first argument and covariant in its second, and therefore it [must form](http://blog.sigfpe.com/2011/07/profunctors-in-haskell.html) a [`Profunctor`](http://hackage.haskell.org/packages/archive/profunctors/3.3.0.1/doc/html/Data-Profunctor.html).

```haskell
instance Profunctor Fold where
  dimap f g (Fold rar r0 rb) = Fold (\r -> rar r . f) r0 (g . rb)
```

All this does is let us tweak the inputs and/or outputs to our `Fold`.

But what perhaps isn't immediately obvious is that `Fold a` forms a `Comonad`!

```haskell
instance Comonad (Fold a) where
  extract (Fold _ r rb) = rb r
  duplicate (Fold rar r0 rb) = Fold rar r0 $ \r -> Fold rar r rb
```

Notice that the `duplicate :: Fold b a -> Fold b (Fold b a)` sneaks in and generates a nested fold _before_  the final tweak at the end that destroys our accumulator is applied! It works a bit like a last second pardon from the governor, a stay of execution if you will.

A Scary Digression
==================
(this is skippable)

It also forms a somewhat scarier sounding (strong) lax semimonoidal comonad, which just is to say that `(<*>)` is well behaved with regards to extract, so we can say:

```haskell
instance ComonadApply (Fold a) where
  (<@>) = (<*>)
```

This enables our `Comonad` to work with the `codo` sugar in Dominic Orchard's [`codo-notation`](http://hackage.haskell.org/package/codo-notation) package. I won't be doing that today, but you may want to download and modify one of the later examples to use it, just to get a feel for it. It is pretty neat.

Folding via Comonad Transformers
================================
(this part of mostly skippable too)

I'll get back to the actual usecases for this `Comonad` shortly, but first I want to start with ways I could have come up with the definition.

It turns out there are a few comonads very closely related to Gabriel's left `Fold`!

If we take the definition of Gabriel's `Fold` and rip off the existential, we get:

```haskell
data FoldX x a b = Fold (x -> a -> x) x (x -> b)
```

If we look through the menagerie supplied by the `comonad-transformers` package, we can pattern match on that with some effort and find:

`FoldX x a b` is isomorphic to both `EnvT (x -> a -> x) (Store x) b` and `StoreT x (Env (x -> a -> x)) b`. That it matches both of these types isn't surprising.

With `Monad` transformers, `State`, `Reader` and `Writer` all commute. In the space of `Comonad` transformers, `Store`, `Env`, and `Traced` all commute similarly.

`Store` is our old friend from the previous post, but `Env` and `EnvT` is something we haven't looked at before.

`Env` is also pretty much the easiest comonad to derive yourself.

Give it a shot!

```active haskell
{-# LANGUAGE DeriveFunctor, ScopedTypeVariables #-}
-- show
import Control.Comonad
-- /show
import Control.Exception
import Control.Monad
-- show
data Env e a = Env e a deriving (Eq,Ord,Show,Read,Functor)

instance Comonad (Env e) where
  -- extract :: Env e a -> a
  extract (Env e a) = error "unimplemented exercise"
  
  -- duplicate :: Env e a -> Env e (Env e a)
  duplicate (Env e a) = error "unimplemented exercise"
-- /show

main = do
  test "extract" $ extract (Env 1 2) == 2
  test "duplicate" $ duplicate (Env 1 2) == Env 1 (Env 1 2)
  
test :: String -> Bool -> IO ()
test s b = try (return $! b) >>= \ ec -> case ec of
  Left (e :: SomeException) -> putStrLn $ s ++ " failed: " ++ show e
  Right True -> putStrLn $ s ++ " is correct!"
  Right False -> putStrLn $ s ++ " is not correct!"
```

When we bolt an extra bit of environment onto our `Store` from the first part, we get

```haskell
data StoreAndEnv s e a = StoreAndEnv e (s -> a) s
```

If we fix `e = (s -> b -> s)`, we get

```haskell
data StoreAndStep s b a = StoreAndStep (s -> b -> s) (s -> a) s
```

then if we existentially tie off the `s` parameter to keep the end-user from fiddling with it we get back to

```haskell
data Fold a b = forall s. Fold (s -> b -> s) (s -> a) s
```

which we could shuffle around into the right place.

You can view tying off `s` as taking a coend if you are so categorically inclined.

It was somewhat unsatisfying that we had to take a coend and make something existential in that type. Can we do without it? 

It turns out we can, as noted by Elliott Hird, we just need to turn to another `Comonad`!

Moore Machines
==============

A [Moore machine](http://en.wikipedia.org/wiki/Moore_machine) is one of the two classic ways to represent a [deterministic finite automaton (DFA)](http://en.wikipedia.org/wiki/Deterministic_finite_automaton). The definition we'll use here is going to allow for deterministic infinite automata for free. 

That sort of thing happens a lot in Haskell.

A Moore machine gives you a result associated with each state in the automaton rather than each edge. We'll make the Moore machine itself represent the state implicitly.

```haskell
data Moore b a = Moore a (b -> Moore b a)
```

You can play around deriving its `extract` method below:


```active haskell
{-# LANGUAGE DeriveFunctor, ScopedTypeVariables #-}
-- show
import Control.Comonad
-- /show
import Control.Exception
import Control.Monad
-- show
data Moore b a = Moore a (b -> Moore b a) deriving Functor

instance Comonad (Moore b) where
  -- extract :: Moore b a -> a
  extract (Moore a as) = error "unimplemented exercise"
  
  -- duplicate :: Moore b a -> Moore b (Moore b a)
  duplicate w@(Moore _ as) = Moore w (duplicate <$> as)

  -- extend :: (Moore b a -> c) -> Moore b a -> Moore b c
  extend f w@(Moore _ as)  = Moore (f w) (extend f <$> as)
-- /show

main = do
  test "extract" $ 1 == extract (Moore 1 $ error "you don't need to look in the tail")
  
test :: String -> Bool -> IO ()
test s b = try (return $! b) >>= \ ec -> case ec of
  Left (e :: SomeException) -> putStrLn $ s ++ " failed: " ++ show e
  Right True -> putStrLn $ s ++ " is correct!"
  Right False -> putStrLn $ s ++ " is not correct!"
```

If you have an eye for this sort of thing, you may have noted that `Moore` is a `Cofree Comonad`!

That is to say, `Moore b a ~ Cofree ((->) b) a`.

`Moore` machines are supplied in my `machines` package.

We can also derive an `Applicative` for `Moore` and all the machinery from the `Fold` package, plus our new toys above.

Here is where I'd love to be able to say that, reformulating things in this simpler way pays off and everything gets faster from using this encoding. Alas, that is not to be. 

The `Moore` machine formulation is about 50% slower than the `Fold` representation in part due to the fact that it has hidden information about the environment for our machine from the optimizer. With `Fold`, the explicit `s` can be manipulated by the inliner very easily. 

Moreover applying an `fmap` is clearly done at the end, and so you pay no real cost for it until after the last iteration of the loop.

However, with the `Moore` representation, we pay for each `fmap`, because it winds up entangled in our core loop forever and we have to 'step over it' to get to the actual core of work we want to do. If we apply the co-`Yoneda` lemma to our Moore machine, we get

```haskell
data YonedaMoore a b = forall r. YonedaMoore (r -> a) (Moore a r)
```

Then you get rid of the overhead for each `fmap`, but we've brought back the existential and just made the optimizer's job harder.

What we do gain is flexibility in exchange for a bit of speed and no need for extensions.

A `Moore` machine can represent a mixture of strict and lazy left folds without extra boxes. The `Fold` type we started with can only represent one or the other easily, but otherwise must use a box around the intermediate value type. The choice is made when you go to apply the `Fold`. Gabriel has chosen (rightly) to focus on strict left folds.

With `Moore` we can define the embedding to either be lazy

```haskell
moorel :: (a -> b -> a) -> a -> Moore b a
moorel f = go where 
  go a = Moore a (go . f a)
```

or strict

```haskell
moorel' :: (a -> b -> a) -> a -> Moore b a
moorel' f = go where 
  go !a = Moore a (go . f a)
```

and because we don't have an explicit 's' parameter, we don't have to put a `Box` around it if we want the lazy version.

Then the kinds of combinators supplied by `Fold` can be implemented as

```haskell
total :: Num a => Moore a a
total = moorel' (+) 0

count :: Num a => Moore b a
count = moorel' (\a _ -> a + 1) 0
```

`Fold` and `Moore` are equivalent in expressive power, so another way to think about a `Fold` is as `Cofree ((->) a)` represented with an explicit seed in the style of `Nu` from my [`recursion-schemes`](http://hackage.haskell.org/package/recursion-schemes) package!

Feeding Machines and Folds
==========================

If we redefine our `Moore` machine using record syntax:

```haskell
data Moore b a = Moore { this :: a, less :: b -> Moore b a }
```

then we can run one of our `Moore` machines by continually calling `less` with new inputs and then extracting the answer for its final result state.

```haskell
more :: Foldable t => t b -> Moore b a -> a
more xs m = extract (F.foldl' less m xs)
```

Note that even though I'm using foldl' here, the thing that is being strictly updated is the Moore machine, not its member, which is only strict if you _built_ the Moore machine using `moorel'` above.

`more xs` is now a `Cokleisli` arrow for our `Comonad`, just like `rule 110` was for our `Store` `Comonad` in the last post.

We can construct a similar version of `more` for `Fold` using Gabriel's `fold` combinator.

```haskell
more :: Foldable t => t b -> Fold b a -> a
more xs m = extract (fold m xs)
```

What does the Comonad for Fold mean?
====================================

The `Comonad` for `Fold a` or `Moore` enables us to partially apply a `Fold` or `Moore` machine to some input and then resume it later.

If we `extend (more xs)` we get the ability to resume it with additional input, having partially driven our `Fold`!

If we turn to `(=<=)` from `Control.Comonad`.

```haskell
(=<=) :: Comonad w => (w b -> c) -> (w a -> b) -> w a -> c
f =<= g = f . extend g
```

Then we can express the laws for `more`:

```haskell
extract = more []
more as =<= more bs = more (as ++ bs)
```

So `more` provides us a monoid homomorphism between Cokleisli composition and concatenation.

Operationally, it sneaks in before you apply the last step to convert from your intermediate accumulator to the final result and lets you continue to do more work on the accumulator.

This strikes me as not intuitively obvious, because unless you look at it carefully, it isn't immediately obvious that you can resume something like a hash function because at the end, you usually tweak the result before giving it to the user. Here because we have access to the internals of the Comonad, we can `duplicate` them into the result before closing it off. 

This is where the explicit seed pays off, because that `duplicate` incurs no overhead during the actual traversal under Gabriel's representation.

This same existential construction works for `foldMap`- and `foldr`-based folds as well, though most of the "stream fusion" benefits require you to be able to stream and so `foldMap`-like structures, sadly, get little benefit.

Resuming a Hash Function
========================

Let us consider a couple of CRC-like functions, to have something non-trivial to fold.

```haskell
data Adler32 = Adler32 {-# UNPACK #-} !Word32 {-# UNPACK #-} !Word32

adler32 :: Moore Word8 Word32
adler32 = done <$> moorel' step (Adler32 1 0) where
  step (Adler32 s1 s2) x = Adler32 s1' s2' where
    s1' = mod (s1 + fromIntegral x) 65521
    s2' = mod (s1' + s2) 65521
  done (Adler32 s1 s2) = unsafeShiftL s2 16 + s1
```

In [Adler32](http://en.wikipedia.org/wiki/Adler-32), the final step of hashing destroys the separation of information between `s1` and `s2`, but we can sneak in with the comonad before we destroy it and resume!

Similarly, but less catastrophically, in [CRC32](http://en.wikipedia.org/wiki/Cyclic_redundancy_check) the final step is to complement the input.

```haskell
crc32 :: Moore Word8 Word32
crc32 = complement <$> moorel' step 0xffffffff where
  step r b = unsafeShiftR r 8 `xor` (crcs Unboxed.! fromIntegral (xor r (fromIntegral b) .&. 0xff))

crcs :: Unboxed.Vector Word32
crcs = Unboxed.generate 256 (go.go.go.go.go.go.go.go.fromIntegral) where
  go c = unsafeShiftR c 1 `xor` if c .&. 1 /= 0 then 0xedb88320 else 0
```

We can describe similar `Moore` machines (or `Fold`s) for common hashing functions, and then we don't need to make up separate functions for initializing the state, feeding them some incremental additional data and finally cleaning up when we're done.

The `Moore` machine provides you with all of that, and the entire API necessary to interact with them comes down to feeding it `more`, extending after doing so to accept more input!

This strikes me as an incredibly clean implementation pattern for HMACs such as [MD5](http://en.wikipedia.org/wiki/MD5) and [SHA](http://en.wikipedia.org/wiki/SHA-1) in Haskell. You don't need to name 3 separate pieces. 

You just name the HMAC itself as the Moore machine that produces it. Then you can feed it `more` data, extending it as needed until you finally go to look at the last result.

Uncompressed PNGs
=================

So let's put our code where our mouth is and show that we can use this to do some software engineering by writing some code to produce a [PNG](http://en.wikipedia.org/wiki/Portable_Network_Graphics) image from scratch in Haskell.

A bit over a year ago, Keegan McAllister wrote a nice [post](http://mainisusuallyafunction.blogspot.com/2012/04/minimal-encoder-for-uncompressed-pngs.html) on how to generate a minimal uncompressed PNG using python. We'll copy his development here, except we'll switch out to the nicer table-based crc32 above.

As he noted, you need to implement two hash functions to actually get through writing an uncompressed PNG. Hrmm. We appear to have those.

We'll use `Data.Binary` to write out the results, mostly because PNG is an annoyingly introspective format, so we'll have to talk about the lengths of fragments we're generating as we go.

We can write the ability to put a PNG 'chunk' out, which consists of a 4 byte header followed by some data, but which first encodes the length of just the data, then emits the header, then the data, and finally closes off the chunk with the CRC32 of both.

Let's generalize more to work over any `Fold` (in the `lens` sense this time!) that yields the input type.

```haskell
moreOf :: Getting (Endo (Endo (Moore b a))) s b -> s -> Moore b a -> a
moreOf l xs m = extract (foldlOf' l less m xs)
```

That somewhat baroque seeming type can be read as a more liberal version of:

```haskell
moreOf :: Fold s b -> s -> Moore b a -> a
```

that just happens to get better inference due to the lack of rank-2 types.

Now we can use it directly on the lazy bytestring fragments we get along the way

```haskell
putChunk :: Lazy.ByteString -> Put -> Put
putChunk h (runPut -> b) = do
  putWord32be $ fromIntegral (Lazy.length b)
  putLazyByteString h
  putLazyByteString b
  putWord32be $ moreOf bytes h =<= moreOf bytes b $ crc32
```

To write out a PNG file, we need to be able to emit the `IHDR` chunk, 1 or more `IDAT` chunks of zlib compressed data, and an `IEND` chunk.

We can break up our zlib data into uncompressed blocks. However, zlib only allows uncompressed runs of 64k at a time, so we let's define the encoding for a nested uncompressed deflate block.

```haskell
deflated :: Bool -> Lazy.ByteString -> Put
deflated final b | l <- fromIntegral (Lazy.length b) = do
  putWord8 $ if final then 1 else 0
  putWord16le l -- yep, now it's little endian!
  putWord16le (complement l)
  putLazyByteString b
```

Then we just rip our input up into 64k blocks, embed each of those blocks in one enormous `IDAT` block, then finally seal everything up with the `Adler32` checksum that we so helpfully supplied as an example above!

```haskell
zlibbed :: Lazy.ByteString -> Put
zlibbed bs = do
  putWord8 0x78
  putWord8 0x01
  go bs
  putWord32be $ moreOf bytes bs adler32
  where
    go (Lazy.splitAt 0xffff -> (xs, ys)) | done <- Lazy.null ys = do
      deflated done xs
      M.unless done (go ys)
```

Now we can write out a PNG header, loop through the data, state that we're not applying any transformation for each row:

```haskell
png :: Int -> [Int -> (Word8, Word8, Word8)] -> Lazy.ByteString
png w fs = runPut $ do
  putLazyByteString "\x89PNG\r\n\x1a\n"
  putChunk "IHDR" $ do
    putWord32be $ fromIntegral w
    putWord32be $ fromIntegral (List.length fs)
    putWord8 8 -- 8 bit color depth
    putWord8 2 -- RGB
    putWord8 0
    putWord8 0
    putWord8 0
  putChunk "IDAT" $ zlibbed (runPut rows)
  putChunk "IEND" $ return ()
  where
    rows = forM_ fs $ \f -> do
      putWord8 0
      forM_ [0..w-1] (put . f)
```

Here I've chosen to tell the PNG the width, but leave height implicit in the length of the list of functions from horizontal position to pixel color. I may revisit that later, but it was the fastest thing I could think of to write.

This lets `png` nicely fit into the recursion pattern from the previous post.

But we've written a lot of code, so it'd be nice to check that we generated a valid PNG.

```active haskell web
-- show
{-# LANGUAGE BangPatterns #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE ViewPatterns #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TypeFamilies #-}
{-# OPTIONS_GHC -Wall #-}
import Control.Applicative
import Control.Comonad
import Control.Lens
import qualified Control.Monad as M
import Data.Bits
import Data.Binary
import Data.Binary.Put
import qualified Data.ByteString.Lazy as Lazy
import Data.ByteString.Lens
import Data.Monoid
import qualified Data.Vector.Unboxed as Unboxed
import Data.Foldable as F
import Data.List as List
import Yesod

-- * Moore machines

data Moore b a = Moore { this :: a, less :: b -> Moore b a }

instance Num a => Num (Moore b a) where
  (+) = liftA2 (+)
  (-) = liftA2 (-)
  (*) = liftA2 (*)
  abs = fmap abs
  signum = fmap signum
  fromInteger = pure . fromInteger

instance Fractional a => Fractional (Moore b a) where
  recip = fmap recip
  (/) = liftA2 (/)
  fromRational = pure . fromRational

instance Functor (Moore b) where
  fmap f = go where go (Moore a k) = Moore (f a) (go . k)

instance Comonad (Moore b) where
  extract = this
  duplicate w@(Moore _ as) = Moore w (duplicate . as)

instance ComonadApply (Moore b) where
  (<@>) = (<*>)

instance Applicative (Moore b) where
  pure a = as where as = Moore a (const as)
  Moore f fs <*> Moore a as = Moore (f a) $ \b -> fs b <*> as b

instance Profunctor Moore where
  dimap f g (Moore a as) = Moore (g a) (dimap f g . as . f)

moorel :: (a -> b -> a) -> a -> Moore b a
moorel f = go where go a = Moore a (go . f a)

moorel' :: (a -> b -> a) -> a -> Moore b a
moorel' f = go where go !a = Moore a (go . f a)

moreOf :: Getting (Endo (Endo (Moore b a))) s b -> s -> Moore b a -> a
moreOf l xs m = extract (foldlOf' l less m xs)

-- * Adler 32

data Adler32 = Adler32 {-# UNPACK #-} !Word32 {-# UNPACK #-} !Word32

adler32 :: Moore Word8 Word32
adler32 = done <$> moorel' step (Adler32 1 0) where
  step (Adler32 s1 s2) x = Adler32 s1' s2' where
    s1' = mod (s1 + fromIntegral x) 65521
    s2' = mod (s1' + s2) 65521
  done (Adler32 s1 s2) = unsafeShiftL s2 16 + s1

-- * CRC32

crc32 :: Moore Word8 Word32
crc32 = complement <$> moorel' step 0xffffffff where
  step r b = unsafeShiftR r 8 `xor` crcs Unboxed.! fromIntegral (xor r (fromIntegral b) .&. 0xff)

crcs :: Unboxed.Vector Word32
crcs = Unboxed.generate 256 (go.go.go.go.go.go.go.go.fromIntegral) where
  go c = unsafeShiftR c 1 `xor` if c .&. 1 /= 0 then 0xedb88320 else 0

-- * PNG

putChunk :: Lazy.ByteString -> Put -> Put
putChunk h (runPut -> b) = do
  putWord32be $ fromIntegral (Lazy.length b)
  putLazyByteString h
  putLazyByteString b
  putWord32be $ moreOf bytes h =<= moreOf bytes b $ crc32

deflated :: Bool -> Lazy.ByteString -> Put
deflated final b | l <- fromIntegral (Lazy.length b) = do
  putWord8 $ if final then 1 else 0
  putWord16le l -- yep, now it's little endian!
  putWord16le (complement l)
  putLazyByteString b

zlibbed :: Lazy.ByteString -> Put
zlibbed bs = do
  putWord8 0x78
  putWord8 0x01
  go bs
  putWord32be $ moreOf bytes bs adler32
  where
    go (Lazy.splitAt 0xffff -> (xs, ys)) | done <- Lazy.null ys = do
      deflated done xs
      M.unless done (go ys)

png :: Int -> [Int -> (Word8, Word8, Word8)] -> Lazy.ByteString
png w fs = runPut $ do
  putLazyByteString "\x89PNG\r\n\x1a\n"
  putChunk "IHDR" $ do
    putWord32be $ fromIntegral w
    putWord32be $ fromIntegral (List.length fs)
    putWord8 8 -- 8 bit color depth
    putWord8 2 -- RGB
    putWord8 0
    putWord8 0
    putWord8 0
  putChunk "IDAT" $ zlibbed (runPut rows)
  putChunk "IEND" $ return ()
  where
    rows = forM_ fs $ \f -> do
      putWord8 0
      forM_ [0..w-1] (put . f)

-- * Yesod

data App = App

instance Yesod App

mkYesod "App" [parseRoutes| / ImageR GET |]

main :: IO ()
main = warpEnv App
-- /show

-- show
getImageR :: MonadHandler m => m TypedContent
getImageR = sendResponse $ toTypedContent (typePng, toContent img) where
  img = png 500 $ take 300 $ pixel <$> [0..]
  pixel y x = (fromIntegral x,fromInteger y,0)
-- /show
```

That image matches up byte for byte with the output of Keegan's sample, so we seem to have an end-to-end test that works.

A lot of this code is redundant, however.

For instance all of the `Moore` code could be taken from the `machines` package, which provides `Data.Machine.Moore` along with all of these instances! Then with a bit of tightening of exposition and removing unnecessary detours we could generate the whole thing in a lot less code.

Automata, Please
================

Of course, this is supposed to be a series about cellular automata. So let's draw one.

1.) I'll be switching to a 4 line minimalist version of Gabriel's `foldl` library, rather than using the `Moore` representation, since we don't need any of the instances. I've renamed his `Fold` to `L` here to avoid conflicts with the `Lens` library.

2.) We don't _need_ to use the `Comonad` for the fold type we spent all that time above building up. Here we're working with lazy bytestrings, so let's just append them in the one case we need!

2.) I'll also be using the `Context` comonad from the `lens` package rather than continuing to roll our own `Store`. That'll be useful next time when I want to abuse the separate indices.

3.) 
  I've tweaked the memoization rule to use

  ```haskell
  loop f = iterate (tab . extend f) . tab
  ```

  instead of 

  ```haskell
  loop f = iterate (extend f . tab) 
  ```

  to get slightly better memoization. I also switched to `representable-tries`, 
  because it'll make it easier to switch to new topologies later.

4.) Finally, to reduce the footprint of the PNGs we generate we'll let the existing `zlib` bindings for Haskell do the compression rather than manually deflate. This reduces the footprint of the generated images a great deal.

Click Run!

```active haskell web
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE ExistentialQuantification #-}
{-# LANGUAGE Rank2Types #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TypeFamilies #-}
import Control.Applicative
import Control.Comonad
import Codec.Compression.Zlib
import Control.Lens.Internal.Context
import Control.Lens as L
import Data.Bits
import Data.Bits.Lens as L
import Data.Monoid
import Data.Binary
import Data.Binary.Put
import qualified Data.ByteString.Lazy as Lazy
import qualified Data.Vector.Unboxed as Unboxed
import Data.Foldable as F
import Data.MemoTrie
import Yesod

rule :: Num s => Word8 -> Context s s Bool -> Bool
rule w (Context f s) = testBit w $ 0 L.& partsOf L.bits .~ [f (s+1), f s, f (s-1)]

loop :: HasTrie s => (Context s s a -> a) -> Context s s a -> [Context s s a]
loop f = iterate (tab . extend f) . tab where
  tab (Context k s) = Context (memo k) s

data L b a = forall x. L (x -> b -> x) x (x -> a)

more :: Lazy.ByteString -> L Word8 a -> a
more bs (L xbx x xa) = xa (Lazy.foldl' xbx x bs)

crc32 :: L Word8 Word32
crc32 = L step 0xffffffff complement where
  step r b = unsafeShiftR r 8 `xor` crcs Unboxed.! fromIntegral (xor r (fromIntegral b) .&. 0xff)

crcs :: Unboxed.Vector Word32
crcs = Unboxed.generate 256 (go.go.go.go.go.go.go.go.fromIntegral) where
  go c = unsafeShiftR c 1 `xor` if c .&. 1 /= 0 then 0xedb88320 else 0

putChunk :: Lazy.ByteString -> Lazy.ByteString -> Put
putChunk h b = do
  putWord32be $ fromIntegral (Lazy.length b)
  putLazyByteString h
  putLazyByteString b
  putWord32be $ more (h <> b) crc32

png :: Int -> Int -> [Int -> (Word8, Word8, Word8)] -> Lazy.ByteString
png w h fs = runPut $ do
  putLazyByteString "\x89PNG\r\n\x1a\n"
  putChunk "IHDR" $ runPut $ do
    putWord32be (fromIntegral w)
    putWord32be (fromIntegral h)
    putWord8 8 -- 8 bit color depth
    putWord8 2 -- RGB
    putWord8 0
    putWord8 0
    putWord8 0
  putChunk "IDAT" $
    compressWith defaultCompressParams { compressLevel = bestSpeed } $
    runPut $ forM_ (take h fs) $ \f -> do
      putWord8 0
      forM_ [0..w-1] (put . f)
  putChunk "IEND" mempty

data App = App
instance Yesod App
mkYesod "App" [parseRoutes| / ImageR GET |]
main = warpEnv App
-- /show

-- show
getImageR :: MonadHandler m => m TypedContent
getImageR = sendResponse $ toTypedContent (typePng, toContent img) where
  img = png 150 150 $ draw <$> loop (rule 110) (Context (==149) 149)
  draw (Context p _) x = if p x then (0,0,0) else (255,255,255)
-- /show
```

That weighs in somewhere around 75 lines, and includes our compressed PNG generator, all the logic for running Wolfram's 2-color rules as before, and our embedded Yesod server. You can feel free to tweak the output above.

In the real world you'd probably just use [JuicyPixels](http://hackage.haskell.org/packages/archive/JuicyPixels/3.1/doc/html/Codec-Picture-Png.html).

Now that we're not shackled by the SVG rendering speed we can generalize this to other topologies and maybe try to improve on our other bottlenecks in future updates.

-[Edward Kmett](mailto:ekmett@gmail.com)

September 1, 2013
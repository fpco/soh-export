# Part III: A World in a Bottle

In [Part I](https://www.fpcomplete.com/user/edwardk/cellular-automata/part-1), I showed how we can build up cellular automata in Haskell and render them to the web as custom SVGs. In [Part II](https://www.fpcomplete.com/user/edwardk/cellular-automata/part-2), I replaced the SVG writer with a hand-rolled PNG writer.

The last article got lost in the weeds playing with PNG writing and the comonadic structure of folds. This time around I want to go back to focusing on the automata themselves. If you haven't read those yet, I'd highly recommend at least skimming at least the first one to familiarize yourself with its contents before proceeding. 

One of the issues I raised back in part I was that `Store` was in some sense too big to describe automata because the automata could know and use their absolute position information when computing their answers. I'd like to fix that.

Another thing I hinted at was that it was possible to build automata on strange topologies. I'd like to show how we can enable interesting topologies through the very act of fixing the previous problem.

Moves like Jagger
=================

The problem that we had was that the `Store`/`Context` comonad gave you direct access to the current location. So nothing prevents a rule from acting very differently in position 34 than it does anywhere else in the space.

We can fix that by introducing a notion of a relative movement rather than just having absolute positioning.

Now, we an define our notion of a rule as something that can use relative position information alone, and can combine answers to questions about nearby locations to generate a local answer.

```haskell
type Rule m a = (m -> a) -> a
```

Under this scheme, our old 2-color Wolfram-style [elementary cellular automata](http://mathworld.wolfram.com/ElementaryCellularAutomaton.html) rules look like:

```haskell
data Move = R | S | L deriving (Eq, Enum, Bounded, Show, Read)

rule :: Word8 -> Rule Move Bool
rule w f = testBit w $ 0 L.& partsOf L.bits .~ (f <$> [minBound .. maxBound])
```

If we want to increase the "speed of light" we can make a new `Move` type, and generalize `rule`.

```haskell
data Move2 = RR | R | S | L | LL deriving (Eq, Enum, Bounded, Show, Read)

rule :: (Enum m, Bounded m, Num n, Bits n) => n -> Rule m Bool
rule w f = testBit w $ 0 L.& partsOf L.bits .~ (f <$> [minBound .. maxBound])
```

Now `rule` can work for pretty much any enumerable, bounded move type and provide you with Wolfram-like rules.

Now we just need some way to act on a `Rule`!

Act Upon The World
==================

It turns out it is better to think of our rules as acting upon our automaton's topology rather than vice versa. 

We can do that by describing the ["action"](http://en.wikipedia.org/wiki/Semigroup_action) of our movements on our position. If our movements were always composable, then we would want this to be a "semigroup action", or even a "monoid action" if we had a unit. If you are curious to know more about monoid actions, I'd encourage you to read [Brent's Pearl](http://www.cis.upenn.edu/~byorgey/pub/monoid-pearl.pdf), but then if you've been following all of the myriad links I've been throwing out from the beginning, then you may already have done so. I cited it for other reasons back in Part I.

However, since we want to ensure we have a discrete "speed of light" governing information travel in our automaton and we don't want to deal with relativistic effects, for now, let's just not let them compose for now.

```haskell
type Act m s = m -> s -> s
```

If we wanted to be more correct, we should likely have different universe types for each action, but it'll suffice to let the action define our topology instead.

An action transforms a move into a function that transforms the current location.

```haskell
flat :: Act Move Int
flat L i = i-1
flat S i = i
flat R i = i+1
```

Another example topology for the first `Move` type we started with would be to just treat the movement as relative in a world.

```haskell
modulo :: Int -> Act Move Int
modulo n m i = flat m i `mod` n
```

Now we just need to define the step function:

I Dream of Genie
================

```haskell
step :: Act m s -> Rule m a -> Context s s a -> a
```

Let's say for a minute that we didn't know how to write this! What could we do?

Back in 2005, Lennart Augustsson [wrote](http://permalink.gmane.org/gmane.comp.lang.haskell.general/12747) a wonderful tool named `djinn` for doing just this sort of thing.

From the release announcement:

> For the curious, Djinn uses a decision procedure for intuitionistic
propositional calculus due to Roy Dyckhoff.  It's a variation of
Gentzen's LJ system.  This means that (in theory) Djinn will always
find a function if one exists, and if one doesn't exist Djinn will
terminate telling you so.

So let's use it!

On [irc.freenode.net](webchat.freenode.net/?channels=haskell&uio=d4), our well-loved mechanical assistant `lambdabot` has a version of `djinn` installed, which is available via the `@djinn` command.

```haskell
[03:35] edwardk:   @djinn a -> a
[03:35] lambdabot: f a = a
[03:35] edwardk:   @djinn a -> b -> c
[03:35] lambdabot: -- f cannot be realized.
[03:35] edwardk:   @djinn a -> b -> a
[03:35] lambdabot: f a _ = a
```

She doesn't know anything about our `Context`, though, so let's help her out.

```haskell
[03:35] edwardk: @djinn-add data Context a b t = Context (b -> t) a
```

Now she's fully capable of deriving for us the definitions for extract:

```haskell
[03:37] edwardk:   @djinn Context a a t -> t
[03:37] lambdabot: f a =
[03:37] lambdabot:  case a of
[03:37] lambdabot:  Context b c -> b c
```

and even how to `extend` `Context` as an indexed comonad:

```haskell
[03:39] edwardk:   @djinn (Context b c t -> r) -> Context a c t -> Context a b r
[03:39] lambdabot: f a b =
[03:39] lambdabot:  case b of
[03:39] lambdabot:  Context c d -> Context (\ e -> a (Context c e)) d
```

So, let's teach her about rules and actions.

```haskell
[03:42] edwardk: @djinn-add type Rule m a = (m -> a) -> a
[03:42] edwardk: @djinn-add type Act m s = m -> s -> s
```

and ask her for a definition for `step`.

```haskell
[03:43] edwardk:   @djinn Act m s -> Rule m a -> Context s s a -> a
[03:43] lambdabot: f _ a b =
[03:43] lambdabot:  case b of
[03:43] lambdabot:  Context c d -> a (\ _ -> c d)
```

Ack! Something went wrong. 

What happened was that the problem was under-constrained. 

She didn't have to apply the action, so she didn't. 

Similarly, if we try for the unindexed version of `extend`, we run into the same problem!

```haskell
[03:45] edwardk:   @djinn (Context a a t -> r) -> Context a a t -> Context a a r
[03:45] lambdabot: f a b =
[03:45] lambdabot:	case b of
[03:45] lambdabot:	Context c d -> Context (\ e -> a (Context (\ _ -> c d) e)) d
```

So, let's split apart the positive and negative uses of 's' in our original problem.

```haskell
[03:46] edwardk:   @djinn-add type Acts m s t = m -> s -> t
[03:46] edwardk:   @djinn Acts m s t -> Rule m a -> Context s t a -> a
[03:46] lambdabot: f a b c =
[03:46] lambdabot:  case c of
[03:46] lambdabot:  Context d e -> b (\ f -> d (a f e))
```

There we have it!

```haskell
step :: Act m s -> Rule m a -> Context s s a -> a
step a b (Context d e) = b (\f -> d (a f e))
```

If we want to clean that up a bit:

```haskell
step :: Act m s -> Rule m a -> Context s s a -> a
step top r (Context f s) = r (f . flip top s)
```

Thats All For Now
=================

Putting all of that together a greyscale version of the PNG writer from my Mandelbrot snippet yields the following code.

Click Run!

```active haskell web
{-# LANGUAGE RankNTypes #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE DeriveFunctor #-}
import Codec.Compression.Zlib
import Control.Comonad
import Control.Lens as L
import Control.Lens.Internal.Context
import Data.Binary
import Data.Binary.Put
import Data.Bits
import Data.Bits.Lens as L
import Data.ByteString.Lazy as Lazy
import Data.Foldable as F
import Data.List as List
import Data.MemoCombinators
import Data.Monoid
import Data.Vector.Unboxed as Unboxed
import Yesod

-- show
type Rule m a = (m -> a) -> a

data Move = R | S | L deriving (Eq, Enum, Bounded, Show, Read)

rule :: (Enum m, Bounded m) => Word8 -> Rule m Bool
rule w f = testBit w $ 0 L.& partsOf L.bits .~ (f <$> [minBound .. maxBound])

type Act m s = m -> s -> s

modulo :: Int -> Act Move Int
modulo m L i = (i-1) `mod` m
modulo _ S i = i
modulo m R i = (i+1) `mod` m
-- /show

step :: Act m s -> Rule m a -> Context s s a -> a
step top r (Context f s) = r (f . flip top s)

loop :: Integral s => (Context s s a -> a) -> Context s s a -> [Context s s a]
loop f = List.iterate (tab . extend f) . tab where
  tab (Context g s) = Context (integral g) s

run :: Word8 -> Int -> Int -> [[Word8]]
run r x m0 = fmap line $ loop (step (modulo m0) (rule r)) $ Context (==0) 0 where
  line = fmap bw . window (x `div` 2)
  bw True  = 0
  bw False = 255
  window w = iexperiment $ \ s -> [s-w..s+w]

crc32 :: Lazy.ByteString -> Word32
crc32 = complement . Lazy.foldl' f 0xffffffff where
  f r b = unsafeShiftR r 8 `xor` crcs Unboxed.! fromIntegral (xor r (fromIntegral b) .&. 0xff)
  crcs = Unboxed.generate 256 (go.go.go.go.go.go.go.go.fromIntegral)
  go c = unsafeShiftR c 1 `xor` if c .&. 1 /= 0 then 0xedb88320 else 0

putChunk :: Lazy.ByteString -> Lazy.ByteString -> Put
putChunk h b = do
  putWord32be $ fromIntegral (Lazy.length b)
  putLazyByteString h
  putLazyByteString b
  putWord32be $ crc32 (h <> b)

png :: Int -> Int -> [[Word8]] -> Lazy.ByteString
png w h fs = runPut $ do
  putLazyByteString "\x89PNG\r\n\x1a\n"
  putChunk "IHDR" $ runPut $ do
    putWord32be (fromIntegral w)
    putWord32be (fromIntegral h)
    putWord8 8 -- 8 bit
    putWord8 0 -- greyscale
    putWord8 0
    putWord8 0
    putWord8 0
  putChunk "IDAT" $
    compressWith defaultCompressParams { compressLevel = bestSpeed } $
    runPut $ F.forM_ (Prelude.take h fs) $ \xs -> do
      putWord8 0
      F.forM_ (Prelude.take w xs) put
  putChunk "IEND" mempty

data App = App
instance Yesod App
mkYesod "App" [parseRoutes| / ImageR GET |]

main :: IO ()
main = warpEnv App

-- show
getImageR :: MonadHandler m => m TypedContent
getImageR = sendResponse $ toTypedContent (typePng, toContent img) where
  img = png 280 280 $ run 110 280 30
-- /show
```

With all of that we're still well under a hundred lines.

We're not limited to simple small world topologies, but if you want to connect random points in space, you're probably better off doing so in a 2d automaton, simply because there are more interesting combinations.

In the interest of full disclosure, Djinn isn't perfect. It can't deal with rank-n types. It also doesn't really understand typeclasses as they behind the scenes _also_ involve rank-n types. It is however, an incredibly useful tool that helps showcase the power of [free theorems](http://ttic.uchicago.edu/~dreyer/course/papers/wadler.pdf) to constrain down the space of possible implementations to the point where only reasonable programs can type check at all. 

By making our programs _more_ generic we are able to restrict them to fewer possible implementations, leaving us with only one reasonable thing that typechecks.

-[Edward Kmett](mailto:ekmett@gmail.com)

September 15th, 2013








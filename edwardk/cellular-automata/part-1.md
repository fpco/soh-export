# Part I: From Theory to Pretty Pictures

Cellular automata are one of the "go to" examples for comonads in Haskell.

Dan Piponi wrote his article on [using comonads to evaluate cellular automata](http://blog.sigfpe.com/2006/12/evaluating-cellular-automata-is.html) back in 2006, and that was pretty much my introduction to comonads in general. He used a [list zipper](http://en.wikipedia.org/wiki/Zipper_%28data_structure%29).

Today, I want to use something a little bit more general and maybe draw some pictures.

Minding The Store
=================

To that end, let's define the `Store` `Comonad`.

```active haskell
{-# LANGUAGE DeriveFunctor #-}
import Control.Comonad

-- show
data Store s a = Store (s -> a) s deriving Functor

instance Comonad (Store s) where
  extract (Store f s) = f s
  duplicate (Store f s) = Store (Store f) s
-- /show
  
experiment :: Functor f => (s -> f s) -> Store s a -> f a
experiment k (Store f s) = f <$> k s 

main = putStrLn "It typechecks, so it must be correct!"
```

A `Store s a` describes some "test" that takes a configuration `s` and will produce a value of type `a`, where we also have some ambient initial configuration of type `s` that is known with which we could start the experiment.

The `experiment` combinator characterizes a `Store` completely. It lets you explore variations on the initial conditions of our test.

```haskell
experiment :: Functor f => (s -> f s) -> Store s a -> f a
experiment k (Store f s) = f <$> k s 
```

`Store` gives you a little bit more power than we want in a cellular automaton, as you can do both relative _and_ global addressing, but it happens to be a very general construction, so we'll start there. It has the benefit that if we decide we want to play with automata in more than 1 dimension all we have to do is change out the state type.

The `Store` comonad has a lot of different uses that aren't immediately obvious. It is used heavily inside of the `lens` library.

A Glimpse Down the Rabbit Hole
==============================

_(This section is completely skippable and is included as a highly technical aside)_

An interesting exercise for the advanced Haskeller is to flip the definition of `experiment` and take that as the definition for `Store`.

```active haskell
{-# LANGUAGE DeriveFunctor #-}
{-# LANGUAGE RankNTypes #-}
import Control.Comonad
-- show
newtype Pretext s a = Pretext { 
    runPretext :: forall f. Functor f => (s -> f s) -> f a 
  } deriving Functor

experiment :: Functor f => (s -> f s) -> Pretext s a -> f a
experiment f (Pretext k) = k f
-- /show
main = putStrLn "It typechecks, so it must be correct!"
```

Defining the `Comonad` instance for that type is a particularly enlightening challenge.

If you replace the `Functor` constraint in the definition above with `Applicative` you get a `Comonad` I call the `Bazaar`. This `Comonad` is used to derive many of the most brain-bending `Traversal` and `uniplate`-derived combinators in `lens`. 

The code for its `Comonad` instance is identical to the instance for `Pretext` above, but it can also be made `Applicative`.

```active haskell
{-# LANGUAGE DeriveFunctor #-}
{-# LANGUAGE RankNTypes #-}
import Control.Comonad
import Control.Applicative
-- show
newtype Bazaar s a = Bazaar { 
    runBazaar :: forall f. Applicative f => (s -> f s) -> f a 
  } deriving Functor
-- /show
main = putStrLn "It typechecks, so it must be correct!"
```

If you try to search for the `Store`-like analogue to the `Bazaar`, you wind you looking at what Twan van Laarhoven called a `FunList` in ["A non-regular data type challenge"](http://twanvl.nl/blog/haskell/non-regular1). 

```active haskell
{-# LANGUAGE DeriveFunctor #-}
import Control.Comonad
import Control.Applicative
-- show
data FunList s a
    = Done a
    | More s (FunList s (s -> a))
    deriving Functor
-- /show
main = putStrLn "It typechecks, so it must be correct!"
```

An interesting exercise is to derive the `Applicative` and `Comonad` instances for `FunList`. This exercise is much easier than the `Pretext` and `Bazaar` derivations, but still quite challenging.

Surprisingly `FunList` is actually a less powerful type than `Bazaar` in the presence of infinite traversals as many tools you can build will not terminate when you manipulate an infinite traversal with them built using a `FunList`, but _will_ terminate when they are constructed using the `Bazaar`!

Following the Rules
===================

Stephen Wolfram described a rather concise encoding of 2-color automata that can only look at their neighbors in ["A New Kind of Science"](http://www.wolframscience.com/).

We can encode his family of 2-color rules as a comonadic action:

```haskell
rule :: Num s => Word8 -> Store s Bool -> Bool
rule w (Store f s) = testBit w $ 
  0 & partsOf (taking 3 bits) .~ [f (s+1), f s, f (s-1)]
```

That is rather dense, so let's unpack it.

Wolfram numbers his rules from 0 to 255 because if you look at the current cell and the neighbor to the left and right of it, we have 3 inputs to consider. Each is a `Bool` so we have 2^3 different results to give. If we bundle all those possible results together as the bits of a `Word8`, the `Word8` perfectly describes all of the possible 2-color cellular automata that can look at the current and neighboring cells.

So now the trick is doing that indexing. To do so, first we need to figure out which bit in our `Word8` we are interested in. To do that we need to use the 3 booleans we obtain by tweaking our position and asking to perform our "experiment" there at the slightly modified positions instead.

Now we want to compose 3 bits together into an `Int`. We could do this with a bunch of conditional logic, etc. but there is a slightly cute encoding we can get when we use `lens`.

`bits` provides a `Traversal` of the individual bits of any instance of `Bits`. (In the case of `Integer`, though, because it is infinite sadly the `Traversal` can never finish reassembling the `Integer`, and so it devolves to merely a `Fold`.

`taking n t` takes a `Traversal t` and yields a `Traversal` that only touches the first `n` targets of the original `Traversal`.

Therefore `taking 3 bits` is the `Traversal` of the first 3 bits of a number.

`partsOf` takes a `Traversal` and gives you a (slightly hinky) `Lens` to a list of all of the targets of the traversal. You can freely replace that list with a new list (of the same length!). It is only a law abiding `Lens` if you do not change the length of the list of targets, but even if you violate these assumptions it is well behaved operationally. In fact you can safely remove `taking n` from the definition of rule above, and its semantics do not change.

And finally, we can use the fact that every `Lens` is a valid `Setter` to make the assignment.

```haskell
0 & partsOf (taking 3 bits) .~ [f (s+1), f s, f (s-1)]
```

then builds an `Int` _n_ between 0 and 7 by starting with a 0 and setting its first 3 bits accordingly.

With that in hand we can now test the _n_th bit of the rule number and obtain our result.

Since `Store s` forms a `Comonad` though, we can `extend` our `rule n` to obtain a new `Store s Bool` from out existing `Store s Bool`.

Now if we, say, `extend (rule 110)` we get a function from one world to a new world, where that
rule has been applied uniformly across the entire world at the same time.

```haskell
extend (rule 110) :: Num s => Store s Bool -> Store s Bool
```

By choosing an appropriate number type for `s` we can choose the topology for our automaton to live on!

We could repeatedly run our rules with 

```haskell
slowLoop :: (Store s a -> a) -> Store s a -> [Store s a]
slowLoop f = iterate (extend f)
```

Got the Memo?
=============

...but we'd get explosive slowdown. Why?

After a each loop iteration we depend on 3x as many evaluations as we did for the iteration before, because each evaluation is asking for all of the other old versions of the old neighbors, etc. 

So the trick is to memoize our function. The easiest way to do that without reasoning about `IO` is to use a memo combinator package like `data-memocombinators` or my own `representable-tries`. I'll buck my trend and use Luke's package instead of mine.

But which function?

We don't want to memoize the comonad algebra itself. The argument to that is of type `Store s a`, and memoizing function spaces of function spaces gets truly messy. Let's make a function that turns a value in our `Store` comonad into one that memoizes its answers by memoizing the experiment it contains.

```haskell
tab :: Memo s -> Store s a -> Store s a
tab opt (Store f s) = Store (opt f) s
```

`tab` takes a way to memoize a function from the context of our `Store` and a `Store` and yields a new `Store` that memoizes its results.

`Memo` comes from `data-memocombinators`. 

```haskell
type Memo a = forall r. (a -> r) -> a -> r
```

A value of type `Memo a` describes a memoization strategy for functions from values of type `a`. It takes a function and turns it into a function that memoizes its results. It does so in a completely pure way that is worth exploring in its own right, but...

If we just use the fact that `integral` provides us with such a memoization strategy that works for any `Integral` type, we can derive a smarter `loop`!

```haskell
loop :: Integral s => (Store s a -> a) -> Store s a -> [Store s a]
loop f = iterate (extend f . tab integral)
```

Here when we are given a new `Store` before each iteration we simply upgrade it to memoize its results for each position as it is asked before handing it to our rule for further evaluation.

Let's Do the Time Warp Again
============================

Now let's timewarp back to the stone age and print out endless reams of paper filled with automaton states.

To do that we need a way to see what some slice of our world looks like:

```active haskell
-- TODO: copy the whole program below here
{-# LANGUAGE RankNTypes #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE QuasiQuotes #-} 
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE DeriveFunctor #-}

import Control.Comonad
import Control.Lens as L
import Data.Bits
import Data.Bits.Lens as L
import qualified Data.ByteString as Strict
import qualified Data.ByteString.Lazy as Lazy
import Data.MemoCombinators
import Data.Word
import Diagrams.Backend.SVG
import Diagrams.Prelude as D
import Text.Blaze.Svg.Renderer.Utf8
import Yesod

data Store s a = Store (s -> a) s deriving Functor

instance Comonad (Store s) where
  extract (Store f s) = f s
  duplicate (Store f s) = Store (Store f) s
  
experiment :: Functor f => (s -> f s) -> Store s a -> f a
experiment k (Store f s) = f <$> k s 

rule :: Num s => Word8 -> Store s Bool -> Bool
rule w (Store f s) = testBit w $ 
  0 L.& partsOf (taking 3 L.bits) .~ [f (s+1), f s, f (s-1)]

tab :: Memo s -> Store s a -> Store s a
tab opt (Store f s) = Store (opt f) s

loop :: Integral s => (Store s a -> a) -> Store s a -> [Store s a]
loop f = iterate (extend f . tab integral)

-- show
window :: (Enum s, Num s) => s -> s -> Store s a -> [a]
window l h = experiment $ \ s -> [s-l..s+h]

xo :: Bool -> Char
xo True  = 'X'
xo False = ' '

main = mapM_ (putStrLn . map xo . window 50 0) $ 
       take 50 $ loop (rule 110) $ Store (==0) 0
-- /show
```

I probably should have told that thing stop printing a little sooner. Sorry. ;)

`window` varies our position on the number line up or down a bit so we can see several data points.

`xo` converts each result into a form we might want to see.

Then we put it all together and run Wolfram's [rule 110](http://en.wikipedia.org/wiki/Rule_110) starting with a single point at position 0 as our initial condition.

Pretty as a Picture
===================

It isn't the stone age any more. 

Matt Sottile did a pretty looking [forest fire](http://syntacticsalt.com/2010/08/30/forest-fire-cellular-automaton-haskell-and-matlab/) cellular automata example a couple of years back. But he had to render everything by hand using OpenGL.

Nowadays we can draw pretty pictures using Brent Yorgey's awesome `diagrams` package rather than carve ASCII `X`'s into the walls of our cave.

Now that we have the windows of data we want, all we need to do is turn each `window` into a a bunch of squares and stitch those rows together into a `Diagram`.

```haskell
grid :: [[Bool]] -> Diagram SVG R2
grid = vcat . map (hcat . cell) where
  cell b = unitSquare D.# fc (if b then black else white)
```

This post was spawned from a discussion with Rein Henrichs on #haskell. He supplied the initial version of the `diagrams` code. His version was much prettier.

`diagrams` supports rendering to a ton of formats including SVG, so we can transform our diagram into a document using `diagrams-svg` and `blaze-svg`. We could also render it directly to `cairo` and get out a PNG, get out an HTML canvas, a postscript document, etc. 

We could use the `renderSVG` function to generate a file on disk, but it also isn't the 80s. Command line tools that spit out files are passé. So lets just get our hands on the file here in memory as a `ByteString` and make sure it's strict to deal with the impedence mismatch between the tools I'm using.

```haskell
svg :: Diagram SVG R2 -> Strict.ByteString
svg = Strict.concat . Lazy.toChunks . 
      renderSvg . renderDia SVG (SVGOptions (Width 400) Nothing)
```

But is it Web Scale?
====================

The School of Haskell supports [running](https://www.fpcomplete.com/blog/2013/08/snap-happstack-anything-else) full-fledged web-based applications from an "active" Haskell snippet, so lets give it a try.

If we put them these pieces of code together you should be able to click run below and get out pretty pictures out of a custom web server that all but fits on your screen.

Click Run!

```active haskell web
{-# LANGUAGE RankNTypes #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE QuasiQuotes #-} 
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE DeriveFunctor #-}

import Control.Comonad
import Control.Lens as L
import Data.Bits
import Data.Bits.Lens as L
import qualified Data.ByteString as Strict
import qualified Data.ByteString.Lazy as Lazy
import Data.MemoCombinators
import Data.Word
import Diagrams.Backend.SVG
import Diagrams.Prelude as D
import Text.Blaze.Svg.Renderer.Utf8
import Yesod

data Store s a = Store (s -> a) s deriving Functor

instance Comonad (Store s) where
  extract (Store f s) = f s
  duplicate (Store f s) = Store (Store f) s
  
experiment :: Functor f => (s -> f s) -> Store s a -> f a
experiment k (Store f s) = f <$> k s 

rule :: Num s => Word8 -> Store s Bool -> Bool
rule w (Store f s) = testBit w $ 0 L.& partsOf (taking 3 L.bits) .~ [f (s+1), f s, f (s-1)]

tab :: Memo s -> Store s a -> Store s a
tab opt (Store f s) = Store (opt f) s

loop :: Integral s => (Store s a -> a) -> Store s a -> [Store s a]
loop f = iterate (extend f . tab integral)

window :: (Enum s, Num s) => s -> s -> Store s a -> [a]
window l h = experiment $ \ s -> [s-l..s+h]

grid :: [[Bool]] -> Diagram SVG R2
grid = cat unitY . reverse . map (hcat . map cell) where
  cell b = unitSquare D.# fc (if b then black else white)

svg :: Diagram SVG R2 -> Strict.ByteString
svg = Strict.concat . Lazy.toChunks . renderSvg . renderDia SVG (SVGOptions (Width 400) Nothing)
 
data App = App

instance Yesod App

mkYesod "App" [parseRoutes| / ImageR GET |]

getImageR :: MonadHandler m => m TypedContent
getImageR = sendResponse $ toTypedContent (typeSvg, toContent img) 

img = svg . grid . map (window 49 0) . take 50 . loop (rule 110) $ Store (==0) (0 :: Int)

main = warpEnv App
```

That clocks in at 60 lines of code. In that much space we defined the `Store` comonad, defined a generic evaluator that can handle any of Wolfram's 2-color automata, built a system of memoization to avoid asymptotic slowdown, took a cross section of our universe, and then rendered it to a diagram and built a custom web server to display that content here on the internet.

Almost all of the components we built are generic. We can define new types of automata, try out new initial conditions, jump around in time, with some work we can support multiple colors, new topologies, render the same diagram to different file formats conditionally based on browser preferences. The code above can be edited live here in your browser or downloaded and run locally on your own machine.

In the interest of full disclosure, the SVG that is rendered is far from optimal. The `diagrams` crew is aware of the issue and they are hard at work improving the way `diagrams` streams primitives to its backends, allowing it to take advantage of all the glorious structure that they have inside the `Diagram` type described in Brent's [excellent functional pearl](http://www.cis.upenn.edu/~byorgey/pub/monoid-pearl.pdf). Currently the communication process between `diagrams` and the backend is duplicating the transformation matrix and styling on a per element basis, and this is resulting in a much inflated document. When those changes go into `diagrams` and `diagrams-svg`, then this example will become _much_ faster with no changes to this code.

I hope this shows how you can use a little bit of theory and some of the more practical components of the Haskell ecosystem to accomplish a lot with very little code.

-- [Edward Kmett](mailto:ekmett@gmail.com)
August 15, 2013

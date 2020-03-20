# Part I: Deamortized ST

A week or two ago I gave a talk at Mozilla, San Francisco on [Cache-Oblivious Maps](http://www.youtube.com/watch?v=P3pLDpbzqCw) to the Bay Area Haskell User Group. 

My goal with this series of posts is to ultimately improve on the conclusion of that talk to generate a purely functional version of a structure like [Bender _et al._](http://supertech.csail.mit.edu/papers/sbtree.pdf)'s cache-oblivious lookahead array and the [stratified B-tree](http://arxiv.org/pdf/1103.4282v2.pdf), closing the gap between the performance of imperative and functional data structures in this space.

In short I want a really fast `Map`, optimized for contiguous use of memory that hits provably optimal asymptotics across the board for a wide array of problems despite exposing a purely functional API.

We won't get there today, but we'll at least establish some of the building blocks.

Today, I want to talk about a new trick that I came up with that allows us work with observably-functional algorithms in Haskell that can provide them with a purely-functional API in the same manner as the `ST` monad does today, but which permits us enough additional control over scheduling that we can deamortize many such algorithms.

## The Price of Purity

In a purely functional language, [amortized analysis](http://en.wikipedia.org/wiki/Amortized_analysis) is further hampered by the fact that every previous version of a structure is available at all times, so you have to consider that any structure can participate in multiple futures. This makes working with amortization much trickier. 

You can't really earn credit by doing things cheaply and then use all that credit to do something big later depending on data not known when you earned the credit because someone may reuse the data structure with different data, and respend the same credit!

In a purely-functional setting, if you don't use your budget this time you won't get more to spend later. It is just gone!

The trick is coming up with the right contortions, so that you can build in enough lag in your data structures to ensure that you always have just enough to work on.

## The Price of Laziness

Chris Okasaki's book [Purely Functional Data Structures](http://www.amazon.com/Purely-Functional-Structures-Chris-Okasaki/dp/0521663504) and [thesis](http://www.cs.cmu.edu/~rwh/theses/okasaki.pdf) cover a suite of tools for reasoning about asymptotics in a pure _lazy_ language.

He notes that you can often set up a thunk that will evaluate to the right result, if you know all the data you'll need to calculate its answer in the end. Then if two versions of the data structure in the future force this same thunk then they'll share the answer, but only compute it once. This preserves the correctness of the asymptotic analysis of the algorithm despite amortization.

We get quite a bit more flexibility with regards to amortization than in the strict setting. Now we can pay into an account for later by building a thunk we'll force in later calculations.

## Slowdown: What is the Worst Case?

When amortized complexity bounds aren't enough you can often follow Okasaki's advice and use even more advanced techniques such as "Implicit Recursive Slowdown", a variant on a technique by Kaplan and Tarjan.  There we set up a series of thunks you'll evaluate over time, each individually relatively cheap, so that you can turn amortized bounds into worst case bounds. Then you just force an appropriate amount of work as you go.

By setting up the thunks and paying them down by forcing them at just the right time you can often derive quite good worst-case performance in a purely functional persistent setting, so long as you have access to laziness.

There are still some algorthms such as Tarjan's [Union-Find](http://en.wikipedia.org/wiki/Disjoint-set_data_structure) problem that don't seem to yield to this technique, but surprisingly many do.

## Partiality as an Effect

So, if I want to set up such a chain of thunks, what would be a good general purpose way to go about it?

[Venanzio Capretta](http://arxiv.org/pdf/cs/0505037v6.pdf) defined a simple partiality monad. I'll play with the names a bit and reproduce a version of it here:

```active haskell
-- show
data Partial a = Stop a | Step (Partial a)

instance Monad Partial where
  return = Stop
  Stop a >>= f = f a
  Step m >>= f = Step (m >>= f)
-- /show

main = putStrLn "It typechecks!"
```

Those of you who watch out for such things will recognize this as just `Free Identity`, though technically there is a distinction to be had when you move beyond Haskell here. This is based on `νx. a + x` rather than `μx. a + x`, but in Haskell, `ν` and `μ` coincide. This means that technically we're using the "completely iterative" monad based on `Identity`, not the free construction, but I digress.

Using it you can run a calculation a number of steps and check to see if it has stopped with an answer yet.

```haskell
run :: Int -> Partial a -> Partial a
run 0 m = m
run n m@Stop{} = m
run n (Step m) = run (n-1) m
```

You can inject steps into a calculation to demarcate time.

```haskell
step :: Partial ()
step = Step (return ())
```

As a trivial example, you can also define a calculation that no matter how many steps you take will spin forever, but where each individual `Step` takes _O(1)_

```haskell
never :: Partial a
never = Step never
```

Using this if you have a purely functional algorithm that builds a sub-structure gradually you could pay it down `Step` by `Step` by burying a `Partial` result somewhere down in your structure. If you are careful about the amount of work within a `Step` then you can reason about the asymptotics of the overall system.

## The Power of Mutation

<div align="center"><img src="http://ekmett.github.com/images/x-men.jpg" style="padding-bottom: 10px"></div>

There is an old result by [Pippenger](http://www.cs.princeton.edu/courses/archive/fall03/cs528/handouts/Pure%20Versus%20Impure%20LISP.pdf) that showed that an algorithm implemented in a pure strict language may need to suffer a logarithmic slowdown relative to an algorithm implemented in a strict language with side-effects.

The fact that the slowdown for some algorithms is at least logarithmic in the absence of mutation can be shown using a fairly easy pigeon-hole argument.

The fact that the slowdown is at most logarithmic derives from the fact that you could always maintain a set of 'references' yourself in a `Map` like structure, in exchange for logarithmic overhead on (de)reference.

The price you pay for this power is that mutation brings with it its own headaches. From a free theorem perspective every mutable input is now effectively an extra output, and you have all sorts of spooky action at a distance concerns entangling distant parts of your program.

## Laziness is its Own Reward

<div align="center"><img src="http://ekmett.github.com/images/jennifer-lawrence-pants.gif" style="padding-bottom: 10px"></div>

Pippenger's analysis relied on the absence of mutation. 

In a non-strict language like Haskell, graph reduction / memoization of thunk evaluation provides us with a limited form of mutation! This renders his analysis inconclusive. 

Many algorithms that are provably slowed down asymptotically in a strict pure language can be implemented just fine in a non-strict language, by creative contortions to exploit this limited form of mutation.

This means we can at least sometimes win a log factor in a strict setting by cheating and using side-effects. There may or may not be cases where we need to pay an extra logarithmic factor for this in a lazy functional setting, but we don't have a definitive proof one way or the other.


My goal today is to help narrow the gap a bit by cheating.

## Is It Cheating If You Don't Get Caught?

If your algorithm builds an immutable result, then does it matter how you built it?

<img src="http://ekmett.github.com/images/cheater.jpg" style="float: right; width:45%;margin-left:10px; margin-bottom:15px">

Secretly cheating and doing mutable stuff behind the user's back but only exposing immutable purely functional trappings has been the job of the `ST s` monad in Haskell since John Launchbury and Simon Peyton Jones introduced it in [Lazy Functional State Threads](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.144.2237&rep=rep1&type=pdf) back in 1994.

The idea behind `ST` is that as long as nobody knows that you cheated and nobody can observe that you did, and no matter how many times they try to catch you you get away with it, does it matter that you cheated? I will definitely _not_ be trying this line of reasoning with my wife, but types are more forgiving.

I'm not going to dive into the use of the `ST s` monad here, beyond noting that it looks a lot like the `IO` monad under the hood with a much reduced palette of operations intended so that the result of the `ST s` calculation should be deterministic.

<br clear="all">

```haskell
newSTRef :: a -> ST s (STRef s a)
readSTRef :: STRef s a -> ST s a
writeSTRef :: STRef s a -> a -> ST s ()
...
```

When you're done, you run the entire `ST s` calculation at once:

```haskell
runST :: (forall s. ST s a) -> a
```

Since you are universally quantified over the choice of `s`, you can't use references produced in one `ST` calculation in another, and encapsulation, referential transparency and all the good things about functional programming are preserved.

However, we had to run the entire effect at once.

If I want to amortize it and pay it down over time, I'm out of luck. In the end I'm interested in building very large vectors but paying for their construction in very small, affordable, chunks.

That's where today's hackery comes in.

## Every Step You Take

My first thought was of course to use something like 

```haskell
walkST :: (forall s. Free (ST s) a) -> Free Identity a
```

or

```haskell
walkST :: (forall s. Free (ST s) a) -> Partial a
```

But this has the problem that [every step you take](http://www.youtube.com/watch?v=OMOGaugKpzs&t=26s) will be observed and must be paid for, even little administrative actions at the end, such as freezing the result vector costs you an extra step. 

If I want to do something like work with existing `Stream` fusion out of `vector` and just have it pay once for every `step`, this approach isn't going to cut it.

You could bandaid this with a coproduct and use `Free (ST s :+: Identity)` and promise not to use the knowledge that the `ST s` calculations were generated separately for evil, but given the amount of time I recently spent talking about [rightsizing abstractions](https://www.fpcomplete.com/user/edwardk/editorial/procrustean-mathematics), it'd be hypocritical for me not to try to find a better way.

## Capretta's Iterative Monad Transformer

We can upgrade Capretta's partiality monad to a monad transformer as done by [Capretta, Altenkirch and Uustalu](http://www.ioc.ee/~tarmo/tday-veskisilla/uustalu-slides.pdf).

```active haskell
-- show
newtype IterT m a = IterT 
  { runIterT :: m (Either a (IterT m a)) 
  }

instance Monad m => Monad (IterT m) where
  return = IterT . return . Left
  IterT m >>= k = IterT $ 
    m >>= either (runIterT . k) (return . Right . (>>= k))
  fail = IterT . fail
-- /show 

main = putStrLn "It typechecks!"
```

I've added `IterT` and its dual to `free` 

Here we can still insert explicit steps:

```haskell
step :: Monad m => IterT m ()
step = IterT . return . Right . return
```

If you've been playing with free monads for a while you'll recognize this as a version of `FreeT Identity m` from the `free` package, rather than the simpler `Free m` above. Again, we face the technical distinction that this is based on `νx. ST s (a + x)` not `μx. ST s (a + x)`.

`IterT` has been added to the `free` package in version 4.2 as a distinct construction from `FreeT Identity` to help drive this distinction home!

Now what we want to do now is generate a slower version of `runST` that takes a properly quantified `ST s` calculation with inserted step markers and walks through it carefully, one step at a time:

```haskell
walkST :: (forall s. IterT (ST s) a) -> Partial a
```

## Walking the Walk

We have a few options for how to implement `walkST`.

It is possible to do this entirely with `unsafeInterleaveST`. 

This is actually a non-trivial exercise and it is very easy to accidentally write a version that is too eager and performs effects too soon. My best version so far requires two uses of `unsafeInterleaveST` to get the right semantics.

I leave this as an exercise for the reader. 

You can also rummage through [λpaste](http://lpaste.net) for old versions of this monad for tips. ;)

## Newsflash: `unsafeInterleaveST` Is Unsafe!

Even if you get that right though, `unsafeInterleaveST` is a whole lot more unsafe for this use case, than the equivalent `unsafeInterleaveIO` operation! To understand why we need to look down in the guts of each of them.

When a thunk is evaluated in GHC there is an ever so tiny race condition. When one thread enters into a thunk there is a tiny 1-2 cycle window between that thread entering and establishing the [greyhole](http://citeseerx.ist.psu.edu/viewdoc/download?rep=rep1&type=pdf&doi=10.1.1.125.857) that will catch other threads and make them block, during which another thread could come along and start evaluating the same thunk at the same time.

In the absence of side-effects this is benign. The risk is so low relative to the astronomical costs of synchronization across threads that we, well, just don't bother synchronizing.

This of course would be bad if the thunk _did_ have side-effects, _e.g._ if it called `unsafePerformIO`.

Internally `unsafePerformIO` calls `noDuplicate` to check to make sure that we're not duplicating effort and effects:

```haskell
unsafePerformIO :: IO a -> a
unsafePerformIO m = unsafeDupablePerformIO (noDuplicate >> m)
```

Similarly `unsafeInterleaveIO` also checks `noDuplicate`.

```haskell
unsafeInterleaveIO :: IO a -> a
unsafeInterleaveIO m = unsafeDupableInterleaveIO (noDuplicate >> m)
```

But `unsafeInterleaveST`, on the other hand rather boldly does not.

```haskell
unsafeInterleaveST :: ST s a -> ST s a
unsafeInterleaveST (ST m) = ST ( \ s ->
    let
        r = case m s of (# _, res #) -> res
    in
    (# s, r #)
  )
```

This means that it may very well wind up duplicating work if the result `r` of the `ST s a` calculation we called `unsafeInterleaveST` on is evaluated via `par`. And since we don't control what users will do with our code, you really do need to allow for that. 

Roman Leschinskiy's cute [Sparking Imperatives](http://unlines.wordpress.com/2010/04/21/sparking-imperatives/) hack even mixes `par` with `ST`, but he is careful to `noDuplicate` as he goes.

Now, there is possibly a very good reason for this distinction. If we look at the haddocks for `noDuplicate` it

> Ensures that the suspensions under evaluation by the current thread
are unique; that is, the current thread is not evaluating anything
that is also under evaluation by another thread that has also executed
'noDuplicate'.

So we're faced with a dilemma (trilemma?), we can either:

1. abandon the use of `unsafeInterleaveST` entirely as too risky.

2. reason through whether `noDuplicate` would be legal to use and how to mix it with the existing `unsafeInterleaveST`.

3. or we can require the end user to only ever perform idempotent operations in the `ST` monad!

For now I'm largely restricting myself to #1 and #3. 

For an API I expect to expose to an end user, I'm most likely to choose option #1. Anything they do with the resulting construction can be branded `Trustworthy` and they don't need to know how it is built in too much detail.

But when I'm writing code myself that merely exposes a pure façade, the performance benefits of #3 may well outweigh the reasoning difficulties. In principle, if my principal operations are merging elements from two immutable input vectors and generating output in another vector, so long as I'm not bumping a counter stored in an `STRef`, everything I do will have idempotent effects.

In the long term, it is probably worth checking to see if `unsafeInterleaveST` should be updated to do `noDuplicate` and thereby close out the concern about #2, effectively merging it performance-wise with option #1. 

This still leaves option #3 open for constant tuning in the same crazy way as the `inlinePerformIO` hackery gets used down in `bytestring`.

## A Safer Alternative: `unsafePerformIO`

You have to love it when `unsafePerformIO` is the safest option.

```active haskell
{-# LANGUAGE RankNTypes #-}

import Control.Monad.ST
import System.IO.Unsafe as Unsafe
import Control.Monad.ST.Unsafe as Unsafe

-- show
walkST :: (forall s. IterT (ST s) a) -> Partial a
walkST m = go m where
  go (IterT m) = 
    case Unsafe.unsafePerformIO $ 
         Unsafe.unsafeSTToIO m of
      Left a  -> Stop a
      Right m -> Step (go m)
-- /show

newtype IterT m a = IterT 
  { runIterT :: m (Either a (IterT m a)) 
  }

instance Monad m => Monad (IterT m) where
  return = IterT . return . Left
  IterT m >>= k = IterT $ 
    m >>= either (runIterT . k) (return . Right . (>>= k))
  fail = IterT . fail

data Partial a = Stop a | Step (Partial a)

instance Monad Partial where
  return = Stop
  Stop a >>= f = f a
  Step m >>= f = Step (m >>= f)
  
main = putStrLn "It typechecks!"
```

Here we're relying on the fact that we perform one step at a time, and that we're evaluating an "entirely sealed" `ST s` calculation. When we're done and have the answer `a` in our `Partial a`, like with conventional `ST s` we can't go back and use any of our references any more. In `walkST` we keep reopening the same ST region, but we only do so after we look around and make sure nobody else is going to catch us and nobody else is doing the same thing.

With this, we can go through and do things like calculate an `n`-element unboxed vector in `n` individually worst-case constant time steps!

## Next Time

This opens up new opportunities for matching worst case asymptotic bounds of algorithms from the imperative world in a purely functional setting.

Next time I'll start to explore how we can mix this approach with a novel (to me) choice of number system to dynamize and deamortize large immutable lookup structures. 

Happy Halloween!

-[Edward Kmett](mailto:ekmett@gmail.com)
Oct 31, 2013
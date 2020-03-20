# Once

I wrote this fragment a few years ago and have talked about it to folks individually, but it came up again in discussion on the `#haskell` channel, so I figured it was worth posting about in a central location.

## Evaluating to Normal Form

The `deepseq` package's `Control.DeepSeq` provides the incredibly useful `NFData` class.

```haskell
class NFData a where
  rnf :: a -> ()
```

`rnf` evaluates its argument fully to normal form.

This class is abused by efficiency afficionados everywhere to ensure that no undue laziness leaks into their data structures. 

However, it is easy to abuse, and often winds up having to do a lot of unnecessary work forcing things we already know to be forced!

## Once and For All

We can prevent that by making a rather tricky little instance of `NFData`:

```active haskell
import Control.DeepSeq
import Control.Lens
import Data.Copointed
import Data.Foldable

-- show
data Once a = Once () a

runOnce :: Once a -> a
runOnce (Once _ a) = a

once :: NFData a => a -> Once a
once a = Once (rnf a) a

instance NFData (Once a) where
  rnf (Once () _) = ()
-- /show

instance Foldable Once where
  foldMap f (Once _ a) = f a

instance Copointed Once where
  copoint (Once _ a) = a

_Once :: NFData a => Iso' (Once a) a
_Once = iso runOnce once

main = putStrLn "It compiled."
```

Now, anywhere you expect to spam `rnf` in your code, just wrap the value in `Once`, and any attempts will only force the fragment underneath once.

This can make a massive difference in the performance of code that happens to abuse `rnf`, enabling it to avoid doing useless work and avoid thrashing caches walking over irrelevant data.

You don't have to use `Once` to make this trick work. You can of course put extra `()` arguments as needed recursively within your data structure and replicate the `NFData` trick above yourself.

## With A Little Bit of Lens

With `lens` you can collapse the entire API for working with `Once` to a single isomorphism:

```haskell
_Once :: NFData a => Iso' (Once a) a
_Once = iso runOnce once
```

-[Edward Kmett](mailto:ekmett@gmail.com)

September 3, 2013
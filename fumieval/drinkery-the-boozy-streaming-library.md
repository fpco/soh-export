# Drinkery the boozy streaming library

In the ecosystem of Haskell, a number of stream processing libraries has been made. The very purpose is to process a sequence of values with effects, in a composable manner. Still, I was not satisfied with the sets of features of the existing packages. Accordingly, I decided to make a new one.

It's called [drinkery](http://hackage.haskell.org/package/drinkery). This package achieves following features:

* __Sequential producer__: You can build a producer using a monadic action like `yield :: s -> Producer s ()`.
* __[ListT done right](https://wiki.haskell.org/ListT_done_right)__: Correct implementation of a list monad transformer.
* __Monadic consumer__: A consumer monad processes a stream involving effects.
* __Transducer__: A mechanism that transforms streams.
* __Full duplex__: Downstream can send a value to the upstream.
* __Decent performance__

## Hello, world

```haskell
import Data.Sinky
import qualified Data.Sinky.Finite as D

main = tapListT' (taste "0H1e2l3l4o5,6 w7o8r9ld!\n")
  +& D.filter (not . isDigit)
  $& traverseFrom_ consume (liftIO . putChar)
```

## Sequential producer

Most libraries offers monads where you can emit a value in a computation; `Producer` for `pipes`, `ConduitM` for `conduit`, `PlanT` for `machines`. Sinky employs a `Producer` which manipulates a `Tap`, non-terminating source. The type parameter `r` is a request type that grants full-duplexity; assume `()` for now.

```haskell
newtype Producer r s m a = Producer { unProducer :: (a -> Tap r s m) -> Tap r s m }

newtype Tap r s m = Tap { unTap :: r -> m (s, Tap r s m) }
```

As well as other implementations, `yield` emits one element.

```haskell
yield :: (Monoid r, Applicative m) => s -> Producer r (Maybe s) m ()
```

`Producer` can be converted to a `Tap`:

```haskell
runProducer :: (Monoid r, Applicative m) => Producer r (Maybe s) m a -> Tap r (Maybe s) m
```

One big difference from other libraries is the explicit use of `Maybe` to have an end of a stream. This allows you to omit `Maybe` and use a specific value instead (e.g. empty bytestring).

## ListT done right

It's a well-known fact that [transformer](https://hackage.haskell.org/package/transformers-0.5.5.0/docs/Control-Monad-Trans-List.html)'s ListT is _not_ a monad transformer. A proper implementation is convenient for writing nested loops. Today, several libraries implement ListT. The essentials are: 

* (a) A monad transformer `T`: `T m` is a monad for any monad `m`.
* (b) An `Alternative` instance: You can convert a list into an action by `asum . map pure`.
* (c) A way to draw `a` from `T m a`.

drinkery implements it as a Boehm-Berarducci encoded list. (a) is satisfied because `ListT r m` is a monad regardless of `m`. Also There is an alternative instance (b). A specialised function is defined for convenience:

```haskell
newtype ListT r m s = ListT
  { unListT :: forall x. (s -> Tap r x m -> Tap r x m) -> Tap r x m -> Tap r x m }

sample :: Foldable f => f s -> ListT r m s
```

You can turn it into a `Tap`.

```haskell
runListT :: (Monoid r, Applicative m) => ListT r m s -> Tap r (Maybe s) m
```

Benchmark time. [This benchmark](https://github.com/fumieval/drinkery/blob/master/benchmarks/benchmark.hs) goes through a triple of loops where inner loops depend on the outer one.

```haskell
sourceAlt :: Monad m => ([Int] -> m Int) -> m Int
sourceAlt k = do
  a <- k [1..50]
  b <- k [1..a]
  c <- k [1..b]
  return $! a + b + c
```

Thanks to the encoding, ListT doesn't impose a slowdown. In fact, it's the fastest implementation!

```
drain/drinkery/Producer                    mean 304.7 μs  ( +- 11.48 μs  )
drain/drinkery/ListT                 mean 304.7 μs  ( +- 24.92 μs  )
drain/pipes/Producer                     mean 372.9 μs  ( +- 17.91 μs  )
drain/pipes/ListT                        mean 770.3 μs  ( +- 75.69 μs  )
drain/list-t                             mean 5.332 ms  ( +- 393.9 μs  )
drain/ListT                              mean 23.69 ms  ( +- 1.331 ms  )
```

## Monadic consumer

Monadic consumption is one of the important abilities of a stream processing library (there's an exception like [streaming](https://hackage.haskell.org/package/streaming), though). The most common implementation is called iteratee. `machines`, `pipes`, and `conduit` have some additions on it.

```haskell
newtype Iteratee s m a = Iteratee
  { runIteratee :: m (Either (s -> Iteratee s m a) a) }
```

On the other hand, drinkery's consumer is dissimilar.

```haskell
newtype Sink t m a = Sink { runSink :: t m -> m (a, t m) }

consume :: (Monoid r, Monad m) => Sink (Tap r s) m s
```

The first type parameter represents the source (usually `Tap r s`). A `Sink` action is a function that consumes a tap and returns the remainder. You can just apply `runSink` to feed a tap.

Several combinators are defined to work with finite streams. Their first argument is usually `consume`.

```haskell
foldlFrom' :: (Monad m) => m (Maybe a) -> (b -> a -> b) -> b -> m b
foldMFrom :: (Monad m) => m (Maybe a) -> (b -> a -> m b) -> b -> m b
traverseFrom_ :: (Monad m) => m (Maybe a) -> (a -> m b) -> m ()
drainFrom :: (Foldable t, Monad m) => m (Maybe a) -> m ()
```

Since it operates on a `Tap` you can push an element back:

```haskell
leftover ::  (Monoid r, Monad m) => s -> Sink (Tap r s) m ()
```

### Fanout

Sometimes we want to distribute an input stream to multiple consumers. This is not possible with `Sink` itself and cloning a tap is not trivial either. For this purpose, drinkery offers a classic iteratee:

```haskell
newtype Awaiter s m a = Awaiter { runAwaiter :: m (Either (s -> Awaiter s m a) a) }

await :: Monad m => Awaiter s m s
```

`serving_` combines a list of `Awaiter`s to one.

```haskell
serving_ :: Monad m => [Awaiter s m a] -> Awaiter s m ()
```

It can be converted into a `Sink`.

```haskell
iterAwaiterT consume :: Awaiter s m a -> Sink s m a
```

### Taste & compare

Not defined in the package yet, but `Sink` can simultaneously consume two streams through the [Product](https://hackage.haskell.org/package/base-4.10.1.0/docs/Data-Functor-Product.html) type. It should also be possible to manipulate an [extensible](https://hackage.haskell.org/package/extensible) record of taps.

```haskell
drinkL :: (Monoid r, Monad m) => Sink (Product (Tap r s) tap) m s
drinkL = drinking $ \(Pair p q) -> fmap (`Pair`q) <$> unTap p mempty
```

Multi-stream is a rare feature. Only machines supports it as far as I know, but it probably won't be long till drinkery gains it in a more flexible way.

## Transducer

A stream transducer receives an input and produces zero or more outputs. There are three ways to represent a stream transducer.

* Concrete structure which can consume and produce values (e.g. machines, pipes, conduit)
* A stream producer where the base monad is a consumer (iteratee)
* A function from a stream producer to another stream producer (streaming)

drinkery took the second approach. `Distiller tap r s m` is a tap which consumes `tap`.

```haskell
type Distiller tap r s m = Tap r s (Sink tap m)
```

Surprisingly, the only combinator introduced is `(++$)`, the composition of a tap and a distiller.

```haskell
(++$) :: (Functor m) => tap m -> Distiller tap r s m -> Tap r s m
```

Since Distiller is a special case of a tap, you can feed a drinker a distiller, and you can also connect two distillers using `(++$)`. Note that the drinker also has an access to the input of the distiller, allowing it to send a request.

```haskell
runSink :: Sink (Tap r s) (Sink tap m) a -> Distiller tap m r s -> Sink tap m a
```

Full-duplexity
----

One distinctive feature of [pipes](https://hackage.haskell.org/package/pipes) is `Proxy`, the base type, having four parameters:

```haskell
data Proxy a' a b' b m r
    = Request a' (a  -> Proxy a' a b' b m r )
    | Respond b  (b' -> Proxy a' a b' b m r )
    | M          (m    (Proxy a' a b' b m r))
    | Pure    r
```

This allows a producer to receive a value of type `b'`, and does a consumer to send `a'`. This interactivity is useful for handling seeking. However, pipes' composition operator fixes the request type to `()`:

```haskell
(>->) :: Monad m
  => Proxy a' a () b m r
  -> Proxy () b c' c m r
  -> Proxy a' a c' c m r
```

You need to resort to one of the special combinators, `(+>>)`. A sad fact is that `Proxy` cannot accumulate requests for one input. You would have to define some custom function.

```haskell
(+>>)
    :: Monad m
    => (b' -> Proxy a' a b' b m r)
    ->        Proxy b' b c' c m r
    ->        Proxy a' a c' c m r
```

The good old [iteratee](https://hackage.haskell.org/package/iteratee)'s composition does propagate requests, but in a rather disappointing fashion: the type of requests is `SomeException`. Honestly, iteratee's combinators and their semantics are quite puzzling.

```haskell
newtype Iteratee s m a = Iteratee{ runIter :: forall r.
          (a -> Stream s -> m r) ->
          ((Stream s -> Iteratee s m a) -> Maybe SomeException -> m r) ->
          m r}
```

Other libraries don't support bidirectionality at all.

As we've seen in the first section, drinkery's Tap (`Producer` and `ListT` likewise) has an extra parameter for reception. You can send requests by calling `request`.

```haskell
request :: (Monoid r, Monad m) => r -> Sink (Tap r s) m ()
```

Producer or ListT are able to receive orders from the drinker, flushing the pending requests.

```haskell
accept :: Monoid r => Producer r s m r
inquire :: Monoid r => ListT r m r
```

Of course composition doesn't take this ability away.

### Resource management

Conduit has a unique mechanism for resource management which makes it a respectably practical library; you can attach a finaliser to a stream producer.

```haskell
addCleanup :: Monad m => (Bool -> m ()) -> ConduitM i o m r -> ConduitM i o m r
```

In drinkery, you can create an instance of `CloseRequest` to finalise a tap. `(+&)`, a specialised version of `runSink`, closes the tap as soon as the drinker finishes.

```haskell
class CloseRequest a where
  -- | A value representing a close request
  closeRequest :: a

instance CloseRequest r => Closable (Tap r s)

(+&) :: (Closable tap, Monad m) => tap m -> Sink tap m a -> m a
```

## Performance

I [benchmark](https://github.com/fumieval/drinkery/blob/master/benchmarks/benchmark.hs)ed a composition of two scanning operations `scan (+) 0 D.++$ scan (+) 0` processing 22100 elements.

```haskell
scan-chain/drinkery/++$                  mean 1.717 ms  ( +- 104.5 μs  )
scan-chain/drinkery/$&                   mean 1.239 ms  ( +- 110.7 μs  )
scan-chain/pipes                         mean 1.210 ms  ( +- 78.40 μs  )
scan-chain/conduit                       mean 1.911 ms  ( +- 97.84 μs  )
scan-chain/machines                      mean 2.731 ms  ( +- 176.9 μs  )
```

It's quite good. Note that there are two ways to compose distiller: `++$` attaches to a tap or a distiller, and `$&` attaches to a drinker. The latter seems to be faster. Notably, a single scan is significantly faster than the rivals:

```haskell
scan/drinkery                            mean 534.6 μs  ( +- 58.07 μs  )
scan/pipes                               mean 736.7 μs  ( +- 54.84 μs  )
scan/conduit                             mean 862.3 μs  ( +- 68.02 μs  )
scan/machines                            mean 1.352 ms  ( +- 84.82 μs  )
```

Conclusion
----

drinkery offers a significantly greater flexibility without losing speed. The API is not complete; I plan to add a lot more combinators in the near future. 

I'm looking forward to your patronage.
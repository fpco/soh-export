# Named extensible effects

Monad transformers are the standard way to extend monads.
The [transformers](https://hackage.haskell.org/package/transformers) package
defines monad transformers.

`StateT`, one of the representative monad transformers, has three parameters:

```haskell
newtype StateT s m a = StateT { runStateT :: s -> m (a, s) }
```

The second parameter `m` allows you to add a new type of effect while preserving
the original monad. This is characterised by `MonadTrans`:

```haskell
class MonadTrans t where
  lift :: Monad m => m a -> t m a

instance MonadTrans (StateT s) where
  lift m = StateT $ \s -> (\a -> (a, s)) <$> m
```

As a monad transformer stack gets deeper, it gets more inconvenient to apply
`lift`s. [mtl](https://hackage.haskell.org/package/mtl) provides type classes
in order to avoid this.

`MonadState s` is such a class; this is the class of monads which have an access
to a state `s`.

```haskell
class MonadState s m | m -> s where
  get :: m s
  put :: s -> m ()
  state :: (s -> (a, s)) -> m a
```

`StateT s m` is an instance of `MonadState s`.

```haskell
instance Monad m => MonadState s (StateT s m) where
  get = StateT $ \s -> return (s, s)
  put s = StateT $ const $ return ((), s)
  ...
```

Also, other monad transformers are also instances, when their inner monads are
`MonadState`. Their methods are just `lift`ed.

```haskell
instance MonadState s m => MonadState (ReaderT r m) where
  get = lift get
  put = lift . put

instance (Monoid w, MonadState s m) => MonadState (WriterT w m) where
  get = lift get
  put = lift . put
```

This way you don't have to care about the number of `lift`s, and you can define
polymorphic functions using these classes.

However, for library designers, this is a big pain. Basically this kind of class
can be defined for each monad; this means adding a monad transformer has a
quadratic development cost. It also makes packaging significantly harder. 

Extensible effects
----

[Extensible effects](http://okmij.org/ftp/Haskell/extensible/exteff.pdf) is a
solution to this problem. The extensible effects framework provides a monad
parameterised by a type-level list of effects.

One recent implementation is [freer-effects](https://hackage.haskell.org/package/freer-effects).

In this framework, you don't have to deal with other monads in order to define
a new type of effect.

`send` makes a big difference; it can be thought of as universal `lift`.
As long as `t` is a member of `r`, you can embed an action to `Eff`.

```haskell
send :: Member t r => t v -> Eff r v
```

A state monad can be defined as follows:

```haskell
data State s v where
  Get :: State s s
  Put :: !s -> State s ()

get :: Member (State s) r => Eff r s
get = send Get

put :: Member (State s) r => s -> Eff r ()
put = send . Put
```

`handleRelayS` decomposes `Eff` in a stateful manner.

```haskell
handleRelayS :: s
  -> (s -> a -> Eff r w)
  -> (forall v. s -> t v -> (s -> v -> Eff r w) -> Eff r w)
  -> Eff (t ': r) a
  -> Eff r w
```

The following example interprets the `State` actions.

```haskell
runState :: Eff (State s ': r) a -> s -> Eff r (a, s)
runState m s0 = handleRelayS s0 (\s a -> return (a, s))
  (\s t k -> case t of
    Get -> k s s
    Put s' -> k s' ()) m
```

While this seems very good, it leaves one issue.

Unlike `MonadState`, In a constraint `Member (State s) r`, `r` doesn't determine
the type of `s`. This means that you can't use a polymorphic effect without a
type signature:

```haskell
increment :: (Num a, Member (State a) r) => Eff r ()
increment = get >>= put . (+1)
```

This is annoying, and I think this is one reason why extensible effects are not popular.

Named extensible effects
----

My [extensible](http://hackage.haskell.org/package/extensible-0.4.2/docs/Data-Extensible-Effect.html)
library solves this problem by giving names to effects.

```haskell
liftEff :: forall s t xs a. Associate s t xs
  => Proxy s
  -> t a
  -> Eff xs a
```

`liftEff` is like `send`, but `Proxy s` specifies the name. `Associate s t xs`
requires that the effect corresponding to name `s` is `t` in `xs`. This version
doesn't suffer from the ambiguity problem because `s` and `xs` determines `t`.

Various operations take names:

```haskell
getEff :: forall k s xs. Associate k (State s) xs => Proxy k -> Eff xs s
putEff :: forall k s xs. Associate k (State s) xs => Proxy k -> s -> Eff xs ()
```

`Eff` in `extensible` is also an instance of `MonadState` so you can simply write

```haskell
increment :: (Num a, MonadState a m) => m ()
increment = get >>= put . (+1)
```

The equivalent of
`handleRelayS` is `peelEff`. Note that the order of arguments is different.

```haskell
peelEff1 :: (a -> b -> Eff xs r) -- ^ return the result
  -> (forall x. t x -> (x -> b -> Eff xs r) -> b -> Eff xs r) -- ^ Handle the foremost type of an action
  -> Eff (k >: t ': xs) a -> b -> Eff xs r
peelEff1 = peelEff rebindEff1
```

Here's an example. You can use `TypeApplications` extension to specify a name.

```haskell
runStateEff :: forall k s xs a. Eff (k >: State s ': xs) a -> s -> Eff xs (a, s)
runStateEff = peelEff1 (\a s -> return (a, s))
  (\m k s -> let (a, s') = runState m s in k a $! s')
```

It's also possible to handle multiple effects which have the same type.

```haskell
{-# LANGUAGE DataKinds, FlexibleContexts, OverloadedLabels #-}
import Data.Extensible

test :: (Associate "foo" ((,) String) xs
  , Associate "bar" ((,) String) xs) => Eff xs ()
test = do
  tellEff #foo "Hello "
  tellEff #bar "foo"
  tellEff #foo "world"
  tellEff #bar "bar"
```

```haskell
> leaveEff $ runWriterEff @ "foo" $ runWriterEff @ "bar" test
(((),"foobar"),"Hello world")
```

Performance
----

I did some [benchmarks](https://github.com/fumieval/extensible/blob/master/benchmarks/eff-comparison.hs) on RWS equivalents. Unfortunately, it's much slower than monad transformers.

[chart](https://cdn.pbrd.co/images/GzURdi6.png)

Conclusion
----

The idea of extensible effects is promising to avoid the issue of quadratic
monad class instances. However, the performance is sluggish.
For time being, there is a trade-off between convenience and performance.
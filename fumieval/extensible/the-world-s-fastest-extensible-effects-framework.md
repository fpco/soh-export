# The world's fastest extensible effects framework

An [operational monad](http://apfelmus.nfshost.com/articles/operational-monad.html) is a data type parameterised by a set of operations `t`, and it gives a `Monad` instance for free.

It upgrades extensible sums, aka. [open unions](http://okmij.org/ftp/Haskell/extensible/index.html#open-union), into an extensible effect monad (Oleg Kiselyov and Hiromi Ishii. [Freer Monads, More Extensible Effects](http://okmij.org/ftp/Haskell/extensible/more.pdf), 2015).
Extensible effects release you from the obligation to write a lot of instances to make your monadic actions lift-free (see also [Named extensible effects](https://www.schoolofhaskell.com/user/fumieval/extensible/named-extensible-effects)).

I uploaded a new version of [monad-skeleton-0.1.4](http://hackage.haskell.org/package/monad-skeleton-0.1.4), an operational monad library. As a result of performance optimisation in the new version,
[extensible](https://hackage.haskell.org/package/extensible), the extensible effects library based on it, is now much faster than the other well-known libraries.

I [benchmark](https://github.com/fumieval/extensible/blob/3d9030397ab39ec539b30ffd73804784386b94bb/benchmarks/eff-comparison.hs)ed several libraries using equivalents of the following code:

```haskell
testMTL :: (MonadReader Int m, MonadState Int m, MonadWriter (Sum Int) m)
  => m ()
testMTL = replicateM_ 100 $ do
  r <- ask
  s <- get
  tell (Sum s)
  put $! s + r
```

Here's the result. `extensible` is twice as fast as `freer-effects`:

```
extensible-0.4.2: benchmarks
Running 1 benchmarks...
Benchmark eff-comparison: RUNNING...
benchmarking rws/extensible
time                 9.752 μs   (9.714 μs .. 9.785 μs)
                     1.000 R²   (1.000 R² .. 1.000 R²)
mean                 9.767 μs   (9.733 μs .. 9.820 μs)
std dev              142.7 ns   (95.88 ns .. 202.9 ns)
variance introduced by outliers: 11% (moderately inflated)
             
benchmarking rws/mtl
time                 794.5 ns   (791.9 ns .. 797.1 ns)
                     1.000 R²   (1.000 R² .. 1.000 R²)
mean                 802.1 ns   (795.9 ns .. 810.3 ns)
std dev              24.06 ns   (16.43 ns .. 30.87 ns)
variance introduced by outliers: 42% (moderately inflated)
             
benchmarking rws/mtl-RWS
time                 586.4 ns   (581.8 ns .. 591.7 ns)
                     1.000 R²   (0.999 R² .. 1.000 R²)
mean                 585.2 ns   (582.6 ns .. 590.2 ns)
std dev              10.94 ns   (7.718 ns .. 17.81 ns)
variance introduced by outliers: 22% (moderately inflated)
             
benchmarking rws/exteff
time                 130.3 μs   (127.5 μs .. 134.0 μs)
                     0.997 R²   (0.993 R² .. 1.000 R²)
mean                 128.4 μs   (127.6 μs .. 130.7 μs)
std dev              3.986 μs   (1.817 μs .. 8.008 μs)
variance introduced by outliers: 28% (moderately inflated)
             
benchmarking rws/effin
time                 29.71 μs   (29.62 μs .. 29.81 μs)
                     1.000 R²   (1.000 R² .. 1.000 R²)
mean                 29.74 μs   (29.67 μs .. 29.83 μs)
std dev              260.9 ns   (208.8 ns .. 328.2 ns)
             
benchmarking rws/freer-effects
time                 19.64 μs   (19.59 μs .. 19.69 μs)
                     1.000 R²   (1.000 R² .. 1.000 R²)
mean                 19.63 μs   (19.58 μs .. 19.69 μs)
std dev              167.0 ns   (133.9 ns .. 203.3 ns)
             
Benchmark eff-comparison: FINISH
Completed 3 action(s).
```

To try it as a replacement for mtl, import [Data.Extensible.Effect.Default](http://hackage.haskell.org/package/extensible-0.4.2/docs/Data-Extensible-Effect-Default.html) along with `Data.Extensible`.

`runWriterDef`, `runStateDef`, `runReaderDef` are similar to the runners of monad transformers, but you need to apply `leaveEff :: Eff '[] a -> a` to extract the result.

```haskell
runExtensible :: Eff '[ReaderDef Int, StateDef Int, WriterDef (Sum Int)] a
  -> ((a, Int), Sum Int)
runExtensible = leaveEff
  . runWriterDef
  . flip runStateDef 0
  . flip runReaderDef 1
```

If you seek for faster extensible effects, this definitely is an option.

Under the hood
----

The definition of `Skeleton` used to be

```haskell
newtype Skeleton t a = Skeleton { unSkeleton :: Spine t (Skeleton t) a }
```

where

```haskell
data Spine t m a where
  Spine :: MonadView t m a -> Cat (Kleisli m) a b -> Spine t m b

data Cat k a b where
  Empty :: Cat k a a
  Leaf :: k a b -> Cat k a b
  Tree :: Cat k a b -> Cat k b c -> Cat k a c
```

The new representation is somewhat simpler, and quite similar to what [freer-effects](http://hackage.haskell.org/package/freer-effects-0.3.0.1/docs/src/Control-Monad-Freer-Internal.html#Eff) has:

```haskell
data Skeleton t a where
  ReturnS :: a -> Skeleton t a
  BindS :: t a -> Cat (Kleisli (Skeleton t)) a b -> Skeleton t b

data Cat k a b where
  Leaf :: k a b -> Cat k a b
  Tree :: Cat k a b -> Cat k b c -> Cat k a c
```

I'm guessing removing `Empty` reduced the number of case analyses `debone` does.
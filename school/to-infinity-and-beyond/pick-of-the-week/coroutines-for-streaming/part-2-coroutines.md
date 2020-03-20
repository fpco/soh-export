# Part 2: Coroutines

# A quick look at coroutines

You've probably heard the word "coroutine" before. What is a coroutine? Well, briefly, it is cooperatively passing control around.

Imagine two computations that are made to be intertwined. They take turns, one runs for a while, produces some value, and hands it off along with The Olympic Torch (or The Floor, or whatever you want to call it: control) to the other. The second runs for a while, produces some value, and hands it off to the first. The one with The Torch is the one running, and will soon produce a value. The one without The Torch is waiting for a value, and control, to be passed back.

This is the kind of coroutine we are going to implement.

# General definition of a coroutine

Lightly repainted after stealing it from the `monad-coroutine` package, I give you, the `Coroutine` monad transformer!

```haskell
newtype Coroutine s m r
  = Coroutine { resume :: m (CoroutineState s m r) }

data CoroutineState s m r
  = Run (s (Coroutine s m r))
  | Done r
```

The `s` type parameter is what `monad-coroutine` calls the "suspension functor." As you can see from the `Run` constructor of `CoroutineState`, when a coroutine suspends itself, it will present you with this functor, containing a way to resume it. Or not, or multiple... it all depends on the functor of course, which may or may not "contain" anything.

Look closely, `Coroutine` is just the free monad transformer in disguise! But we'll ignore that for now, and keep talking about it as a coroutine abstraction.

Exercise to the reader: implement the `Monad`, and `MonadTrans` instances for `Functor s => Coroutine s`. Bonus: implement the `MFunctor` instance (type class from the `mmorph` package). Solutions are in the source of monad-coroutine, `hoist = mapMonad`.

`Coroutine`, as a language transformer, comes with one primary command: `suspend`. This just *wraps* up a suspension functor into the Coroutine monad. *(Ahem, did I just say `wrap`? Speaking of FreeT...)*

```haskell
suspend :: (Monad m, Functor s) => s (Coroutine s m x) -> Coroutine s m x
suspend s = Coroutine (return (Run s))
```

# Interface: our suspension functor

Note that if we select the following suspension functor, we have recovered the equivalent of `PauseT` from last time:

```haskell
newtype PauseF x = PauseF x
instance Functor PauseF where fmap f (PauseF x) = PauseF (f x)
type PauseT = Coroutine PauseF

pause :: Monad m => PauseT m ()
pause = suspend $ PauseF (return ())
```

We are going to select a specific suspension functor to work with. Our functor is an enhancement over `PauseF` in two ways: it will allow suspensions to provide a value, as well as take in a value, before moving on with the next step.

```haskell
data Interface i o x
  = Produced o (i -> x)

instance Functor (Interface i o) where
  fmap f (Produced o k) = Produced o (f . k)
```

Note that, as promised, by setting the "in" and "out" parameters to the trivial message, `()`, we *again* recover `PauseT`.

```haskell
type PauseF' = Interface () ()
type PauseT' = Coroutine PauseF'

pause' :: Monad m => PauseT' m ()
pause' = suspend $ Produced () (\() -> return ())
```

# Synonyms and operations

Coroutines are in one of two states, which I call "Producing" and "Consuming". In the Producing state, they need no input, and should be run until they produce something, at which point they switch into the Consuming state. In the Consuming state, they need an input. After receiving input, they switch into the Producing state. I will adopt the convention of using the type parameter `o` for "out", and `i` for "in", `m` for "monad", and `r` for "result".

```haskell
type Producing o i = Coroutine (Interface i o)
type Consuming r m i o = i -> Producing o i m r
```

Let's pause for a second to talk about this abstraction. Even though there is an input end and an output end on an Interface, this is *not* the same as the pipes or conduits abstraction. The main difference here is that an Interface bundles a single output with a single input, always. Of course, your output type can have multiple components, but you surrender control with a single value, and control returns to you with a single value. (If you want some other way to communicate with the outside world, then use a different Interface.) This choice was inspired by `Control.Proxy`; as we will see later, we can mimic Proxy by simply layering two interfaces: an "upstream" interface and a "downstream" interface. Spoilers!

There are two main operations that this abstraction gives us. One is an action, `yield`, which is similar to `pause`, but communicates information across the interface. The other is a combining operator, `$$`, which attaches a `Producing` coroutine to a `Consuming` coroutine with compatible interfaces. Go ahead and try implementing these operations yourself before I show you how I did it.

You may think that the order I put the types in is a little odd. There is reason to this madness which will be explained later. (Partly, it's just a matter of taste, though.)

```active haskell
-- show Given this...
import Control.Monad
import Control.Monad.Trans.Class

newtype Coroutine s m r
  = Coroutine { resume :: m (CoroutineState s m r) }

data CoroutineState s m r
  = Run (s (Coroutine s m r))
  | Done r

instance (Functor s, Functor m) => Functor (Coroutine s m) where
instance (Monad m, Functor s) => Monad (Coroutine s m) where
instance Functor s => MonadTrans (Coroutine s) where

suspend :: (Monad m, Functor s) => s (Coroutine s m x) -> Coroutine s m x
suspend s = Coroutine (return (Run s))

data Interface i o x = Produced o (i -> x)
instance Functor (Interface i o) where

type Producing o i = Coroutine (Interface i o)
type Consuming r m i o = i -> Producing o i m r

-- show implement this
yield :: Monad m => o -> Producing o i m i
yield o = undefined

($$) :: Monad m => Producing a b m r -> Consuming r m a b -> m r
producing $$ consuming = undefined

-- show ...and see if it compiles.
-- Don't try testing it unless you fill in all the instances.
-- Just play type tetris instead.
main = putStrLn "it compiles"
```

@@@ Here's how I implemented them
```haskell
yield :: Monad m => o -> Producing o i m i
yield o = suspend $ Produced o return

($$) :: Monad m => Producing a b m r -> Consuming r m a b -> m r
producing $$ consuming = resume producing >>= \co -> case co of
  Done r -> return r
  Run (Produced o k) -> consuming o $$ k -- They just swap places! So cool.
```
@@@

It turns out that `($$)` can have a very short and meaningful implementation, and it will come in handy next time, when we deal with multiple layers.

# Next time

As I hinted, next time we are going to look at what happens when we put stack multiple `Producing` transformers on top of each other. With two interfaces, we will have the equivalent of a `Proxy` (or interface transformation). We will implement the conduit-style fusion operators `($=)`, `(=$)`, and `(=$=)` in terms of `($$)`. After that, we will look with more detail at the `Consuming r m` instance of Category, as well as the `Proxy` instance.

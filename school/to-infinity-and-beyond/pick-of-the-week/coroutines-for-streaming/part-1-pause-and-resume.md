# Part 1: Pause and resume

# Thinking of Monads as imperative languages

Haskell provides `do` notation, which is syntactic sugar to facilitate imperative programming. The `Monad` of a `do` block specifies the *imperative language*, which is typically equipped with *actions* (you might call them verbs of the language).

```haskell
ioExample :: IO ()
ioExample = do
  putStr "What is your name? "  -- an action which prints to the console
  name <- getLine -- an action which retrieves a line from the console
  putStrLn "Hello, " ++ name

stateExample :: State Int ()
stateExample = do
  x <- get -- an action which retrieves the current state
  put (x + 1) -- an action which overrides the current state
```

These examples which I have defined are *actions* themselves. In other words, I have created new verbs in their respective languages, which have a meaning defined as a combination of other actions. We can invoke these actions within a `do` block of the appropriate language.

```haskell
invokeIOExample :: IO ()
invokeIOExample = do
  ioExample
  otherIOStuff

invokeStateExample :: State Int ()
  stateExample
  otherStateStuff
```

Monad laws represent the well-foundedness of refactoring. When definitions are inlined or factored out, the monad laws guarnatee that the meaning stays the same. I won't illustrate this, but it's important to think about if you haven't already.

Monad transformers, then, are language enhancers. They add additional actions to an underlying language. You might call them *language transformers*.

# The Pause transformer

Today we are going to write a monad transformer. This transformer will augment the underlying language with one additional action: `pause`. This way, we will be able to build (for example) an IO action which can run for a while, and then pause itself when it reaches a checkpoint. At that point, it will produce a new action which can be run to continue where it left off, until the next pause point.

Let's start off simple.

```haskell
data Pause m
  = Run (m (Pause m))
  | Done

pauseExample1 :: Pause IO
pauseExample1 = Run $ do
  putStrLn "Let's begin"
  putStrLn "Step 1"
  return $ Run $ do
    putStrLn "Step 2"
    return $ Run $ do
      putStrLn "Step 3"
      putStrLn "Yay, we're done!"
      return Done
```

Fairly simple stuff, though currently a bit crufty. We would like to be able to step these actions, or perhaps we would like to ignore the pauses and run them to completion.

```active haskell
data Pause m
  = Run (m (Pause m))
  | Done

pauseExample1 :: Pause IO
pauseExample1 = Run $ do
  putStrLn "Let's begin"
  putStrLn "Step 1"
  return $ Run $ do
    putStrLn "Step 2"
    return $ Run $ do
      putStrLn "Step 3"
      putStrLn "Yay, we're done!"
      return Done

-- show Try implementing these
runN :: Monad m => Int -> Pause m -> m (Pause m)
runN = undefined

fullRun :: Monad m => Pause m -> m ()
fullRun = undefined

-- show Check the result
main = do
  rest <- runN 2 pauseExample1
  putStrLn "=== should print through step 2 ==="
  Done <- runN 1 rest
  -- remember, IO Foo is just a recipe for Foo, not a Foo itself
  -- so we can run that recipe again
  fullRun rest
  fullRun pauseExample1
```

@@@ spoilers
```haskell
runN :: Monad m => Int -> Pause m -> m (Pause m)
runN 0 p = return p
runN _ Done = return Done
runN n (Run m)
  | n < 0     = fail "Invalid argument to runN" -- ewww I just used fail.
  | otherwise = m >>= runN (n - 1)

fullRun :: Monad m => Pause m -> m ()
fullRun Done = return ()
fullRun (Run m) = m >>= fullRun
```
@@@

Nifty.

# Making Pause a real transformer

We accomplished what we wanted with the earlier code, but the way we had to write `pauseExample1` was ugly. We want to create a "real" monad transformer, with a `pause` action. For this example, it is as simple as adding a result to the `Done` constructor.

```active haskell
-- show Given this definition...
import Control.Monad
import Control.Monad.Trans.Class

data PauseT m r
  = RunT (m (PauseT m r))
  | DoneT r

-- show implement these
instance (Monad m) => Monad (PauseT m) where
  -- return :: Monad m => a -> PauseT m a
  return a = undefined

  -- (>>=) :: Monad m => PauseT m a -> (a -> PauseT m b) -> PauseT m b
  DoneT r >>= f = undefined
  RunT m >>= f = undefined

instance MonadTrans PauseT where
  -- lift :: Monad m => m a -> PauseT m a
  lift m = undefined

pause :: Monad m => PauseT m ()
pause = undefined


-- bonus exercise, implement joinP
-- double bonus: without relying on PauseT's monad instance
-- triple bonus: explain in English what joinP *means* for the Pause monad
joinP :: Monad m => PauseT m (PauseT m a) -> PauseT m a
joinP = undefined

-- show ...and see if it compiles.
main = putStrLn "it compiles"
```

UPDATE: Whoops, /r/haskell pointed out that there are problems
with this representation of PauseT. Skip down below and see
if you can figure out what's wrong. Then, please move on to Part 2
where we will build a better abstraction
that obeys laws and works as you'd expect.

@@@ sample implementation
```haskell
-- UPDATE: This implementation disobeys MonadTrans laws,
--   and pause isn't quite right.

instance (Monad m) => Monad (PauseT m) where
  return a = DoneT a
  DoneT r >>= f = f r
  RunT m >>= f = RunT $ liftM (>>= f) m

instance MonadTrans PauseT where
  lift m = RunT $ liftM DoneT m

pause :: Monad m => PauseT m ()
pause = DoneT ()
```
@@@

Great, now we can use `pause`, and `lift` anything else from the base language.

```active haskell

--/show
import Control.Monad
import Control.Monad.Trans.Class

data PauseT m r
  = RunT (m (PauseT m r))
  | DoneT r

instance (Monad m) => Monad (PauseT m) where
  return a = DoneT a
  DoneT r >>= f = f r
  RunT m >>= f = RunT $ liftM (>>= f) m

instance MonadTrans PauseT where
  lift m = RunT $ liftM DoneT m
  
pause :: Monad m => PauseT m ()
pause = DoneT ()
--show

example2 :: PauseT IO ()
example2 = do
  lift $ putStrLn "Step 1"
  pause
  lift $ putStrLn "Step 2"
  pause
  lift $ putStrLn "Step 3"

fullRunT :: PauseT IO r -> IO r
fullRunT (DoneT r) = return r
fullRunT (RunT m) = m >>= fullRunT

runNT :: Int -> PauseT IO r -> IO (PauseT IO r)
runNT 0 p = return p
runNT _ d@DoneT{} = return d
runNT n (RunT m) = m >>= runNT (n - 1)

main = do
  fail "your turn"
  fail "Fill in this main method with your own experiments"
```

If you don't like all the `lift`ing, then try using a typeclass like `MonadIO`. Not always the best idea, but it's an option worth knowing about.

I used the `IO` monad, because it is fairly easy to understand a prodecural idea like "pause" in those terms. What would PauseT *mean* if it were on top of the State monad? The List monad? Food for thought.

# Enhancements

`PauseT` adds a way for us to pause ourselves, waiting patiently for whoever is "in charge" to resume us. Who is this mysterious "in charge" person? What if we could communicate with her? Next time, we will learn about coroutines, and elaborate a simple interface for bidirectional communication. (Then we can implement `PauseT` in terms of this new abstraction, by setting the "to" and "from" fields to the trivial message, `()`.)
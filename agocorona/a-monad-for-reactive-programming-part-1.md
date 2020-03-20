# A monad for reactive programming. Part 1

For the impatient: you can find something runnable at the bottom.

Functional reactive programming has no notion of event scope.  A functional,
declarative, reactive computation is affected as a whole by a signal, and must
re-start from the beginning in some way, since it is declarative.  A monad can
contain the scope of the signal, so part of the computation already done
upstream could be kept when a signal is injected at some level of the
computation by an active component.

A monad can decompose the computation in a chain of event handlers virtually set
up by the monadic computation when it is called by the top signal, that is, when
it is run for the first time.

This has many applications.  Imagine this computation:

``` haskell
profits = do
    quantity <- waitEvent "quantity"
    liftIO $ do
        putStr "quantity="
        print quantity
        getChar
    price <- waitEvent "price"
    liftIO $ do
        putStr "price="
        print price
        getChar
    liftIO $ do
        putStr $ "total="
        print $ quantity * price
        getChar
    return $ quantity * price
```

Suppose that quantity and price changes along the day, so there are two events,
one when the stock sold (`quantity` changes) and one when `price` changes.
Suppose that, instead of printing in the console, the process update a display
in an executive dashboard and the computation is much more complicated and
costly with access to databases and OLAP reports, with a lot of different
events.  We need only that when the price changes, the monad execute just the
code needed to change the price and the profits without touching the other
costly branches of the monad, including not only the part of the computation
that is above, upstream, but also some branches that may be after the event, if
we wish.

A monad suitable for configuring event handlers when it is executed is the one
below:

``` haskell
data Transient m x = Transient (m (Maybe x))

instance (MonadState m, StateType m ~ EventF) => Monad (Transient m) where
    return x = Transient $ return $ Just x
    x >>=  f = Transient $ do
        setEventCont f
        mk <-  runTrans x
        case mk of
            Just k  ->  runTrans $ f k
            Nothing -> return Nothing
```

The name `Transient` comes from my old idea of closures in a monadic expression
as transient pure states that are only re-evaluated when some change appears
upstream.

Let's use following primitive to tell the rest of the computation downstream to
stop:

``` haskell
stop = empty

empty :: Monad m => Transient m a
empty = Transient $ return Nothing
```

This `empty` will be the empty of the applicative instance for Transient, but I
donÂ´t want to introduce it right now.

It uses the state monad since we need to "transport" `EventF` data.  This kind
of data are continuations _(f)_ for each computation _(x)_.

``` haskell
data EventF = forall m b c. EventF (b -> Transient m c)
```

`Transient` uses Maybe because I want to stop the computation to flow down
whenever further computations are not necesssary. The monad does not install
event handlers, it simply store the next continuation with `setEventCont`:

``` haskell
setEventCont f = do
    f' <- get
    put $ EventF f
    return f'
```

The _real_ computation that set the event handlers is `waitEvent`, it is an
active element that set up an event handler. That event handler is active
element's continuation (set by the monad) for the event that is waiting.

(Note: I could have done it more simple obviating the state but as we should
see, the final solution need it).


At the end, `waitEvent` stop the computation (returning `Nothing`):

``` haskell
waitEvent name = Transient $ do
    f   <- get
    evs <- liftIO $ takeMVar eventHandlers
    liftIO . putMVar eventHandlers . M.insert name f $ evs  -- !> ("set "++name)
    return Nothing
    ```

It uses a map of event handlers:

``` haskell
eventHandlers âˆ·  MVar (M.Map String EventF)
eventHandlers= unsafePerformIO $ newMVar M.empty
```

Then, after the monadic expression is executed, this structure has at least the
first event handler set.  `eventLoop` waits for events and executes the
corresponding event handlers:

``` haskell
type EvType = String
data Event  = forall a. Event EvType a

eventLoop [] = return ()
eventLoop (Event name r : evs) = do
    liftIO . print $ "new event: " ++ name
    ths <- liftIO $ readMVar eventHandlers
    case M.lookup name ths of
      Just (EventF f) -> runTrans'' $ (unsafeCoerce f) r
      Nothing -> return ()
    eventLoop evs
```

An `Event` contains its type (that is used to lookup for event handler)
and its value.

Instead of a list, `eventLoop` could be defined as a process that reads a queue.

Note the `unsafeCoerce` there.  By construction I know that it must work, since
it applies the `x` of the monad with the `f` of the continuation stored in the
state, but the compiler has no such information.  For this time, as a matter of
experiment I will allow it.  I'm a physicist, not an mathematician neither an
engineer.  If I were a mathematician I would spent three years struggling with
the denotational semantics of event handling, producing abstruse papers before
doing anything usable.  If I were an engineer I would do a visual basic program
for the concrete problem.  I love math and engineering, but I would happily
sacrifice conceptual beauty and delivery time to get the general problem solved.

`runTrans''` is the computation that executes the continuation assuming a
`StateT` transformer that transports the `EventF` state.

``` haskell
runTrans'' :: Transient (StateT EventF IO) a â†’ IO ()
runTrans'' tmx = runTrans' tmx >> return ()

runTrans' ::  Monad m => Transient (StateT EventF m) x -> m (Maybe x)
runTrans' tmx = evalStateT (runTrans tmx) undefined
```

The continuation called by `runTrans''` may contain further `waitEvent`
sentences that will add further event handlers to the loop by the same
mechanism.

LetÂ´s give a set of events to the monster:

``` haskell
eventList =
    [ Event "quantity" 10
    , Event "price"     2
    , Event "price"     3
    , Event "quantity" 30
    , Event "price"     4
    ]
```

And run it (**press <kbd>Enter</kbd> to make advance the computation**, since
`profits` has `getChar`).

```active haskell
{-# LANGUAGE ExistentialQuantification #-}
{-# LANGUAGE FlexibleContexts          #-}
{-# LANGUAGE UndecidableInstances      #-}
{-# LANGUAGE MultiParamTypeClasses     #-}

import Control.Concurrent.MVar
import Control.Monad.State
import Data.Map                as M
import System.IO.Unsafe
import Unsafe.Coerce


data EventF = forall m b c. EventF (b -> Transient m c)
data Transient m x = Transient (m (Maybe x))

runTrans :: Transient m x -> m (Maybe x)
runTrans (Transient mx) = mx

setEventCont f = do
    f' <- get
    put $ EventF f
    return f'

empty :: Monad m => Transient m a
empty = Transient $ return Nothing

instance (MonadState EventF m) => Monad (Transient m) where
    return x = Transient . return . Just $ x
    x >>= f = Transient $ do
        setEventCont f
        mk <- runTrans x
        case mk of
          Just k  -> runTrans (f k)
          Nothing -> return Nothing

instance MonadTrans Transient where
    lift mx = Transient $ mx >>= return . Just

instance (MonadState EventF m, MonadIO m) => MonadIO (Transient m) where
    liftIO io = let x = liftIO io in x `seq` lift x

instance (MonadState EventF m) => MonadState EventF (Transient  m) where
--  type StateType (Transient m) = EventF
    get = Transient $ get >>= return . Just
    put x = Transient $ put x >> return (Just ())

eventHandlers :: MVar (M.Map String EventF)
eventHandlers = unsafePerformIO $ newMVar M.empty

type EvType = String
data Event = forall a. Event EvType a

waitEvent name = Transient $ do
    f <- get
    evs <- liftIO $ takeMVar eventHandlers
    liftIO . putMVar eventHandlers . M.insert name f $ evs
    return Nothing

eventLoop [] = return ()
eventLoop (Event name r : evs) = do
    liftIO . putStrLn $ "new event: " ++ name
    ths <- liftIO . readMVar $ eventHandlers
    case M.lookup name ths of
      Just (EventF f) -> runTrans'' $ (unsafeCoerce f)  r
      Nothing         -> return ()
    eventLoop evs

runTrans' :: Monad m => Transient (StateT EventF m) x -> m (Maybe x)
runTrans' tmx = evalStateT (runTrans tmx) undefined

runTrans'' :: Transient (StateT EventF IO) a -> IO ()
runTrans'' tmx = runTrans' tmx >> return ()

-- show
main = do
  runTrans'' profits
  eventLoop eventList
  putStrLn "END"

eventList :: [Event]
eventList =
    [ Event "quantity" 10
    , Event "price"     2
    , Event "price"     3
    , Event "quantity" 30
    , Event "price"     4
    ]

profits :: Transient (StateT EventF IO) Integer
profits = do
    quantity <- waitEvent "quantity"
    liftIO $ do
        putStr "quantity="
        print quantity
        getChar
    price <- waitEvent "price"
    liftIO $ do
        putStr "price="
        print price
        getChar
    liftIO $ do
        putStr $ "total="
        print $ quantity * price
        getChar
    return $ quantity * price

-- /show
```

You see that only the code after each event is executed and that the context
upstream necessary for the continuation is maintained when each event is called.

And that's all.

No. Look at this other example.

``` haskell
main = do
    runTrans'' $ do
    let threshold = 100
    pr <- profits
    liftIO $ do
        when (pr > threshold) $
            putStr "Passed threshold! Mail sent to boss."
        print pr
    eventLoop eventList
    putStrLn "END"
```

It may run the same `profits` computation and the same events, but at the end of
it if a threshold of profits is surpassed, it has to produce an extra message.

If you change the main expression in the program by the above one, you would
expect that the last message should be

```
"Passed threshold!â€¦"
```

but that is not the
case.  This means that the monad, as is defined, does not trigger the entire set
of event handlers necessary.  It is necesary to store in the state not the
individual continuation but a stack of all the nested continuations in all
subcomputations that are affected by the event.

Moreover, the `unsafeCoerce` may produce segmentation faults, if instead of

``` haskell
waitEvent "quantity"
```

alone in a monadic computation we'll put

``` haskell
(*) <$> return units <*> waitEvent "quantity"
```
.

But this in the second part.

By the way the improvement of this monad that I will present in the second part
is the one used in [hplayground, a client-side framework][hplay].

[hplay]: https://github.com/agocorona/hplayground
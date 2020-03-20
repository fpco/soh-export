# A monad for reactive programming. Part 2

Introduction
------------

Some time has passed since I published the [first part of this tutorial](https://www.fpcomplete.com/user/agocorona/a-monad-for-reactive-programming-part-1). The time has been spent in achieving a deeper understanding of events and event processing.  In this second part I solve the problems of the simple approach depicted at the end of the part 1. 

The motivating example to solve is the same, a console application with a basic event scheduler, a monadic computation that  has some statements which wait for different kinds of events. You can run the two examples here in the tutorial.

This tutorial  also can be considered as the description of an event sourcing mechanism without inversion of control. A event sourcing application is basically a set of event handlers for each type of event with a central state. A functional reactive mechanism is a single event handler with event preprocessing on top. while a monadic reactive mechanism like this is a monadic computation that automatically generates a cascade of event handlers. 

Besides monadic reactive, I will demonstrate more classical functional reactive behaviours using applicative combinators, and how to mix monadic, reactive and alternative to mimic the creation of new signals, with thresholds.


Motivation
----------

Why am I jumping into the crowded reactive business? Because the current functional reactive solutions do not address the problem created by side effects, like computation costs, input output and whatever. Monadic reactive programming has been discarded by functional purists because it looks "imperative". As a result monadic reactive is rarely considered. But it is the natural solution for many reactive scenarios.

Functional reactive programming has no notion of event scope. A functional, reactive computation is declarative. It is affected as a whole by every signal, so it must re-compute from the beginning in some way. 

In the other side, a monad can contain the scope of the signal, so part of the computation already done, upstream, is not re-executed when a signal is injected at some level of the computation by an active component.

A monad can decompose the computation in a chain of event handlers virtually set up by the monadic computation when it is called by the top signal, that is, when it is run for the first time. Each event handler executes the continuation of the computation with the execution state of the closure coming from what was executed previously. The monadic sequence, althoug it has a imperative look, actually it is a declarative expression that configure event handlers. It is not a imperative description of what the computer will do.


This cascading of computations for different events has many applications. Imagine this computation:

    profits= do
      quantity ←  waitEvent "quantity"
      liftIO $ do
            putStr "quantity="
            print quantity
            getChar
      price ←  waitEvent "price"
      liftIO $ do
            putStr "price="
            print price
            getChar
      liftIO $ do
         putStr $ "total="
         print $ quantity * price
         getChar
      return $ quantity * price


Suppose that quantity and price changes along the day, so there are two events, one when the stock sold ("quantity") changes and one when price changes. Suppose that instead of printing in the console, the process update a display in an executive dashboard. Suppose that the computation is much more complicated and costly with access to databases, OLAP queries, with a lot of different events. Maybe some message has been sent to the president and he launched some missiles. We need only that when the price changes, the monad execute just the code below the price event, to change the price and the calculated total, without touching the other costly branches of the monad, that may call the president to launch more missiles when it is not necessary. Maybe  not only the part of the computation that is above, upstream of the event insertion must remain untouched, but also some branches that may be after the event.

Alternatively, consider that the computation is cheap, but events happens very fast, for example there is a mouse movement event, but the scene must be redraw only when another second event happens and this redraw is the expensive part of the process. The cheap re-calculation can be done before and only from time to time the expensive part will be executed. 

In these  scenarios monadic reactive is the natural solution.


Shortcomings of part 1
----------------------

In the part 1 we see that the solution proposed does not handle more than one bind level. This program,  uses the `profits` expression defined above:

    threshold= 120
    do
     pr <- profits
     liftIO $ if pr > threshold
         then  putStr "Passed threshold!.. mail sent to boss: " >> print pr
         else  print pr


would never show the message "Passed Threshold..." since the event handler set by the monad "forget" the continuation that is outside of the bind chain where the event occurs, in `profits`.

To see that this new code print the message when the condition is met, run [the code](#run-it)


Additionally, the part 1 can not handle applicative combinations of events. The code below is the same `profits` calculation, but this time it is using an applicative expression:

    profits' :: TransientIO Int
    profits'=  do
      total<- (*) <$> currentEventValue "quantity" <*> currentEventValue "price"
      liftIO $ do
         putStr $ "total="
         print $ total
         getChar

See how the new code run this applicative combination [running here](#running-the-appicative-case)

Creating events
---------------

In the part I there is a tentative monad that do not fully configure the event handlers when it is executed. This time there are some enhancements :

    data Transient m x= Transient  (m (Maybe x)

    type StateIO= StateT EventF  IO

    type TransientIO= Transient StateIO
    
    instance Monad TransientIO where
      return x = Transient $ return $ Just x
      x >>= f = Transient $ do
        cont <- setEventCont x  f
        mk <- runTrans x
        resetEventCont cont
        case mk of
           Just k  -> runTrans $ f k
           Nothing -> return Nothing

Don´t worry, all this would be explained in the following paragraphs. the name `Transient` comes from my old idea of closures in a monadic expression as transient pure states that are only reevaluated when some change appears upstream.

To tell the rest of the monadic computation to stop:

    stop= empty

This `empty` will be the empty of the applicative instance for Transient. Now we define these instances that were absent in part 1:

    instance   Functor TransientIO where
      fmap f x= Transient $ fmap (fmap f) $ runTrans x


    instance Applicative TransientIO where
      pure a  = Transient  .  return $ Just a
      Transient f <*> Transient g= Transient $ 
          f >>= \ k ->
          g >>= \ x ->
          return $  k <*> x

    instance  Alternative TransientIO where
      empty= Transient $ return  Nothing
      Transient f <|> Transient g= Transient $ do
          x <- f
          y <- g
          return $ x <|> y


These instances would permit nice combinations of signals, for the creation of new ones.


It uses the state monad since we need to 'transport' `EventF` data. This kind of data are continuations (f) for each computation (x). 

    data EventF  = forall a b . EventF{eventHandlers:: (M.Map String EventF )
                                      ,currentEvent :: Maybe Event
                                      ,eventValues :: M.Map String Dynamic
                                      ,xcomp :: (TransientIO a)
                                      ,fcomp :: [a -> TransientIO b]}

This time the state is more complicated because now the event handlers and the last arrived event are carried out in the state. They are not stored in MVars like in the part 1. Moreover, to simulate discrete signals whose values change when the corresponding event arrives, I stored all the current event values for each type.  `eventHandlers` contains the state at which the computation is for each event waiting. from this state is possible to construct the event handler.

Note that now the computation state has two components. `xcomp` is the code of the active component that initiate the event handler, and carries out the state of all the computation upto now. `fcomp` is the rest of the computation. There is a list because the rest of the computation may be a subcomputation of a calling computation, therefore it has his own continuation and so on. in the example the additional continuation is threshold comparison. More on that later.


`Transient` uses Maybe because I want to stop the computation when further computations are not necessary. The monad does not install event handlers properly, it simply store the next continuation with `setEventCont`:

    setEventCont ::   (TransientIO a) -> (a -> TransientIO b) -> StateIO EventF
    setEventCont x f  = do
       st@(EventF es c vs x' fs)  <- get
       put $ EventF es c vs  x ( f: unsafeCoerce fs) 
       return st

`setEventConf` stores the current x in the monad, that may contain active combinators that set event handlers. it also `flattens` the hierarchy of nested computations that compose the remaining to be done. both will compose the event handler code. togeter with `resetEventConf` (not shown) they push and pop subcomputations in the state in the monad instance, so that `waitEvent` can set the right event handler.

The *real* computation that set the event handlers is `waitEvent` , it is the active element that set up an event handler. It need to get his own continuation (set by `setEventCont` in the monad) and set it as the event handler for the event that is waiting. 

The event handler is set in the state itself, in the `eventHandlers` field. Since `waitEvent` is called two times, one when the monad find it to set up the event and once or more times when his event arrives, he distinguish both cases. if the event has not been set, it insert itself and stop the computation returning Nothing. If it has been set previosly, it updates the event handler and return the value of the current event if it is of the type expected. Nothing if it is not. `eventValue` just verifies if the current event match the  event type that was waiting for or not.


    waitEvent :: Typeable a => EvType -> TransientIO a
    waitEvent name = Transient $ do
      st <- get                                              !> "waitEvent"
      let evs = eventHandlers  st 

      case  M.lookup name evs of
        Nothing ->  do
           put st{ eventHandlers=  M.insert name st  evs}    !> ("created event handler for: "++ name)
           return Nothing 
        Just _ ->  do
           put st{ eventHandlers=  M.insert name st evs}     !> ("upadated event handler for: "++ name)
           eventValue name

    eventValue name =  do
       st <- get
       let me= currentEvent st 
       case me of
         Nothing -> return Nothing                           !> "NO current EVENT"
         Just (Event name' r) -> do
          if name /= name' then return Nothing  else do
            case fromDynamic r of
              Nothing -> return Nothing 
              Just x -> do 
                liftIO $ putStrLn $ "read event: " ++ name
                put st{eventValues= M.insert name (toDyn x) $ eventValues st}
                return $ Just x


the operator `!>` is used for debugging. set it to  `(!>)= flip trace` to have execution traces.

Now the event is defined using `Dynamic`. An `Event` contains a type -that is used to lookup for its event handler- and his value.


    type EvType = String
    data Event = Event EvType Dynamic


Then, after the monadic expression is executed, this structure has at least the first event handler set.

`eventLoop` is the event scheduler that wait for events and execute the corresponding event handlers:


    eventLoop :: [Event] ->  StateIO ()
    eventLoop []= return()
    eventLoop (ev@(Event name _):evs)= do
       modify (\st -> st{currentEvent= Just ev ,eventValues= M.delete name $ eventValues st})  
                                        !> ("inject event:" ++ name)
       ths <- gets eventHandlers
       case M.lookup name ths of
          Just st -> runCont st         !> ("execute event handler for: "++ name) 
          Nothing -> return ()          !> "no handler for the event"
       eventLoop evs


Instead of consuming a list, `eventLoop` could have been defined as a process that read a queue.

The continuation called by eventLoop  may contain further `waitEvent` sentences that will add further event handlers to the loop by the same mechanism.

Let´s give a set of events to the monster:

    eventList=[ Event "quantity" 10, Event "price" 2, Event "price" 3
               ,Event "quantity" 30, Event "price" 4]


And run it 

##Run it

(**Press return to make advance the computation**, since `profits` has `getChar`)


``` active haskell

{-# START_FILE main.hs #-}

module Main where

import Base
import Control.Monad.State.Strict
import Unsafe.Coerce
import System.IO.Unsafe
import Control.Applicative
import qualified Data.Map as M
import Data.Dynamic

-- show

threshold = 100

main= (flip runStateT) eventf0 $ do
  runTrans  $ do
     pr <- profits
     liftIO $ if pr > threshold
         then  putStr "Passed threshold!.. mail sent to boss: " >> print pr
         else  print pr
   
  eventLoop eventList
  
  liftIO $ putStrLn "END"
  

eventList=[Event "quantity" $ int2Dyn 10
                ,Event "price" $ int2Dyn 2
                ,Event "price" $ int2Dyn 3
          ,Event "quantity" $ int2Dyn 30
                ,Event "price" $ int2Dyn 4]

  where
  int2Dyn :: Int -> Dynamic
  int2Dyn= toDyn
  


profits :: TransientIO Int
profits= do
      quantity <-   waitEvent "quantity"
      liftIO $ do
            putStr "quantity="
            print (quantity :: Int)
            getChar
      price <-   waitEvent "price"
      liftIO $ do
            putStr "price="
            print price
            getChar
      let total= quantity * price
      liftIO $ do
         putStr $ "total="
         print total
         getChar
      return total

-- /show



{-# START_FILE Base.hs #-}
{-# LANGUAGE ExistentialQuantification,FlexibleContexts,
             FlexibleInstances, UndecidableInstances, MultiParamTypeClasses #-}

-- show

module Base where

--/show

import Control.Monad.State.Strict
import Unsafe.Coerce
import System.IO.Unsafe
import Control.Applicative
import qualified Data.Map as M
import Data.Dynamic
import Debug.Trace

(!>) = const . id -- flip trace


data Transient m x= Transient  (m (Maybe x))


data EventF  = forall a b . EventF{eventHandlers:: (M.Map String EventF )
                                  ,currentEvent :: Maybe Event
                                  ,eventValues :: M.Map String Dynamic
                                  ,xcomp :: (TransientIO a)
                                  ,fcomp :: [a -> TransientIO b]}


eventf0= EventF M.empty Nothing M.empty (empty) [const $ empty]


instance MonadState EventF  TransientIO where
  get=  Transient $ get >>= return . Just
  put x= Transient $ put x >> return (Just ())

type StateIO= StateT EventF  IO

type TransientIO= Transient StateIO

runTrans ::  TransientIO x -> StateT EventF  IO (Maybe x)
runTrans (Transient mx) = mx

setEventCont ::   (TransientIO a) -> (a -> TransientIO b) -> StateIO EventF
setEventCont x f  = do
   st@(EventF es c vs x' fs)  <- get
   put $ EventF es c vs  x ( f: unsafeCoerce fs) -- st{xcomp=  x, fcomp=  f: unsafeCoerce fs}
   return st

resetEventCont cont=do
      st <- get
      put cont {eventHandlers=eventHandlers st, eventValues=eventValues st, currentEvent= currentEvent st}

getCont ::(MonadState EventF  m) => m EventF
getCont = get

runCont :: EventF -> StateIO ()
runCont (EventF _ _ _ x fs)= do runIt x (unsafeCoerce fs); return ()
   where
      runIt x fs= runTrans $  x >>= compose fs



      compose []= const empty
      compose (f: fs)= \x -> f x >>= compose fs


eventLoop :: [Event] ->  StateIO ()
eventLoop []= return()
eventLoop (ev@(Event name _):evs)= do
   (modify $ \st -> st{currentEvent= Just ev ,eventValues= M.delete name $ eventValues st})  !> ("inject event:" ++ name)
   ths <- gets eventHandlers
   case M.lookup name ths of
      Just st -> runCont st  !> ("execute event handler for: "++ name) 
      Nothing -> return ()   !> "no handler for the event"
   eventLoop evs

instance   Functor TransientIO where
  fmap f x= Transient $ fmap (fmap f) $ runTrans x


instance Applicative TransientIO where
  pure a  = Transient  .  return $ Just a
  Transient f <*> Transient g= Transient $ 
       f >>= \ k ->
       g >>= \ x ->
       return $  k <*> x

instance  Alternative TransientIO where
  empty= Transient $ return  Nothing
  Transient f <|> Transient g= Transient $ do
       x <- f
       y <- g
       return $ x <|> y


instance Monad TransientIO where
      return x = Transient $ return $ Just x
      x >>= f = Transient $ do
        cont <- setEventCont x  f
        mk <- runTrans x
        resetEventCont cont
        case mk of
           Just k  -> runTrans $ f k
           Nothing -> return Nothing

instance MonadTrans (Transient ) where
  lift mx = Transient $ mx >>= return . Just

instance MonadIO TransientIO where
  liftIO = lift . liftIO --     let x= liftIO io in x `seq` lift x



type EvType = String
data Event = Event EvType Dynamic



currentEventValue :: Typeable a => String -> TransientIO a
currentEventValue name =  do
  st <- get !> "currValue"
  let vals= eventValues st
  case M.lookup name vals of
      Nothing -> waitEvent name
      Just v  -> return $ fromDyn v (error "currentEventValue: type error") 
      
  
waitEvent :: Typeable a => String -> TransientIO a
waitEvent name = Transient $ do
  st <- get !> "waitEvent"
  let evs = eventHandlers  st 

  case  M.lookup name evs of
    Nothing ->  do
       put st{ eventHandlers=  M.insert name st  evs} !> ("created event handler for: "++ name)
       return Nothing 
    Just _ ->  do
       put st{ eventHandlers=  M.insert name st evs} !> ("upadated event handler for: "++ name)
       eventValue name

eventValue name =  do
   st <- get
   let me= currentEvent st 
   case me of
     Nothing -> return Nothing   !> "NO current EVENT"
     Just (Event name' r) -> do
      if name /= name' then return Nothing  else do
        case fromDynamic r of
          Nothing -> return Nothing 
          Just x -> do 
            liftIO $ putStrLn $ "read event: " ++ name
            put st{eventValues= M.insert name (toDyn x) $ eventValues st}
            return $ Just x
```

## Running the applicative case
(**Press return to make advance the computation**)

And now an applicative combination of events values. This case is more similar to a  traditional , non monadic, functional reactive framework. Note that each event produce a `total` result, instead of executing partially the computation. Here `getCurrentEvent` get the current event of the signal associated with the event, that is nothing less, nothing more thant the value of the last event of this type.

``` active haskell
{-# START_FILE Main.hs #-}
module Main where

import Base
import Control.Monad.State.Strict
import Unsafe.Coerce
import System.IO.Unsafe
import Control.Applicative
import qualified Data.Map as M
import Data.Dynamic

-- show

threshold = 100

main= (flip runStateT) eventf0 $ do
  runTrans  $ do
     pr <- profits'
     liftIO $ if pr > threshold
         then  putStr "Passed threshold!.. mail sent to boss: " >> print pr
         else  print pr
   
  eventLoop eventList
  
  liftIO $ putStrLn "END"
  

eventList=[Event "quantity" $ int2Dyn 10
                ,Event "price" $ int2Dyn 2
                ,Event "price" $ int2Dyn 3
          ,Event "quantity" $ int2Dyn 30
                ,Event "price" $ int2Dyn 4]

  where
  int2Dyn :: Int -> Dynamic
  int2Dyn= toDyn
  

getSignal= currentEventValue

profits' :: TransientIO Int
profits'=  do
    total <- (*) <$> getSignal "quantity" <*> getSignal "price"
    liftIO $ do
         putStr $ "total="
         print $ total
         getChar
         
    return total

--/show

{-# START_FILE Base.hs #-}

{-# LANGUAGE ExistentialQuantification,FlexibleContexts,
             FlexibleInstances, UndecidableInstances, MultiParamTypeClasses #-}

-- show

module Base where

--/show

import Control.Monad.State.Strict
import Unsafe.Coerce
import System.IO.Unsafe
import Control.Applicative
import qualified Data.Map as M
import Data.Dynamic
import Debug.Trace

(!>) = const . id -- flip trace


data Transient m x= Transient  (m (Maybe x))


data EventF  = forall a b . EventF{eventHandlers:: (M.Map String EventF )
                                  ,currentEvent :: Maybe Event
                                  ,eventValues :: M.Map String Dynamic
                                  ,xcomp :: (TransientIO a)
                                  ,fcomp :: [a -> TransientIO b]}


eventf0= EventF M.empty Nothing M.empty (empty) [const $ empty]


instance MonadState EventF  TransientIO where
  get=  Transient $ get >>= return . Just
  put x= Transient $ put x >> return (Just ())

type StateIO= StateT EventF  IO

type TransientIO= Transient StateIO

runTrans ::  TransientIO x -> StateT EventF  IO (Maybe x)
runTrans (Transient mx) = mx

setEventCont ::   (TransientIO a) -> (a -> TransientIO b) -> StateIO EventF
setEventCont x f  = do
   st@(EventF es c vs x' fs)  <- get
   put $ EventF es c vs  x ( f: unsafeCoerce fs) -- st{xcomp=  x, fcomp=  f: unsafeCoerce fs}
   return st

resetEventCont cont=do
      st <- get
      put cont {eventHandlers=eventHandlers st, eventValues=eventValues st, currentEvent= currentEvent st}

getCont ::(MonadState EventF  m) => m EventF
getCont = get

runCont :: EventF -> StateIO ()
runCont (EventF _ _ _ x fs)= do runIt x (unsafeCoerce fs); return ()
   where
      runIt x fs= runTrans $  x >>= compose fs



      compose []= const empty
      compose (f: fs)= \x -> f x >>= compose fs


eventLoop :: [Event] ->  StateIO ()
eventLoop []= return()
eventLoop (ev@(Event name _):evs)= do
   (modify $ \st -> st{currentEvent= Just ev ,eventValues= M.delete name $ eventValues st})  !> ("inject event:" ++ name)
   ths <- gets eventHandlers
   case M.lookup name ths of
      Just st -> runCont st  !> ("execute event handler for: "++ name) 
      Nothing -> return ()   !> "no handler for the event"
   eventLoop evs

instance   Functor TransientIO where
  fmap f x= Transient $ fmap (fmap f) $ runTrans x


instance Applicative TransientIO where
  pure a  = Transient  .  return $ Just a
  Transient f <*> Transient g= Transient $ 
       f >>= \ k ->
       g >>= \ x ->
       return $  k <*> x

instance  Alternative TransientIO where
  empty= Transient $ return  Nothing
  Transient f <|> Transient g= Transient $ do
       x <- f
       y <- g
       return $ x <|> y


instance Monad TransientIO where
      return x = Transient $ return $ Just x
      x >>= f = Transient $ do
        cont <- setEventCont x  f
        mk <- runTrans x
        resetEventCont cont
        case mk of
           Just k  -> runTrans $ f k
           Nothing -> return Nothing

instance MonadTrans (Transient ) where
  lift mx = Transient $ mx >>= return . Just

instance MonadIO TransientIO where
  liftIO = lift . liftIO --     let x= liftIO io in x `seq` lift x



type EvType = String
data Event = Event EvType Dynamic



currentEventValue :: Typeable a => String -> TransientIO a
currentEventValue name =  do
  st <- get !> "currValue"
  let vals= eventValues st
  case M.lookup name vals of
      Nothing -> waitEvent name
      Just v  -> return $ fromDyn v (error "currentEventValue: type error") 
      
  
waitEvent :: Typeable a => String -> TransientIO a
waitEvent name = Transient $ do
  st <- get !> "waitEvent"
  let evs = eventHandlers  st 

  case  M.lookup name evs of
    Nothing ->  do
       put st{ eventHandlers=  M.insert name st  evs} !> ("created event handler for: "++ name)
       return Nothing 
    Just _ ->  do
       put st{ eventHandlers=  M.insert name st evs} !> ("upadated event handler for: "++ name)
       eventValue name

eventValue name =  do
   st <- get
   let me= currentEvent st 
   case me of
     Nothing -> return Nothing   !> "NO current EVENT"
     Just (Event name' r) -> do
      if name /= name' then return Nothing  else do
        case fromDynamic r of
          Nothing -> return Nothing 
          Just x -> do 
            liftIO $ putStrLn $ "read event: " ++ name
            put st{eventValues= M.insert name (toDyn x) $ eventValues st}
            return $ Just x


```
If there is something for this event in the `eventValues` state, then `getCurrentEvent` will return it. If not, it will call `waitEvent` to get one:


    currentEventValue :: Typeable a => EvType -> TransientIO a
    currentEventValue name =  do
      st <- get !> "currValue"
      let vals= eventValues st
      case M.lookup name vals of
          Nothing -> waitEvent name
          Just v  -> return $ fromDyn v (error "currentEventValue: type error") 


It can be said that the applicative combination of quantity and price creates a new signal: the `total` signal. When this signal pass the threshold, the computation continues, so it "send" the message to the boss.

It is easy to create computations with a combination of applicative and monadic computations as we please. The last snippet is an example of it

In the other side observe that there are no loops. That is because the monadic reactive expression is a declaration. It is not what the computer really does. Is the event scheduler, that inject the event the one that execute event handlers and run the code, not the monadic expression. The latter only configure event handlers. 

That is beatiful IMHO, and show the endless power of monads for industrial applications adaptable to any execution model and, more important, the hability to encapsulate the plumbing behind the monad instance. In this case, the nighmare of the callback hell.

## The code

is [here](https://github.com/agocorona/transient)

## Exercises

+ You can redefine the operator `(!>)` in Base.hs to have execution traces in both examples.

+ The use of the Alternative instance has not been demonstrated in this example, but can you imagine an scenario where the `<|>` operator can be used to create more complex signals?

## Other similar projects

Atze van der Ploeg - [Monadic Functional Reactive Programming](http://homepages.cwi.nl/~ploeg/papers/monfrp.pdf) and the hackage library drClickOn.

I possibly looked previously at a different version of the paper which did not mention the hackage library. I don´t remember the box examples. Anyway I looked for literature after creating hplayground, which implement basically the same monadic mechanism of this tutorial and I did not looked at the paper in detail until now. But the problems that solves and the advantages of the approach are the ones that I appreciated the most from the paper.

As far as I can understand the execution model described in that paper uses blocking, since the process is running and it advances when it receives the events expected. Therefor is not as flexible as the one proposed here, that uses a non-blocking semantics.

What I see is that the library of the paper seems to use blocking semantics. Since it uses SDL https://www.libsdl.org/ for the example. So there is a single listening point active at each computation that is looking for events. The computation continues when it receives the right event. Please correct me if someone know the example more, but it seems that the computation is in a blocking state waiting, like a console application.

My approach is different: the monad simply configure an event handler for the first `waitEvent` of the computation and then finishes. The rest is done by event handlers. when an event arrives, the event scheduler call the event handler just configured. That event handler execution may configure more event handlers if there are more waitEvent sentences downstream. Once configured, there are many listening points (callbacks), one for each waitEvent. Since there is no central IO state, the event actions can be executed fully in parallel (but, wait, this latter is not as easy....)
Once configured, there is no valid and invalid sequences of events to consider but any combination of them are valid. That means that it may execute the event handler corresponding to a previous event, upstream from the one that has received previously, while the one of the paper can not. Like in the case of the inputs of a console application, they have a strict order since the computation has a single execution point. That is his weak point. It can not manage the example of my tutorial without enclosing it in a loop and adding conditionals since the sequence has to contemplate explicitly all possible combinations of events.

But I may be very wrong. Tell me by mail or [in this reddit discussion](https://www.reddit.com/r/haskell/comments/2mrjg8/a_monad_for_reactive_programming_part_2_school_of/)

In the other side it mentioned in the paper that there are primitives to emit signals. I understand that as if the computations can send new events sort to speak. I don´t know very well the mechanism. I don not understand very well all the concepts used by functonal reactive frameworks. Neither I´m an expert in signal processing....The paper describes a more complete signal processing API, but in essence it uses also Applicative operators for composition of signals (it call it <^>). And all the primitives are in essence combinations of the basic monadic-applicative combinators that I also have defined.


## Conclusions

Monadic functional reactive frameworks have many advantages. They permit to avoid to repeat costly computations, so they may perform effects in a more controlled way. With the help of applicative combinators they can express the complex signal processing associated with specialized declarative reactive frameworks, but using standard haskell combinators.

It has been used in the web browser with the [hplayground](https://github.com/agocorona/hplayground) client-side framework. Although it uses combinators that produce rendering as well as output values (formlets) and the event scheduler is the one of the web browser, in essence the event mechanism is the same. 

Additionally it is easy to create an scheduler that may work independently from window or web programming. It can be used in event sourcing applications. Is much more convenient  and pleasant since the entire application can be programmed in a single flow, instead of splitting it in event handlers with a common state.

AMDG



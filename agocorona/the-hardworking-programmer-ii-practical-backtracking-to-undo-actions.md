# The hardworking programmer II: practical backtracking to undo actions

#Intro#
Compensations are a general mechanism for restoring state that permits to undo even IO actions. I present two primitives for transient:  `onUndo` that "annotates" undo actions attached to the monadic statements in a monadic sequence. These statements are executed in reverse order when `undo` is called. It is a kind of backtracking.

`onUndo`statements can be used to restore state variables, close streams, restore database registers, respond to the back button in a web navigation or whatever. `undo` can be invoked as part of the execution flow or to express some exceptional condition.

Additionally, with `undoCut`  the `onUndo` statements can fix some problem and resume the execution forward from this point instead of undoing further actions.

#A word on continuations#

In Haskell, the continuation is the second parameter in the bind operation. Other languages use imperative and eager execution, so they have to resort to continuations to implement special kinds of flows so  a kind of special mechanism is necessary in these languages for capturing continuations. Haskell does not have such problem: It uses continuations natively. The monad instance defines what each kind of computation has to do with these continuations.

A bind has two parameters: a closure and a continuation.

So at every moment you know what is in the continuation, in any monad. You don't need any special `Cont` structure for this. It is in the monad instance: it is the second parameter in the bind operation:

     instance Monad ...
        x >>= f = ....
        ...

In this expression:

    exp= x >>= (f1>>= f2) >>=f3 
  
When executing the second `>>=` in this expression, the closure is the result of the execution of `x>>=f1` The continuation is `f2>>=f3`. so that at every point of the execution, the expression is identical to his closure composed with his continuation:

    exp= closure >>= continuation

while the computation executes, the closure and the continuation changes, but the equality holds.  It is like if the monadic tree were expressed in terms of a [zipper](http://learnyouahaskell.com/zippers).

What the transient Monad does is to store the closure and the continuation of each bind operation in the state of a state monad. Then each statement, for example `f1`, can access to both of them to create new effectful primitives. a such statement can modify the execution flow so that it is possible to create new effects without modifying the monad instance neither stacking monad transformers.

Suppose that in `f1` depend on two events from a GUI, or, for example, the finalization of two blocking IO operations. 

    exp= x >>= ((op1 <|> op2) >>= f2) >>=f3

then `op1` and `op2` share the same closure and the same continuation. since both block, I can set up the watching of both events, stop execution of the current thread and wait. when any of the two events fire, I can store the event value in a buffer, execute the closure with each of the two buffer result  and then execute the continuation. That is how events and non blocking IO can make use of the closure and the continuation. See the first article of this serie for more details.

The Transient state contains both of them.

``` haskell
data EventF  = forall a b . EventF{xcomp :: (TransientIO a)
                                  ,fcomp :: [a -> TransientIO b]
                                  mfData :: M.Map TypeRep SData
                                  ...
                                  }
```

It also contains a Data.Map of Dynamic values, to store arbitrary data, that can be stored and retrieved by his type. This map can store any programmer-defined data. it also can store continuations of previous statements, this permits to modify the execution of the monadic expression in very sophisticated ways. 

By editing the monadic statements as if they were arrows in a graph, you can construct new effects.  That is what my Transient monad does.  In [the previous article](https://www.fpcomplete.com/user/agocorona/EDSL-for-hard-working-IT-programmers) it has been used to implement asynchronous event handling, parallelism and thread control. Now I will use it for another exotic effect, that may be very useful.


#Backtracking#

The Transient monad of [the previous article](https://www.fpcomplete.com/user/agocorona/EDSL-for-hard-working-IT-programmers) has user state management, event/signal handling, thread control, parallelism, and early termination effects. But another important effect that I wish to make available for the hard working programmer is backtracking. With this additional effect I can undo transactions and I can express a Web navigation. As I demonstrated [here](https://www.fpcomplete.com/user/agocorona/how-haskell-can-solve-the-integration-problem) with the MFlow package. The backtracking in MFlow is done using a different mechanism, explained in this article in [The monad reader](https://themonadreader.wordpress.com/2014/04/23/issue-23/). This time I will use the Transient monad for the implementation of this effect.

In the previous article I presented the Transient monad, that stores a closure and a continuation in a state monad. Can we implement backtracking without touching the Base package where the Transient monad is defined?. Yes, we can.

Instead of using intimidating words like "backtracking" as a concept, let's start with an application of it. Let's code some primitives like `undo` and `onUndo` so that we can, for example, undo the reservation of some product when the payment process fails because the user gave up for whatever reason. The semantics of these two primitives can be understood by looking at this example:

``` haskell
transaction=   do
       option "back" "backtracking test" 
       productNavigation   
       reserve   
       payment

       liftIO $ print "done!"

       where

       productNavigation = liftIO $ putStrLn "product navigation"

       reserve= liftIO (putStrLn "product reserved,added to cart")
                 `onUndo` liftIO (putStrLn "product un-reserved")

       payment = do
           liftIO $ putStrLn "Payment failed"
           undo
```

Instead of undoing the reservation manually when the fail is verified, I call `undo` and let each action undo itself, I give the responsibility to the actions themselves. The advantage is that the programmer of the flow doesn't need to care about such low level things.

To implement these primitives I will define a registration method `registerUndo` that registers a statement to be re-executed when backtracking.

I need a definition of the backtrack stack, which will contain a flag that indicates if backtracking is being executed and also will contain all the continuations of the backtracking points.

The call `registerUndo` (below) gets the continuation and stores it in the Backtrack structure.
This `Backtrack` data will be stored in the session state using `getSessionData` and `setSessionData`


``` haskell
data Backtrack= forall a b.Backtrack{backtracking :: Bool
                                    ,backStack :: [EventF]}


registerUndo :: TransientIO a -> TransientIO a
registerUndo f  = Transient $ do
      cont  <- getCont 
      md  <- getSessionData
      setSessionData $   case md of
         Just bs ->  Backtrack b $ cont:bs
         Nothing ->  Backtrack False [cont]
      runTrans f
```

`getCont` is the Transient primitive that gives the computation state at that point, including the closure and the continuation.

Then, we define the `onUndo` primitive, that has two actions as parameters:

``` haskell
    onUndo :: TransientIO a -> TransientIO a -> TransientIO a
    onUndo ac bac= registerUndo $ do
       Backtrack back _ <- getSData <|> return (Backtrack False [])
       if back then bac else ac
```

When going forward the first action is executed, but when the flag signals that `onUndo` is being executed under backtracking, the second action is executed.

And now the primitive that executes the backtracking:

``` haskell
undo :: TransientIO a
undo= Transient $ do
  bs <- getSessionData  `onNothing` return nullBack             
  goBackt  bs

  where
  nullBack= Backtrack False []
  goBackt (Backtrack _ [])= return Nothing                     
  goBackt (Backtrack b (stack@(first: bs)))= do
        put first
        setSData $ Backtrack True stack
        mr <-  runClosure first
        Backtrack back _ <- getSessionData `onNothing` return nullBack
        case back of
           True ->  goBackt $ Backtrack True bs                
           False -> case mr of
                   Nothing -> return Nothing                   
                   Just x -> runContinuation first x            

```


First It get the backtracking stack, which contains closures and continuations of different backtracking points. Then it sets the backtracking flag and executes the first closure (that is the last statement registered). If the closure changed the backtracking flag, (False) then the continuation of that closure is executed, so the flow continues forward from that statement on. If the closure returns Nothing (early termination) then `undo` and stop.

If the closure doesn't change the backtracking flag, the next backtracking point in the stack is executed in the same way until there is no more backtracking points.


This code below contains all the programs of the [Hard working programmer 1](https://www.fpcomplete.com/user/agocorona/EDSL-for-hard-working-IT-programmers) plus the backtracking example(s).


``` active haskell



{-# START_FILE main.hs #-}

{-# LANGUAGE ScopedTypeVariables #-}

module Main where

import           Base
import           Backtrack
import           Control.Applicative
import           Control.Concurrent
import           Control.Exception
import           Control.Monad.State
import           Data.Monoid
import           System.IO.Unsafe

import           Network.HTTP

import           Network
import           System.IO

-- show

main=  do
    runTransient $ do  
      async inputLoop  <|> return ()
     
      option "main" "to return to the main menu"  <|> return ""
      liftIO $ putStrLn "MAIN MENU"

      transaction <|> transaction2 <|> colors <|>
        app  <|> sum1 <|> sum2 <|> server <|> menu

    stay

transaction=   do
       option "back" "backtracking test"
       productNavigation
       reserve
       payment
      
transaction2= do
       option "back2" "backtracking test 2"
       productNavigation
       reserveAndSendMsg
       payment


       liftIO $ print "done!"

      
productNavigation = liftIO $ putStrLn "product navigation"

reserve= liftIO (putStrLn "product reserved,added to cart")
                 `onUndo` liftIO (putStrLn "product un-reserved")

payment = do
           liftIO $ putStrLn "Payment failed"
           undo

reserveAndSendMsg= do
            reserve
            liftIO $ print "MIDDLE"
            liftIO  (putStrLn "update other database necesary for the reservation")
                 `onUndo` liftIO (putStrLn "database update undone")

colors :: TransientIO ()
colors= do
       option "colors" "choose between three colors"
       r <-  color 1  "red"  <|> color 2 "green" <|> color 3 "blue"
       liftIO $ print r
       where
       color :: Int -> String -> TransientIO String
       color n str= option (show n) str >> return  str

app :: TransientIO ()
app= do
       option "app" "applicative expression that return a counter in 2-tuples every second"
       r <-  (,) <$>  number  <*> number
       liftIO $ putStrLn $ "result=" ++ show r
       where
       number= waitEvents $ do
          threadDelay 1000000
          n <- takeMVar counter
          putMVar counter (n+1)
          return  n

       counter=unsafePerformIO $ newMVar (0 :: Int)

sum1 :: TransientIO ()
sum1= do
       option "sum1" "access to two web pages concurrently and sum the number of words using Applicative"
       (r,r') <- (,) <$> async  (worker "http://www.haskell.org/")
                     <*> async  (worker "http://www.google.com/")

       liftIO $ putStrLn $ "result="  ++ show (r + r')

getURL= simpleHTTP . getRequest

worker :: String -> IO Int
worker url=do
      r <- getURL url
      body <- getResponseBody r
      putStrLn $ "number of words in " ++ url ++" is: " ++ show(length (words body))
      return . length . words $ body

sum2 :: TransientIO ()
sum2= do
       option "sum2" "access to N web pages concurrenty and sum the number of words using map-fold"
       rs <- foldl (<>) (return 0) $ map (async . worker)
                  [ "http://www.haskell.org/"
                  , "http://www.google.com/"]

       liftIO $ putStrLn $ "result="  ++ show rs

instance Monoid Int where
      mappend= (+)
      mempty= 0

server :: TransientIO ()
server=  do
       option "server" "A web server in the port 8080"
       liftIO $ print "Server Stated"
       sock <-  liftIO $  listenOn $ PortNumber 8080
       (h,_,_) <- spawn $ accept sock
       liftIO $ do
           hPutStr h msg
           putStrLn "new request"
           hFlush h
           hClose h
         `catch` (\(e::SomeException) -> sClose sock)

msg = "HTTP/1.0 200 OK\r\nContent-Length: 5\r\n\r\nPong!\r\n"


menu :: TransientIO ()
menu=  do
     option "menu"  "a submenu with two options"
     colors  <|> sum2

-- / show






{-# START_FILE Backtrack.hs #-}

{-# LANGUAGE DeriveDataTypeable #-}
{-# LANGUAGE ExistentialQuantification #-}

-- show
module Backtrack (registerUndo, onUndo, undo, retry, undoCut) where
-- /show

import Base
import Data.Typeable
import Control.Applicative
import Control.Monad.State
import Unsafe.Coerce

data Backtrack= forall a b.Backtrack{backtracking :: Bool
                                    ,backStack :: [EventF]}
                                    deriving Typeable

-- | Assures that backtracking will not go further
undoCut :: TransientIO ()
undoCut= Transient $ do
     delSessionData $ Backtrack False []
     return $ Just ()

-- | The second parameter will be executed when backtracking
{-# NOINLINE onUndo #-}
onUndo :: TransientIO a -> TransientIO a -> TransientIO a
onUndo ac bac= do
   r<-registerUndo $ Transient $ do
     Backtrack back _ <- getSessionData `onNothing` return (Backtrack False [])
     runTrans $ if back then bac  else ac
   return r
  
-- | Register an action that will be executed when backtracking
{-# NOINLINE registerUndo #-}
registerUndo :: TransientIO a -> TransientIO a
registerUndo f  = Transient $ do
   cont@(EventF _ _ _ i _ _ )  <- get   !> "backregister"
   md  <- getSessionData
   setSessionData $   case md of
        Just (bss@(Backtrack b (bs@((EventF _ _ _ i' _ _ ):_)))) -> if False then bss else  Backtrack b $ cont:bs
        Nothing ->  Backtrack False [cont]
   runTrans f

-- | Restart the flow forward from this point on
retry :: TransientIO ()
retry= do
    Backtrack _ stack <- getSessionData `onNothing` return (Backtrack False [])
    setSData $ Backtrack False stack

-- | Execute backtracking. It executes the registered actions in reverse order.
--
-- If the backtracking flag is changed, the flow proceeds forward from that point on.
--
-- If the backtrack stack is finished or undoCut executed, `undo` will stop.
undo :: TransientIO a
undo= Transient $ do
  bs <- getSessionData  `onNothing` return nullBack            !>"GOBACK"
  goBackt  bs

  where
  nullBack= Backtrack False []
  goBackt (Backtrack _ [])= return Nothing                     !> "END"
  goBackt (Backtrack b (stack@(first@(EventF x fs _ _ _  _ ): bs)))= do
        put first{replay=True}
        setSData $ Backtrack True stack
        mr <-  runClosure first                                !> "RUNCLOSURE"
        Backtrack back _ <- getSessionData `onNothing` return nullBack
                                                               !>"END RUNCLOSURE"
        case back of
           True ->  goBackt $ Backtrack True bs                !> "BACK AGAIN"
           False -> case mr of
                   Nothing -> return empty                     !> "FORWARD END"
                   Just x ->  runContinuation first x          !> "FORWARD EXEC"


{-# START_FILE Base.hs #-}

-----------------------------------------------------------------------------
--
-- Module      :  Base
-- Copyright   :
-- License     :  GPL (Just (Version {versionBranch = [3], versionTags = []}))
--
-- Maintainer  :  agocorona@gmail.com
-- Stability   :
-- Portability :
--
-- |
--
-----------------------------------------------------------------------------
{-# LANGUAGE ExistentialQuantification,FlexibleContexts,
             FlexibleInstances, MultiParamTypeClasses #-}

-- show
module Base  where
-- /show

import Control.Monad.State
import Unsafe.Coerce
import System.IO.Unsafe
import Control.Applicative
import qualified Data.Map as M
import Data.Dynamic
import Debug.Trace
import Data.Monoid

--import Data.IORef
import Control.Concurrent
import Control.Concurrent.STM
import GHC.Conc
import Data.Maybe
import System.Mem.StableName
import Data.List

(!>) =  const . id -- flip trace
infixr 0 !>

data Transient m x= Transient  {runTrans :: m (Maybe x)}
type SData= ()

type EventId= Int



data EventF  = forall a b . EventF{xcomp :: TransientIO a
                                  ,fcomp :: a -> TransientIO b
                                  ,mfData :: M.Map TypeRep SData
                                  ,mfSequence :: Int
                                  ,row :: P RowElem
                                  ,replay :: Bool
                                  }

type P= MVar

type Buffer= Maybe ()
type NodeTuple= (EventId, ThreadId, Buffer)

type Children=  Maybe (P RowElem)

data RowElem=   Node NodeTuple |  RowList Row Children

instance Show RowElem where
  show (Node (e,_,_))= show e
  show (RowList r ch)= show ( reverse r)  ++ "->" ++ show ch

type Row = [P RowElem]

instance Eq NodeTuple where
     (i,_,_) ==  (i',_,_)= i == i'


instance Show x => Show (MVar x) where
  show  x = show (unsafePerformIO $ readMVar x)

eventf0= EventF  empty (const  empty) M.empty 0
          rootRef False

-- {-# NOINLINE topNode #-}
-- topNode= (-1 :: Int,unsafePerformIO $ myThreadId,False,Nothing)

{-# NOINLINE rootRef #-}
rootRef :: MVar RowElem
rootRef=  unsafePerformIO $ newMVar $ RowList []  Nothing                     

instance MonadState EventF  TransientIO where
  get=  Transient $ get >>= return . Just
  put x= Transient $ put x >> return (Just ())

type StateIO= StateT EventF  IO

type TransientIO= Transient StateIO

--runTrans ::  TransientIO x -> StateT EventF  IO (Maybe x)
--runTrans (Transient mx) = mx

runTransient :: TransientIO x -> IO (Maybe x, EventF)
runTransient t= runStateT (runTrans t) eventf0


newRow :: MonadIO m => m (P RowElem)
newRow= liftIO $ newMVar $ RowList [] Nothing

setEventCont ::   TransientIO a -> (a -> TransientIO b) -> StateIO EventF
setEventCont x f  = do
   st@(EventF   _ fs d _  ro r)  <- get
   n <- if replay st then return $ mfSequence st
     else  liftIO $ readMVar refSequence
   ro' <- newRow
   ro `eat` ro'
   put $ EventF   x ( \x -> f x >>= unsafeCoerce fs) d n  ro' r !> ("stored " ++ show n)
   return st

eat ro ro'= liftIO $
 modifyMVar_  ro $ \(RowList es t) -> return $ RowList (ro':es) t

resetEventCont (EventF x fs _ _  _ _)=do
   st@(EventF   _ _ d  n  ro r )  <- get
   put $ EventF  x fs d n  ro r


getCont ::(MonadState EventF  m) => m EventF
getCont = get

runCont :: EventF -> StateIO ()
runCont (EventF  x fs _ _  _ _)= do runIt  x (unsafeCoerce fs); return ()
   where
   runIt  x fs= runTrans $ do
         st <- get
         --put st{mfSequence=i}
         r <- x
         put st
         fs r


runClosure :: EventF -> StateIO (Maybe a)
runClosure (EventF x _ _ _ _ _) =  unsafeCoerce $ runTrans x

runContinuation ::  EventF -> a -> StateIO (Maybe b)
runContinuation (EventF _ fs _ _ _ _ ) x= runTrans $  (unsafeCoerce fs) x

instance   Functor TransientIO where
  fmap f x=   Transient $ fmap (fmap f) $ runTrans x --


instance Applicative TransientIO where
  pure a  = Transient  .  return $ Just a
  Transient f <*> Transient g= Transient $ do
       k <- f
       x <- g
       return $  k <*> x

instance  Alternative TransientIO where
  empty= Transient $ return  Nothing
  Transient f <|> Transient g= Transient $ do
       k <- f
       x <- g
       return $  k <|> x


-- | A synonym of empty that can be used in a monadic expression. It stops the
-- computation.
stop :: TransientIO a
stop= Control.Applicative.empty

instance Monoid a => Monoid (TransientIO a) where
  mappend x y = mappend <$> x <*> y 
  mempty= return mempty

instance Monad TransientIO where
      return x = Transient $ return $ Just x
      x >>= f  = Transient $ do
        cont <- setEventCont x  f
        mk <- runTrans x
        resetEventCont cont
        case mk of
           Just k  -> do addDescent' !> "ADDROW" ; runTrans $ f k

           Nothing -> return Nothing

        where
        addDescent'= do
            r <- gets row
            n <- addDescent r
            modify $ \s -> s{row= n}
addDescent r=
            liftIO $ do
              n <- newMVar $ RowList [] Nothing
              modifyMVar_ r $ \(RowList ns ch) ->  return $ RowList  ns $ Just n
             --   case ch of
             --     Just x -> error $ "children not empty: "++ show x
             --     Nothing ->  return $ RowList  ns $ Just n
              return n

addChild row ref= modifyMVar_  row $ \(RowList ns t) -> return $  RowList (ref : ns) t
  
instance MonadTrans (Transient ) where
  lift mx = Transient $ mx >>= return . Just

instance MonadIO TransientIO where
  liftIO = lift . liftIO --     let x= liftIO io in x `seq` lift x



-- | Get the session data of the desired type if there is any.
getSessionData ::  (MonadState EventF m,Typeable a) =>  m (Maybe a)
getSessionData =  resp where
 resp= gets mfData >>= \list  ->
    case M.lookup ( typeOf $ typeResp resp ) list of
      Just x  -> return . Just $ unsafeCoerce x
      Nothing -> return $ Nothing
 typeResp :: m (Maybe x) -> x
 typeResp= undefined

-- | getSessionData specialized for the View monad. If Nothing, the monadic computation
-- does not continue. getSData is a widget that does not validate when there is no data
--  of that type in the session.
getSData :: MonadState EventF m => Typeable a =>Transient m  a
getSData= Transient getSessionData


-- | setSessionData ::  (StateType m ~ MFlowState, Typeable a) => a -> m ()
setSessionData  x=
  modify $ \st -> st{mfData= M.insert  (typeOf x ) (unsafeCoerce x) (mfData st)}

-- | A shorter name for setSessionData.
setSData ::  ( MonadState EventF m,Typeable a) => a -> m ()
setSData= setSessionData

delSessionData x=
  modify $ \st -> st{mfData= M.delete (typeOf x ) (mfData st)}

delSData :: ( MonadState EventF m,Typeable a) => a -> m ()
delSData= delSessionData

withSData ::  ( MonadState EventF m,Typeable a) => (Maybe a -> a) -> m ()
withSData f= modify $ \st -> st{mfData=
    let dat = mfData st
        mx= M.lookup typeofx dat
        mx'= case mx of Nothing -> Nothing; Just x -> unsafeCoerce x
        fx=  f mx'
        typeofx= typeOf $ typeoff f
    in  M.insert typeofx  (unsafeCoerce fx) dat}
    where
    typeoff :: (Maybe a -> a) -> a
    typeoff = undefined
----

genNewId :: MonadIO m => MonadState EventF m =>  m Int
genNewId=  do
      st <- get
      case replay st of
        True -> do
          let n= mfSequence st
          put $ st{mfSequence= n+1}
          return n
        False -> liftIO $
          modifyMVar refSequence $ \n -> return (n+1,n)

{-# NOINLINE refSequence #-}
refSequence :: MVar Int
refSequence= unsafePerformIO $ newMVar 0


--- IO events

--buffers :: IORef [(EventId,Dynamic)]
--buffers= unsafePerformIO $ newIORef []

data Loop= Once | Loop | Multithread deriving Eq

waitEvents ::  IO b -> TransientIO b
waitEvents= parallel Loop


async  :: IO b -> TransientIO b
async = parallel Once

spawn= parallel Multithread

parallel  ::  Loop ->  IO b -> TransientIO b
parallel hasloop receive =  Transient $ do
      cont <- getCont
      id <- genNewId
      liftIO $ forkCont id hasloop receive cont

forkCont::  EventId -> Loop -> IO a -> EventF -> IO (Maybe a)
forkCont id hasloop receive cont= do
      let currentRow= row cont
      mnode  <-   liftIO $ lookTree id currentRow !> ("idToLook="++ show id++ " in: "++ show currentRow)
     
      case mnode of
        Nothing ->do
                 return () !> "NOT FOUND"
                 forkCont' id cont hasloop receive
                 return Nothing
       
        Just (node@(id',th', mrec)) -> do
         -- modify $ \cont -> cont{nodeInfo=Nothing}
          return $ if isJust mrec then Just $ unsafeCoerce $ fromJust mrec else Nothing

        where
        forkCont' id cont hasloop receive= liftIO $ forkIO $ do
                     th <- myThreadId
                     ref <-newMVar  $  Node (id,th,Nothing)
                     addChild (row cont) ref

                     loop hasloop  receive $ \r -> do
                       modifyMVar_  ref $ \(Node(i,th,_)) -> return
                                       $ Node(i,th,Just $ unsafeCoerce r)
                       (flip runStateT) cont $ do
                           cont@(EventF  x fs _  _ _ _) <- get
                          
                           put cont{replay= True{-,-mfSequence=i,-}{-nodeInfo=Just ref-}}
                          
                           mr <- runClosure cont
                           case mr  of
                             Nothing ->return Nothing
                             Just r ->do
                               row1 <- gets row
                               liftIO $ delEvents  row1              !> ("delEvents: "++ show row1)
                               id <- liftIO $ readMVar refSequence
                               n <-  addDescent  row1
                               modify $ \cont -> cont{row=n,replay= False,mfSequence=id } !> ("SEQ=" ++ show(mfSequence cont))
                               runContinuation cont r
                       return ()



        loop Once rec x  = rec >>= x
        loop Loop rec f = do
            r <- rec
            f r
            loop Loop rec f

        loop Multithread rec f = do
            r <- rec
            forkIO $ f r
            loop Multithread rec f

        lookTree :: EventId -> P RowElem -> IO (Maybe NodeTuple)
        lookTree id ref=  do
            RowList ns _<- readMVar ref
            lookList id ns



        lookList id mn= case mn of
              [] -> return Nothing
              (p:nodes) -> do
                  me <- readMVar p
                  case me of
                    Node(node@((id',_,_))) ->
                      if id== id'
                         then return $ Just node
                         else lookList id nodes
                    RowList row _ -> do
                         mx <- lookList id nodes
                         case mx of
                           Nothing -> lookList id row
                           Just x -> return $ Just x
        delEvents :: P RowElem  -> IO()
        delEvents ref = do
            RowList mevs mch <- takeMVar ref
            maybeDel mch
            putMVar ref $ RowList mevs Nothing

        maybeDel mch=  case mch of
              Nothing -> return ()
              Just p -> do
                  RowList es mch' <- readMVar p
                  delList es !> ("toDelete="++ show es)
                  maybeDel mch'


        delList es=  mapM_ del es where
          del p = readMVar p >>= del'
          del' (Node(node@(_,th,_)))= killThread th !> ("DELETING " ++ show node)
          del' (RowList l mch)= delList l >> maybeDel mch


type EventSetter eventdata response= (eventdata ->  IO response) -> IO ()
type ToReturn  response=  IO response
react
  :: Typeable eventdata
  => EventSetter eventdata response
  -> ToReturn  response
  -> TransientIO eventdata

react setHandler iob= Transient $ do
        cont    <- getCont
        mEvData <- getSessionData
        case mEvData of
          Nothing -> do
            liftIO $ setHandler $ \dat ->do
--              let cont'= cont{mfData = M.insert (typeOf dat)(unsafeCoerce dat) (mfData cont)}
              runStateT (setSData dat >> runCont cont) cont
              iob
            return Nothing
          Just dat -> delSessionData dat >> return (Just  dat)


{-# NOINLINE getLineRef #-}       
getLineRef= unsafePerformIO $ newTVarIO Nothing


option1 x  message=  inputLoop `seq` (waitEvents  $ do
     liftIO $ putStrLn $ message++"("++show x++")"
     atomically $ do
       mr <- readTVar getLineRef
       th <- unsafeIOToSTM myThreadId
       case mr of
         Nothing -> retry
         Just r ->
            case reads1 r !> ("received " ++  show r ++  show th) of
            (s,_):_ -> if  s == x  !> ("waiting" ++ show x)
                     then do
                       writeTVar  getLineRef Nothing !>"match"
                       return s

                     else retry
            _ -> retry)
     where
     reads1 s=x where
      x= if typeOf(typeOfr x) == typeOf "" then unsafeCoerce[(s,"")] else readsPrec 0 s
      typeOfr :: [(a,String)] ->  a
      typeOfr  = undefined

option ret message= do
    liftIO $ putStrLn $"Enter "++show ret++"\tto: " ++ message
    waitEvents  $ getLine' (==ret)
    liftIO $do putStrLn $ show ret ++ " chosen"
    return ret
   
getLine' cond=   inputLoop `seq` do
     atomically $ do
       mr <- readTVar getLineRef
       th <- unsafeIOToSTM myThreadId
       case mr of
         Nothing -> retry
         Just r ->
            case reads1 r !> ("received " ++  show r ++ show th) of
            (s,_):_ -> if cond s  !> show (cond s)
                     then do
                       writeTVar  getLineRef Nothing !>"match"
                       return s

                     else retry
            _ -> retry
     where
     reads1 s=x where
      x= if typeOf(typeOfr x) == typeOf "" then unsafeCoerce[(s,"")] else readsPrec 0 s
      typeOfr :: [(a,String)] ->  a
      typeOfr  = undefined

inputLoop=  do
    print "Press end to exit"
    inputLoop'
    where
        inputLoop'= do
           r<- getLine                      !> "started inputLoop"
           if r=="end" then putMVar rexit () else do
              atomically . writeTVar  getLineRef $ Just r
              inputLoop'


rexit= unsafePerformIO newEmptyMVar

stay=  takeMVar rexit

onNothing iox iox'= do
       mx <- iox
       case mx of
           Just x -> return x
           Nothing -> iox'

          
```


If you press the option "back", it executes the backtracking test, corresponding to the first snippet of code in this article. The sequence portrayed here is the one intended:

      "back" chosen
      product navigation
      product reserved,added to cart
      Payment failed
      product un-reserved

This is a simple `undo` with one single backtracking point, but suppose that the `reserve` call updates a database, but, for some reason, it is necessary in the future to update a second database, so you add to `reserve` this modification without changing the main flow:

``` haskell
reserve= do
            liftIO (putStrLn "product reserved,added to cart")
                 `onUndo` liftIO (putStrLn "product un-reserved")
                
            liftIO  (putStrLn "update other database necessary for the reservation")
                 `onUndo` liftIO (putStrLn "database update undone")
```

The `undo` in the main flow will undo both changes.

There are two more primitives in the library

- `undoCut` to empty the stack, so previous back points will not be executed by the next `undo`
- `retry` changes the backtracking flag, so the flow will proceed forward from that point on

You can play with them and tell me the about the results.

The Transient repo:

        https://github.com/agocorona/transient


#Conclusions and future work#
With the use of session state and backtracking it is possible to do complex navigations when exploring tree structures and even doing web navigations.  I plan to adapt MFlow to this transient Monad.

Execution state persistence, like the Workflow and MFlow packages is also necessary for the hardworking programmer. This can be done by storing events and replaying them. Check them out!

This is one more effect added to my hardworking programmer super-monad. It is intended to super-charge the Haskell newbie with a set of powerful but intuitive primitives ad combinators to give unprecendented expressive power without adding complexity.

More effects to come!



Thanks to: Aistis Raulinaitis
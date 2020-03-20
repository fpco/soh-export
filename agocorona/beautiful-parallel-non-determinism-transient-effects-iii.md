# Beautiful parallel non-determinism (Transient effects III)

What?
=====


I will demonstrate how the Transient monad can process many elements and filter them at the same time using `guard`, like the list monad, but in this case, running parallel threads. The programmer can decide how many threads will execute. 

I will also use the same primitives for a parallel search in a filesystem, followin an [example](http://tldrify.com/9hg) on the book "parallel and concurrent Haskell" in the chapter [Parallel programming using threads](http://chimera.labs.oreilly.com/books/1230000000929/ch13.html) from Simon Marlow. We will see that the high level approach used for non-determinism solve the problem of parallel search with more generality and with less modifications of the straighforward single threaded code.

Finally I will show the relation between logic programming in a language like Prolog or curry with the nondeterminist examples.

The naive parallelization of non deterministic computations may produce potentially a big amount of threads and the granularity may be smaller than the optimal if each element has a light processing that does not justify spawning a thread for it. To control this situation, the Transient monad now incorporates primitives for such purpose. These primitives can be used in any other scenario where granularity of paralelization is a problem.

#About Transient

In the hardworking programmer series, I develop a "supercharged" EDSL, based on the Transient monad with many high level effects out of the box. Some are classical effect with a new twist, like early termination or state. Some others are high level, including some ones that languages and operating systems do not bring or bring them at a low level, like implicit threading, event handling or backtracking. 

The purpose is to enpower the programmer. I just want to ease the introduction of people to haskell and let them be productive from the beginning. Transient has some uncommon effects that makes it to look like a very high level language. In the first article I argued that these effects will permit to program composable software even in the presence of multithreading and events.

Unlike other EDSLs, the transient effects are not created by aggregating monad transformers, neither using an effect system. It is done manipulating continuations in a new way, a kind of meta-programming.

The repository with the last version of Transient and the examples is at

[https://github.com/agocorona/transient](https://github.com/agocorona/transient)


# About the Transient Monad

In the transient monad each statement can have access to his own continuation. The monad also carries programmer-defined state. Therefore, it is possible to edit the continuation at run time. It is also possible to notify other statements about some condition by adding information to the state. The best example of this manipulation is the [backtracking effect](https://www.fpcomplete.com/user/agocorona/the-hardworking-programmer-ii-practical-backtracking-to-undo-actions). 

*Effect primitives in Transient are like the nodes of a  living flowchart that could rewire it at run time*, to do more than what the original flowchart supposed to do. That kind of metaprogramming is the way the transient statements can create new effects. In category theoretical terms, the effects edit the arrows of the continuations.

This is an unexplored territory. **I´m currently researching new possibilities. It is impossible for me to explain the details every step, but I hope that you may catch the power of the concept and forgive me for my bad writing habits. I´m rewriting this from time to time since this is an unfinished work**. 

from the previous version, I added better and simpler internals, and primitives for thread control `threads`, `oneThread` and `freeThreads`, added for solving parallel indeterminst problems, as well as some primitives for the same purpose: `choose` and  `collect`. 

I will describe them here. 

But first, a long disquisition that you may obviate if you are not interested in the details. step over it if you want to go to the "meat"

# About Backtracking and Indeterminism

There is a legitimate confusion between backtracking and indeterminism. Backtracking in `Transient` means to resume execution at a previous statement, but not only that. The execution can go further back until a cut of backtracking is found. Alternatively it can continue forward depending on some primitives used for flow control. 

In the other side, Indeterminism, exemplified in the List monad, is the effect that process and return more than one value. However some people understand that the List monad perform a kind of backtracking. Details about this follows:

After publishing the [Backtracking example](https://www.fpcomplete.com/user/agocorona/the-hardworking-programmer-ii-practical-backtracking-to-undo-actions)  Jello_Raptor  asked me:

*Err, Why not just use List as a monad to get the same effect? I mean that's just what using List for recursive descent parsers does, no? If you need state and things, there's always ListT (or one of many implementations thereof).

And I answered this:

I do not know how to use the list monad for backtracking. I know how to use it for indeterminism

This problem (undoing IO actions) is not similar to the one of the example here that I think is what you have in your mind:

refering to [this article](http://www.randomhacks.net/2007/03/12/monads-in-15-minutes)

In the above article each combination is filtered against the guard condition. Really in that example there is no backtracking, but indeterminism. It is a list monad that compute all the responses "at the same time". Only the ones that pass the guard condition are shown. As always to remove the magic and understand what is going on, the best thing is to desugar everything. In this case, if you change the bind operation by `concatMap`. You will see that it is ordinary list processing going on there.

Lazy evaluation in fact makes the backtracking simulated by the list monad a bit more real, since the list is created on demand, and the guard forces the evaluation of each element of the list by executing the code within 'choose', but this is no different from how any other monad des it executing lazy statements. That is not privative of the List monad. Laziness assures that every pure computation implement "backtracking in some way", since it forces the execution of unevaluated expressions coming from statements above them.  But is the hability to carry many values (indeterminism) of the List monad the effect that permits to simulate the execution of different alternatives.

To summarize, non-determinism can simulate backtracking in some problems like this, where a tree of combinations are explored. A deterministic program would need either execute explicit loops or do true backtracking to solve the problem of the above link, but in the case of my article about backtracking, there is no tree of alternatives to explore. The problem is to undo actions which have undo "annotations" attached. and the list monad can do nothing with it.

The backtracking effect demonstrated in my article is different; While in the case of the list it explores alternatives simulating being restarted from the same point again and again choosing of another alternative each time (it is not that, but it simulates this behavior), in my case the backtracking does his job while it is going back (by undoing or compensating actions) This is beyond what any indeterministic monad can do. However my monad can reverse the execution flow and go forward (with retry) then backward (with undo), then forward etc and can simulate an indeterministic effect if `onUndo` feed the flow with different values each time. In this case it would be backtracking simulating indeterminism, while the list monad execute indeterminism simulating backtracking. That is closely related, but not the same than exploring a tree deep first versus doing it breath first. Laziness complicates the analogy in the case of the list monad

--- end of response ----

I will not simulate indeterminism doing backtracking, because it is easier to perform indeterminism as such, by mimicking the behaviour of the List monad. But the list monad does not perform parallelization. In addition Transient permits paralelization.

# Non-determinism in the List monad

This is the code of the indeterminist example  above (that the author call backtracking) using the list monad:

```active haskell

import Control.Monad

solveConstraint :: [(Int,Int)]
solveConstraint = do
  x <- choose [1,2,3]
  y <- choose [4,5,6]
  guard (x*y == 8)
  return (x,y)
  where
  choose = id

main= print solveConstraint

```

If we execute and print the result of `solveConstraint` we get 

>[(2,4)]

the magic is operated by the List monad and the `guard` operation. I will not enter in the details. The best way to understand it, as i said above, is to desugar everything until the monad is expressed in terms of pure function application and see that `solveConstraint` is normal list processing. 

# MonadPlus and guard

`guard` need the [MonadPlus](http://en.wikibooks.org/wiki/Haskell/MonadPlus) class. It is the indeterminism class. It is in charge of adding results to a computation.  the methods of `MonadPlus` are `mzero`, when no result is added, and `mplus`  when a result is added to the previous ones.

In the list monad, `MonadPlus`is defined as:


```haskell
instance MonadPlus [] where
  mzero = []
  mplus = (++)
```

`guard` is defined polimorphically for all the monads that have the `MonadPlus` instance as:

```haskell
guard :: MonadPlus m => Bool -> m ()
guard True  = return ()
guard _ = mzero
```

look at the desugaring of [guard in the MonadPlus](http://tldrify.com/7w8) document to understand how it filter result in the list monad.

# MonadPlus and guard in Transient

Let's do it for the Transient monad. The MonadPlus instance would be:

```haskell
instance MonadPlus TransientIO where
    mzero= stop
    mplus (Transient x) (Transient y)=  Transient $ do
         mx <- x
         case mx of
             Nothing -> y
             justx -> return justx
```
`stop`  is the call that interrupt the execution of the monad. It is defined as a Transient statement that return Nothing:

```haskell
stop= Transient $ return Nothing
```

`mplus` executes the first expression, if the first gives Nothing, then it executes the second.

# Input a list of values and process them in parallel

We need to introduce a list of values in the monad.

```haskell
-- | slurp a list of values and process them in parallel . To limit the number of processing
-- threads, use `threads`
choose  ::  [a] -> TransientIO a
choose   xs = do
    evs <- liftIO  $ newMVar xs  
    parallel  $ do
           es <- takeMVar evs
           putMVar evs $ tail es  

           case es of
            [x] -> return $ Left x
                else
            (x:_)  return $ Right x 
```

`choose` stores the set of values in a IORef, then `parallel` extract each value from thar IORef. If there is no more, it stop the computation returning a Left value 

choose uses `parallel`, the basic primitive of transient, that executes his continuation every time that the IO computation return a value. it has changed. Now the signature is simpler:

```haskell
parallel  ::   IO (Either a a) -> TransientIO a
```

When the IO computation return a `Left` value, parallel executes and finish. When it retun a `Rigth` value, it does the same, but reexecutes the IO computation until a `Left` value arrives

Well, there is a much more elegant way to do `choose`:

```haskell
choose' :: [a] -> TransientIO a
choose'  xs = foldl (<|>) empty $ map (parallel . return . Left) xs

```

But it is a bit slower, since it uses a number of `parallel` statements equal to the number of elements, instead of one. 


# Control of parallelization

`parallel` launch a new thread. This thread wait for outputs of the IO procedure passed as the parameter. Then it executes the continuation. Everithing that happens after parallel is in this new thread.

Another new thing is `freeThreads`. It avoids thread control. All the threads created after it control their lifecycle themselves. A non free thread can be killed by the parent process with `killchilds`

It is possible however to restrict the number of threads. The example above would produce nine threads. One for each element of the first `choose`. Each one of them launches one thread for each element of the second choose. if there are more elements (N) and more choose's (NC), the number of threads would grow exponentially  N^NC.  That is not optimal, taking into account also that the processing done by each thread may be very light. In this case, only the processing is just the guard operation. To be effective, one thread must process more than one guard.

To avoid it,  `threads` set the maximum number of threads that may run simultaneously. If the maximum is reached, each new `parallel` statement does not spawn a thread, but execute his loop within the current thread, until some other thread dies. Then the next `parallel` will use this free slot.

finally `oneThread` assures that all the threads generated after this combinator are killed before continuing the execution. In the previous versions, this was done automatically by `parallel`. This decoupling allows for more flexibility.
````haskell
oneThread :: TransientIO a -> TransientIO a
```

# List-like processing using the Transient monad

Finally, with the definitions of `choose` and `guard` the code in the Transient monad is almost the same than for the list monad. Remember that `choose` is essentially the primitive `parallel` feeded with different values:

```haskell
import Base
import Indeterminism

solveConstraint=  do
      x <- choose  [1,2,3]
      y <- choose  [4,5,6]
      guard $ x * y == 8
      return (x,y)

main= keep $ do
      s <- threads 4 solveConstraint
      liftIO $ print s

```

`keep` is a new call that run the transient monad and keep the application running. It also wait for text input.

# Using the parallel non-determinist effects in Transient to perform parallel search

The true power of parallel non-determinism can only be extracted when working with large datasets and  combined with other effects. Then, non-determinism can magically convert a single-threaded program into a multithreaded program almost without changes.

Let's see the Simon Marlow [example](http://tldrify.com/9hg) in the Book *"parallel and Concurrent programming in Haskell"*. 
The program, using a single thread, crawl the filesystem from a given directory to all the subdirectories looking for a file. The program return when a file is found.



```haskell
find :: String -> FilePath -> IO (Maybe FilePath)
find s d = do
  fs <- getDirectoryContents d                         -- 1
  let fs' = sort $ filter (`notElem` [".",".."]) fs    -- 2
  if any (== s) fs'                                    -- 3
     then return (Just (d </> s))
     else loop fs'                                     -- 4
 where
  loop [] = return Nothing                             -- 5
  loop (f:fs)  = do
    let d' = d </> f                                   -- 6
    isdir <- doesDirectoryExist d'                     -- 7
    if isdir
       then do r <- find s d'                          -- 8
               case r of
                 Just _  -> return r                   -- 9
                 Nothing -> loop fs                    -- 10
       else loop fs                                    -- 11
```

*Parallelizing this program is not entirely straightforward because doing it naively could waste a lot of work; if we search multiple subdirectories concurrently and we find the file in one subdirectory, then we would like to stop searching the others as soon as possible. Moreover, if an error is encountered at any point, then we need to propagate the exception correctly. We must be careful to keep the deterministic behavior of the sequential version, too. If we encounter an error while searching a subtree, then the error should not prevent the return of a correct result if the sequential program would have done so.*

To use the async library, it is necessary to transform the single threaded program in some contorted ways. Instead, the non-determinist effect with paralelization using the Transient monad as explained above, essentially allows you to parallelize the single-threaded program with few changes. 

The transformed program is:


```haskell
find' :: String -> FilePath -> TransientIO FilePath
find' s d = do
  fs <- liftIO $ getDirectoryContents d                -- 1
  let fs' = sort $ filter (`notElem` [".",".."]) fs    -- 2
  if any (== s) fs'                                    -- 3
     then  return $ d</> s

     else do
       f <- choose fs'                                 -- 4
       let d' = d </> f                                -- 6
       isdir <- liftIO $ doesDirectoryExist d'         -- 7
       if isdir then find' s d'                        -- 8
                else stop        -- not a dir, not the file 
                                 -- looked for, end
```

Instead of the loop, I use  `choose`, defined above, to carry out the recursive search in the tree of directories.

That's all. This is the whole of it. Well, find' is executed in parallel many times and calls itself. It can return many matches and can execute up to one thread for each `choose` if not limited. If we run `find'` "as is", it will return files and it will not stop until all the directory tree is explored. In a big filesystem probably will allocate too much RAM since the number of simultaneous threads is unlimited.

To control  the execution, we use `collect`:


```haskell
main= keep $  do
    r<- threads 10 . collect 1 $ find' "Main.hs"  ".."
    liftIO $ putStrLn $ "Found= "++ show  r
```


`collect` monitors the threads spawned by `find'` and the number of responses. When the number of responses are the number requested, it stop all the running `find'`processes. If there is no thread active because the process has finished and there a not enough responses, it return the ones found. In this example the search uses ten threads and we look for one file.

Yo can see that there are separated combinators for each functionality: thread control, monitoring and thread spawning. That allows for more use cases and more flexible programs with a few of these combinators. 


# Running examples

All the examples above and some more can be run below. You must choose the option "nondet". Then you see a submenu with all the examples.

To see the effect of restricting the number of threads, one of the samples, the thread sample, display the result together with the identifier of the thread that generated it. You can input the max number of simultaneous threads and see how the order of the responses and the thread identifiers change.

I also included all the examples of the previous articles, that demonstrates backtracking for undoing IO actions, parallelism, composability, event handling, async-like/futures like/map-reduce-like processing:


```active haskell

{-# START_FILE Main.hs #-}

{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE DeriveDataTypeable #-}

module Main where

import           Data.Typeable
import           Base
import           Backtrack
import           Indeterminism
import           Control.Applicative
import           Control.Concurrent
import           Control.Exception
import           Control.Monad.State
import           Data.Monoid
import           System.IO.Unsafe
import           System.Directory
import           System.FilePath
import           Network.HTTP
import           qualified Data.Map as M
import           Network
import           System.IO
import           Data.IORef
import Data.List hiding (find,map, group)

-- show

solveConstraint=  do
      x <- choose  [1,2,3]
      y <- choose  [4,5,6]

      guard $ x * y == 8

      return (x,y)

pythags = do
  x <- choose [1..50]
  y <- choose ([1..x] :: [Int])
  z <- choose [1..round $ sqrt(fromIntegral $ 2*x*x)]

  guard (x*x+y*y==z*z)

  return (x, y,z)

example1= do
    option "ex1" "example 1"
    r <- threads 4 solveConstraint
    liftIO $ print r

example2= do
    option "pyt" "pythagoras"
    r<- threads 1 pythags
    liftIO $ print r

groupSample= threads 4 $ do
    option "coll" "group sample: return results in a list"
    r <- group 9 $ do
      x <- choose  [1,2,3]
      y <- choose  [4,5,6]
      return (x,y)

    liftIO $ print r

threadSample= freeThreads $ do
     option "th" "threads sample"
     liftIO $ print "number of threads? (< 10)"

     n <- input  ( < 10)

     threads n $ do
        x <- choose  [1,2,3]
        y <- choose  [4,5,6]

        -- added some delay to show tread interleaving better in Linux
        th <- liftIO $ threadDelay 100000 >> myThreadId

        liftIO $ print (x,y,th)

nonDeterminsm= do
      option "nondet" "Non determinism examples"
      example1 <|> example2
               <|> groupSample
               <|> threadSample
               <|> fileSearch



find' :: String -> FilePath -> TransientIO FilePath
find' s d = do
  fs <- liftIO $ getDirectoryContents d
       `catch` \(e:: SomeException) -> return []       -- 1
  let fs' = sort $ filter (`notElem` [".",".."]) fs    -- 2
  if any (== s) fs'                                    -- 3
     then return $ d</> s
     else do
       f <- choose fs'                                 -- 4
       let d' = d </> f                                -- 6
       isdir <- liftIO $ doesDirectoryExist d'         -- 7
       if isdir then find' s d'                        -- 8
                else stop


------------------

fileSearch=   do
    option "file" "example of file search"
    r<- threads 3 $ collect 1 $ find' "Main.hs"  ".."
    liftIO $ putStrLn $ "Found= "++ show  r



main= keep $ do
      oneThread $ option "main" "to return to the main menu"   <|> return ""
      liftIO $ putStrLn "MAIN MENU"

      nonDeterminsm <|> trans <|>
             colors <|> app   <|>
            futures <|> server

-- / show

trans= do
       option "trans" "transaction examples with backtracking for undoing actions"
       transaction <|> transaction2

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
            liftIO  (putStrLn "update other database necesary for the reservation")
                 `onUndo` liftIO (putStrLn "database update undone")



colors :: TransientIO ()
colors= do
       option "colors" "choose between three colors"
       r <-  color 1  "red"  <|> color 2 "green" <|> color 3 "blue"
       liftIO $ print r
       where
       color :: Int -> String -> TransientIO String
       color n str= do
         option (show n) str
         liftIO . print $ str ++ " color"
         return  str

app :: TransientIO ()
app= do
       option "app" "applicative expression that return a counter in 2-tuples every second"
       liftIO $ putStrLn "to stop the sequence, write main(enter)"
       counter <- liftIO $ newMVar 0

       r <-  (,) <$>  number  counter 1 <*> number counter 1


       liftIO $ putStrLn $ "result=" ++ show r
       where
       number counter n= waitEvents $ do
          threadDelay $ n * 1000000
          n <- takeMVar counter
          putMVar counter (n+1)
          return n

futures= do
       option "async" "for parallelization of IO actions with applicative and monioidal combinators"
       sum1 <|> sum2

sum1 :: TransientIO ()
sum1= do
       option "sum1" "access to two web pages concurrently and sum the number of words using Applicative"
       liftIO $ print " downloading data..."
       (r,r') <- (,) <$> async  (worker "http://www.haskell.org/")
                     <*> async  (worker "http://www.google.com/")

       liftIO $ putStrLn $ "result="  ++ show  (r + r')

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
      liftIO $ print " downloading data..."
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



{-# START_FILE Backtrack.hs #-}
-- show
-- /show
{-# LANGUAGE DeriveDataTypeable #-}
{-# LANGUAGE ExistentialQuantification #-}

{-# LANGUAGE DeriveDataTypeable #-}
{-# LANGUAGE ExistentialQuantification #-}
module Backtrack (registerUndo, onUndo, undo, retry, undoCut) where

import Base
import Data.Typeable
import Control.Applicative
import Control.Monad.State
import Unsafe.Coerce
import System.Mem.StableName

data Backtrack= forall a b.Backtrack{backtracking :: Bool
                                    ,backStack :: [EventF]}
                                    deriving Typeable

-- | assures that backtracking will not go further
undoCut :: TransientIO ()
undoCut= Transient $ do
     delSessionData $ Backtrack False []
     return $ Just ()

-- | the secod parameter will be executed when backtracking 
{-# NOINLINE onUndo #-}
onUndo :: TransientIO a -> TransientIO a -> TransientIO a
onUndo ac bac= do
   r<-registerUndo $ Transient $ do 
     Backtrack back _ <- getSessionData `onNothing` return (Backtrack False [])
     runTrans $ if back then bac  else ac 
   return r
   
-- | register an actions that will be executed when backtracking
{-# NOINLINE registerUndo #-}
registerUndo :: TransientIO a -> TransientIO a
registerUndo f  = Transient $ do
   cont@(EventF x _ _ _ _ _ _ _ _)  <- get   !> "backregister"
   md  <- getSessionData 
   ss <- case md of
        Just (bss@(Backtrack b (bs@((EventF x'  _ _ _ _ _ _ _ _):_)))) -> do
            addrx  <- addr x
            addrx' <- addr x'         -- to avoid duplicate backtracking points
            return $ if addrx == addrx' then bss else  Backtrack b $ cont:bs
        Nothing ->  return $ Backtrack False [cont]
   setSessionData ss
   runTrans f
   where
   addr x = liftIO $ return . hashStableName =<< (makeStableName $! x)

-- | restart the flow forward from this point on
retry :: TransientIO ()
retry= do
    Backtrack _ stack <- getSessionData `onNothing` return (Backtrack False [])
    setSData $ Backtrack False stack

-- | execute backtracking. It execute the registered actions in reverse order. 
--
-- If the backtracking flag is changed the flow proceed  forward from that point on. 
--
--If the backtrack stack is finished or undoCut executed, `undo` will stop.
undo :: TransientIO a
undo= Transient $ do
  bs <- getSessionData  `onNothing` return nullBack            !>"GOBACK"
  goBackt  bs

  where
  nullBack= Backtrack False []
  goBackt (Backtrack _ [])= return Nothing                     !> "END"
  goBackt (Backtrack b (stack@(first@(EventF x fs _ _  _ _ _ _ _): bs)))= do
--        put first{replay=True} 
        setSData $ Backtrack True stack
        mr <-  runClosure first                                !> "RUNCLOSURE"
        Backtrack back _ <- getSessionData `onNothing` return nullBack 
                                                               !>"END RUNCLOSURE"
        case back of
           True ->  goBackt $ Backtrack True bs                !> "BACK AGAIN"
           False -> case mr of
                   Nothing -> return empty                     !> "FORWARD END"
                   Just x ->  runContinuation first x          !> "FORWARD EXEC"



{-# START_FILE Indeterminism.hs #-}
-- show
-- /show
-----------------------------------------------------------------------------
--
-- Module      :  Transient.Indeterminism
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
{-# LANGUAGE BangPatterns, DeriveDataTypeable #-}
module Indeterminism (
choose, choose', collect, group --, found
) where

import Base
import Control.Monad.IO.Class
import Data.IORef
import Control.Applicative
import Data.Monoid
import Control.Concurrent
import Data.Typeable
import Control.Monad.State
import Control.Concurrent.STM as STM
import GHC.Conc


-- | slurp a list of values and process them in parallel . To limit the number of processing
-- threads, use `threads`
choose  ::  [a] -> TransientIO a
choose []= empty
choose   xs = do
    evs <- liftIO  $ newIORef xs
    parallel   $ do
           es <- atomicModifyIORef' evs $ \es -> let !tes= tail es in (tes,es)
           case es of
            [x]  -> return $ Left $ head es
            x:_  -> return $ Right x

-- | group the output of a possible mmultithreaded process in groups of n elements.
group :: Int -> TransientIO a -> TransientIO [a]
group num proc =  do
    v <- liftIO $ newIORef (0,[])
    x <- proc
    n <- liftIO $ atomicModifyIORef' v $ \(n,xs) -> let !n'=n +1 in ((n', x:xs),n')
    if n < num
      then stop
      else do
       liftIO $ atomicModifyIORef v $ \(n,xs) ->  ((0,[]),xs)

choose' :: [a] -> TransientIO a
choose'  xs = foldl (<|>) empty $ map (parallel . return . Left) xs


--newtype Collect a= Collect (MVar (Int, [a])) deriving Typeable

-- collect the results of a search done in parallel, usually initiated by
-- `choose` . The results are added to the collection with `found`
--
--

(**>) x y=   Transient $do
       runTrans x
       runTrans y

(<**) x y= Transient $ do
       r <- runTrans x
       runTrans y
       return r

-- execute a process and get the first n solutions.
-- if the process end without finding the number of solutions requested, it return the fond ones
-- if he find the number of solutions requested, it kill the threads of the process and return
-- It works monitoring the solutions found and the number of active threads.
collect :: Typeable a => Int -> TransientIO a -> TransientIO [a]
collect n search=  do
  rv <- liftIO $ atomically $ newTVar (0,[]) !> "NEWMVAR"
  endflag <- liftIO $ newTVarIO False
  st <- get
  let any1 = do
        r <- search   !> "ANY"
        liftIO $ atomically $ do
            (n1,rs) <- readTVar rv
            writeTVar  rv (n1+1,r:rs) !> "MODIFY"
        stop

      detect= do
        stnow <- get
        freeThreads $ async $ do
             threadDelay 1000 -- to allow the spawning of worker threads
             xs <- atomically $ do
                (n',xs) <- (readTVar rv ) !> "read"
                ns <- readTVar $ children st
--                unsafeIOToSTM $ putStrLn $ "LEN="++show (length ns)++ " "++ show n'++ " "++ show n

                if (n' >= n) || (length ns == 1)
                  then return xs
                  else retry

             th <- myThreadId !> "KILL"
             free th stnow
             killChildren st
             addThread st stnow
             return  xs

  any1  **> detect




{-# START_FILE Base.hs #-}
-- show
-- /show
{-# LANGUAGE ScopedTypeVariables #-}
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
{-# LANGUAGE ExistentialQuantification #-}
{-# LANGUAGE FlexibleContexts          #-}
{-# LANGUAGE FlexibleInstances         #-}
{-# LANGUAGE MultiParamTypeClasses     #-}
{-# LANGUAGE DeriveDataTypeable        #-}
-- show
module Base  where
-- /show

import           Control.Applicative
import           Control.Monad.State
import           Data.Dynamic
import qualified Data.Map               as M
import           Data.Monoid
import           Debug.Trace
import           System.IO.Unsafe
import           Unsafe.Coerce
import           Control.Exception
import           Control.Concurrent
import           Control.Concurrent.STM
import           System.Mem.StableName
import           Data.Maybe
import           GHC.Conc
import           Data.List
import           Data.IORef

(!>) =const . id -- flip trace
infixr 0 !>

data Transient m x= Transient  {runTrans :: m (Maybe x)}
type SData= ()

type EventId= Int



data EventF  = forall a b . EventF{xcomp       :: TransientIO a
                                  ,fcomp       :: [b -> TransientIO b]
                                  ,mfData      :: M.Map TypeRep SData
                                  ,mfSequence  :: Int
                                  ,threadId    :: ThreadId
                                  ,freeTh      :: Bool
                                  ,parent      :: Maybe EventF
                                  ,children    :: TVar[EventF]
                                  ,maxThread     :: Maybe (P Int)
                                  }
                                  deriving Typeable

type P= IORef
newp= newIORef


--(=:) :: P a  -> (a -> a) -> IO()
(=:) n f= liftIO $ atomicModifyIORef' n $  \v ->  ((f v),())

addr x= show $ unsafePerformIO $ do
       st <- makeStableName $! x
       return $ hashStableName st

waitQSemB sem= atomicModifyIORef' sem $ \n -> if n > 0 then(n-1,True) else (n,False)
signalQSemB sem= atomicModifyIORef' sem  $ \n ->  (n + 1,())

-- | set the maximun number of threads for a procedure. It is useful for the
threads :: Int -> TransientIO a -> TransientIO a
threads n proc= Transient $do
   msem <- gets maxThread
   sem <- liftIO $ newIORef n
   modify $ \s -> s{maxThread= Just sem}
   r <- runTrans proc
   modify $ \s -> s{maxThread = msem} -- restore it
   return r


instance MonadState EventF  TransientIO where
  get=  Transient $ get >>= return . Just
  put x= Transient $ put x >> return (Just ())

type StateIO= StateT EventF  IO

type TransientIO= Transient StateIO

--runTrans ::  TransientIO x -> StateT EventF  IO (Maybe x)
--runTrans (Transient mx) = mx


runTransient :: TransientIO x -> IO (Maybe x, EventF)
runTransient t= do

  th <- myThreadId
  let eventf0=  EventF  empty [] M.empty 0
          th False  Nothing  (unsafePerformIO $ newTVarIO []) Nothing
   

  runStateT (runTrans t) eventf0{threadId=th} !> "MAIN="++show th


setEventCont ::   TransientIO a -> (a -> TransientIO b) -> StateIO ()
setEventCont x f  = do
   st@(EventF   _ fs d n  r applic  ch rc bs)  <- get
--   if applic
--    then return () else
   put $ EventF x ( unsafeCoerce f : fs) d n  r applic  ch rc bs



resetEventCont :: Maybe a -> StateIO ()
resetEventCont mx =do
   st@(EventF _ fs d n  r nr  ch rc bs)  <- get
--   if nr
--    then return ()
--    else do
   let f= \mx ->  case mx of
                       Nothing -> empty
                       Just x  -> (unsafeCoerce $ head fs)  x
   put $ EventF  (f mx) ( tailsafe fs)d n  r nr  ch rc bs
   where
   tailsafe []=[]
   tailsafe (x:xs)= xs



getCont ::(MonadState EventF  m) => m EventF
getCont = get


runCont :: EventF -> StateIO ()
runCont (EventF  x fs _ _  _ _  _ _ _)= runTrans ((unsafeCoerce x') >>= compose ( fs)) >> return ()
    where
    x'=  do
--           modify $ \s -> s{replay=True}
           r<- x
--           modify $ \s -> s{replay=False}
           return r

{-
runCont cont= do
     mr <- runClosure cont
     case mr of
         Nothing -> return Nothing
         Just r -> runContinuation cont r
-}

compose []= const empty
compose (f: fs)= \x -> f x >>= compose fs



runClosure :: EventF -> StateIO (Maybe a)
runClosure (EventF x _ _ _ _ _ _ _ _) =  unsafeCoerce $ runTrans x

runContinuation ::  EventF -> a -> StateIO (Maybe b)
runContinuation (EventF _ fs _ _ _ _  _ _ _) x= runTrans $  (unsafeCoerce $ compose $  fs) x

instance   Functor TransientIO where
  fmap f mx=   -- Transient $ fmap (fmap f) $ runTrans mx
    do
     x <- mx
     return $ f x

instance Applicative TransientIO where
  pure a  = Transient . return $ Just a


  f <*> g = Transient $ do

         rf <- liftIO $ newIORef Nothing
         rg <- liftIO $ newIORef Nothing   -- !> "NEWIOREF"

         cont@(EventF _ fs a b c d peers children g1) <- get   -- !> "APLICATIVE DOIT"

         let
             appg x = Transient $  do
                   liftIO $ writeIORef rg $ Just x :: StateIO ()
                   k <- liftIO $ readIORef rf

                   return $ k <*> Just x  -- !> "RETURNED: " ++ show(isJust k)++ show(isJust x)


             appf k = Transient $  do
                   liftIO $ writeIORef rf  $ Just k :: StateIO ()
                   x<- liftIO $ readIORef rg

                   return $ Just k <*> x  --  !> "RETURNED: " ++ show(isJust k)++ show(isJust x)



         put $ EventF f (unsafeCoerce appf:  fs)
                                          a b c d peers children g1
         k <- runTrans f
         liftIO $ writeIORef rf  k -- :: StateIO ()

         put $ EventF g (unsafeCoerce appg :  fs)
                                          a b c d peers  children g1
         x <- runTrans g
         liftIO $ writeIORef rg  x

         return $ k <*> x


instance  Alternative TransientIO where
  empty= Transient $ return  Nothing
  Transient f <|> Transient g= Transient $ do
         k <-   f
         x <-   g
         return $ k <|> x

-- | delete all the previous childs generated by the expressions and continue execution
-- of the current thread.
oneThread :: TransientIO a -> TransientIO a
oneThread comp=  do
   chs <- liftIO $ newTVarIO []
   r <- comp
   modify $ \ s -> s{children= chs}
   killchilds
   return r

-- | internal. use `oneThread`
killchilds :: TransientIO()
killchilds= Transient $  do
   cont <- get
   liftIO $  killChildren cont
   return $ Just ()

-- | The threads generated in the process passed as parameter will not be killed.
freeThreads :: TransientIO a -> TransientIO a
freeThreads proc= Transient $ do
     st <- get
     put st{freeTh= True}
     r <- runTrans proc
     modify $ \st -> st{freeTh= freeTh st}
     return r

-- | The threads will be killed when the parent thread dies
hookedThreads proc= Transient $ do
     st <- get
     put st{freeTh= False}
     r <- runTrans proc
     modify $ \st -> st{freeTh= freeTh st}
     return r

instance MonadPlus TransientIO where
    mzero= stop
    mplus (Transient x) (Transient y)=  Transient $ do
         mx <- x
         case mx of
             Nothing -> y
             justx -> return justx

-- | a sinonym of empty that can be used in a monadic expression. it stop the
-- computation
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
        resetEventCont mk
        case mk of
           Just k  -> do
               runTrans $ f k

           Nothing -> return Nothing



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

-- | getSessionData specialized for the View monad. if Nothing, the
-- monadic computation does not continue. getSData is a widget that does
-- not validate when there is no data of that type in the session.
getSData :: MonadState EventF m => Typeable a =>Transient m  a
getSData= Transient getSessionData


-- | setSessionData ::  (StateType m ~ MFlowState, Typeable a) => a -> m ()
setSessionData  x=
  modify $ \st -> st{mfData= M.insert  (typeOf x ) (unsafeCoerce x) (mfData st)}

-- | a shorter name for setSessionData
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

          let n= mfSequence st
          put $ st{mfSequence= n+1}
          return n

refSequence :: IORef Int
refSequence= unsafePerformIO $ newp 0



data Loop= Once | Loop | Multithread deriving Eq

waitEvents ::   IO b -> TransientIO b
waitEvents io= do
   r <- parallel (Right <$> io)
   killchilds
   return r


async  ::  IO b -> TransientIO b
async io= do
   r <- parallel  (Left <$>io)
   killchilds
   return r

spawn ::  IO b -> TransientIO b
spawn io= freeThreads $ do
   r <- parallel (Right <$>io)
   return r

data EventValue= EventValue SData deriving Typeable

parallel  ::    IO (Either b b) -> TransientIO b
parallel  ioaction= Transient$   do
        cont <- getCont                    -- !> "PARALLEL"
        mv <- getSessionData
        case mv  of
         Just (EventValue v)  -> do
            delSessionData $ EventValue () -- !> "ISJUST "
            return  $ Just $ unsafeCoerce v
         Nothing -> do
            liftIO  $ loop cont    ioaction
            return Nothing




loop (cont'@(EventF x fs a b c d peers childs g))  rec  =  do
  chs <- liftIO $ newTVarIO []
  let cont = EventF x fs a b c d (Just cont') chs g
      iocont dat=
          runStateT ( do
             setSessionData . EventValue $ unsafeCoerce dat
             runCont cont
             ) cont
             >> return ()
      loop'= do
        mdat <- rec
        case mdat of
         Left dat -> iocont dat

         Right dat -> do
              forkMaybe  $ iocont dat
              loop'

      forkMaybe  proc = do
         dofork <- case maxThread cont of
                  Nothing -> return True
                  Just sem -> do
                    dofork <- waitQSemB sem
                    if dofork then  return True else return False

         if dofork
            then  do
                 th <- forkFinally proc $ \me -> do
                         case me of -- !> "THREAD ENDED" of
                          Left  e -> do
                           when (fromException e /= Just ThreadKilled)$ liftIO $ print e
                           killChildren  cont  !> "KILL RECEIVED" ++ (show $ unsafePerformIO myThreadId)

                          Right _ -> do
                             --  if parent is alive
                             --  then remove himself from the list (with free)
                             --  and pass his active children to his parent
                             return ()
                             th <- myThreadId
                             mparent <- free th cont
                             case mparent of
                              Nothing  ->  return()
                              Just parent -> atomically $ do
                                     chs' <- readTVar $ children cont
                                     chs  <- (readTVar $ children parent) 
                                     writeTVar (children parent)$ chs ++ chs'
                                     return ()

                         case maxThread cont of
                           Just sem -> signalQSemB sem
                           Nothing -> return ()


                 addThread cont' cont{threadId=th}  -- !>  "thread created: "++ show th

            else proc  -- !> "NO THREAD"

  forkMaybe  loop'


free th env= do
  if isNothing $ parent env
   then  return Nothing  !>  show th ++ " orphan"
   else do
    let msibling= fmap children $ parent env
    
    case msibling of
     Nothing -> return Nothing
     Just sibling  -> do
       found <- atomically $ do
                sbs <- readTVar sibling  
                let (sbs', found) = drop [] th  sbs !> "search "++show th ++ " in " ++ show (map threadId sbs)
                when found $ writeTVar sibling sbs'
                return found
       if (not found && isJust (parent env)) 
         then free th $ fromJust $ parent env !> "toparent"
         else return $ Just env

   where
   drop processed th []= (processed,False)
   drop processed th (ev:evts)| th == threadId ev= (processed ++ evts, True)
                    | otherwise= drop (ev:processed) th evts


addThread parent child = when(not $ freeTh parent) $ do
   let headpths= children parent
   atomically $ do
       ths <- readTVar headpths
       writeTVar headpths $  child:ths

killChildren  cont = do
     forkIO $ do
        let childs= children cont --  !> "killChildren list= "++ addr (children cont)
        ths <- atomically $ do
           ths <- readTVar childs
           writeTVar childs []
           return ths
        mapM_ (killThread . threadId) ths  !> "KILLEVENT " ++ show (map threadId ths)
     return ()


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



getLineRef= unsafePerformIO $ newTVarIO Nothing


option1 x  message= do

 waitEvents  $ do
     liftIO $ putStrLn $ message++"("++show x++")"
     th <-  myThreadId
     atomically $ do
       mr <- readTVar getLineRef

       case mr of
         Nothing -> retry
         Just r ->
            case reads1 r of  -- !> ("received " ++  show r ++  show th) of
            (s,_):_ -> if  s == x  -- !> ("waiting" ++ show x)
                     then do
                       writeTVar  getLineRef Nothing -- !>"match"
                       return s

                     else retry
            _ -> retry
     where
     reads1 s=x where
      x= if typeOf(typeOfr x) == typeOf "" then unsafeCoerce[(s,"")] else readsPrec 0 s
      typeOfr :: [(a,String)] ->  a
      typeOfr  = undefined


roption= unsafePerformIO $ newMVar []

-- | install a event receiver that wait for a string and trigger the continuation when this string arrives.
option :: (Typeable b, Show b, Read b, Eq b) =>
     b -> [Char] -> TransientIO b
option ret message= do
    let sret= show ret

    liftIO $ putStrLn $ "Enter "++sret++"\tto: " ++ message
    liftIO $ modifyMVar_ roption $ \msgs-> return $ sret:msgs
    waitEvents  $ getLine' (==ret)
    liftIO $ putStrLn $ show ret ++ " chosen"
    return ret


-- | validates an input entered in the keyboard in non blocking mode. non blocking means that
-- the user can enter also anything else to activate other option
-- unlike `option`, input only wait for one valid response
input :: (Typeable a, Read a) => (a -> Bool) -> TransientIO a
input cond= Transient . liftIO . atomically $ do
       mr <- readTVar getLineRef
       case mr of
         Nothing -> retry
         Just r ->
            case reads1 r  of
            (s,_):_ -> if cond s  --  !> show (cond s)
                     then do
                       writeTVar  getLineRef Nothing -- !>"match"
                       return $ Just s

                     else return Nothing
            _ -> return Nothing


getLine' cond=    do
     atomically $ do
       mr <- readTVar getLineRef
       case mr of
         Nothing -> retry
         Just r ->
            case reads1 r of --  !> ("received " ++  show r ++ show (unsafePerformIO myThreadId)) of
            (s,_):_ -> if cond s -- !> show (cond s)
                     then do
                       writeTVar  getLineRef Nothing -- !>"match"
                       return s

                     else retry
            _ -> retry

reads1 s=x where
      x= if typeOf(typeOfr x) == typeOf "" then unsafeCoerce[(s,"")] else readsPrec 0 s
      typeOfr :: [(a,String)] ->  a
      typeOfr  = undefined

inputLoop=  do
    putStrLn "Press end to exit"
    inputLoop'  -- !> "started inputLoop"
    where
        inputLoop'= do
           r<- getLine
           if r=="end" then putMVar rexit () else do
              atomically . writeTVar  getLineRef $ Just r
              inputLoop'


rexit= unsafePerformIO newEmptyMVar

stay=  takeMVar rexit

keep mx = do
   forkIO $ inputLoop
   forkIO $ runTransient mx  >> return ()
   stay

exit :: TransientIO a
exit= do
  liftIO $ putStrLn "Tempus fugit: exit"
  liftIO $ putMVar rexit   True
  return undefined

onNothing iox iox'= do
       mx <- iox
       case mx of
           Just x -> return x
           Nothing -> iox'



```

`collect` can also be used in context of map-reduce.

To see how the transient EDSL compose programs and how the internals of the EDSL work, see the [first article](https://www.fpcomplete.com/user/agocorona/EDSL-for-hard-working-IT-programmers)

# Constraint Logic programming
In the paper ["transforming functional logic programs into monadic functional programs"](http://www.researchgate.net/profile/Sebastian_Fischer9/publication/221557789_Transforming_Functional_Logic_Programs_into_Monadic_Functional_Programs/links/02e7e537488887d824000000.pdf?origin=publication_detail) the authors propose a generic solution that translate the logic expression to any monad with the MonadPlus instance if the logic variables (free variables whose value is unknown) are substituted by generators. Each value generated in each iteration must be shared in the expression being evaluated. 

So once we define a generator class, Transient can perform logic programming for all the instances of this class. It is better to show it with an example:

To create a generator it is enough to use `choose` defined above, applied to a list of possible values of the data type. 

Using `choose` we can create the class `Free`:

```haskell
class  Free m a where
    free :: m [a]


instance Free TransientIO Bool where
    free= choose[True,False]

instance Free TransientIO Int where
    free= choose[0..]  -- exclude negatives
...
```

So this  Transient routine

```haskell

pairs :: TransientIO Int
pairs= do
   x <- free
   y <- free
   guard (x/y == 4)
   return (x,y)
```

or expressed with monad comprehension:

```haskell
pairs=[(x,y)| x/y == 4, x <- free, y <- free]
```
Is  a constraint logic program quite similar in syntax to the syntax in a functional logic program such is curry with constraints:

```haskell
pairs= (x,y) where x /y == 4, x free, y free
```

But also all the rest of the above programs are  constraint logic programs too, since `choose` is a way to express constraints on the possible values of the logic variables. 

I guess that there is a way to program  classical examples of logic programming with logic clauses besides constraints. [This example designe for the list monad](https://wiki.haskell.org/Logic_programming_example) I suppose that is straighforward to translate to Transient. 
   
# Conclusions and future work

I used Transient here for some typical toy problems of non-determinism, but also I have show how **for some realistic IT problems the non-determinism built-in in the Transient monad can generate elegant multithreaded solutions with the same list monad syntax, while doing the same without this monad requires complex code transformations**.

The advantage of using Transient for problems with multiple solutions is that **the evaluation model of Transient is multithreaded so it can make use of all the machine cores for the exploration of the possible answers.**

And, perhaps more important, **the Transient monad is unique for all the effects: It is not necessary to switch from a monad to other with complicated monadic stacks to use multithreading, events, non-determinism, transactions or distributed computing (soon) among others. All the effects can be mixed freely. The code is as simple and clean as it could be**.

`parallel`, the basic primitive of Transient is inherently non-deterministic, since it implicitly uses multithreading and execute the continuation for each event in a new thread. But non-determinism is typically represented by the List monad. Here I demonstrated how Transient can process in parallel and filter many results using the semantics of the List monad with simple but effective and high level threading control.

I have used the parallel non-determinism of Transient to easily paralelize a search program with very little changes and at a higher level programming, more close to the programmer intuitions. In fact it does not need more but less code!. This shows how non-determinism can be in Transient to solve practical problems while maintaining a high-level and soundness of the solution and thus increase the maintainability of the software. 

Since all of this machinery can be mixed freely with other effects, there is nothing that I may know with the generality and seamlessness of Transient for parallel processing.

`guard` and `collect` can also be used in map-reduce scenarios.

For greater flexibility, thread control has been decoupled from `parallel`. Now paralell does not perform thread control. there are explicit primitives for it. 

Continuations are stored in the state with type erasure. This makes the edition of them difficult since the type system does not help. Using Dynamic would restrict the intermediate result to the Typeable ones and it would slow down execution. Moreover that would only produce errors at runtime, not at compile time. Perhaps some trick can aleviate these problems. Definitively something better than the hacky edition of state is necessary.

Transient will include more effects: thread state persistence in files and source emit are necessary for the integration of [MFlow](http://mflowdemo.herokuapp.com/) and [hplayground](http://tryplayg.herokuapp.com/) as a single "isomorphic" web framework.

Thread state persistence is the effect now provided by the Workflow package. Source emit is the solution for the serialization of code and the execution in another node, including a web browser. It is not clear that it can be feasible.

A cloud effect for the distribution of computing resources, where the transient monad does the distribution automatically depending on the load of each node and his capabilities, including browsers, may be possible in the medium term.

AMDG
# Moving Haskell processes between nodes (Transient effects IV)

This article has been rewritten the 19/08/2015

## Motivation (yours not mine)

The distributed framework that I´m developing is a completely different way to consider distributed computing in the same sense that Transient is a different way to program in Haskell or in any other language. It is so straightforward that is difficult to understand after decades of inversion of control, programming state machines, objects frameworks, routes, configurations and callbacks. If you dare to think different you will benefit from this higher level way of programming that I'm convinced that is the future.

It is the future because it is functionality oriented, not object oriented. A software functionality is a business process, that naturally compose with other business processes. A process is like physiology, while OOP is like anatomy. Functional programming is about composability, whether efectful or not. Transient is deeply functional and can express business functionalities in compact category theoretical expressions that compose. Have you seen two distributed programs that can be composed to create a wider one?

The main obstacle for such composability is events and blocking. events are asynchronous and need callbacks. Alternatively the process may avoid callback spagettization if the thread blocks and wait for events. That is what the synchronous IO call does. 

And this is what Erlang does. but that impairs composability, since to handle other events it is necessary to spawn another thread. Both threads would need to communicate and so on. That is why an Erlang program is made of many small independent distributed elements. But then synchronizations and extra communications and monitorizations are necessary. That accidental complexity complicate the development and the code is again difficult to understand.

A third solution is to use continuations. But continuations are either hard to master or have a akward syntax or are hard to impossible to serialize in order to save state or to transport them. In particular, haskell has no support for the serialization of continuations. That is the reason why continuation frameworks have been not accepted neither in haskell neither in other languages, except perhaps in the lisp family where these problems are less critical. A remaining problem not solved by continuations is the composition of parallel programs.

Transient *does* compose arbitrary pieces of software in concurrent parallel and sequential arangements not by communicating them trough channels neither by means of special constructions or configurations, but by means of standard Haskell combinators. Therefore these compositions are checked at compilation time, even if the elements will run in different machines.

# A sad story

Two years ago I created [MFlow](http://mflowdemo.herokuapp.com), the only web framework that uses [Haskell category theoretical abstractions](https://themonadreader.files.wordpress.com/2014/04/mflow.pdf) for the creation of the high level elements of web applications such are routes and navigations among other things. For the first time navigation elements are composable. For example:

       login >> navigateCatalog >>= shoppingCartManagement >>= payment

This is an application, assembled with four navigations. Back button  and arbitrary URLs entered works fine. It has not the problems associated with continuations because it does not use continuations as such. 

Besides having more features than the widely used frameworks and more innovative ones, apart from the above mentioned, MFlow has not been sucessful. What was the cause? My careless and clumsy English? My lack of attention to details? Lack of social skills? For sure. 

But there are other reasons why this framework or other early continuation frameworks like WASH has not gained support; For example, the haskellers that program web apps comes from other languages and find that strange.

But there is also a fear of novelty, a reluctance to think different and a fear to do things in a different way than what is academically or industrially established.

There is also a tendency to wait and see. "mmm is interesting... let's look what happens... this guy does interesting things but nobody knows him... I will not shoot myself in the foot. I will not risk my investments; The more traditional path is more sure".

And finally, there's tradition of inversion of control that is a second nature in the programming community since the invention of OOP and the mouse. The invention of the mouse, and then a second source of input, ended the golden age of imperfect but effective composability with the console apps and the unix pipes. Simply, the unix pipes model, that composed single streams was not valid because a second stream of events was added. The solution was the state machine, divided in small pieces each one called classes and objects. That was OOP. But objects are not composable.

Later on, I created [hplayground](http://tryplayg.herokuapp.com), a client side framework with the same characteristics of composability. For this purpose I created the Transient monad. Transient is [a new way of using continuations and a new way of looking at the Monad instance, as a manipulation of continuations](https://www.fpcomplete.com/user/agocorona/the-hardworking-programmer-ii-practical-backtracking-to-undo-actions). Again, a different way to think about both concepts. The whole history is narrated in the series of articles in FPComplete.

The history repeated with hplayground sadly, since again, it is the only fully composable FRP, 100% category theoretical standard haskell for programming the shortest and cleanest code ever seen in a Web Browser.

But I don't care because **this is THE future**. I did not invented it. It was there and there is no better way. Perhaps someone will reinvent it in Java 14 or in  Scala 10 and composable de-inverted programming gain acceptance. I will see it from Heaven.

# Composable distributed computing

Having the possibility to express multithreaded programs that deal with IO in a composable way, I was persuaded that a composable form of distributed computing was possible using the non-blocking calls of transient. Composable in this context means that a distributed computing functionality can be invoked within a monadic, monoidal, applicative or alternative expression to create a wider application that are composable too.

In mathematical terms, composability means that there is a form of algebra for combining the elements of the domain problem. In this case, it is the abstract algebra that include monads, applicatives, alternatives and monoids. Composable programming is like teaching the computer to do maths without greek letters. In the same way that we combine numbers in complicated algebraic expressions -every  programming language know how to do that algebra- we can teach computers to create expressions that include concurrent, parallel, sequential and distributed computations. And we can combine these expressions to create even more complex expressions.

For example, a map-reduce operation that may involve many nodes can be inserted in a monadic expression to access data that later will feed a  browser application. All this programmed in a single expression of beautiful haskell code with standard combinators.

And it is possible as you may verify below. Not only because map-reduce is now doable that way, but because in the future I will integrate MFlow and hplayground, the web frameworks, with Transient.

# The plan

I plan to develop various levels of distributed computing, all of them composable:

- Level 1: basic calls for node-to-node communication: `beamTo`, `forkTo`, `callTo`, `beamInit`, `listen`

- Level 2: clusted oriented computing: process execution replicated in al connected nodes:  `clustered`, `asynchronize`, `connect`

- level 3: location independent communications: processes are connected without mention of the hosts and ports. Process location static and stablished at configuration time.

- level 4: location independent, processes movable from node to node depending on run-time conditions according with programmer defined strategies. (A cloud monad)
 
At this moment I have developed the two first levels

The repository is at:

[https://github.com/agocorona/transient](https://github.com/agocorona/transient)


##Level 1: basic calls for node-node commmunication

I defined three  primitives, the level 1 of the distributed framework, but at the level in which they are defined are quite powerful: 

```haskell
beamTo :: HostName -> PortID -> TransientIO ()
forkTo :: HostName -> PortID -> TransientIO ()
callTo :: HostName -> PortID -> TransientIO a -> TransientIO a

```

The second action execute a copy of the running process in the other node with the same execution state.

The first continue the execution of the monadic sequence in other node, but "destroy the original". That means that the running process continues in the other node.

The first two send the process and forget about it. The third call a procedure in the other node, and  receive the result.

There are other auxiliary primitives that I will detail. 

`beamTo` can be used for that kind of sequential process that may need to do different things in different nodes. and perhaps may return with the results. Web applications can start in the data tier, continue in the server and migrate to the browser for a long interaction, then can return to the server and so on.

`forkTo` may be used for example, to  configure a node interactively and then clone that configuration and execution state in many nodes. 

# Map-reduce

`callTo` may be used for fast roundtrip calls and also to start subtasks in other nodes. Combined with the monoid instance of Transient, it can be used to create  mapreduce calls which spawn worker task to other nodes: 

```haskell
rs <- foldl (<>) mempty $ map (\(h,p) -> callTo h p worker) nodes
```

The three primitives can be combined for complex scenarios.

NOTE: the reduce operation in this case is a fold in the receiving node. For largue volumes of data this is not realistic. The reduction should be also distributed. That is something that [I did here] (https://www.fpcomplete.com/user/agocorona/estimation-of-using-distributed-computing-streaming-transient-effects-vi-1#distributed-datasets)


# forwarding events across the network

`callTo` is not a simple remote call by various reasons. Not only it works in de-inverted mode, it means that it does not set any callback. Instead, the result return to the routine that called it. That permit much more understandable code.

But that is not done sacrificing composability since it is also non blocking when used with applicative, alternative and monoidal expressions, since the rest of the expression does not depend on the result of `callTo` to continue. The map-reduce snippet would not work in parallel if not for this property.

But there is more. Since `callTo` uses the Transient monad, that is inherently non deterministic and multithreaded. the local node uses `parallel` to receive successive responses, each one in a different thread. This avoid buffer overruns and contentions. An example, running below, is as follows:

```haskell
networkEvents rh rp= do
     r <- callTo rh rp $ do
                         option "fire"  "fire event"
                         return "event fired"
     putStrLnhp  rp $ r ++ " in remote node"

putStrLnhp p msg= liftIO $ putStr (show p) >> putStr " ->" >> putStrLn msg
```

`option`is executed in the remote node, when the user write "fire" the remote node return the message to the caller node and another message appears in the local node.

since `option` produces an event for each "fire" entered, callTo works here as an event pipeline between the two nodes.

As an example, a single `callTo` can be used to interact with a client side widget or it can retrieve chunks from a query to a large database.

## Level 2

The higher level cloud primitives that I am developping now are:

`connect` will connect a new node to a network of nodes in a Wide Area Network. It will connect to one of the nodes and all the rest will receive notifications about it.

```haskell
connect :: Node -> TransientIO ()
```

`clustered` will execute a computation in all the connected nodes, like for example, to update/query a distributed database. This will not return until all computations succeed. It gets the return values and `mconcat` them. 

```haskell
clustered :: Monoid a => TransientIO a -> TransientIO a
```

it is the map-reduce snippet of the second paragraph, but applied to all the connected nodes. Later on, a system of capabilities will filter the nodes, so that the nodes will specialize in some calls. For example, some nodes may store data and other can process the data and respond to user requests.

`asynchronize` Will do the same that clustered, but asynchronously. The primitive will schedule the remote execution, but it will return immediately.

```haskell
asynchronize :: TransientIO () -> TransientIO ()
```


`connect` can be expressed in terms of clustered:


```haskell
connect (host,port) = do
     nodes <- callTo host port $ clustered $ do
                           updateNodeList 
                           myNode <- getMyNode
                           return [myNode]
     setNodes nodes
```

Since one of the nodes in the list is the local one, reentrant invocation of `callTo` is necessary

That's all for the moment.

### distributed Chat application in four lines

Wow. Perhaps this may be too mind blowing. So much that may be anticlimactic, but this snippet is a distributed chat application. If  `nodes` contain all the connected chat programs:

```haskell
chat :: [(HostName, PortID)] -> Transient StateIO ()
chat nodes = do 
    name  <- step $ do liftIO $ putStrLn "Name?" ; input (const True)
    text  <- step $ waitEvents  $ putStr ">" >>hFlush stdout >> getLine' (const True)
    let line= name ++": "++ text 
    foldl (<>) mempty $ map (\(h,n) -> callTo h  n  . liftIO $ putStrLn line) nodes 
```

This is part of the executable examples below.

`input` is a non-blocking execute-once version of getLine. It has a parameter that is the validator.
`getLine'` is a non-blocking version of `getLine` .  
`waitEvents` is another Transient primitive that executes the continuation for each input. So it will execute the last line for each line entered with the keyboard. 

The last line uses the Monoid instance of Transient to execute `callTo` in parallel for all the nodes.

This would also do it:

```haskell
mapM_ (\(h,n) -> callTo h  n  . liftIO $ putStrLn line) nodes
```

But this last one does not do it in parallel, but sequentially.



# How it works

This may  be a case of the [egg of Columbus](https://en.wikipedia.org/wiki/Egg_of_Columbus) . The migration of the program is done through logging and replaying. The first who used replaying to transport thread execution trough the network - that I may know - is Jeff Epstein using Cloud Haskell and my [Workflow](http://hackage.haskell.org/Workflow) library. The workflow library log the intermediate results of a monadic sequence. Using the log it is possible to recover the execution state. After replaying the log, the program will be executing a closure identical to the one that was when it was interrupted. 

Migration of continuations or strong mobility of the kind of `beamTo` was done at an early time in Haskell by [Dubois et al](http://www.dcs.gla.ac.uk/~trinder/papers/strongm.pdf). It is a shame that this project was not continued. Probably one of the reasons of the abandonment is related with something that is mentioned in the paper in the point 3; Moving the execution state includes everything, upto registers, stack and memory. This happens also for every language that compiles to machine code. The binary serialization of state may be huge and this cause many problems. Moreover, the source and destination architecture, and the binary program must be identical. Static closures or to be more mundane, pointers to static functions, defined at compilation time can be transmitted to invoke foreign code provided that both source and destination are two identical binary executables. That is the mechanism used in Cloud Haskell.

logging and replaying is lighter than execution state serialization and deserialization. It is also architecture independent and the effect is the same. The replaying of the log reconstruct the execution state. That includes the values of all the local variables. If the log is transferred, the state can be recovered in another node, even if it has different architecture.

It is important to understand the freedom that this mechanism gives. For example `callTo` is not a RPC call, as is the case in cloud haskell. The remotely executed routine has access to all the local variables (that have been logged) of the caller, so I can do this:


```haskell
do 
    name <- step input
    callTo host  node  $ writeIORef refname name
    callTo host' node' $ writeIORef refname name
    ...
```


This example writes the value of `name` in the variables of each remote node and resume the execution of the calling action.

The above code is executed sequentially. To do it in parallel for all the nodes:


```haskell
do 
    name <- step input
    (,) <$> callTo host node (writeIORef refname name) <*> callTo host' node' (writeIORef refname name) 
```

or more generally, using the monoid instance of Transient:

```haskell
do 
    name <- step input
    fold (<>) mempty $ map (\(h,n) -> callTo h  n  . writeIORef $ refname name) nodes
```


`step` is the primitive that update the log and replay it.  It must appear explicitly in all the previous monadic statements before the moving primitives. In this toy implementation step is not enforced, so if a monadic statement has no step, it will be executed in the receiving node. That may be intended, in order to access remote variables for example.


## Logging

Logging as an effect in the Transient monad. It is implemented now in the `Transient.Logged` module. Currently it is only used for remote execution, but it will be used for other purposes. It has sophisticated optimizations: for example when a long action return a value, it is possible to erase his log and replace it with the value returned, so the log/replay mechanism must not  store and execute each and every intermediate result but only the top level ones. But if a failure happens deeper inside this action, then the detail log of this particular action will be saved.


# Implementation details

I did it using the Transient monad. It permits a much cleaner code, since it does manage state, events and threading implicitly, without breaking the specification in separate actions. 

The user program executes in two ways: directly at the main procedure and when a request arrive. This is in the first line of this Transient expression.
 
```haskell
beamInit port program=   do
    listen port  <|> return ()
    (program >> empty)  <|> close
    where
    close= do
       (h,sock) <- getSData
       liftIO $ hClose h  `catch` (\(e::SomeException) -> sClose sock)
```

`beamInit` initialize the user program in these two modes.

in the first line, `listen` generates a thread and continues the execution when a request arrives. the second operand, return() just executes the continuation in "normal" mode, not when a request arrive. This means that in this mode `program` will be executed without a log, so the statements of the programs will be executed and logged. In the other side, when `listen` receive a log, the `program` is replayed from the log.

The next sentence is the user program. At the end, if the thread was initiated by listen, the handle is closed.  If there is no such data, `getSData` will stop the execution.

To see the execution model of Transient, see the previous articles of transient in my FPcomplete space. A Transient process can be stopped in a thread and be continued in another thread when an event arrives.

`listen` is defined as:

```haskell
listen :: PortID -> TrasientIO ()
listen  port = do
       sock <- liftIO $ listenOn  port
       (h,_,_) <- spawn $ accept sock
       liftIO $ hSetBuffering h LineBuffering
       slog <- liftIO $ hGetLine  h
       setSData (h,sock)
       setSData $ Log True $ read slog
```

`spawn` is the Transient primitive that launch a thread for each `accept`ed message. the process continues within this new thread. The message has an execution log, that is stored in the session state. The (handler,socket) is also stored.

More info about the mechanism of logging and recovery below.

This is `beamTo`:

```haskell
beamTo :: HostName -> PortID -> TransientIO ()
beamTo node port= do
      Log rec log <- getSData <|> return (Log False [])
      if rec then return () else do
          h <- liftIO $ connectTo node port
          liftIO $ do
             hSetBuffering h LineBuffering
             hPutStrLn h (show $ reverse log)  >> hFlush h
             hClose h
          delSData h
          stop
```

The other two moving primitives are similar. 

`beamTo` get the log, if it is running in normal mode (that means not in recovery mode) it just send the log to the receiving node. Then it closes the handler and remove the handler from the session state. Then it finish.

# Supervision

all the network processes are initiated by `listen` responding to received messages. `listen` and all the running processes will be killed if the parent thread or any of their ancestors call a kill childs primitive or if they die. 

Typically as the name indicates, the Transient processes, including the distributed ones are short lived. Since there is no inversion of control, the processes and the communications are created and destroyed for the needs of the main monadic process while it advance in the execution. 

For example, the map-reduce snippet does not need some special receiving dispatcher processes waiting permanently in the remote node. The only watching process is the one of `listen` which can schedule many different calls downstream. Once finished the work, the process dies.

There is no need of monitoring for long living processes although it is necessary the detection of failed communications in order to reschedule them.


# The example program, how to run it

The program has two options: one uses `callTo` to read an IORef in the other program. The other uses `beamTo` to input a string locally and then moves to the other node where it print a message and update the IORef. The other two are the ones mentioned above.

In this example the other node is not only in the same machine, but in the same executable. In the example `listen` is invoked and later some distribute primitives are invoked against the `listen` port. A single machine is not the rigth environment for testing distributed computing but this is all I can do with here in the School of Haskell. I´m very grateful to you, FP guys. You are doing a great job and a big service to the Haskell community.

By executing the options in this order   call -> move -> call, you can see how the interaction among nodes takes place. This demo was intended for two different processes instead of one, but the FPcomplete environment does not permit it.

As always, the distributed examples have been mixed with the previous ones that demonstrate other effects of the transient monad. This show the wonderful composability of this way of programming.

simply press "distr" and choose one of the examples.


You can also intentionally provoke some errors to see how the absence of monad transformers simplify the task of solving bugs.

```active haskell

{-# START_FILE Main.hs #-}

{-# LANGUAGE ScopedTypeVariables #-}

{-# LANGUAGE DeriveDataTypeable  #-}

module Main where

import           Data.Typeable
import           Base
import           Backtrack
import           Indeterminism
import           Logged
import           Move
import           Vars
import           Control.Applicative
import           Control.Concurrent
import           Control.Exception
import           Control.Monad.State
import           Data.Monoid
import           System.IO.Unsafe
import           System.Directory
import           System.FilePath
import           Network.HTTP
import           Network
import           System.IO
import           Data.IORef

import Data.List hiding (find,map, group)

-- show
main= keep $ do
      oneThread $ option "main" "to kill previous spawned processes and return to the main menu"   <|> return ""
      liftIO $ putStrLn "MAIN MENU"

      nonDeterminsm <|> trans <|>
             colors <|> app   <|>
            futures <|> server <|> 
            distributed <|> pubSub

-- /show

solveConstraint=  do
      x <- choose  [1,2,3]
      y <- choose  [4,5,6]

      guard $ x * y == 8

      return (x,y)

pythags = freeThreads $ do
  x <- choose [1..50]
  y <- choose ([1..x] :: [Int])
  z <- choose [1..round $ sqrt(fromIntegral $ 2*x*x)]

  guard (x*x+y*y==z*z)
  th <- liftIO $  myThreadId
  return (x, y, z, th)

example1= do
    option "ex1" "example 1"
    r <- threads 4 solveConstraint
    liftIO $ print r

example2= do
    option "pyt" "pythagoras"
    r<- threads 2 pythags
    liftIO $ print r

collectSample= threads 4 $ do
    option "coll" "group sample: return results in a list"
    r <- collect 0 $ do
      x <- choose  [1,2,3]
      y <- choose  [4,5,6]
      th <- liftIO $ threadDelay 1000 >> myThreadId

      return (x,y,th)

    liftIO $ print r

threadSample= do
     option "th" "threads sample"
     liftIO $ print "number of threads? (< 10)"

     n <- input  ( < 10)

     threads n $ do
        x <- choose  [1,2,3]
        y <- choose  [4,5,6]
        th <- liftIO $ myThreadId
        liftIO $ print (x,y,th)

nonDeterminsm= do
      option "nondet" "Non determinism examples"
      example1 <|> example2
               <|> collectSample
               <|> threadSample
               <|> fileSearch



find' :: String -> FilePath -> TransientIO FilePath
find' s d = do
  fs <- liftIO $ getDirectoryContents d
       `catch` \(e:: SomeException) -> return []       -- 1
  let fs' = sort $ filter (`notElem` [".",".."]) fs    -- 2
  if any (== s) fs'                                    -- 3
     then do
       liftIO $ print $ d </> s
       return $ d</> s
     else do
       f <- choose fs'                                 -- 4
       let d' = d </> f                                -- 6
       isdir <- liftIO $ doesDirectoryExist d'         -- 7
       if isdir then find' s d'                        -- 8
                else stop


------------------

fileSearch=   do
    option "file" "example of file search"
    r<- threads 3 $ collect 10 $ find' "Main.hs"  "."
    liftIO $ putStrLn $ "SOLUTION= "++ show  r
--    exit







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
       option "async" "for parallelization of IO actions with applicative and monoidal combinators"
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

       (h,_,_) <- spawn $ accept sock `catch` (\(e::SomeException) -> sClose sock >> throw e)
       liftIO $ do
           hPutStr h msg
           putStrLn "new request"
           hFlush h
           hClose h
         `catch` (\(e::SomeException) -> sClose sock)

msg = "HTTP/1.0 200 OK\r\nContent-Length: 5\r\n\r\nPong!\r\n"


-- show

-- distributed computing

distributed= do
      option "distr" "examples of distributed computing"
      let port1 = PortNumber 2000


      addNodes [(host,port1)]
      listen port1 <|> return ()-- conn port1 port1 <|> conn port2 port1

      examples' host port1
      where
      host= "localhost"
      conn p p'=  connect host p host p'

examples' remoteHost remotePort= do
   logged $ option "maind"  "to see this menu" <|> return ""
   r <-logged    $ option "move" "move to another node"
               <|> option "call" "call a function in another node"
               <|> option "chat" "chat"
               <|> option "netev" "events propagating trough the network"
   case r of
       "call"  -> callExample remoteHost remotePort
       "move"  -> moveExample remoteHost remotePort
       "chat"  -> chat
       "netev" -> networkEvents remoteHost remotePort


callExample host port= do
   logged $ putStrLnhp  port "asking for the remote data"
   s <- callTo host port $ liftIO $ do
                       putStrLnhp  port "remote callTo request"
                       readIORef environ


   liftIO $ putStrLn $ "resp=" ++ show s


environ= unsafePerformIO $ newIORef "Not Changed"

moveExample host port= do
   logged $ putStrLnhp  port "enter a string. It will be inserted in the other node by a migrating program"
   name <- logged $ input (const True)
   beamTo host port
   putStrLnhp  port "moved!"
   putStrLnhp  port $ "inserting "++ name ++" as new data in this node"
   liftIO $ writeIORef environ name
   return()


chat ::  TransIO ()
chat  = do
    name  <- logged $ do liftIO $ putStrLn "Name?" ; input (const True)
    text <- logged $  waitEvents  $ putStr ">" >> hFlush stdout >> getLine' (const True)
    let line= name ++": "++ text
    clustered $   liftIO $ putStrLn line


networkEvents rh rp= do
     logged $  do
       putStrLnhp  rp "callTo is not  a simole remote call. it stablish a connection"
       putStrLnhp  rp "between transient processes in different nodes"
       putStrLnhp  rp "in this example, events are piped back from a remote node to the local node"

     r <- callTo rh rp $ do
                         option "fire"  "fire event"
                         return "event fired"
     putStrLnhp  rp $ r ++ " in remote node"

putStrLnhp p msg= liftIO $ putStr (show p) >> putStr " ->" >> putStrLn msg

-- /show

pubSub=  do
  option "pubs" "an example of publish-suscribe using Event Vars (EVars)"
  v <- newEVar  :: TransIO (EVar String)
  suscribe v <|> publish v
  where  
  publish v= do
    liftIO $ putStrLn "Enter a message to publish"
    msg <- input(const True)
    writeEVar v msg
    liftIO $ print "after writing the EVar"

  suscribe :: EVar String -> TransIO ()
  suscribe v= proc1 v  <|>  (proc2 v)
   

  proc1 v=  do
    msg <- readEVar v 
    liftIO $ putStrLn $  "proc1 readed var: " ++ show msg
    
  proc2 v= do
    msg <- readEVar v 
    liftIO $ putStrLn $ "proc2 readed var: " ++ show msg

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

module Base  where


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


{-# INLINE (!>) #-}
(!>) = const. id --  flip trace
infixr 0 !>

data TransIO  x = Transient  {runTrans :: StateT EventF IO (Maybe x)}
type SData= ()

type EventId= Int

type TransientIO= TransIO

data EventF  = forall a b . EventF{xcomp       :: TransientIO a
                                  ,fcomp       :: [b -> TransientIO b]
                                  ,mfData      :: M.Map TypeRep SData
                                  ,mfSequence  :: Int
                                  ,threadId    :: ThreadId
                                  ,freeTh      :: Bool
                                  ,parent      :: Maybe EventF
                                  ,children    :: TVar[EventF]
                                  ,maxThread   :: Maybe (P Int)
                                  }
                                  deriving Typeable

type P= IORef
newp= newIORef


--(=:) :: P a  -> (a -> a) -> IO()
(=:) n f= liftIO $ atomicModifyIORef' n $  \v ->  ((f v),())

addr x= show $ unsafePerformIO $ do
       st <- makeStableName $! x
       return $ hashStableName st




instance MonadState EventF  TransientIO where
  get=  Transient $ get >>= return . Just
  put x= Transient $ put x >> return (Just ())

type StateIO= StateT EventF  IO

--type TransientIO= Transient StateIO

--runTrans ::  TransientIO x -> StateT EventF  IO (Maybe x)
--runTrans (Transient mx) = mx


runTransient :: TransientIO x -> IO (Maybe x, EventF)
runTransient t= do

  th <- myThreadId
  let eventf0=  EventF  empty [] M.empty 0
          th False  Nothing  (unsafePerformIO $ newTVarIO []) Nothing


  runStateT (runTrans t) eventf0{threadId=th} !> "MAIN="++show th





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
         was <- getSessionData `onNothing` return NoRemote
         if was== WasRemote
           then  return Nothing
           else do
             liftIO $ writeIORef rf  k -- :: StateIO ()

             mfdata <- gets mfData
             put $ EventF g (unsafeCoerce appg :  fs) mfdata b c d peers  children g1


             x <- runTrans g              !> "RUN g"
             liftIO $ writeIORef rg  x
             return $ k <*> x


data IDynamic= IDyns String | forall a.(Read a, Show a,Typeable a) => IDynamic a

instance Show IDynamic where
  show (IDynamic x)= show $ show x
  show (IDyns s)= show s

instance Read IDynamic where
  readsPrec n str= map (\(x,s) -> (IDyns x,s)) $ readsPrec n str


type Recover= Bool
type CurrentPointer= [LogElem]
type LogEntries= [LogElem]
data LogElem=  WaitRemote | Exec | Step IDynamic deriving (Read,Show)
data Log= Log Recover  CurrentPointer LogEntries deriving Typeable


instance  Alternative TransientIO where
  empty = Transient $ return  Nothing
  (<|>) = mplus

--  Transient f <|> Transient g= Transient $ do
--         k <-   f
--         x <-   g
--         return $ k <|> x

data RemoteStatus=  WasRemote | NoRemote deriving (Typeable, Eq)

instance MonadPlus TransientIO where
    mzero= empty
    mplus  x y=  Transient $ do
         mx <- runTrans x    -- !> "RUNTRANS11111"
         was <- getSessionData `onNothing` return NoRemote
         if was== WasRemote
           then return Nothing
           else case mx of
             Nothing -> runTrans y   --  !> "RUNTRANS22222"
             justx -> return justx

-- | a sinonym of empty that can be used in a monadic expression. it stop the
-- computation
stop :: TransientIO a
stop= Control.Applicative.empty

instance Monoid a => Monoid (TransientIO a) where
  mappend x y = mappend <$> x <*> y
  mempty= return mempty

-- | set the current closure and continuation for the current statement
setEventCont ::   TransientIO a -> (a -> TransientIO b) -> StateIO ()
setEventCont x f  = do
   st@(EventF   _ fs d n  r applic  ch rc bs)  <- get
   put $ EventF x ( unsafeCoerce f : fs) d n  r applic  ch rc bs


-- | reset the closure and continuation. remove inner binds than the prevous computations may have stacked
-- in the list of continuations.
resetEventCont :: Maybe a -> StateIO ()
resetEventCont mx =do
   st@(EventF _ fs d n  r nr  ch rc bs)  <- get

   let f= \mx ->  case mx of
                       Nothing -> empty
                       Just x  -> (unsafeCoerce $ head fs)  x
   put $ EventF  (f mx) ( tailsafe fs)d n  r nr  ch rc bs
   where
   tailsafe []=[]
   tailsafe (x:xs)= xs


instance Monad TransientIO where
      return x = Transient $ return $ Just x
      x >>= f  = Transient $ do
        setEventCont x  f

        mk <- runTrans x
        resetEventCont mk
        case mk of
           Just k  -> do
               runTrans $ f k

           Nothing -> return Nothing




--instance MonadTrans (Transient ) where
--  lift mx = Transient $ mx >>= return . Just

instance MonadIO TransientIO where
  liftIO x = Transient $ liftIO x >>= return . Just --     let x= liftIO io in x `seq` lift x




-- Threads

waitQSemB sem= atomicModifyIORef' sem $ \n -> if n > 0 then(n-1,True) else (n,False)
signalQSemB sem= atomicModifyIORef' sem  $ \n ->  (n + 1,())

-- | set the maximun number of threads for a procedure. It is useful for the
threads :: Int -> TransientIO a -> TransientIO a
threads n proc= Transient $ do
   msem <- gets maxThread
   sem <- liftIO $ newIORef n
   modify $ \s -> s{maxThread= Just sem}
   r <- runTrans proc
   modify $ \s -> s{maxThread = msem} -- restore it
   return r
-- | delete all the previous childs generated by the expressions and continue execution
-- of the current thread.
oneThread :: TransientIO a -> TransientIO a
oneThread comp=  do
   chs <- liftIO $ newTVarIO []
   r <- comp
   modify $ \ s -> s{children= chs}
   killChilds
   return r



-- | The threads generated in the process passed as parameter will not be killed.
freeThreads :: TransientIO a -> TransientIO a
freeThreads proc= Transient $ do
     st <- get
     put st{freeTh= True}
     r <- runTrans proc
     modify $ \st -> st{freeTh= freeTh st}
     return r

-- | The threads will be killed when the parent thread dies. That is the default
-- This can be invoked to revert the effect of `freeThreads`
hookedThreads :: TransientIO a -> TransientIO a
hookedThreads proc= Transient $ do
     st <- get
     put st{freeTh= False}
     r <- runTrans proc
     modify $ \st -> st{freeTh= freeTh st}
     return r


-- | kill all the child processes
killChilds :: TransientIO()
killChilds= Transient $  do
   cont <- get
   liftIO $  killChildren cont
   return $ Just ()


-- | Get the session data of the desired type if there is any.
getSessionData ::  (MonadState EventF m,Typeable a) =>  m (Maybe a)
getSessionData =  resp where
 resp= gets mfData >>= \list  ->
    case M.lookup ( typeOf $ typeResp resp ) list  of
      Just x  -> return . Just $ unsafeCoerce x
      Nothing -> return $ Nothing
 typeResp :: m (Maybe x) -> x
 typeResp= undefined

-- | getSessionData specialized for the View monad. if Nothing, the
-- monadic computation does not continue. getSData is a widget that does
-- not validate when there is no data of that type in the session.
--
-- If there is no such data, `getSData`  silently stop the computation.
-- That may or may not be the desired behaviour.
-- To make sure that this does not get unnoticed, use this construction:
--
-- >  getSData <|> error "no data"
getSData ::  Typeable a => TransIO  a
getSData= Transient getSessionData


-- | set session data for this type. retrieved with getSessionData orr getSData
setSessionData ::  (MonadState EventF m, Typeable a) => a -> m ()
setSessionData  x= do
  let t= typeOf x in  modify $ \st -> st{mfData= M.insert  t (unsafeCoerce x) (mfData st)}

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
   killChilds
   return r


async  ::  IO b -> TransientIO b
async io= do
   r <- parallel  (Left <$>io)
   killChilds
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
              forkMaybe cont $ iocont dat
              loop'

  forkMaybe  cont loop'
  where
      forkMaybe cont proc = do
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
                let (sbs', found) = drop [] th  sbs -- !> "search "++show th ++ " in " ++ show (map threadId sbs)
                when found $ writeTVar sibling sbs'
                return found
       if (not found && isJust (parent env))
         then free th $ fromJust $ parent env -- !> "toparent"
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
        let childs= children cont   !> "killChildren list= "++ addr (children cont)
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


-- * non-blocking keyboard input

getLineRef= unsafePerformIO $ newTVarIO Nothing


roption= unsafePerformIO $ newMVar []

-- | install a event receiver that wait for a string and trigger the continuation when this string arrives.
option :: (Typeable b, Show b, Read b, Eq b) =>
     b -> [Char] -> TransientIO b
option ret message= do
    let sret= show ret

    liftIO $ putStrLn $ "Enter  "++sret++"\tto: " ++ message
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
              let rs = breakSlash [] r
              mapM_ (\ r -> do threadDelay 1000
                               atomically . writeTVar  getLineRef $ Just r) rs
              inputLoop'

    breakSlash :: [String] -> String -> [String]
    breakSlash s ""= s
    breakSlash res s=
      let (r,rest) = span(/= '/') s
      in breakSlash (res++[r]) $ tail1 rest
      where
      tail1 []=[]
      tail1 x= tail x

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

{-# START_FILE Backtrack.hs #-}
-- show
-- /show

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
{-# LANGUAGE BangPatterns #-}


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
      else liftIO $ atomicModifyIORef v $ \(n,xs) ->  ((0,[]),xs)

choose' :: [a] -> TransientIO a
choose'  xs = foldl (<|>) empty $ map (parallel . return . Left) xs


--newtype Collect a= Collect (MVar (Int, [a])) deriving Typeable

-- collect the results of a search done in parallel, usually initiated by
-- `choose` . The results are added to the collection with `found`
--
--


-- execute a process and get the first n solutions.
-- if the process end without finding the number of solutions requested, it return the fond ones
-- if he find the number of solutions requested, it kill the threads of the process and return
-- It works monitoring the solutions found and the number of active threads.
-- If the first parameter is 0, collect will return all the results
collect ::  Int -> TransientIO a -> TransientIO [a]
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

      detect= freeThreads $ do
          xs <- async $ do
             threadDelay 1000 -- to allow some activity before monitoring it
             atomically $ do
                (n',xs) <- readTVar rv
                ns <- readTVar $ children st

                if (n > 0 && n' >= n) ||  null ns  !> show (n,n') !> (show $ length ns)
                  then return xs
                  else retry

          th <- liftIO $ myThreadId !> "KILL"
          stnow <- get
          liftIO $ killChildren st
          liftIO $ addThread st stnow
          return  xs

  (any1 >> stop)  <|> detect

{-# START_FILE Logged.hs #-}

-- show
-- /show

-----------------------------------------------------------------------------
--
-- Module      :  Transient.Logged
-- Copyright   :
-- License     :  GPL-3
--
-- Maintainer  :  agocorona@gmail.com
-- Stability   :
-- Portability :
--
-- |
--
-----------------------------------------------------------------------------
{-# LANGUAGE  ExistentialQuantification #-}
module Logged  where

import Data.Typeable
import Unsafe.Coerce
import Base
import Control.Applicative
import Control.Monad.IO.Class



fromIDyn :: (Read a, Show a, Typeable a) => IDynamic -> a
fromIDyn (IDynamic x)= unsafeCoerce x

fromIDyn (IDyns s)=r where r= read s !> "read " ++ s ++ "to type "++ show (typeOf r)

toIDyn x= IDynamic x

-- | synonymous of `step`
logged :: (Show a, Read a, Typeable a) => TransientIO a -> TransientIO a
logged= step



-- | write the result of the computation in the log and return it.
-- but if there is data in the internal log, it read the data from the log and
-- do not execute the computation.
--
-- It accept nested step's. The effect is that if the outer step is executed completely
-- the log of the inner steps are erased. If it is not the case, the inner steps are logged
-- this reduce the log of large computations to the minimum. That is a feature not present
-- in the package Workflow.
--
-- >  r <- step $ do
-- >          step this :: TransIO ()
-- >          step that :: TransIO ()
-- >          step thatOther
-- >  liftIO $ print r
--
--  when `print` is executed, the log is just the value of r.
--
--  but when `thatOther` is executed the log is: [Exec,(), ()]
--
step :: (Show a, Read a, Typeable a) => TransientIO a -> TransientIO a
step mx=  do
    Log recover rs full <- getSData <|> return ( Log False  [][])

    case (recover,rs) of
      (True, Step x: rs') -> do
            setSData $ Log recover rs' full
            return $ fromIDyn x  !>  "read in step:" ++ show x

      (True,Exec:rs') -> do
            setSData $ Log recover rs' full
            mx

      (True, WaitRemote:rs') -> do
            setSData (Log recover rs' full) !> "waitRemote2"
            empty

      _ -> do
            let add= Exec:  full
            setSData $ Log False add add
            r <-  mx
            let add= Step (toIDyn r): full
            setSData $ Log False add add
            return  r

{-# START_FILE Move.hs #-}

--show
-- /show

-----------------------------------------------------------------------------
--
-- Module      :  Transient.Move
-- Copyright   :
-- License     :  GPL-3
--
-- Maintainer  :  agocorona@gmail.com
-- Stability   :
-- Portability :
--
-- |
--
-----------------------------------------------------------------------------
{-# LANGUAGE DeriveDataTypeable , ExistentialQuantification
    ,ScopedTypeVariables, StandaloneDeriving #-}
module Move where
import Base
import Logged
import Data.Typeable
import Control.Applicative
import Network
import Network.HTTP
import Control.Monad.IO.Class
import System.IO
import Control.Exception
import Data.Maybe
import Unsafe.Coerce
import System.Process
import System.Directory
import Control.Monad
import Network.Info
import System.IO.Unsafe
import Control.Concurrent.STM as STM
import Data.Monoid
import qualified Data.Map as M
import Data.List (nub)



-- | install in a remote node a haskell package with an executable transient service initialized with `listen`
-- the package, the git repository and the main exectable must have the same name
installService node port servport package= do
  beamTo node port
  liftIO $ do
     let packagename= name package
     exist <- doesDirectoryExist  packagename
     when (not exist) $ do
         runCommand $ "git clone "++ package
         runCommand $ "cd "++ packagename
         runCommand "cabal install"
         createProcess $ shell $ "./dist/build/"++ packagename++"/"++packagename
                                       ++ " " ++ show port
         return()
  where
  name path=
     let x= dropWhile (/= '/') path
     in if x== "" then tail path else name $ tail    x


beamTo :: HostName -> PortID -> TransientIO ()
beamTo host port= do
  Log rec log _ <- getSData <|> return (Log False [][])
  if rec then return () else do
      h <- liftIO $ connectTo host port
      liftIO $ hSetBuffering h LineBuffering
      liftIO $ hPutStrLn h (show $ reverse log) >> hFlush h
      liftIO $ hClose h
      delSData h
      stop

forkTo  :: HostName -> PortID -> TransientIO ()
forkTo host port= do
  Log rec log _<- getSData <|> return (Log False [][])
  if rec then return () else do
      h <- liftIO $ connectTo host port
      liftIO $ hSetBuffering h LineBuffering
      liftIO $ hPutStrLn h (show $ reverse log)  >> hFlush h
      liftIO $ hClose h
      delSData h


callTo :: (Show a, Read a,Typeable a) => HostName -> PortID -> TransIO a -> TransIO a
callTo host port remoteProc= logged $ Transient $ do
--      liftIO $ print "callto"
      Log rec log fulLog <- getSessionData `onNothing` return (Log False [][])
      if rec
         then
          runTrans $ do

               Connection port  h sock <- getSData <|> error "callto: no hander"
               r <- remoteProc !> "executing remoteProc" !> "CALLTO REMOTE" -- LOg="++ show fulLog
               liftIO $ hPutStrLn h (show r) --  `catch` (\(e::SomeException) -> sClose sock)
                -- !> "sent response, HANDLE="++ show h
               setSData WasRemote
               stop


         else do

            h <- liftIO $ connectTo host port
            liftIO $ hPutStrLn h (show $ reverse fulLog) >> hFlush h !> "CALLTO LOCAL" -- send "++ show  log
            let log'= WaitRemote:tail log
            setSessionData $ Log rec log' log'
            runTrans $ waitEvents $ do -- local side
                   liftIO $ hSetBuffering h LineBuffering
                   s <- hGetLine h
--                   hClose h

                   let r = read s

                   return r   !> "read: " ++ s ++" response type= "++show( typeOf r)




data Connection= Connection PortID Handle Socket deriving Typeable

-- | Wait for messages and replay the rest of the monadic sequence with the log received.
listen :: PortID ->  TransIO ()
listen  port = do
       setSData $ Log False [] []
       sock <- liftIO $ withSocketsDo $ listenOn  port

       (h,host,port1) <- parallel $ Right <$> accept sock
                          `catch` (\(e::SomeException) -> sClose sock >> throw e)

       liftIO $  hSetBuffering h LineBuffering  -- !> "LISTEN in "++ show (h,host,port1)

       slog <- Transient $ liftIO $ (Just <$> hGetLine  h)
                          `catch` (\(e::SomeException) -> print "ERR" >>  return Nothing)

       setSData $ Connection port h sock  -- !> "setdata port=" ++ show port

       let log= read slog   -- !> "read1 " ++ slog
       setSData $ Log True log (reverse log)


-- | init a Transient process in a interactive as well as in a replay mode.
-- It is intended for twin processes that interact among them in different nodes.
beamInit :: PortID -> TransIO a -> IO b
beamInit port program=  keep $ do
    listen port   <|> return ()
    program
--    (program >> stop)   <|> close
--    where
--    close= do
--       Connection _ h sock <- getSData
--       liftIO $ hClose h  `catch` (\(e::SomeException) -> sClose sock)




instance Read PortNumber where
  readsPrec n str= let [(n,s)]=   readsPrec n str in [(fromIntegral n,s)]


deriving instance Read PortID
deriving instance Typeable PortID

-- * Level 2: connections node lists and operations with the node list

type Node= (HostName,PortID)

nodeList :: TVar  [Node]
nodeList = unsafePerformIO $ newTVarIO []

deriving instance Ord PortID

getNodes :: TransIO [Node]
getNodes  = Transient $ Just <$> (liftIO $ atomically $ readTVar  nodeList)




addNodes   nodes= Transient . liftIO . atomically $ do
  prevnodes <- readTVar nodeList

  writeTVar nodeList $ nub $ prevnodes ++  nodes
  return $ Just ()



-- | execute a Transient action in each of the nodes connected. The results are mappend'ed
clustered :: (Typeable a, Show a, Read a) => Monoid a => TransIO a -> TransIO a
clustered proc= logged $ do
     nodes <- logged getNodes
     logged $ foldr (<>) mempty $ map (\(h,p) -> callTo h p proc) nodes !> "fold"

-- | Connect to a new node to another. The other node will notify about this connection to
-- all the nodes connected to him. the new connected node will receive the list of connected nodes
-- the nodes will be updated with this list. it can be retrieved with `getNodes`
connect ::   HostName ->  PortID ->  HostName ->  PortID -> TransientIO ()
connect host  port   remotehost remoteport=  do
    listen port <|> return ()
    logged $ do
        logged $ addNodes [(host,port)]
        logged $ liftIO $ putStrLn $ "connecting to: "++ show (remotehost,remoteport)
        host <- logged $ return host
        port <- logged $ return port
        nodes <- callTo remotehost remoteport   $ do
                   clustered $  addNodes [(host, port)]
                   getNodes

        logged $ addNodes nodes
        logged $ liftIO $ putStrLn $ "Connected to modes: " ++ show nodes


{-# START_FILE Vars.hs #-}
-- show
-- /show

{-# LANGUAGE DeriveDataTypeable #-}
module Vars where

import Base
import qualified Data.Map as M
import Data.Typeable

import Control.Concurrent
import Control.Applicative
import Data.IORef
import Control.Monad.IO.Class
import Control.Monad.State

newtype EVars= EVars  (IORef (M.Map Int [EventF])) deriving Typeable

data EVar a= EVar Int (IORef (Maybe a))

-- * concurrency effect 
-- Evars are event vars. `writeEVar` trigger the execution of all the continuations associated to the  `readEVar` of this variable
-- (the code that is after them) as  stack: the most recent reads are executed first.
--
-- It is like the publish-suscribe pattern but without inversion of control, since a readEVar can be inserted at any place in the
-- Transient flow.
--
-- EVars are created upstream and can be used to communicate two subbranches of the monad. Following the Transient philosophy they 
-- do not block his own thread if used with alternative operators, unlike the IO and STM vars. 
-- 

newEVar ::  TransientIO (EVar a)
newEVar  = Transient $ do
   EVars ref <- getSessionData `onNothing`  do
                            ref <- liftIO $ newIORef M.empty
                            setSData $ EVars ref
                            return  (EVars ref) 
   id <- genNewId
   ref <- liftIO $ newIORef Nothing
   return . Just $ EVar id ref

readEVar :: EVar a -> TransIO a
readEVar (EVar id ref1)= Transient $ do 
         cont <- getCont
         EVars ref <- getSessionData `onNothing` error "No Events context"
         map <- liftIO $ readIORef ref
         let Just conts=  M.lookup  id map <|> Just []
         liftIO $ writeIORef ref $  M.insert id (cont:conts) map
         return Nothing
         liftIO $ readIORef ref1
         
writeEVar (EVar id ref1) x= Transient $ do
   EVars ref <- getSessionData `onNothing`  error "No Events context" 
   liftIO $ writeIORef ref1 $ Just x
   map <- liftIO $ readIORef ref
   let Just conts= M.lookup id map <|> Just []
   mapM runCont conts 
   return $ Just ()
 

```


`step` (`logged` is a synonym) is the primitive that perform the logging and recovery. it add entries to the log. But when a log has been received, it return the corresponding entry in the log instead of executing his action. Until there is no more entries in the log. In this case it resumes normal execution and logging.

```haskell
step :: (Show a, Read a, Typeable a) => TransientIO a -> TransientIO a
step mx= do

    Log recover rs <- getSData <|> return (Log False [])

    if recover && not (null rs)
      then do
        setSData $ Log recover $ tail rs
        return $ fromIDyn $ head rs
      else do
        r <- mx
        setSData . Log False $ toIDyn r:rs
        return r
```

Note that the user program is a single monadic expression. there is no callbacks neither handlers, there is no routing, no registering of remote procedures, no dispatching. 

The dispatching is done by the replaying of the log. As always, I use the Transient philosophy of a complete de-inversion of control.

To execute the example clone the transient package:

    git clone https://github.com/agocorona/transient

the user program is in the file main.hs at the end. There is another test program called move.hs that contains some tests of distributed computing for N nodes.  The moving services are defined in the module "Transient.Move" in the "src" folder.

The move.hs program can be executed in two or more nodes. The program need three parameters:

    program  localPort  remoteHost remotePort

I use it locally in two instances. If I name the program "move":

```
   runghc -isrc move 8080 localhost 8081
   runghc -isrc move 8081 localhost 8080
```

Each node can act as master and operate with the other node.



#Current limitations and future plans

At this moment, the  program must be running in all the nodes, but the first remote service that I will develop with this basic infrastructure will be for moving code.  It is not necessary to have all the code in the other node, just the code that will be called remotely. For example, for a web application, the code that interact with the user interface can be in a haskell file. it can be compiled with Haste or GHCJS before sending it to the browser. In cloud applications, if the nodes have different architectures, the remote node will receive the source code and will compile it locally. 

Hot swapping of code is also a goal. Once a program is substituted by a new version, replaying from the log would recover the node state.

In the medium-long term my goal is to develop the level 3 and 4 services that can move executions among nodes at runtime, depending on runtime conditions, capabilities of the nodes, failure events, changes in the topology of the network etc. the monitor would execute continuations in any location. That is why the code need to be seamless. The code must not reflect the topology of the network, it must be abstracted from it. That is why all the program is a single monadic expression executed in different nodes, in the same way that currently Transient execute a single monadic expression in different threads. The distributed mechanism just extend it to run the thread in a different node.




# Moving code

Installing a new service that respond requests in a remote node is easy in the era of Internet: it is a matter of impersonating in the other node with `beamTo` and install the corresponding package:

```haskell
-- | install in a remote node a haskell package with an executable transient service initialized with `listen`
-- the package, the git repository and the main exectable must have the same name

installService node port servport gitrepo= do
  beamTo node port
  liftIO $ do
     let packagename= name gitrepo
     exist <- doesDirectoryExist  packagename
     when (not exist) $ do
         runCommand $ "git clone "++ gitrepo
         runCommand $ "cd "++ packagename
         runCommand "cabal install"
         createProcess $ shell $ "./dist/build/"++ packagename++"/"++packagename
                                       ++ " " ++ show port

```

# Comparison with Cloud Haskell and Erlang

I don't know erlang neither cloud haskell very well. This development is at the beginning and the erlang model is mature.
The Erlang model is intentionally low level, it tries to make the network and the communication costs visible. It focuses on reliability

Mine tries to make the network transparent and emphasizes composability and high level programming. An example of composability is in the mapReduce snippet. My goal is to make the network transparent, in the way of [Akka](http://akka.io) . 

But akka and Erlang OTP are asynchronous and blocking. Well, akka is event-based. The green threads of Erlang is a higher level way of using callbacks, where the callback is managed by the threads library. That permits a code more readable, since the flow does continue after the blocking call. But that does not change the fact that both Erlang OTP and Akka use a blocking semantics for synchronous calls. That precludes parallel composition of the kind of my mapReduce snippet: simply, there is no way to combine two Erlang or akka elements that work in parallel within the same expression. All the seaming has to be done by hand. Usually by means of auxiliary monitor processes that execute what in haskell would be an applicative expression where the elements are embedded.

For this reason, the Actor model is asynchronous, in order to avoid blocking. Transient does not enforce asynchronous communications since it is non blocking, this permits more integrated and seamless code, more readable and robust since the compiler can verify more invariants.

In fact **a distributed transient program is a single monadic, seamless expression or a subexpression of the whole Transient program**, replicated in all the nodes totally or partially. This expression is not monolithic as is the case of other reactive frameworks, but made of pieces, each one with a functional meaning. 

For example, *a sender has no meaning without the receiver. However, they are separate expressions in all the other frameworks. Even if both computations are part of the same object. To use them, the programmer has to do some manual wiring to glue them together. In Transient both sender and receiver are codified in a single expression, that is type checked and composable*. That splitting of sender and receiver in Erlang and other cloud solutions is not a feature as such, but the result of a necessity, due to the lack of composability associated with inversion of control.

These are two examples of the ping-pong app in Transient, using the latest version in gitHub:

```haskell
pingPong :: TransIO ()
pingPong= do
     (node1, node2) <- logged $ return (node1,node2) -- will work with the node1 context
     beamTo node2
     step $ liftIO $ putStrLn "PING"

     beamTo node1
     step $ liftIO $ putStrLn "PONG"
```

```haskell
pingPong :: TransIO ()
pingPong= callTo node2  liftIO (putStrLn "PING") >> liftIO (putStrLn "PONG")
```

It is also the opposite of what Erlang and other similar distributed architectures propose but maintaining their advantages. Transient implement a shared nothing semantics, and the transported data is explicitly labeled with logging calls, but it behaves as if sharing actually happens. Once eliminated the surprise, there is very little cognitive impedance in doing a distributed call using Transient.

The final objective is a cloud monad, or a cloud effect where distribution of resources happens automatically depending on an strategy for the distribution of resources that will be defined by the programmer. The software will be the one that will evaluate the costs. The platform will be agnostic about the concrete architecture; This will depend on the application and will be dynamic. 

For example, if a database program has too much load, it can split into two shards in two different nodes, the requests will be redirected to the appropriate node, but this is upto the application logic. The framework will provide facilities for request forwarding and moving code and data.

Concerning inversion of control, the approach is more close to this: http://lampwww.epfl.ch/~odersky/papers/jmlc06.pdf. This paper seems to be a precursor of more advanced works in scala, like the Scala futures and Akka. Although Scala  can do half of the inversion of control since the handling code can never return to the main flow, since this main flow is imperative (And the paper makes this clear). They use typically monoidal (List like) processing using futures. flatMap, is concatMap in Haskell, the bind (>>=) of the list monad.

However this kind of processing can not return the result to the main imperative procedure that called him, since it is not monadic. So it need to use callbacks. 

Concerning mobility of the software and functionality, It is more closer to Objectspace Voyager: http://www.inf.fu-berlin.de/lehre/WS99/VS/Misc/Voyager/API/doc/orb.pdf

I dedicate this software to Graham Glass, the genius behind Objectspace Voyager that developed a cloud platform 15 years ago that is still years ahead of anything else.

[https://github.com/agocorona/transient](https://github.com/agocorona/transient)





AMDG




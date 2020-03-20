# Estimation of π using distributed computing. Streaming. Transient effects VI

AMDG

#Distributed streaming#

** Run the source code of this article at:**

[https://www.fpcomplete.com/user/agocorona/autocloned/transient-streaming](https://www.fpcomplete.com/user/agocorona/autocloned/transient-streaming)

With Transient, It is possible to make an streaming expression distributed simply by adding an extra combinator.

This program stream even numbers and print them using monad comprehension in the same machine[ (see the article about parallel non-determinism)](https://www.fpcomplete.com/user/agocorona/beautiful-parallel-non-determinism-transient-effects-iii):

```haskell
{-# LANGUAGE MonadComprehensions #-}
import Transient.Base
import Transient.Indeterminism

main= keep $ [2*x| x <- choose[1..]]) >>= liftIO . print
```

This program stream even numbers from a remote node to the local node, where they are printed: 

```haskell
{-# LANGUAGE MonadComprehensions #-}
import Transient.Base
import Transient.Indeterminism
import Transient.Move

main= do
  let myNode    = createNode "localhost"     2000
      remoteNode= createNode "rem.oteho.st"  2000

  
  beamInit myNode $ do
    logged $ option "start" "start"
    runAt remoteNode [2*x| x <- choose[1..]] >>= liftIO . print

```
The remote host must run the same program with the roles of the nodes changed in the other way around. A command line parameter can be added to switch them. So the same program in both cases can be used.

"start[ENTER]"  should be entered to init the streaming in one of the two nodes (or both)

Both the first stand-alone program and the distributed one are multithreaded. By default, `choose` spawn a thread for each entry produced. The second snippet also spawn a new thread in the receiving node for each received element. This happens currently if it is not limited with the thread management primitives, but this is easy. Just prefix the expression with the number of threads that you want to use:

```haskell
main= beamInit myNode $ threads 1 
        $  (runAt remoteNode [2*x| x <- choose[1..]]) >>= liftIO . print
```

`runAt` is another name for `callTo` explained in [another article](https://www.fpcomplete.com/user/agocorona/moving-haskell-processes-between-nodes-transient-effects-iv) as well as `beamInit`.

There are many problems related with streaming: how to open and close resources? Since it is multithreaded, how to adjust the troughput of sender and receiver? contention of the sender, socket block sizes, buffering etc.

For example, if we do not limit the number of threads of the sender, since the print operation is slower than the generation of numbers, the buffers of the socket will be filled so the current and the next produced will be blocked, but since the number of them is not limited, new threads will be spawned and will block continuously. The sender program will occupy more and more memory. With `threads n`  this can not happen. incidentally this limitation would apply both to the sender and the receiver.

To illustrate a more complex example of distributed computing I will show it with a calculation of the number π using random numbers. The method can be seen in the [spark examples](http://tldrify.com/brb)

This code estimates π by "throwing darts" at a circle. We pick random points in the unit square ((0, 0) to (1,1)) and see how many fall in the unit circle. The fraction should be π / 4, so we use this to get our estimate.

The direct translation of the spark example would be something similar to this:

```haskell
     xs <- collect numSamples 
           $ clustered [if x * x + y * (y ::Double) < 1 then 1 else (0 :: Int)
                                 | x <- random, y <-random] 
     liftIO $ print $ 4.0 * (fromIntegral $ sum xs) / (fromIntegral numSamples)
```
Spark as well as any distributed framework has configurations behind to set up the cluster and instantiate the supervisors and workers that "materialize" the expression. 

This is the complete program in Haskell including the configuration. All the nodes are simulated within the same process space. But, otherwise, they communicate using the same socket and stream mechanism:

```haskell
main= do
   let numNodes= 5
       numSamples= 1000
       ports= [2000.. 2000+ numNodes -1]
       createLocalNode p= createNode "localhost" (PortNumber p)
       nodes= map createLocalNode ports

   addNodes nodes
   keep $ do

     xs <- collect numSamples $ do
                   foldl (<|>) empty (map listen nodes) <|> return()
                   clustered [if x * x + y * y  < 1 then 1 else (0 :: Int)| x <- random, y <-random] 
     liftIO $ print $ 4.0 * (fromIntegral $ sum xs) / (fromIntegral numSamples)
     
     where
     random= waitEvents randomIO :: TransIO Double
```
where is the wiring that connect processes in different nodes? It is implicit in `clustered`. More on that below

There are many things scrambled in the above code, which is very dense. the code execute the calculation in all the nodes until `numSamples` are `collect`'ed in the calling node.

`listen` wait for requests from each of the nodes. To see how it works, see [the previous article](https://www.fpcomplete.com/user/agocorona/moving-haskell-processes-between-nodes-transient-effects-iv)

In transient, the alternative `<|>` operator is used for parallelism ever that the left side spawn a new thread and return `empty` to the current thread. It is like if a second process has forked the operand in the alternative expression followed by the continuation.

so thanks to `<|>` repeated by the `foldl` expression, all the listeners for the simulated nodes are waiting for requests. In the `foldl` line, five processes in five threads -one for each node- have been started (since `listen` invokes `parallel`) plus the current one, that continues thanks to the `return ()` statement. this local thread is the one that invokes all the rest using clustered:

```haskell
clustered :: Loggable a  => TransIO a -> TransIO a
clustered proc= logged $ do
     nodes <- step getNodes
     logged $ foldr (<|>) empty $ map (\node -> callTo node proc) nodes !> "fold"
```
`clustered` get the nodes and launch a `callTo` with the computation to all of them.
Sort to speak, each request includes a pointer to the concrete code in the application that want to execute remotely and with which parameters. This pointer is transported as a log which is transported by the request.

The previous example execute nodes within the same process space. The nodes are created in the local machine.


To run it in a real cluster of nodes, it is necessary to `connect` them:

```haskell
main= do
    args <- getArgs
    localHost <- args !! 0
    localPort <- args !! 1
    seedHost  <- args !! 2
    seedPort  <- args !! 3
    mynode= createNode localHost localPort
    seedNode= createNode seedHost seedPort
    
    keep $ do
        connect mynode seednode

        logged $ option  "start"  "start"

        xs <- collect numSamples $
                   clustered [if x * x + y * y  < 1 then 1 else (0 :: Int)| x <- random, y <-random] 
        liftIO $ print $ 4.0 * (fromIntegral $ sum xs) / (fromIntegral numSamples)
        exit
    where
    random= waitEvents randomIO :: TransIO Double
```
to invoke the program:

```
> program localHost localPort seedHost seedPort
```

The seedNode is a node already connected. A node can connect to himself in order to be a seed node. The `option` statement wait for the "start"[ENTER] to be entered in the console. This is in order to allow the connection of the rest of the nodes before executing the process. Since all the nodes share the same program, anyone can take the role of master by  simply starting the process.

all statements between the listen point and the remote call must be `logged`. otherwise the statement not logged will be executed in the remote node. This latter is interesting for some purposes that can not be discussed here.

Note: currently, `collect` has problems for terminating remote computations.

#Streaming#

Let's construct a variant of the same program that can be used to demonstrate a more continuous streaming. The new program perform the same calculation but it does not stop, and the results are accumulated in in a mutable reference within the calling node, so the precision in the value of pi is printed with more and more precision. every 1000 calculations.

Here instead of `collect` that finish the calculation when the number of samples has been reached, i use `group` which simply group the number of results in a list

Since `group` do not finish the calculation, new sums are streamed from the nodes again and again. 



```haskell
main= do
   let numNodes= 5
       numCalcsNode= 100
       ports= [2000.. 2000+ numNodes -1]
       createLocalNode p= createNode "localhost" (PortNumber p)
       nodes= map createLocalNode ports

   rresults <- newIORef (0,0)
   keep $ freeThreads $ threads 1 $ do
--     setBufSize 1024
     addNodes nodes
     foldl (<|>) empty (map listen nodes) <|> return()

     r <- clustered $ do
--               Connection (Just (_,h,_,_)) _ <- getSData <|> error "no connection"
--               liftIO $ hSetBuffering h $ BlockBuffering Nothing
               r <- group numCalcsNode $ do
                    n <- liftIO  getNumCapabilities
                    threads n . 
                        spawn $ do
                            x <- randomIO :: IO Double
                            y <- randomIO
                            return $ if x * x + y * y < 1 then 1 else (0 :: Int)
               return $ sum r

     (n,c) <- liftIO $ atomicModifyIORef' rresults $ \(num, count) ->
                let num' = num + r
                    count'= count + numCalcsNode
                in ((num', count'),(num',count'))

     when ( c `rem` 1000 ==0) $ liftIO $ do
           th <- myThreadId
           putStrLn $ "Samples: "++ show c ++ " -> " ++
             show( 4.0 * fromIntegral n / fromIntegral c)++ "\t" ++ show th
```

There are more explicit configuration options put there. Socket Buffering parameters and stream buffering parameters also can be changed. 

Of course all of this is experimental. There is a long way before this has industrial strenght. I want to make some of these parameter adjustments more automatic, so that they may disappear under higher level primitives. That's why I don´t want external configurations.

An intelligent cluster would balance the load among the nodes and adjust to the output to what the reduce stage can process. Some nodes can ignore the requests when they can not respond to the requests because heavy load or because they do not handle that particular request.

#Managing resources#

I defined some operators to open and close resources during streaming so that the elements that manage them would be composable. 

This stream get lines from a file, transform the text to uppercase and write the content in another file:

```haskell
main=  keep . threads 0  $ do
         chunk <- sourceFile "../src/Main.hs"
         liftIO $ print chunk
         return $ map toUpper chunk
       `sinkFile` "outfile""
```

At the end of the processing the files are closed. Any step in the process can send a signal of termination upstream and downstream. For this purpose [event vars `EVars`](https://www.fpcomplete.com/user/agocorona/publish-subscribe-variables-transient-effects-v) are used.

EVent vars are a more powerful mechanism than exceptions.. 

this is all the stack of definitions for  `sinkFile`. 

```haskell
sinkFile :: TransIO String -> String -> TransIO ()
sinkFile input file= 
      process input (openFile file WriteMode)  hClose' hPutStrLn'
  where
  hClose' h= putStr "closing " >> putStrLn file >> hClose h
  hPutStrLn' h  x= (SMore <$>  hPutStrLn h x)
                  `catch` (\(e::SomeException)-> return $ SError (show e))


process
     :: TransIO a
     -> IO handle
     -> (handle -> IO ())
     -> (handle -> a -> IO (StreamData b))
     -> TransIO b
process input open close process=do
   h <- liftIO open
   onFinish (liftIO (close h) >> stop) <|> return()
   some <- input
   process' h  some
   where
   process' h something = do
       v <-  liftIO $ process h  something
       checkFinalize v

checkFinalize v=
   case v of
       SDone ->  finish  >> stop
       SLast x ->  finish >> return x
       SError e -> liftIO (putStr "slurp: " >> putStrLn e) >> finish  >> stop
       SMore x -> return x
              
newtype Finish= Finish (EVar Bool) deriving Typeable

initFinish :: TransIO Finish
initFinish= do
      fin <- newEVar
      let f = Finish fin
      setSData  f
      return f

onFinish :: TransIO () -> TransIO a
onFinish  close= do
       Finish finish <- getSData <|> initFinish
       readEVar finish
       close
       stop

finish :: TransIO ()
finish = do
    liftIO $ putStrLn "finish Called"
    Finish finish <- getSData
    writeEVar finish True
```

instead of defining a new monad transformer for resource management that would make `sinkFile` a more computation in the flow, I defined it within the Transient monad, The drawback is that it is necessary to use infix notation. This arrangement allows to be used in combinations with any other Transient effects such are distributed computing and multithreading without adding extra boilerplate code.

So it is possible to add `sinkFile` at the end of the calculation of Pi to store the values returned in a file.

`process` is the general operation for processing a streamed input. `sourceFile` is also defined in terms of `process`. It open  resources before processing and close them when finish is called. 

Also when the processing return `SDone` or `SError` it invokes `finish` 


#Distributed Datasets#

To make full use of distributed computing and to manage large datasets it is necessary to process data in a location-independent way. map-reduce frameworks like spark manage distributed datasets, which are partitioned among the machines of the cluster. Each partition is processed by the node where it is located. 

Now I´m trying to manage lists of elements located in different nodes. By defining map and reduce operations over these datasets, I can chain map-reduce operations in the cluster as in a single node. Automatic failover can be implemented using logging and replaying, as well as duplication of data.

Not only the map, but also the local reduce operation can be processed at the location of the data with this arrangement.


I defined:
```haskell
data DDS a= Loggable a => DDS (TransIO [Elem [a]])
data Elem a= Ref Node Path deriving (Typeable,Read,Show)

distribute :: Loggable a => [a] -> DDS a             -- distribute the list among the nodes

cmap :: Loggable b => (a -> b) -> DDS a -> DDS b     -- map

reduce ::  Loggable b => ([a] -> b) -> (b -> b -> b)-> b -> DDS a ->TransientIO b    -- NOTE: change
```

*NOTE: reduce has been changed. Not it is not a mere fold since this is not expressive enough. Now it takes a function that get the whole block of data in each node and convert it in something different, then uses another function to sum these results of all the blocks in the master node. It is not perfect. Probably it will change since multi-stage reduction is necessary for some problems*

Each Distributed Data Set  (DDS) contains a Transient computation that return an array of references to files distributed on the nodes. caching (with TCache) has been implemented now. So the processing happens fully in memory.

And It works: This program return the correct answers. It calculates the number od odd and even numbers in a list, both in N simulated nodes in the same process as well as with two nodes in separate processes.

```haskell
main= do 
     let numNodes = 5
         ports = [2000 .. 2000 + numNodes - 1]
         createLocalNode = createNode "localhost"
         nodes = map createLocalNode ports
     addNodes nodes
     keep' $
       do runNodes nodes
          let cdata = distribute [1 .. 10000 :: Int]
          let cdata' = cmap (*3) cdata
          r <- reduce (sumOddEven 0 0) sumIt (0,0) cdata'
          liftIO $ print r
          exit
          
sumOddEven:: Int -> Int -> [Int] -> (Int,Int)          
sumOddEven o e []= (o,e)
sumOddEven o e (x:xs)= 
  if x `rem` 2== 0 then sumOddEven (o+1) e xs
    else sumOddEven o (e+1) xs

sumIt :: (Int,Int) -> (Int,Int) -> (Int,Int)
sumIt (o,e) (o',e')= (o+o',e+e')

runNodes nodes= foldl (<|>) empty (map listen nodes) <|> return()


```

A partial reduction is performed in each remote node. This code is now working at the FPcomplete project
[https://www.fpcomplete.com/user/agocorona/autocloned/transient-streaming](https://www.fpcomplete.com/user/agocorona/autocloned/transient-streaming)

Select DistibDataSets as the Main executable.

Right now `distribute` divide the entry in chunks (partitions). Each one is stored in a file in each node.
then `cmap` and `reduce` operate in each node-partition until the last small reduction that happens in the calling node.

Remarcably, the pure and lazy semantics of the code is almost identical to the [spark map-reduce primitives of distributed datasets](http://tldrify.com/bs1) (in Spark DDS's are called Resilient Distributed Datasets (RDD)) . here `distribute` and `cmap` are pure, while `reduce` is an action that trigger the execution of the previous stages.


#The paradise of composability#

But since Scala/Spark can not fully de-invert the control, the main flow can not receive the results of the execution. That means that it can not  have  the two last lines of the above snippet. It can do it using  [shared mutable variables](http://tldrify.com/bs1) called accumulators, And must wait for the termination of the task. 

In the other side, since Transient fully de-invert the control and the monad share the underlying effects of `map` and `reduce`, it can sequence distributed computations as if they were normal computations.

It can even execute two or more map-reduce operations in parallel and feed the result to the monad:

The applicative operator `<*>` is also the operator for concurrency in Transient when the operands spawn threads:

```haskell
do
    r <- (,) <$> mapReduce [1..10::Int]  <*>  mapReduce [10..20 :: Int]
    liftIO $ print r
    r' <- choose[1.. fst r]
    liftIO $ print (r,r')
    
    where
    mapReduce set=  reduce (+) . cmap (*2) $ distribute dset
```

In the first line, two map-reduce operations over  two different datasets *are executed in parallel and distributed among the nodes*. Then the result is a 2-tuple that feed `choose`a non-deterministic computation that generate as much entries as the first element of the tuple.

The paradise of composability!


AMDG
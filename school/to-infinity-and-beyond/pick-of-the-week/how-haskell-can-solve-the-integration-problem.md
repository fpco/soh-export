# How Haskell can solve the integration problem

#Introduction

This tutorial:

  - Will show how the development of web applications, workflows, asynchronous tasks, Enterprise Application Integration(EAI), Service Integration and Business Process Management (BPM) face the same underlying problem, that may be called the integration problem. That is, in essence, a problem of inversion of control.

  - Will show how the traditional architecture that solves that problem, the state-transition system, end up in the creation of complex frameworks with excessive configurations, standards, APIs and development environments specialized for each particular problem that hides the commonalities and make the integration prohibitively expensive and difficult to program and maintain, while the language Haskell permits the implementation of a common underlying solution for all these problems that is vastly simpler and intuitive.
  
  - For a practical problem, you will learn how to program and test a long running task (a workflow) in a monadic sequence that will automatically checkpoint its own state so that it can be restarted, after intended or unintended shutdowns.
  
  - Will learn how to program long timeouts while waiting for the completion of some task, using STM transactions.
  
  - Will learn to create a web application that will start, monitor, restart and make use of the results of the previous task
  
  - Will learn how to use transactional persistent queues to communicate asynchronously between different tasks
  
  - Will learn how to  rollback a long running transaction

  - Will learn how to rollback transactions in a online Web application using the same primitives

  - All of this done with the same framework, making a practical demonstration of how  different problems such is the Web application and a running asynchronous workflow have the same underlying problems and the same underlying solution.
  

# The integration problem: Inversion of control
Creating a single application in the imperative style is easy and intuitive, because the programmer is in control of the sequence of things to do. But when it comes the time to integrate two or more autonomous entities that send events at any time in its own sequence then is when the programmer is not in control, so a different programming model is necessary. Such problem happens when trying to integrate the users with  backoffice applications via web applications,  but also when it is necessary to integrate two or more backoffice applications, company departments, web sites, web services etc.  In some of the cases is the inherent nature of the entities to integrate, that are really in control: For example, the user in a web application. In other is due to the loose coupling  of the integration as is the case of web services that may produce timeouts or other error conditions. A single element that produce an error or exception  can force the undo of many operations.

# The standard solution
The standard model that solves this inversion of control problem has various names but essentially is the same architecture with different names: finite state machine, state-transition system, a state machine system or a event handling model. That is the architecture of the main web frameworks, Enterprise Application Integration (EAI) frameworks, Orchestration frameworks, Workflow frameworks, Service Oriented Architecture (SOA) frameworks and Business Process Management (BPM) frameworks, that solve respectively, the individual above mentioned integration problems. 

## The problems of the standard solution

But state machines are difficult to program and debug. Almost every consideration about the goto statement applies to the event handling mechanism that manage state transitions. The event handlers do not share variable scopes, so, like in the case of the goto, the code rely in global variables, or in global session variables, when they are created dynamically. There is no top level structure in the code that may represent the main sequence. For the maintainer, it is very difficult to know what the code is trying to achieve. Just like it would happen when using the goto statement. 

### Declarative configurations make the problem grow
This lack of clarity and maintainability drives a tendency to standardization and to the use of declarative languages and configurations. A typical solution is the use of declarative languages such are XML to express the top level of the state changes, with different  degrees of sophistication: such is the case of routing in  web applications, or the page flow in the case of flow oriented web frameworks (such is the case of the *Seam* or the *Spring Web Flow*  frameworks). This solution is also typical in orchestration of integration scenarios such is the case of EAI, SOA or BPM for the same reasons. 

### Standards divide the integration problem
It is supposed that this declarative top level configuration, separated from the imperative code of the event/integration handlers makes things a little more readable and maintainable. But when the ambition of the declarative strategy grows it becomes almost equally or more complicated than the equivalent imperative code. Moreover, the code that implement the event handlers must be broken into small imperative pieces invoked by the declarative top level, since both are different languages. The change of something in the program involves the edition of many files, with magic identifiers that link this or that feature across the ensemble of files. 

### GUIs hide the problem
A further step, typical in machine state programming is to use a Graphical User Interface that will be in charge of handling the ensemble of configurations and imperative files by coordinating the multiple editions of variables, methods etc in different files when the user edit graphically an action in the flow. Many EAI, SOA, BPM, and Workflow frameworks have a graphical flow editing tool. And also in the case of some Web frameworks. At the same time the framework authors hope that the graphical interface will make evident the sequence that is not present in the code.

However that graphical solution carries out only the edition of the top level. there are a number of things to be done by hand in the configurations and in the code here and there. So the lower level can not be taken as a black box but must be documented for architecture programmers, administrators and so on. At this moment, the framework is a monster with hundred of documentation pages, wizards that help for many particulars cases -but not for your case- and arcane primitives, keywords, and configurations.

# A better solution
All that complication is a consequence of the low level of the machine state and his event model. Instead of a mix of an imperative language that define sequences and some declarative languages that define transitions between sequences, we need a single language that can define declarative sequences. Such declarative sequences describe explicitly the normal flow of execution in a intuitive imperative-like style, and the exception conditions appear in the code as exceptions to the main flow. 

But unlike imperative code, the declarative sequence may be read as a recipe by the program scheduler: Like in that case, the flow can be stopped and restarted again, it can be backtracked to rollback actions,  and it must  be capable of making transitions at any depth from a branch to another to respond to unexpected events or errors. That scheduler  of declarative sequences can be programmed in Haskell using Monad transformers with effects for application state logging and recovery, tracking and backtracking. With these effects, it is possible to run long running applications that integrate, orchestrate and dispatch out of order events, stop and restart and rollback actions in case of errors  in integration scenarios while at the same time are as simple and inexpensive to develop, understand and maintain as ordinary imperative applications.

## Workflow 
The library <hoogle>Workflow</hoogle>  executes processes that are restartable. A process in the workflow monad can be stopped and restarted without losing the execution state, so it can run for days, months or years. The stored state is not the one of a state machine but the execution log  in a sequential program. a workflow is normally an application that run for a lot of time, so checkpointing the state and to recover its state of execution after a  shutdown is one of the main characteristic of a workflow. This library uses STM to execute transactions and to implement timeouts. 

## TCache
The library <hoogle>tcache</hoogle> implement transactional <hoogle>STM</hoogle> variables with persistence in a backend defined by the user. STM can be used not only for concurrency but to manage complex combinations of events in a composable way. `tcache` also implement persistent queues that can be used for asynchronous communications with other workflows or processes. It provides file persistence by default for your data, so prototyping is very fast. workflow uses tcache to manage and persist its state in a coherent state with the rest of the data.

## MFlow
The <hoogle>mflow</hoogle> library is a web framework that uses the Workflow library. It takes web applications as workflow sequences. It add tracking and backtracking effects to respond to out of order user requests. The same backtracking mechanism can be used with <hoogle>Workflow</hoogle> to roll-back long running transactions, something that happens in many integration scenarios. We will see how to do it later in this tutorial.



# The example application
In this example we will show how a web application with some pages and an asynchronous workflow can be coded in an intuitive way in a few lines of Haskell code. 

A small bookseller sell special books by request to selected clients. The seller need to offer special services. For example if a book is not in stock, he will have the option to reserve it for a time as soon as it appears, let say  for five days. If the client do not buy it, the book will be unreserved. 

## How Workflow works
A simple example of a workflow using the <hoogle>Workflow</hoogle> package is as such:


```Haskell
exec1 "workflowname" $ do
   result <- step $ process1
   step $ process2 result
   ...
```

Where `process1`, `process2` etc are IO actions and <hoogle search="Control.Workflow.step">step</hoogle> is the lifting of the monad transformer that log the results of the execution and is involved in the recovery of the application state from the log created. <hoogle search="Control.Workflow.exec1">exec1`</hoogle> is the scheduler that start or restart the workflow. It is guaranteed that once the result of each IO process is logged by <hoogle search="Control.Workflow.step">step</hoogle>, the process is not executed again when it is restarted. It also can be restarted in another machine using the log, so it has a natural fallback mechanism in case of hardware failure.

Since a workflow, like any integration activity is the coordination or orchestration of autonomous entities such are persons and processes, most of the processes in a workflow, like in any integration problem, involves sending a request and wait for a response, or send a response and wait for the next request. In both cases what is received may not be the one expected and it may arrive late. Sometimes it is necessary to execute a different branch depending on different events. In the worst case, one of the autonomous elements integrated can force an undo of the task as a consequence of an error until some handler up in the flow manages the situation and restore the flow forward. Perhaps in another branch of the declarative sequence. At this moment you can see certain similarities with in a web application when the user forces going back in the navigation sequence and goes through another branch. 

## How TCache works
The Workflow library uses STM transactions as the main way to wait for external events. The package <hoogle search="Data.TCache">tcache</hoogle> is a transactional cache using STM, with persistence in different backends. The user define the persistence of his data by means of a class instance. 

First, a definition for the book metadata:

``` haskell
data Book= Book{btitle :: String, stock, reserved :: Int} 
           deriving (Read,Show, Eq,Typeable)
```

The library TCache defines STM variables called `DBRefs` that are persistent in a programmer-defined backend. Such persistence is defined by class instances. Apart from that, <hoogle search="Data.TCache">tcache</hoogle> references are like <hoogle>TVar</hoogle> references.


``` haskell
import Data.TCache.DefaultPersistence

instance Indexable Book where key= btitle

instance Serializable Book where
  serialize= pack. show
  deserialize= read . unpack
```

Here we defined a key for the `Book` register and a simple serialization instance defined in terms of `Read` and `Show`. By default this instance uses files for persistence, unless the <hoogle search="Data.TCache.Defs.setPersist">setPersist</hoogle> method is overridden in the <hoogle>Serializable</hoogle> instance. Later we will substitute it by a database persistence. But now for the first steps, lets use this simple persistence in files. 

``` haskell
keyBook= "booktitle" 

rbook= getDBRef $  keyBook
```
Here `rbook` is a TCache reference to the book of this title. That reference will point to a register in the cache and in the storage (or `Nothing`). The content can be read with <hoogle>readDBRef</hoogle> written with <hoogle>writeDBRef</hoogle> or deleted with <hoogle>delDBRef</hoogle> in the STM monad.

The key of the object can also be obtained from the reference by using <hoogle>keyObjDBRef</hoogle>.

Here below are the first lines of the workflow. That code wait for stock in the book for a time `timereserve`:

``` Haskell
buyReserve timereserve  keyBook= do
    let rbook = getDBRef keyBook
    logWF $  "Reserve workflow start for: "++ keyBook
    t <- getTimeoutFlag timereserve  

    r <- WF.step . atomically $ (reserveIt rbook >> return True)
                      `orElse`  (waitUntilSTM t >> return False)
```

Here the timeout is defined in <hoogle>getTimeoutFlag</hoogle>. The parameter is the timeout in seconds but it can be unlimited, for days,  months or years since it is an <hoogle>Integer<hoogle>. this timeout can wait beyond process stop and restart. If the timeout is in execution and the process is interrupted, when restarted the timeout takes into account the time during which the process was stopped. What it returns is a STM variable with `False` as its content. <hoogle>waitUntilSTM</hoogle> simply wait until the STM variable becomes `True` as result of the timeout. <hoogle>orElse</hoogle> and <hoogle>atomically</hoogle> are part of the standard STM semantics.  What the second statement does is to wait either for the reservation of the book, in which case will return `True` or timeout in which case it return `False`.  More on the `reserveIt` definition later.

The rest of the workflow get the result of the previous step and either log about the timeout condition or wait `timereserve` time until the reservation period finish or the product is bought using a similar STM transaction arrangement:

``` haskell
    if not r
     then do
       logWF "reservation period ended, no stock available"
       return ()

     else do
       logWF "The book entered in stock, reserved "
       t <- getTimeoutFlag timereserve 
       r <- WF.step . atomically $ (waitUntilSTM t >> return False)
                          `orElse` (testBought rbook >> return True)

       if r
        then do
          logWF "Book was bought at this time"
        else do
          logWF "Reserved for a time, but reserve period ended"
          WF.step . atomically $ unreserveIt rbook
          return ()
```

<hoogle>logWF</hoogle> add a message to the log, so the history of the execution of the process can be known by the log content. This trace will be presented to the user or the administrator.  `testBought` return when the user buy the book. If not bought, after the timeout, <hoogle>waitUntilSTM</hoogle> return `False` and the reserve is dropped.

Actually this kind of wait for a condition with timeout is defined in the package workflow as <hoogle>withTimeout</hoogle>:

``` haskell
withTimeout ∷ (MonadIO m, Typeable a, Serialize a)
            ⇒ Integer → STM a → Workflow m (Maybe a)
withTimeout time  f = do
  flag ←  getTimeoutFlag time
  step . liftIO . atomically $ (f >>= return  .  Just )
                      `orElse`
                      (waitUntilSTM flag  >> return  Nothing)

```
But the explicit STM expression is more general and it show how STM can be used for handling the scheduling of complex event combinations.


What is the definition of `reserve`,  `unreserve` ?


``` haskell
reserveIt rbook = do
   mr <- readDBRef rbook
   case mr of
     Nothing -> retry
     Just (Book t s r) -> writeDBRef rbook $ Book t (s-1)  (r+1)


unreserveIt rbook= do
   mr <- readDBRef rbook
   case mr of
     Nothing -> error "unreserveIt: where is the book register?"
     Just (Book t s r) -> writeDBRef rbook $ Book t (s+1)  (r-1)
```

Here `reserveIt` wait for the appearance of the book in stock. Otherwise it retries. <hoogle>retry</hoogle> is defined in the standard STM package. `unreserveIt` does the opposite. To make the code simpler, border cases are not considered. stocks and reservations in the register are supposed to be positive etc.

## Queues
What if we want to reserve it and send a mail to the user? That is a typical problem in workflows and EAI applications. To keep it in the STM monad and have the nice automatic benefits of transactionality we can use the transactional persistent queues of the package TCache: <hoogle> Data.Persistent.Collection</hoogle>

``` haskell
userMail= "user@mail.com"
mailQueue= "mailqueue"

reserveAndMailIt rbook=  do
   let qref = getQRef mailQueue
   pushSTM qref ( userMail :: String
                , "your book "++ keyObjDBRef rbook ++ " received" :: String
                , "Hello, your book...." :: String)
   reserveIt rbook
```

<hoogle>getQRef</hoogle> return the reference to a queue identifier string and <hoogle>pushSTM</hoogle> push a tuple with the recipient mail, the subject and the content. A different process can read it  (using <hoogle>popSTM</hoogle>) and send it through  mail using any of the haskell mail libraries. The write to the queue is transactional so any retry will not add duplicate registers to the queue.

Alternatively, for the execution of IO computations, for example, to send the mail instead of queuing it,  <hoogle>safeIOToSTM</hoogle> can be used. unlike <hoogle>unsafeIOToSTM</hoogle>, the former execute  the IO action completely even if the STM transaction is aborted and retried. However, the IO process invoked must be idempotent. That may involve the detection of duplicated executions since the STM transaction can be retried and the process invoked by `safeIOToSTM` can be re-executed. 

Sometimes it necessary the execution of various tasks that may take certain time, for example the invocation of  other workflow or even to ask some user. depending on the response, something done time ago may need to be undone. These [long running transactions](http://en.wikipedia.org/wiki/Long-running_transaction) usually are rolled back using complensations, and require more declarative configurations and imperative code in traditional solutions. Workflow can use the backtracking mechanism of the package <hoogle>MFlow</hoogle> to roll back such transactions almost transparently, without cluttering the expression of the main flow. That will be explained later.

To simulate the entry of stock, a process called by main wait for a certain amount of time and update the register:

``` haskell
enterStock delay rbook= forkIO $ do
   liftIO $ threadDelay $ delay * 1000000
   putStrLn "ENTER STOCK"
   atomically $ writeDBRef rbook $ Book "booktitle" 5  0

```

Finally `buy` is simulated also with a process that wait for some time. if the book is reserved it decrement the reservation. If not, it decrement the stock:

``` haskell
(!>)= flip trace

buy delay rbook=  forkIO $ do
  threadDelay $ delay * 1000000
  atomically $ do
   mr <- readDBRef rbook
   case mr of
     Nothing -> error "Not in stock"
     Just (Book t n n') ->
        if n' > 0  !> show mr then writeDBRef rbook $ Book t n (n'-1)
                       !> "There is in Stock and reserved, BOUGHT"
        else if n > 0 then
                      writeDBRef rbook $ Book t (n-1) 0
                       !> "No reserved, but stock available, BOUGHT"
        else error "buy: neither stock nor reserve"
```

In the workflow, the detection of the book being bought in the buy process is done checking if the reservation becomes 0. This only works for a single client, but I try to keep it as simple as possible.

``` haskell
testBought rbook= do
    mr <- readDBRef rbook
    case mr of
       Nothing -> retry !> ("testbought: the register does not exist: " ++ show rbook)
       Just (Book t stock reserve) ->
           case reserve  of
              0 -> return()
              n -> retry
```

Now `test` is a method that integrate all the elements  and can invoke the workflow with different timings:

``` haskell
test stockdelay buydelay timereserve stopdelay = do

  let keyBook= "booktitle"
      rbook= getDBRef  keyBook

  enterStock stockdelay rbook
  buy buydelay rbook
  th <- forkIO $ exec "buyreserve" (buyReserve  timereserve) keyBook
      
  stopRestart stopdelay timereserve th

  threadDelay $ (buydelay- stopdelay+1)  * 1000000
  putStrLn  "FINISHED"
  atomically $ delDBRef rbook
  putStrLn "----------------WORKFLOW HISTORY:--------------"
  h <- getHistory "buyreserve" keyBook
  putStrLn $ unlines h
  putStrLn "---------------END WORKFLOW HISTORY------------"
  delWF "buyreserve" keyBook
  
```

`stopRestart` kill the workflow and restart it to simulate a shutdown and a restart. As expected, nothing change with the shutdown and the restart. `getTimeoutFlag` take into account the time during which the process has been stopped.

``` haskell
stopRestart delay timereserve th=  do
    threadDelay $ delay * 1000000
    killThread th                              !> "workflow KILLED"
    syncCache  
    atomically flushAll                                      
    restartWorkflows ( fromList [("buyreserve", buyReserve timereserve)] ) 
                !> "workflow RESTARTED"
```

<hoogle search="Control.Workflow.restartWorkflows">restartWorkflows</hoogle> restart all the "buyreserve" workflows for all the book keys. these keys are all the book keys for which a WF has been started <hoogle search="Control.Workflow.exec"> exec<hoogle>. In this case there is only one key, `"booktitle"`, but if there were many workflows initiated for the reservation of different books, they would have been restarted. if the workflow has been already restarted, `restartWorkflow` does nothing. <hoogle>syncCache</hoogle> synchronize the cache with the storage and <hoogle>flushAll</hoogle> delete the cache.

Becase there is just a single workflow, instead of `restartWorkflows`  <hoogle search="Control.Workflow.exec"> exec<hoogle> could have been used to restar it again:

``` haskell
forkIO $ exec "buyreserve" (buyReserve  timereserve) keyBook
```

since <hoogle search="Control.Workflow.exec">exec</hoogle> recover the state and restart the workflow if it was already initiated.

The `getHistory` procedure read the workflow log and filter all the strings logged by the workflow. 

``` haskell
getHistory name x= liftIO $ do
   let wfname= keyWF name x
   let key= keyResource stat0{wfName=wfname}
   atomically $ flushKey key
   mh <- atomically . readDBRef . getDBRef $ key
   case mh of
      Nothing -> return ["No Log"]
      Just h  -> return  . catMaybes
                         . map eitherToMaybe
                         . map safeFromIDyn
                         $ versions h   ::  IO [String]
   where
   eitherToMaybe (Right r)= Just r
   eitherToMaybe (Left _) = Nothing
```

The elements of the log are stored as a list of <hoogle>IDynamic</hoogle> values. <hoogle>safeFromIDyn</hoogle> is used to filter all the values of type String. The log is retrieved as a DBRef register.


This console program test the whole code by injecting some interesting cases. We may have generated aleatory tests using <hoogle>quickcheck</hoogle>, but probably these interesting cases would have not been produced randomly:

** Warning ** Do not execute this snippet after a web snippet has been executed. Wait until it time out. Some background tasks of the Web example can interfere with this one.

``` haskell active
{-# LANGUAGE DeriveDataTypeable #-}

import Control.Workflow as WF
import Data.TCache
import Data.TCache.DefaultPersistence
import Control.Concurrent.STM
import Data.ByteString.Lazy.Char8(pack,unpack)
import Data.Typeable
import Control.Concurrent(forkIO,threadDelay, killThread)
import Control.Monad.IO.Class(liftIO)
import Control.Workflow.Stat
import Data.Maybe
import Data.Map (fromList)

import Debug.Trace
(!>)= flip trace

data Book= Book{btitle :: String, stock,reserved :: Int}
           deriving (Read,Show, Eq,Typeable)

instance Indexable Book where key= btitle

instance Serializable Book where
  serialize= pack. show
  deserialize= read . unpack

-- show
main= do

  putStrLn "\nFIRST CASE: the stock appears at 20 seconds.\n\
           \The WF is killed and restarted at 30 simulating \
           \a shutdown and restart.\n\
           \It is bought at 40.\n\
           \The reserve timeouts (at 50) is not reached.\n"
  test 20  40 50 30

  putStrLn "press any key to start the second case"
  getChar

  putStrLn "\nSECOND CASE: the stock appears at 20. \n\
           \It is killed at 10 simulating a shutdown \
           \and restart.\n\
           \It is bought at 60, after the end of the \
           \reserve (20+25)\n"
  test 20 60 25 10

  putStrLn "press a letter to start the third case"
  getChar

  putStrLn "\nTHIRD CASE: the product enter in stock at 25,\
           \n when the reservation period was finished.\n\
           \At 30 but the buyer appears shortly after and \
           \buy the product.\n\
           \At 15 the WF is killed to simulate a shutdown\n"
  test 25 30 20 15

  putStrLn "END"

-- /show

test stockdelay buydelay timereserve stopdelay = do
  let keyBook= "booktitle"
      rbook= getDBRef  keyBook

  enterStock stockdelay rbook

  buy buydelay rbook


  th <- forkIO $ exec "buyreserve" (buyReserve  timereserve) keyBook

  stopRestart stopdelay timereserve th

  threadDelay $ (buydelay- stopdelay+1) * 1000000
  putStrLn  "FINISHED"
  atomically $ delDBRef rbook
  putStrLn "----------------WORKFLOW HISTORY:--------------"
  h <- getHistory "buyreserve" keyBook
  putStrLn $ unlines h
  putStrLn "---------------END WORKFLOW HISTORY------------"
  delWF "buyreserve" keyBook




buyReserve timereserve  keyBook= do
    let rbook = getDBRef keyBook
    logWF $  "Reserve workflow start for: "++ keyBook
    t <- getTimeoutFlag timereserve  -- $ 5 * 24 * 60 * 60

    r <- WF.step . atomically $ (reserveIt rbook >> return True)
                      `orElse` (waitUntilSTM t >> return False)
    if not r
     then do
       logWF "reservation period ended, no stock available"
       return ()

     else do
       logWF "The book entered in stock, reserved "
       t <- getTimeoutFlag timereserve -- $ 5 * 24 *60 * 60
       r <- WF.step . atomically $ (waitUntilSTM t >> return False)
                          `orElse` (testBought rbook >> return True)

       if r
        then do
          logWF "Book was bought at this time"
        else do
          logWF "Reserved for a time, but reserve period ended"
          WF.step . atomically $ unreserveIt rbook
          return ()



reserveIt rbook = do
   mr <- readDBRef rbook
   case mr of
     Nothing -> retry
     Just (Book t s r) -> writeDBRef rbook $ Book t (s-1) (r+1)


unreserveIt rbook= do
   mr <- readDBRef rbook
   case mr of
     Nothing -> error "where is the book?"
     Just (Book t s r) -> writeDBRef rbook $ Book t (s+1) (r-1)

enterStock delay rbook= forkIO $ do
   liftIO $ threadDelay $ delay * 1000000
   putStrLn "ENTER STOCK"
   atomically $ writeDBRef rbook $ Book "booktitle" 5  0

buy delay rbook= forkIO $ do
  threadDelay $ delay * 1000000
  atomically $ do
   mr <- readDBRef rbook
   case mr of
     Nothing -> error "Not in stock"
     Just (Book t n n') ->
        if n' > 0 then writeDBRef rbook $ Book t n (n'-1)
                       !> "There is in Stock and reserved, BOUGHT"
        else if n > 0 then
                      writeDBRef rbook $ Book t (n-1) 0
                       !> "No reserved, but stock available, BOUGHT"
        else error "buy: neither stock nor reserve"

testBought rbook= do
    mr <- readDBRef rbook
    case mr of
       Nothing -> retry !>  ("testbought: the register does not exist: " 
                              ++ show rbook)
       Just (Book t stock reserve) ->
           case reserve  of
              0 -> return()
              n -> retry

stopRestart delay timereserve th=  do
    threadDelay $ delay * 1000000
    killThread th  !> "workflow KILLED"
    syncCache
    atomically flushAll
    restartWorkflows ( fromList [("buyreserve", buyReserve timereserve)] ) 
                   !> "workflow RESTARTED"

getHistory name x= liftIO $ do
   let wfname= keyWF name x
   let key= keyResource stat0{wfName=wfname}
   atomically $ flushKey key
   mh <- atomically . readDBRef . getDBRef $ key
   case mh of
      Nothing -> return ["No Log"]
      Just h  -> return  . catMaybes
                         . map eitherToMaybe
                         . map safeFromIDyn
                         $ versions h   :: IO [String]
   where
   eitherToMaybe (Right r)= Just r
   eitherToMaybe (Left _) = Nothing



```


Depending on the delays, any kind of combination is possible. This program is just in order to test the workflow and the subprocesses used in the final application. You can change the delays to add more cases and see what happens.


# Enter MFlow

The Web application will use the <hoogle>MFlow</hoogle> library. It permits the creation of applications with a structure very similar to a console applications, where the text inputs and outputs are substituted by web pages with type safe responses from links, forms and Ajax requests.

The navigation monad add effects of tracking, backtracking to the login and recovery of the workflow package. Then, the monadic code is read by the monad as  a "declarative sequence" in the sense explained above. The routing is expressed as conditionals statements depending on the REST path in the URL or as a result of a link clicked. Even if the application does not have to manage much state along the pages, to arrange the logic as sequences of pages  makes the development more intuitive and thus more maintainable in the same way than the workflow expressed as a sequence is more intuitive than when expressed under a state machine paradigm.


``` haskell active web

    {-# LANGUAGE   OverloadedStrings, DeriveDataTypeable , NoMonomorphismRestriction #-}
import MFlow.Wai.Blaze.Html.All hiding (footer, step, push)
import Control.Monad.State
import Data.Monoid
import Control.Applicative
import Control.Concurrent
import Control.Workflow as WF
import Control.Workflow.Stat
import Control.Concurrent.STM
import Data.Typeable
import Data.TCache.DefaultPersistence
import Data.Persistent.Collection
import Data.ByteString.Lazy.Char8(pack,unpack)
import Data.Map  as M (fromList)
import Data.List(isPrefixOf)
import Data.Maybe
import Debug.Trace
import System.IO.Unsafe
(!>) = flip trace



data Book= Book{btitle :: String, stock,reserved :: Int} deriving (Read,Show,Eq,Typeable)

instance Indexable Book where key= btitle

-- implicitly defined:
-- instance Serializable Book where
--   serialize= pack. show
--   deserialize= read . unpack

keyBook= "booktitle" :: String

rbook= getDBRef $  keyBook

stm= liftIO . atomically

-- show
reservetime= 5* 24 * 60 * 60  -- five days waiting for reserve and  five days reserved

data RouteOptions= Buy | Other | Reserve | NoReserve deriving (Typeable,Show)

main= do
 enterStock 30 rbook
 restartWorkflows $ M.fromList [("buyreserve",  buyReserve reservetime)]

 runNavigation "" . transientNav $ do
  op <-  page $ wlink Buy "buy or reserve the book" <++ br 
            <|> wlink Other "Do other things"
  case op of
   Other -> page $ "doing other things" ++> wlink () "home"
   Buy -> do
     reserved  <- stm $ do
             mr <- readDBRef rbook
             case mr of
               Nothing -> return False
               Just r  -> 
                 if reserved r > 0 then return True
                 else if stock r > 0 then reserveIt rbook >> return True
                 else return False
                 
     if reserved then page $ buyIt keyBook

     else  reserveOffline keyBook
         
-- /show

buyIt keyBook= do
      mh <- getHistory "buyreserve" keyBook

      p  "there is one book for you in stock " 
         ++> case mh of
              Nothing -> p "The book was in stock and reserved online right now"
              Just hist ->
                let histmarkup= mconcat[p << l | l <- hist]
                in  h2 "History of your reserve:"
                    <> histmarkup
         ++> wlink keyBook "buy?"
                 `waction` (\keyBook -> do
                     stm $ buy rbook
                     page $  "bought! " ++> wlink () "home"
                     delWF "buyreserve" keyBook)

reserveOffline keyBook = do
     v <- getState "buyreserve" (buyReserve reservetime) keyBook
     case v of
         Left AlreadyRunning -> lookReserve keyBook
         Left err -> error $ show err
         Right (name, f, stat) -> do
           r <- page $ wlink Reserve 
                        "not in stock. Press to reserve it when available in \
                        \the next five days. It will be reserved for five days "
                   <|> br
                   ++> wlink NoReserve "no thanks, go to home"
           case r of
             Reserve -> do

           
            
                  liftIO $ forkIO $ runWF1 name (buyReserve reservetime keyBook) stat True
                  return ()

             NoReserve -> return()


lookReserve keyBook= do
    hist <- getHistory "buyreserve" keyBook `onNothing ` return ["No workflow log"] 
    let histmarkup= mconcat[p << l | l <- hist]
    page $ do
        mr <-   stm   $ readDBRef rbook
        if mr== Nothing
              || fmap stock mr == Just 0
              && fmap reserved mr  == Just 0
           then 
             "Sorry, not available but you already demanded a reservation when the book\
                           \ enter in stock"
                   ++> wlink () << p "press here to go home if the book has not arrived"
                   <++ p "you can refresh or enter this url to verify availability"
                   <> h2 "status of your request for reservation upto now:"
                   <> histmarkup
           else 
           h2 "Good! things changed: the book arrived and was reserved" 
            ++> buyIt keyBook 


buyReserve timereserve  keyBook= do
    let rbook = getDBRef keyBook
    logWF $  "You requested the reserve for: "++ keyBook
    t <- getTimeoutFlag timereserve  -- $ 5 * 24 * 60 * 60

    r <- WF.step . atomically $ (reserveAndMailIt rbook >> return True)
                      `orElse` (waitUntilSTM t >> return False)
    if not r
     then do
       logWF "reservation period ended, no stock available"
       return ()

     else do
       logWF "The book entered in stock, reserved "
       t <- getTimeoutFlag timereserve -- $ 5 * 24 *60 * 60
       r <- WF.step . atomically $ (waitUntilSTM t >> return False)
                          `orElse` (testBought rbook >> return True)

       if r
        then do
          logWF "Book was bought at this time"
        else do
          logWF "Reserved for a time, but reserve period ended"
          WF.step . atomically $ unreserveIt rbook
          return ()

userMail= "user@mail.com"

mailQueue= "mailqueue"
 
reserveAndMailIt rbook=  do
   let qref = getQRef mailQueue
   pushSTM qref ( userMail :: String
                , "your book "++ keyObjDBRef rbook ++ " received" :: String
                , "Hello, your book...." :: String)
   reserveIt rbook

reserveIt rbook = do
   mr <- readDBRef rbook
   case mr of
     Nothing -> retry
     Just (Book t s  r) ->
       if s >0 then writeDBRef rbook $ Book t (s-1) (r+1)
               else retry


unreserveIt rbook= do
   mr <- readDBRef rbook
   case mr of
     Nothing -> error "unreserveIt: where is the book?"
     Just (Book t s r) ->
       if r >0 then writeDBRef rbook $ Book t (s+1) (r-1)
               else return()

enterStock delay rbook= forkIO $ loop enter
  where
  loop f= f >> loop f
  enter= do
   threadDelay $ delay * 1000000
   atomically $ do
     Book _ n  r <- readDBRef rbook `onNothing` return (Book keyBook 0 0)
     writeDBRef rbook $ Book "booktitle" (n +1) r
              !> "Added 1 more book to the stock"


buy rbook=  do
   mr <- readDBRef rbook
   case mr of
     Nothing -> error "Not in stock"
     Just (Book t n n') ->
        if n' > 0  !> show mr then writeDBRef rbook $ Book t n (n'-1)
                       !> "There is in Stock and reserved, BOUGHT"
        else if n > 0 then
                      writeDBRef rbook $ Book t (n-1) 0
                       !> "No reserved, but stock available, BOUGHT"
        else error "buy: neither stock nor reserve"

   
testBought rbook= do
    mr <- readDBRef rbook
    case mr of
       Nothing -> retry
                  !>  ("testbought: the register does not exist: " 
                       ++ show rbook)
       Just (Book t stock reserve) ->
           case reserve  of
              0 -> return()
              n -> retry

stopRestart delay timereserve th=  do
    threadDelay $ delay * 1000000
    killThread th  !> "workflow KILLED"
    syncCache
    atomically flushAll
    restartWorkflows ( fromList [("buyreserve", buyReserve timereserve)] ) 
                        !> "workflow RESTARTED"

getHistory name x= liftIO $ do
   let wfname= keyWF name x
   let key= keyResource stat0{wfName=wfname}
   atomically $ flushKey key
   mh <- atomically . readDBRef . getDBRef $ key
   case mh of
      Nothing -> return Nothing
      Just h  -> return  . Just
                         . catMaybes
                         . map eitherToMaybe
                         . map safeFromIDyn
                         $ versions h   ::  IO (Maybe [String])
   where
   eitherToMaybe (Right r)= Just r
   eitherToMaybe (Left _) = Nothing

```

The methods of the same name are the ones explained previously, but `enterStock` this time enter a new copy of the book in stock every 30 seconds. So if you buy the books faster than the stock is added, the offline reservation WF will be started and you will see the status of your reserve. if not, the Web App. will do it online. 

After each start <hoogle>restartWorkflows</hoogle> restart all the active workflows. That happens if the application is stopped and restarted. In this case there is only one book and one user for the sake of simplicity.

<hoogle>runNavigation </hoogle> execute a web navigation. 

See the [MFlow Introduction](https://www.fpcomplete.com/user/agocorona/MFlow-tutoria) for more information about how to use formatting, and how to use the applicative and monadic operatos used in the page and in the flow. For how state is managed and about the different kinds of possible web applications, see [MFlow, a DSL for web applications](https://www.fpcomplete.com/user/agocorona/MFlowDSL). 

A navigation is a sequence of pages with conditionals. When the sequence end, it is restarted from the beginning. 

The first page has a menu with two alternative links, with values defined in `RouteOptions`. The routes defined are`http://host/`  `http://host/noscript/buy` and `http://host/noscript/other`. they can be entered in the browser and the response will be the expected. That is because the navigation monad and <hoogle>page</hoogle> have a  mechanism that tracks the REST path and route the execution to the appropriate page.

If the user press the `Buy` option (or if it invokes the `http://host/noscript/buy` URL in the browser), the application check for the availability of the book. If there is stock available, it reserve it (in order to prevent race conditions with other users) and goes to the page where the user is informed about the availability, so he can buy it (explained later). If the book is not available, `reserveOffline` will notify the user of this condition and will present the option to reserve it when available:

``` haskell
reserveOffline keyBook = do
   v <- getState "buyreserve" (buyReserve reservetime) keyBook
   case v of
      Left AlreadyRunning -> lookReserve keyBook
      Left err -> error $ show err
      Right (name, f, stat) -> do
           r <- page $ wlink Reserve "not in stock. Press to reserve\
                \ it when available in the next five days. It will \
                \ be reserved for five days "
                   <|> br
                   ++> wlink NoReserve "no thanks, go to home"
           case r of
             Reserve -> do

                  liftIO $ forkIO $ runWF1 name 
                                    (buyReserve reservetime keyBook) 
                                    stat True
                  return ()

             NoReserve -> return()
```

Since `reserveOffline` will execute the reservation workflow, and may be called recurrently, every time the /buy verb is invoked and there is no stock, it is necessary to check if the WF has been executed already. This is done with <hoogle>getState</hoogle>. If it is not running, the user can choose to init the reservation WF or not. This time the WF is initiated with a more low level primitive, <hoogle>runWF1</hoogle> that uses the state returned by `getState`. Essentially <hoogle search="Control.Workflow.exec">exec</hoogle>, used in the previous example, perform a `getState` followed by a `runWF1`.

When the WF is already running, the user is informed about the state of things in the workflow with `lookReserve`.

``` haskell
lookReserve keyBook= do
    hist <- getHistory "buyreserve" keyBook 
              `onNothing ` return ["No reserve log"] 
    let histmarkup= mconcat[p << l | l <- hist]
    page $ do
        mr <-   stm   $ readDBRef rbook
        if mr== Nothing
              || fmap stock mr == Just 0
              && fmap reserved mr  == Just 0
           then 
            "Sorry, not available but you already demanded a reservation\ 
            \when the book would enter in stock"
              ++> p "you can refresh or enter this url to verify availability"
              ++> h2 "status of your request for reservation upto now:"
              ++> histmarkup
              ++> wlink () << p "press here to go home if the book do not arrive"

           else 
            h2 "Good! things changed: the book arrived and was reserved" 
              ++> buyIt keyBook 
```

`lookReserve` is one page where the Workflow history and the book register are checked. If there is no stock, the history of the WF up to the moment is presented. If there is stock, the `buyIt` content is presented. 

The first alternative has a link to the home page. But since the page code is executed again when the link is pressed, if in the meantime the book arrives and is reserved by the workflow, then the page will present the second alternative instead of returning with the value of the link pressed.  `runNavigation`would restart the navigation again and present the home page otherwise. 

Since the inspection of the reserve status is done within the page, the lookReserve page can be refreshed in the browser to detect the changes.

``` haskell
buyIt keyBook= do
      mh <- getHistory "buyreserve" keyBook
      p "there is one book for you in stock " 
         ++> case mh of
              Nothing -> p "The book was in stock and reserved online right now"
              Just hist ->
                let histmarkup= mconcat[p << l | l <- hist]
                in  h2 "History of your reserve:"
                    <> histmarkup
         ++> wlink ("buyit":: String) "buy?"
              `waction` const (do
                  stm $ buy rbook
                  page $  "bought! " ++> wlink () "home"
                  delWF "buyreserve" keyBook)
```

`buyIt` is not a page, but the content of a page. It is part of both the `lookReserve` and also is invoked in the `Buy` option as a separate page. It informs either if the reservation was made online by the web app or offline by the workflow, while the user was away. The user is informed about when it was reserved and implicitly, for how long the reserve will be kept if the user does not buy immediately. For the sake of simplicity, only the essential options are programmed.

The page show the reserve information and a single link with the "buy" option. If the link is pressed <hoogle search="MFlow.Forms.waction">waction</hoogle> executes a navigation that buy the book (using the same `buy` procedure above mentioned) and present a page with the "bought!" message and a link, after which the flow of execution return to the `buyIt` code, and BuyIt return the flow control to whatever may have called it. In this case, in both call locations to `buyIt` the flow finalizes, so the home menu will be presented again by `runNavigation`. `buyIt` is an example of reusable piece that can be inserted in whatever page of the flow.

With the exception of these useful peculiarities, the flow and structure of the application is pretty similar to a console application. Another peculiarity is that the  back button works because the backtracking mechanism implemented in the navigation monad goes back to the page that handle each request. 


# Long running transactions

the  navigation monad of MFlow is essentially a IO monad lifted with effects for backtracking that includes primitives for user interaction through a web interface. In fact the navigation monad can lift not only IO, but also the Workflow monad. Then, the navigation monad itself can implement a workflow and the backtracking mechanism can be used to undo things in case of failure in a workflow. It also gives to a workflow the capability to interact with the user. Or seen from the other side, in permits a web application to persist his session state thanks to the Workflow monad.

The backtracking is performed by <hoogle search="MFlow.Forms.Internals.Sup">Sup</hoogle> , an embedded monad. `Sup` comes from "Supervisor".

In the navigation monad it is possible to determine if a sentence has been called in backtracking mode or not with <hoogle>goingBack</hoogle>. A sentence is called back when backtracking if it uses <hoogle>breturn<hoogle> as the last executed sentence instead of `return` to yield something. The initiation of the backtracking is performed with the method `fail`. When a <hoogle>page</hoogle> find a request that can not handle it call `fail` and backtrack in the computation to find the appropriate page that handle it.

## Compensation for a long running transaction

A compensable action in the navigation monad can be coded as:

``` haskell
compensate :: Monad m => FlowM v m a -> FlowM v m a -> FlowM v m a
compensate doit undoit= do
     back <- goingBack
     case  back of
        False -> doit >>= breturn
        True  -> undoit
```

When going forward, `compensate` (that can be used in infix mode) execute `doit` and mark itself to be called again when backtracking, because it `breturn` the result of `doit`.

When backtraking, `undoit` is executed. But that is not all.  Again, depending on if `undoit` uses <hoogle>breturn</hoogle>  or `fail` as the last sentence, the flow will continue forward or backward. Depending on that `undoit` can be a fix in order to continue the sequence forward, or a true compensation that, after done, leave the flow going back to undo further actions or to find a fix. In the first case, if the flow reach the top, <hoogle>runFlowOnce</hoogle> will restart the workflow again.

`undoit` can check the session information to decide either to compensate and fail back or fix the problem and continue. If the flow is online such is in the case of a web application during a complex operation, for example, a payment, it can even ask the user  what to do. Seen this way, a workflow with exception handling for long running transactions can be equivalent to a navigation.

Lets codify the workflow that we tested before in the flow monad. Now the `unreserveIt` will be carried out as a compensation of the `reserveAndMaiIt`. There is a first compensate element at the beginning that abort the computation if the backtracking reach this statement, to avoid runFlowOnce to re-schedule of the flow. Also since the Flow monad is a lifting of the Workflow monad, it has to `lift` the workflow statements. Otherwise the flow is the same: 

## Same workflow with compensation
``` haskell
withTimeoutIO flag f  = liftIO $ atomically $ (f  >> return True)
                    `orElse` (waitUntilSTM flag >> return False)

buyReserve timereserve  keyBook= do runFlowOnce f (error "token not found") where
 f :: FlowM Html (Workflow IO) ()
 f= do
    compensate (return()) $ do
        lift $ logWF $ "aborting"
        error "aborted"
        
    let rbook = getDBRef keyBook
    lift . logWF $  "You requested the reserve for: "++ keyBook
    
    t <- lift $ getTimeoutFlag timereserve  -- $ 5 * 24 * 60 * 60

    r <- compensate (step . withTimeoutIO t $ reserveIt rbook)
                     (do
                       lift $ logWF "Unreserving the book"
                       step $  (liftIO . atomically $ unreserveIt rbook) >> fail "")

--     liftIO $ atomically $ (reserveIt rbook >> return True)
--                    `orElse` (waitUntilSTM t >> return False)
    if not r
     then do
       lift $ logWF "reservation period ended, no stock available"
       return ()

     else do
       lift $ logWF "The book entered in stock, reserved "
       t <- lift $ getTimeoutFlag timereserve -- $ 5 * 24 *60 * 60
       r <- step . liftIO $ atomically $ (waitUntilSTM t >> return False)
                          `orElse` (testBought rbook >> return True)

       if r
        then do
          lift $ logWF "Book was bought at this time"
        else do
          lift $ logWF "Reserved for a time, but reserve period ended"
          fail ""

--        now it is compensated above
--        step . liftIO $ atomically $ unreserveIt rbook


```


Now, when the reserve time is finished and the book is not bought, the `fail` method fires a backtracking that call back the compensate statements in reverse order. in the second compensate statement, the event is logged, `unreserveIt` is executed and the flow continue back,  to the first compensate that log something and abort the computation.

Remember that this process can run for days waiting for stock.

We have packaged the reserve and unreserve in a single line, so we can forget in the rest of the workflow about how to undo the reservation. We simply call `fail` and all the compensable transactions will be undone. At the same time, we see the normal flow of execution clearly expressed in the code. Additionally, the rollback is also restartable because its state is also logged since the rollback actions use `step`. The rollback is itself a long running process.

Note that backtracking yield the control to previous statements within the sequence (the ones marked with breturn), while a thrown exception would yield control up to the caller of the sequence.

This is a test program that executes this last workflow, add a book to the stock, and wait for the second timeout. The compensation actions are executed and finally it present the execution log of the flow.


** Warning ** Do not execute an snippet after the web snippet has been executed. Wait until the web app time out. Some background tasks of the Web example can interfere with this one. 

``` haskell active

{-# LANGUAGE DeriveDataTypeable #-}

import MFlow.Wai.Blaze.Html.All hiding (footer, push)
import Control.Monad.State
import Data.Monoid
import Control.Applicative
import Control.Concurrent
import Control.Workflow as WF hiding(step)
import Control.Workflow.Stat
import Control.Concurrent.STM
import Data.Typeable
import Data.TCache.DefaultPersistence
import Data.Persistent.Collection
import Data.ByteString.Lazy.Char8(pack,unpack)
import Data.Map  as M (fromList)
import Data.Maybe
import Debug.Trace
import System.IO.Unsafe
(!>) = flip trace

data Book= Book{btitle :: String, stock,reserved :: Int}
           deriving (Read,Show, Eq,Typeable)

instance Indexable Book where key= btitle

-- instance Serializable Book where
--   serialize= pack. show
--   deserialize= read . unpack

keyBook= "booktitle" :: String

rbook= getDBRef $  keyBook :: DBRef Book

-- show
main= do
  enterStock 10 rbook
  forkIO $ exec "buyreserve" (buyReserve  30) keyBook

  threadDelay 45000000
  
  putStrLn  "FINISHED"
  putStrLn "----------------WORKFLOW HISTORY:--------------"
  h <- getHistory "buyreserve" keyBook
  putStrLn $ unlines h
  putStrLn "---------------END WORKFLOW HISTORY------------"
  delWF "buyreserve" keyBook
  atomically $ delDBRef rbook


compensate :: Monad m => FlowM v m a -> FlowM v m a -> FlowM v m a
compensate doit undoit= do
     back <- goingBack
     case  back of
        False -> doit >>= breturn
        True  -> undoit

withTimeoutIO flag f  = liftIO $ atomically $ (f  >> return True)
                    `orElse` (waitUntilSTM flag >> return False)

buyReserve timereserve  keyBook= do runFlowOnce f (error "token not found") where
 f :: FlowM Html (Workflow IO) ()
 f= do
 
    compensate (return()) $ do   
        lift . logWF $ "Aborting"
        error "Aborted"

    let rbook = getDBRef keyBook
    lift . logWF $  "You requested the reserve for: "++ keyBook
    
    t <- lift $ getTimeoutFlag timereserve  -- $ 5 * 24 * 60 * 60

    r <- compensate (step . withTimeoutIO t $ reserveIt rbook)
                     (do
                       lift $ logWF "Unreserving the book"
                       step $  (liftIO . atomically $ unreserveIt rbook) >> fail "")

    if not r
     then do
       lift $ logWF "reservation period ended, no stock available"
       return ()

     else do
       lift $ logWF "The book entered in stock, reserved "
       t <- lift $ getTimeoutFlag timereserve -- $ 5 * 24 *60 * 60
       r <- step . liftIO $ atomically $ (waitUntilSTM t >> return False)
                          `orElse` (testBought rbook >> return True)

       if r
        then do
          lift $ logWF "Book was bought at this time"
        else do
          lift $ logWF "Reserved for a time, but reserve period ended"
          fail ""


-- /show

reserveIt rbook = do
   mr <- readDBRef rbook
   case mr of
     Nothing -> retry
     Just (Book t s  r) ->
       if s >0 then writeDBRef rbook $ Book t (s-1) (r+1)
               else retry


unreserveIt rbook= do
   mr <- readDBRef rbook  !> "UNRESERVE"
   case mr of
     Nothing -> error "unreserveIt: where is the book?"
     Just (Book t s r) ->
       if r >0 then writeDBRef rbook $ Book t (s+1) (r-1) 
               else return() 

enterStock delay rbook= forkIO $ do
   liftIO $ threadDelay $ delay * 1000000
   putStrLn "ENTER STOCK"
   atomically $ writeDBRef rbook $ Book "booktitle" 5  0

buy delay rbook= forkIO $ do
  threadDelay $ delay * 1000000 
  atomically $ do
   mr <- readDBRef rbook
   case mr of
     Nothing -> error "Not in stock"
     Just (Book t n n') ->
        if n' > 0 then writeDBRef rbook $ Book t n (n'-1)
                       !> "There is in Stock and reserved, BOUGHT"
        else if n > 0 then
                      writeDBRef rbook $ Book t (n-1) 0
                       !> "No reserved, but stock available, BOUGHT"
        else error "buy: neither stock nor reserve"

testBought rbook= do
    mr <- readDBRef rbook
    case mr of
       Nothing -> retry    !>  ("testbought: the register does not exist: " ++ show rbook)
       Just (Book t stock reserve) ->
           case reserve  of
              0 -> return()
              n -> retry

stopRestart delay timereserve th=  do
    threadDelay $ delay * 1000000
    killThread th  !> "workflow KILLED"
    syncCache
    atomically flushAll
    restartWorkflows ( fromList [("buyreserve", buyReserve timereserve)] ) !> "workflow RESTARTED"

getHistory name x= liftIO $ do
   let wfname= keyWF name x
   let key= keyResource stat0{wfName=wfname}
   atomically $ flushKey key
   mh <- atomically . readDBRef . getDBRef $ key
   case mh of
      Nothing -> return ["No Log"]
      Just h  -> return  . catMaybes
                         . map eitherToMaybe
                         . map safeFromIDyn
                         $ versions h   :: IO [String]
   where
   eitherToMaybe (Right r)= Just r
   eitherToMaybe (Left _) = Nothing

```

## Travel back in time
I said that the `compensate` mechanism besides undoing, it can also  fix the problem and continue the flow forward. To realize the fix,  the code may need more specific information about the cause of the `fail`. Suppose that the fail may be the result of a timeout or the consequence of a fail on the approval of a document. the the compensate undo may need to know the cause:


``` haskell
data conditions = Timeout |  Reject Causes | Approbal

...
document <-  edit `compensate`  handleConditions

r <- withTimeout  t approbal 
case r of
   Nothing -> setSessionData Timeout
   Just (reject@Reject _) -> setSessionData $ reject
   Just other -> forward other
...
   
handleConditions= do
     r <- getSessionData
     case r of
       Timeout ->….
       Reject cause -> do
                newDoc <- edit
                breturn newDoc
```

In the code above, `handleConditions` will receive rejections and timeouts. In the first case , the user edit a new document and send it back for approval. The info about the rejection travel  back using <hoogle>setSessionData</hoogle> and <hoogle>getSessionData</hoogle>. 

Note that in a traditional workflow framework, that iteration would need a separate loop, because a plain compensation can not do that, while here the iteration  is part of the main sequence. So the code is more concise. You see how the workflow becomes more close to a navigation back and forth within the main sequence.

# Solving rollback in the Web code
But wait! the web application can show to the user the buy link, but if he does not click it, but instead, it press the back button, the register stay reserved!. It is necessary to detect that condition and unreserve tje book.

That can be done with a single line using `compensate`:

``` haskell

main= do
 enterStock 30 rbook
 restartWorkflows $ M.fromList [("buyreserve",  buyReserve reservetime)]

 runNavigation "" . transientNav $ do
  op <-  page $ wlink Buy "buy or reserve the book" <++ br <|> wlink Other "Do other things"
  case op of
   Other -> page $ "doing other things" ++> wlink () "home"
   Buy -> do
     reserved  <- stm (do
         mr <- readDBRef rbook !> "RESERVING"
         case mr of
           Nothing -> return False
           Just r  ->
             if reserved r > 0 then return True
             else if stock r > 0 then reserveIt rbook >> return True
             else return False)
              
  {-hi-}`compensate` stm (unreserveIt rbook) >> fail "" {-/hi-} 
       
     if reserved then do
                     page $ buyIt keyBook 
                     return() !> "buyit forward"
                  

     else  reserveOffline keyBook
     
```

When the back button is pressed and the user see the home page, when he press some link of this page, then `page` detect the desynchronization, the navigation backtrack, and the undo part of compensate is executed, which unreserve the book and continue backtracking to the previous page, the home page, that handles the request. if the reservation has been done asynchronously, `testBought` will detect the unreserve and will finish the workflow as if the user would have bought the book.

We have solved the same problem in the Web app with the same code used in the long running task. The web app is not a long running transaction since <hoogle>step</hoogle> is not used, so the effect of logging and recovery is not available in the web application. However it can be used, if the web app need it. (I will show it in future articles) 

# Defining persistence

TCache registers persist in files by default. The folder `.tcachedata` will contain the files and folders of the book register and the workflow log. 

Of course it is possible to access a database directly but this does not permit the convenience of composable STM transactions. To permit tcache to read and write to a database for a certain data type, it is a matter of defining a <hoogle search="Data.TCache.Def.setPersist">setPersist</hoogle> in the in the <hoogle>Serializable</hoogle> instance. That is for key-value databases.  The package <hoogle> tcache-AWS</hoogle> uses this method.

A more general mechanism for TCache persistence is to define an instance of the<hoogle search="Data.TCache.IResource">IResource</hoogle> class. It defines the key of the register and how to read, write and delete a register. the package [tcache-persist](https://github.com/agocorona/tcache-persistent) shows an instance of IResource over datatypes defined with the package <hoogle>persistent</hoogle>. 

mflow and workflow work synchronously by default, that means that `step` forces a synchronization of the cache with the persistent storages (there may be many of them) by writing only the modified registers. To change this behaviour, use <hoogle>syncWrite</hoogle>. When writing to a database, with <hoogle>setConditions</hoogle> the programmer configure two procedures to be called before and after the synchronization, for example a commit, if the programmer does not want to use autocommit.

# Conclusion

We have seen how to create an  asynchronous task, with long running transactions and a web application using the same framework, by exploiting the commonalities that the two kinds of applications have. The code is more clean, maintainable and concise than traditional solutions. 

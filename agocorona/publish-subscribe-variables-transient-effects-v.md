# Publish-subscribe Variables. Transient effects V

The Transient monad propagate the effects of the appearance of events downstream. But what happens when I need to notify that something happened to a computation that is in another branch of the execution tree?  


Since I don´t want to break my code with callbacks, I can  do it with TVars, but this new kind of vars, called event vars (EVars) or pubish-subscribe vars have "last in-first out" semantics and can be combined like the rest of the Transient code. Morever the code continues normally after the `writeEvar` statement when all the subscribed actions have been executed. There is no breaking of the flow, and contrary to STM, the execution is deterministic: All the subscribed actions are executed in a single thread with last-in-first-out preferences.

As you could see below, the siminarities and differences with STM does not end there.


This is an example snippet which uses an EVar:

```haskell
main=  keep $ do
  option "pubs" "an example of publish-subscribe using Event Vars (EVars)"
  v <- newEVar  :: TransIO (EVar String)
  susbcribe v <|> publish v

  where 

  publish :: String -> TransIO ()
  publish v= do
    liftIO $ putStrLn "Enter a message to publish"
    msg <- input(const True)
    writeEVar v msg
    liftIO $ putStrLn "after writing the EVar"

  susbcribe :: EVar String -> TransIO ()
  susbcribe v= proc1 v  <|>  (proc2 v)

  proc1 :: EVar String -> TransIO ()
  proc1 v=  do
    msg <- readEVar v 
    liftIO $ putStrLn $  "proc1 readed var: " ++ show msg

  proc2 ::  EVar String -> TransIO () 
  proc2 v= do
    msg <- readEVar v 
    liftIO $ putStrLn $ "proc2 readed var: " ++ show msg
```

The output is

```
Enter a message to publish
> aaaaa
proc2 readed var: "aaaaa"
proc1 readed var: "aaaaa"
after writing the EVar
```

`publish`waits for a text input (`option`) and then it updates the EVar. Inmediately `proc2` and then `proc1` are executed since they invoked `readEVar` of this variable.

Each time that `writeEVar`is executed, the process is reproduced again. It is not necessary to invoke `readEVar` again. 

On the contrary, if your code invoke `readEVar` multiple times, since each continuation is potentially different, the effect is that a new reference to the continuation will be added, so it will be called multiple times.

To avoid this effect, call `unsuscribe var` after `readEVar var` to avoid such duplication. `unsuscribe` removes the current continuation from the list.

Moreover it is possible to combine two or more EVars:

```haskell

    do
        (x,y) <- (,) <$> readEVar v1 <*> readEVar v2
        if x > y 
          then stop
          else do
                .....
```
As you can see, EVars, like TVars, do not block. That is why both are composable.

This look like STM but it is not: when the first event is produced, it waits for the second, while in STM there is no waiting, since a STM var (TVar) ever is filled with a value. 

There is no transaction here, but I guess that it is possible to implement the TVar semantics with EVars.

`stop` is a Transient primitive that interrupts the execution. 

The next update of the variables will trigger the code again. 

In the last example, the order of the events is indiferent, but in the example below, the order is honored:


```haskell
   do
    r1 <- readEVar v1
    r2 <- readEVar v2
    ....

```
This could be useful is some cases.


The example above can be executed online. Ii is in the [the executable snippet in this article](https://www.fpcomplete.com/user/agocorona/moving-haskell-processes-between-nodes-transient-effects-iv#the-example-program--how-to-run-it) . Is the last option of the menu.

The implementation of the EVars is at the [git repository](https://github.com/agocorona/transient/blob/master/src/Transient/EVars.hs) of transient.

It is very concise thanks to the magic of Transient continuations: basically `readEVar` stores in the state the list of subscribed continuations for each variable. When the variable is written `writeEVar`itself extract the list of continuations from the state and execute them before resuming to normal execution.




AMDG
# Nearly Effortless Monad Transformers for Pipes, Conduit and More

# Effortless Monad Transformers for Pipes, Conduit and more

One of the great features provided by Pipes is the ability to utilize monad transformers in isolated sections of a pipeline. First consider an element counter pipe that sends the number of elements it has recieved downstream for every element recieved form upstream. Using the state monad transformer is a natural way to approach this problem, like the following:

```haskell

counter = go
  where
    go = do
        x <- await
        lift $ S.modify (+1)
        r <- lift S.get
        yield r
        go

> (`S.evalStateT` 0) . runEffect $ each "hi there" >-> counter >-> P.print
1
2
3
4
5
6
7
8
```

Combining several independant counters or several pipes unilizing diverse monad transformers into a single pipeline is problematic however.

* Every section of pipeline can get access to the state. The state is global to the pipeline
* Combining several necessitates the need for nesting `hoist lift` in ever more complexity.

To solve these issues `Pipes.Lift` from the main pipes package provides facilities to isolate the most common monad transformers to sections of a pipeline with `runStateP`, `runMaybeP`, ...

```haskell
> import qualified Pipes.Lift as PL
> let localCounter = PL.evalStateP 0 counter
> runEffect $ each "hi there" >-> localCounter >-> P.print
1
2
3
4
5
6
7
8
```

The `Pipes.Lift` facilities solve the above problems in isolation then composed together without inducing any overhead.

The current implementation of the facilities in `Pipes.Lift` requires touch the internal representation of both the `Pipe`s type and the monad transformer in question. Below I will demonstrate how to simply implement most monad transformers for pipes with out touch either the internal representation of the `Pipe` type or that of the monad transformer. This make it easy to add existing monad transform functionality to any existing pipeline.

Example monad transformers that this applies to are:

* [Transfomers](http://hackage.haskell.org/package/transformers)
    * ErrorT
    * MaybeT
    * WriterT
    * ReaderT
    * StateT
    * RWST
    * [EitherT](http://beta.hackage.haskell.org/package/either)
* [MonadRandom](http://hackage.haskell.org/package/MonadRandom)
* And more.

Any monad that has a [MFunctor](http://hackage.haskell.org/package/mmorph-1.0.0/docs/Control-Monad-Morph.html#t:MFunctor) form the [mmorph](http://hackage.haskell.org/package/mmorph) package instance works flawlessly and composing seemlessly with other such monads.

Other monads that can not have a MFunctor instance, or just do not, can be used as well but they do not automatically compose and a little extra care is needed when writing streaming facilities using them.

* ContT - From [Transfomers](http://hackage.haskell.org/package/transformers). An indexed/paramaterized ContT has a MFunctor instnace however.
* [ST Monad Transformer](http://hackage.haskell.org/package/STMonadTrans) - A type guarantee that the STRefs are not escaping down the pipeline is required, this can be as simple as specifing a concrete type like `Int`.
* [Monad-logger](http://hackage.haskell.org/package/monad-logger) - I do not know if a MFunctor can exist for this monad.
* I am sure many more.

I will be focusing on monad transformers that have a MFunctor instance for this article. I will be posting examples of both types soon, you can get in contact with me at Patrick dot John dot Wheeler at gmail dot com if you would like a few rough exmaples sooner.

Here are some example implementations explanations will follow below:

```haskell

runStateP s = (`evalStateT` s) . runSubPipeT

runErrorP   = E.runErrorT . runSubPipeT

runMaybeP   = M.runMaybeT . runSubPipeT

runRandP    = Rand.runRandT . runSubPipeT

-- Monads with out MFunctor instances

runContP mf = (`runContT` mf) . runSubPipeT'

-- log debug information to stdout.
runStdoutLoggerP = runStdoutLoggingT . runSubPipeT'

-- Conduit example
> CL.sourceList [1..5] $= evalStateC 0 counterC  $$ sinkPrint 
1
2
3
4
5

```

Similar implementations exist for bidirectional `Proxy`s and at the end of the day the unidirectional `runStateP` and friends above are just specialization's from the bidirectional compatible forms as will be shown later.

The outline of the technique is to have a section of pipeline `await` from two monad layers lower then itself, `catFromLifted`, and `yield` back to a monad level two lower then itself, `catToLifted`, when the section needing the monad is done.   

``` haskell
{-
catFromLifted = go
  where
    go = do
        x <- lift . lift $ await
        yield x
        go
-}
catFromLifted = (lift . lift . request) >\\ cat

{-
catToLifted = go
  where
    go = do
        x <- await
        lift . lift $ yield x
        go
-}
catToLifted = cat //> (lift . lift . respond)
```

Lets look at the type signature for catFromLifted a little closer.

``` haskell
catFromLifted
  :: (Monad (t (Proxy () y y2 y1 m)), Monad m, MonadTrans t) =>
     Proxy a1 a () y (t (Proxy () y y2 y1 m)) c
```     
(note: I am not clear why this section ends up in a code block, let me know, if you do)

The `t` monad transformer layer is where the monad transformer will end up being run.

Lets look at an example using the same `counter` pipe used above:

``` haskell

counter = go
  where
    go = do
        await -- discard upstream values
        s <- lift get
        lift $ modify (+1)
        yield s
        go

localState = (`evalStateT` 0) . runEffect $ catFromLifted >-> hoist (hoist lift) counter >-> catToLifted


mainPipeLIne = runEffect $ each "hi" >-> localState >-> P.print

```

generalizing this some we have:

``` haskell

runSubPipeT p = runEffect $ catFromLifted >-> hoist (hoist lift) p >-> catToLifted


-- runStateP s p = (`evalStateT` s) (runSubPipeT p)

runStateP s = (`evalStateT` s) . runSubPipeT

```

Compare the above `runStateP` to the this is the current implementation in `Pipes.Lift`.

``` haskell

runStateP
    :: (Monad m)
    => s -> Proxy a' a b' b (S.StateT s m) r -> Proxy a' a b' b m (r, s)
runStateP = go
  where
    go s p = case p of
        Request a' fa  -> Request a' (\a  -> go s (fa  a ))
        Respond b  fb' -> Respond b  (\b' -> go s (fb' b'))
        Pure    r      -> Pure (r, s)
        M          m   -> M (do
            (p', s') <- S.runStateT m s
            return (go s' p') )
            
```

As shown above this approach generalizes to all of the core transformers monads and more besides.

## Bidirectionality

This approach easily generalizes to bidirectional proxies. `await` becomes `request`, `yield` becomes `respond`, `cat` becomes `pull`.  You will noticed that there is some extra function composition with `.` this is to work the functions around the the initial argument to `pull :: Monad m => a' -> Proxy a' a a' a m r`.

``` haskell

pullFromLifted = ((lift . lift . request) >\\) . pull

pullToLifted =  (//> (lift . lift . respond)) . pull

-- Wraps a proxy so it requests and resonds to a proxy two levels lower then itself
fromToLiftedB p = (//> (lift . lift . respond)) 
                . ((lift . lift .request) >\\)
                . hoist (hoist lift) 
                . p

runSubPipeTB =  (runEffect' .) . fromToLiftedB

runErrorPB    = (E.runErrorT .) . runSubPipeTB

runMaybePB    = (M.runMaybeT .)  . runSubPipeTB

runReaderPB r = ((`R.runReaderT` r) .) . runSubPipeTB

runWriterPB   = (W.runWriterT .) . runSubPipeTB

runStatePB  s = ((`S.runStateT` s) .) . runSubPipeTB

```

The approach in Pipes 4.0.0 focuses on unidirectional code but the base data type works with bidirectional data streams. So it is convenient to be able to take a bidirectional compatible function and turn it into a function compatible with the unidirectional `Pipe`. We can do this with only a few additional functions.

``` haskell
 
specialize p = p ()
generalize f = (\_ -> f)

-- | This takes a one arrity function on a Proxies and turns it into a function 
-- on Pipes.
directionalize f = specialize . f . generalize

```

With directionalize it is straight forward to produce the unidirectional variants form the bidirectional functions.


``` haskell

runErrorP    = directionalize runErrorPB

runMaybeP    = directionalize runMaybePB

runWriterP   = directionalize runWriterPB

runStateP    = directionalize . runStatePB

runRWSP      = (directionalize .) . runRWSPB

```


## Compare Core

Lets investigate that the new internal unaware implementation does not adversely effect the end result by looking at the core representation of a simple program.

``` haskell

{-# language NoMonomorphismRestriction #-}

module Main where 
          
import Pipes
import qualified Pipes.Prelude as P
         
import Lift

import qualified Pipes.Lift as PL


standardState = PL.evalStateP 0 counter 
    
newState = evalStateP 0 counter

testStdFunc = runEffect $ each "hi" >-> standardState >-> P.print
    
testnewFunc = runEffect $ each "hi" >-> newState >-> P.print
    
main = print "done"

```


core:

``` haskell

Result size of Tidy Core = {terms: 88, types: 182, coercions: 0}

main
main = print ($fShow[] $fShowChar) (unpackCString# "done")

newState
newState =
  \ @ a_a36u @ a1_a36v @ r_a36w @ m_a36x $dMonad_a36y $dNum_a36z ->
    evalStateP'
      $dMonad_a36y
      (fromInteger $dNum_a36z (__integer 0))
      (counter ($fMonadProxy $dMonad_a36y) $dNum_a36z)

testnewFunc
testnewFunc =
  \ @ m_a3c1 $dMonadIO_a3c2 ->
    let {
      $dMonad_a36I
      $dMonad_a36I = $p1MonadIO $dMonadIO_a3c2 } in
    $ (runEffect $dMonad_a36I)
      (>->
         $dMonad_a36I
         (>->
            $dMonad_a36I
            (each $dMonad_a36I $fFoldable[] (unpackCString# "hi"))
            (newState $dMonad_a36I $fNumInteger))
         (print $dMonadIO_a3c2 $fShowInteger))

standardState
standardState =
  \ @ s_a3cw @ a_a3cx @ m_a3cy @ r_a3cz $dMonad_a3cA $dNum_a3cB ->
    evalStateP
      $dMonad_a3cA
      (fromInteger $dNum_a3cB (__integer 0))
      (counter $dMonad_a3cA $dNum_a3cB)

testStdFunc
testStdFunc =
  \ @ m_a3d1 $dMonadIO_a3d2 ->
    let {
      $dMonad_a3cK
      $dMonad_a3cK = $p1MonadIO $dMonadIO_a3d2 } in
    $ (runEffect $dMonad_a3cK)
      (>->
         $dMonad_a3cK
         (>->
            $dMonad_a3cK
            (each $dMonad_a3cK $fFoldable[] (unpackCString# "hi"))
            (standardState $dMonad_a3cK $fNumInteger))
         (print $dMonadIO_a3d2 $fShowInteger))

main
main = runMainIO main

```

As can be seen from the core, `standardState` using `evalState` from `Pipes.Lift` is identical in all important respects to `newState` using the new `evalStateP`. At least for the simple cases both the method presented here and those currently in `Pipes.Lift` produces the same core representation.


## Conduit and Oother Streaming Libraries

It seems likely that conduit and other streaming libaries can benefit of from the same method.  Below I include a rough implmentation for conduit and the state monad transformer.

```active haskell
{-# language NoMonomorphismRestriction #-}

import Data.Conduit
import qualified Data.Conduit.List as CL

import Control.Monad.Morph
import Control.Monad.Trans
import qualified Control.Monad.Trans.State as S


catFromLifted = go
  where
    go = do
        x <- lift . lift $ await
        case x of
            Nothing -> return ()
            Just x -> do
                yield x
                go
catToLifted = go
  where
    go = do
        x <- lift . lift $ await
        case x of
            Nothing -> return ()
            Just x -> do
                lift . lift $ yield x
                go

counterC = go
  where
    go = do
        x0 <-  await
        case x0 of
            Nothing -> return ()
            Just x -> do
                lift $ S.modify (+1)
                r <- lift S.get
                yield r
                go
                
sinkPrint = CL.mapM_ print

runSubConduit p = catFromLifted $= hoist (hoist lift) p $$ catToLifted

evalStateC s = (`S.evalStateT` s) . runSubConduit

main = CL.sourceList [1..5] $= evalStateC 0 counterC  $$ sinkPrint 

{-
> CL.sourceList [1..5] $= evalStateC 0 counterC  $$ sinkPrint 
1
2
3
4
5
-}
```

## Conclusion

Hopefully the above makes it easier for everyone to incorporate a streaming library into your existing monad transformer stack and/or vs versa. Shuffling monad layers around like above technique seems incredibly useful so if you know of similar techniques let me know.


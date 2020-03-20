# Readme for Data.Conduit.Lift

One of the great features provided by Pipes package is the ability to utilize monad transformers in isolated sections of a pipeline. Conduit historically has not had this advantage because it did not quite [follow the monad transformer laws.](https://github.com/snoyberg/conduit/wiki/Dealing-with-monad-transformers).

`Data.Conduit.Lift` gives a method to get around this limitation so that Conduit can utilize monad transformers in isolate sections of conduit much like the `Pipes.Lift` module provides for Pipes. It is also a rather mechanical translation of the work I recently provided for `Pipes.Lift`.  The technique seems to be quite general and applies to both Pipes, Conduit and probably other streaming systems as well.

Here is an example fro the [link above](https://github.com/snoyberg/conduit/wiki/Dealing-with-monad-transformers) demonstrating the problem with running the state transformer in an isolate section of conduit.

``` active haskell
{-# LANGUAGE OverloadedStrings #-}
import Data.Conduit
import qualified Data.Conduit.List as CL
import Control.Monad.State
import Control.Monad.Trans.Class

source :: Source IO ()
source = CL.sourceList $ replicate 10 ()

replaceNum :: Conduit () (StateT Int IO) Int
replaceNum = awaitForever $ \() -> do
    i <- lift get
    lift $ put $ i + 1
    yield i

main :: IO ()
main = do
    x <- source $$ transPipe (flip evalStateT 1) replaceNum =$ CL.consume
    print x

    y <- flip evalStateT 1
       $ transPipe lift source $$ replaceNum =$ CL.consume
    print y
```

`Data.Conduit.Lift` provides evalStateC to handle this evaluating the state transformer locally rather then globally along the entire conduit:

``` haskell

test = do
    x <- source3 $$ evalStateC 1 replaceNum =$ CL.consume
    print x

> test
[1,2,3,4,5,6,7,8,9,10]

```

This has several advantages:

* Conduit sections that need to eval a state transform can be implemented and used with out having to worry about adding an evalStateT outside the conduit
* Monad transformers can be isolated to only where they are needed. If you need state it no longer needs to be global to the entire conduit.
* 


`Data.Conduit.Lift` implements functionality for the:
* ErrorT
* MaybeT
* ReaderT
* StateT
* WriterT
* rwsT

This technique will work for any monad transformer you can write a [`MFunctor`](http://hackage.haskell.org/package/mmorph-1.0.0/docs/Control-Monad-Morph.html#t:MFunctor) instance for.

I hope this makes it easier to include your favorite monad transformer in your Conduit code or vs versa.




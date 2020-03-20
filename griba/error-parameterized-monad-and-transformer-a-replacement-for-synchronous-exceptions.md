# Error parameterized monad and transformer, a replacement for synchronous exceptions

## Monadic computations

A [Monad](https://wiki.haskell.org/Monad) is an abstract data type for modelling the sequentiality of side effect capable computations that return a result value.

```haskell
class Monad m where
  -- return: generates a simple computation with the parameter as result
  return :: a -> m a  

  -- (>>=) binds a monadic action with a monadic function on its result
  (>>=) :: m a -> (a -> m b) -> m b
  
  -- (>>) binds with a second computation ignoring the result of the first one
  (>>) :: m a -> m b -> m b
  
  x >> y = x >>= \_ -> y      -- (>>) can be defined in terms of (>>=)
  ...
```

The actual definition extends the Applicative typeclass (sequencing computations while combining its results) with `return = pure` and `(>>) = (*>)`, but I want to base this guide on Monads.

The *Do* block is a syntax construction that translates to a Monadic composition of computations. See [do notation](https://wiki.haskell.org/Monad#do-notation)

## Failable computations. Left absorbing elements

A failable monad domain is one that has a [left absorbing element](https://en.wikipedia.org/wiki/Absorbing_element) in `(>>=)`, meaning the monad composition result will be the left operand's one, stopping the evaluation of subsequent computations.

In the *Either* monad all the values constructed with *Left* are left absorbing. See [Data.Either source](http://hackage.haskell.org/package/base/docs/src/Data.Either.html#line-151)

```haskell
instance Monad (Either e) where

   Left e >>= _ = Left e          -- left absorbing, subsequent comp. are ignored
   
   Right x >>= f = f x
   
   return = Right
```

## Exceptions

The package *exceptions* generalizes the exception context to more monads than IO. See [Control.Monad.Catch](https://hackage.haskell.org/package/exceptions/docs/Control-Monad-Catch.html)

```haskell
-- from Control.Monad.Catch

class Monad m => MonadThrow m where

  throwM :: Exception e => e -> m a
  
class MonadThrow m => MonadCatch m where  

  catch :: Exception e => m a -> (e -> m a) -> m a
  
-- MonadCatch instances should obey the following law:
catch (throwM e) f ≡ f e
  
```

You may forget catching exceptions, that finally crash your application.

```haskell
{-# LANGUAGE PackageImports #-}

import "exceptions" Control.Monad.Catch (MonadThrow(..), MonadCatch(..), try)

data MyException = MyExcCase1 String | MyExcCase2 
  deriving (Eq, Show)

instance Exception MyException  -- makes an Exception instance

action1 :: MonadThrow m => m Int
action1 = do
            ... 
            when condition $ throwM $ MyExcCase1 "message"


-- try :: (MonadCatch m, Exception e) => m a -> m (Either e a) 

action2A :: (MonadCatch m) => m (Either MyException Int)
action2A = do
            ...
            try action1   -- catches exception that returns as an Either
            
-- forgetting to catch your exceptions
action2B :: (MonadCatch m) => m Int
acrion2B = do
             ...
             action1  -- it may crash if exception not catched in the code upwards
             ...

```

See also [Catching all exceptions](https://www.schoolofhaskell.com/user/snoyberg/general-haskell/exceptions/catching-all-exceptions) although it is about problems with asynchronous exceptions. 

```haskell 
-- exception pkg replacement as recommended in "Catching all exceptions"

-- import "exceptions" Control.Monad.Catch

import "safe-exceptions" Control.Exception.Safe
```


## Throwable errors that cannot escape the monad - ExceptT


[ExceptT](http://hackage.haskell.org/package/transformers/docs/Control-Monad-Trans-Except.html) is a monad [transformer](https://wiki.haskell.org/Monad_Transformers_Explained) that parameterises computations with the error type.

Because the *main* function must have type `IO a`, you will have to unwrap your ExceptT transformer (with *runExceptT*) at some point, and **your thrown error will not escape the transformed monad!!**, giving you an `Either error result` to analyze, unlike non-caught exceptions. 


```haskell  
-- The ExceptT monad transformer

newtype ExceptT err m a = ExceptT (m (Either err a))
-- ^ err: the error type
-- ^ m: the inner monad

-- the ExceptT unwrapper
runExceptT :: ExceptT err m a -> m (Either err a)

throwE :: (Monad m) => err -> ExceptT err m a
throwE = ExceptT . return . Left

catchE :: (Monad m) =>
    ExceptT err m a               -- ^ the inner computation
    -> (err -> ExceptT err' m a)    -- ^ a handler for exceptions in the inner
                                -- computation
    -> ExceptT err' m a
    
catchE m handler = ExceptT $ do
    a <- runExceptT m
    case a of
        Left  err -> runExceptT (handler err)
        Right r -> return (Right r)

-- `withExceptT` evaluates a monadic action of a different error type, 
-- lifting the error to the present action error type, as shown below
withExceptT :: Functor m => (e -> e') -> ExceptT e m a -> ExceptT e' m a
```

### The Except monad

*(Except err)* is a wrapper for failable functional computations in the Identity monad

```haskell  
-- Identity is the identity monad from "base" Data.Functor.Identity

type Except err :: * -> * = ExceptT e Identity

-- the wrapper
except :: Either err a -> Except err a
except = Identity >>> ExceptT

-- the unwrapper
runExcept :: Except err a -> Either err a
runExcept = runExceptT >>> runIdentity
```

Example of use: Simple nesting of error throwing actions. The error type of the outer action is a discriminated union (sum type) of own and child actions errors. I don't use the type alias *Except* to put emphasis on ExceptT

```haskell
import "transformers" Control.Monad.Trans.Except
import Control.Monad (when)
import Data.Functor.Identity -- this would not be necessary using the type alias Except

data Err2 = Err2Cons1 String | Err2_Other deriving (Eq, Show)

data Err1 = Err1Cons1 String | Err1Cons2 Err2 deriving (Eq, Show)

-- throwing an error bypasses the rest of the computation

action3 :: ExceptT Err2 Identity Int
action3 = do
            x <- return someCalculation
            when (x `notElem` expectedValues) $ throwE $ Err2Cons1 "Unexpected!" 
            when condition2 $ throwE Err2_Other
            return x
            
action2 :: ExceptT Err2 Identity Int
action2 = catchE action3 $ \err -> case err of
                                Err2Cons1 msg -> return 0
                                Err2_Other -> throwE err -- rethrow

action1 :: ExceptT Err1 Identity Int
action1 = do
            when condition $ throwE $ Err1Cons1 "err1 msg"
            
            withExceptT Err1Cons2 action2   -- evaluates action2 lifting the Err2 type to an Err1
       
main :: IO ()
main = do
          let eRes = runExcept action1
          case eRes of
            Left err1 -> putStrLn $ "error: " ++ show err1
            Right v -> putStrLn $ "ok: " ++ show v
```

## Leveraging exceptions into monad errors

```haskell
{-# LANGUAGE PackageImports #-}

import Control.Monad (when)
import "transformers" Control.Monad.Trans.Except
import "exceptions" Control.Monad.Catch

data Err2 = Err2Cons1 String | Err2_Other deriving (Eq, Show)

instance Exception Err2

data Err1 = Err1Cons1 String | Err1Cons2 Err2 deriving (Eq, Show)

-- an exception throwing action
action2 :: IO Int
action2 = do
            x <- return someCalculation
            when (x `notElem` expectedValues) $ throwM $ Err2Cons1 "Unexpected!" 
            ...
            return x

-- an error throwing action
action1 :: ExceptT Err1 IO Int
action1 = do
            -- set the condition at will
            when condition $ throwE $ Err1Cons1 "err1 msg"

            -- wrapping an exception throwing action
            withExceptT Err1Cons2 $ ExceptT (try action2)
       
main :: IO ()
main = do
          eRes <- runExceptT action1
          case eRes of
            Left err1 -> putStrLn $ "error: " ++ show err1
            Right v -> putStrLn $ "ok: " ++ show v
```

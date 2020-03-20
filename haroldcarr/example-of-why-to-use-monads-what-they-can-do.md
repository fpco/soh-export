# example of why to use monads - what they can do

# Introduction

This article is part of a [series of articles on monads](http://haroldcarr.com/posts/2014-02-19-monad-series.html).

Its purpose is to motivate the *usage* of monads (not to explain how
they work, nor to explain in details the particular examples shown
here).

This article is a condensed version of

- <http://www.grabmueller.de/martin/www/pub/Transformers.pdf>

- <http://www.cs.virginia.edu/~wh5a/personal/Transformers.lhs>

This article will show

-   monads as a flexible, extensible way structuring of programs

-   monads hide book-keeping/plumbing, removing clutter from main algorithm

-   using monad transformers (a way to use two or more monads at the same time)

---

## setup

``` haskell
{-# LANGUAGE PackageImports #-}

import           "mtl" Control.Monad.Identity
import           "mtl" Control.Monad.Error
import           "mtl" Control.Monad.Reader
import           "mtl" Control.Monad.State
import           "mtl" Control.Monad.Writer

import                 Data.Maybe
import qualified       Data.Map as Map
```


---

# non-monadic expression evaluator

An expression evaluator will be used as a running example:

``` haskell
type Name   =  String                -- variable names

data Exp    =  Lit  Integer          -- expressions
            |  Var  Name
            |  Plus Exp  Exp
            |  Abs  Name Exp
            |  App  Exp  Exp
            deriving (Eq, Show)

data Value  =  IntVal Integer        -- values
            |  FunVal Env Name Exp
            deriving (Eq, Show)

type Env    =  Map.Map Name Value    -- from names to values

eval0                 :: Env -> Exp -> Value
eval0 env (Lit i)      = IntVal i
eval0 env (Var n)      = fromJust (Map.lookup n env)
eval0 env (Plus e1 e2) = let  IntVal i1  = eval0 env e1
                              IntVal i2  = eval0 env e2
                         in IntVal (i1 + i2)
eval0 env (Abs  n  e)  = FunVal env n e
eval0 env (App  e1 e2) = let  val1  = eval0 env e1
                              val2  = eval0 env e2
                         in case val1 of
                              FunVal env' n body ->
                                  eval0 (Map.insert n val2 env') body
```

Given the above, evaluating `12 + (\x -> x) (4 + 2)` will result in `18` :

``` active haskell
{-# LANGUAGE PackageImports #-}

import           "mtl" Control.Monad.Identity
import           "mtl" Control.Monad.Error
import           "mtl" Control.Monad.Reader
import           "mtl" Control.Monad.State
import           "mtl" Control.Monad.Writer

import                 Data.Maybe
import qualified       Data.Map as Map

type Name   =  String                -- variable names

data Exp    =  Lit  Integer          -- expressions
            |  Var  Name
            |  Plus Exp  Exp
            |  Abs  Name Exp
            |  App  Exp  Exp
            deriving (Eq, Show)

data Value  =  IntVal Integer        -- values
            |  FunVal Env Name Exp
            deriving (Eq, Show)

type Env    =  Map.Map Name Value    -- from names to values

eval0                 :: Env -> Exp -> Value
eval0 env (Lit i)      = IntVal i
eval0 env (Var n)      = fromJust (Map.lookup n env)
eval0 env (Plus e1 e2) = let  IntVal i1  = eval0 env e1
                              IntVal i2  = eval0 env e2
                         in IntVal (i1 + i2)
eval0 env (Abs  n  e)  = FunVal env n e
eval0 env (App  e1 e2) = let  val1  = eval0 env e1
                              val2  = eval0 env e2
                         in case val1 of
                              FunVal env' n body ->
                                  eval0 (Map.insert n val2 env') body

-- show
exampleExp = Plus (Lit 12) (App (Abs "x" (Var "x")) (Plus (Lit 4) (Lit 2)))

main = putStrLn $ show $ eval0 Map.empty exampleExp
-- ==> IntVal 18
-- /show
```

The above evaluator works fine for the "happy path", but does not give useful
error messages when things go wrong, such as an unbound variable:

``` active haskell
{-# LANGUAGE PackageImports #-}

import           "mtl" Control.Monad.Identity
import           "mtl" Control.Monad.Error
import           "mtl" Control.Monad.Reader
import           "mtl" Control.Monad.State
import           "mtl" Control.Monad.Writer

import                 Data.Maybe
import qualified       Data.Map as Map

type Name   =  String                -- variable names

data Exp    =  Lit  Integer          -- expressions
            |  Var  Name
            |  Plus Exp  Exp
            |  Abs  Name Exp
            |  App  Exp  Exp
            deriving (Eq, Show)

data Value  =  IntVal Integer        -- values
            |  FunVal Env Name Exp
            deriving (Eq, Show)

type Env    =  Map.Map Name Value    -- from names to values

eval0                 :: Env -> Exp -> Value
eval0 env (Lit i)      = IntVal i
eval0 env (Var n)      = fromJust (Map.lookup n env)
eval0 env (Plus e1 e2) = let  IntVal i1  = eval0 env e1
                              IntVal i2  = eval0 env e2
                         in IntVal (i1 + i2)
eval0 env (Abs  n  e)  = FunVal env n e
eval0 env (App  e1 e2) = let  val1  = eval0 env e1
                              val2  = eval0 env e2
                         in case val1 of
                              FunVal env' n body ->
                                  eval0 (Map.insert n val2 env') body

-- show
exampleExp = Plus (Lit 12) (App (Abs "x" (Var "x")) (Plus (Lit 4) (Lit 2)))

main = putStrLn $ show $ eval0 Map.empty (Var "x")
-- results in an error: Maybe.fromJust: Nothing
```

That can be "fixed" by using `Either`:

``` active haskell
{-# LANGUAGE PackageImports #-}

import           "mtl" Control.Monad.Identity
import           "mtl" Control.Monad.Error
import           "mtl" Control.Monad.Reader
import           "mtl" Control.Monad.State
import           "mtl" Control.Monad.Writer

import                 Data.Maybe
import qualified       Data.Map as Map

type Name   =  String                -- variable names

data Exp    =  Lit  Integer          -- expressions
            |  Var  Name
            |  Plus Exp  Exp
            |  Abs  Name Exp
            |  App  Exp  Exp
            deriving (Eq, Show)

data Value  =  IntVal Integer        -- values
            |  FunVal Env Name Exp
            deriving (Eq, Show)

type Env    =  Map.Map Name Value    -- from names to values


-- show
eval0e                 :: Env -> Exp -> Either String Value
eval0e env (Lit  i)     = Right $ IntVal i
eval0e env (Var  n)     = case Map.lookup n env of
                              Nothing -> Left $ "unbound var: " ++ n
                              Just v  -> Right v
eval0e env (Plus e1 e2) = let  Right (IntVal i1)  = eval0e env e1
                               Right (IntVal i2)  = eval0e env e2
                          in Right $ IntVal (i1 + i2)
eval0e env (Abs  n  e)  = Right $ FunVal env n e
eval0e env (App  e1 e2) = let  Right val1  = eval0e env e1
                               Right val2  = eval0e env e2
                          in case val1 of
                              FunVal env' n body ->
                                  eval0e (Map.insert n val2 env') body

main = putStrLn $ show $ eval0e Map.empty (Var "x")
-- ==> Left "unbound var: x"
-- /show
```

That works, but the code gets ugly fast, adding/removing `Left` and
`Right` all over the place &#x2014; and the fix only dealt with unbound
variables, not other problems like a non- `IntVal` given to `Plus`
(shown later).


---

# conversion to monadic structure

A better solution is to write the code in a "monadic" style that makes
it relatively easy to add, remove or change monads.  The monads to be
added will handle error conditions, state, etc., (as will be seen
later).

``` haskell
type Eval1 alpha  =   Identity alpha

runEval1          ::  Eval1 alpha -> alpha
runEval1 ev       =   runIdentity ev

eval1                 :: Env -> Exp -> Eval1 Value
eval1 env (Lit  i)     = return $ IntVal i
eval1 env (Var  n)     = return $ fromJust (Map.lookup n env)
eval1 env (Plus e1 e2) = do  IntVal i1  <- eval1 env e1
                             IntVal i2  <- eval1 env e2
                             return $ IntVal (i1 + i2)
eval1 env (Abs  n  e)  = return $ FunVal env n e
eval1 env (App  e1 e2) = do  val1  <- eval1 env e1
                             val2  <- eval1 env e2
                             case val1 of
                                 FunVal env' n body ->
                                     eval1 (Map.insert n val2 env') body
```

Monadic `eval1` is very similar to non-monadic `eval0`.  The only
difference is the type signature and the addition of `return`, `do`,
and using `<-` instead of `let`.

Again, it is not necessary, in this article, to understand *how* monads
work in this example.  The point is what various monads can do, which
will be seen below.  In other words, *why* use monads.


`eval1` has the same behavior as `eval0`

``` active haskell
{-# LANGUAGE PackageImports #-}

import           "mtl" Control.Monad.Identity
import           "mtl" Control.Monad.Error
import           "mtl" Control.Monad.Reader
import           "mtl" Control.Monad.State
import           "mtl" Control.Monad.Writer

import                 Data.Maybe
import qualified       Data.Map as Map

type Name   =  String                -- variable names

data Exp    =  Lit  Integer          -- expressions
            |  Var  Name
            |  Plus Exp  Exp
            |  Abs  Name Exp
            |  App  Exp  Exp
            deriving (Eq, Show)

data Value  =  IntVal Integer        -- values
            |  FunVal Env Name Exp
            deriving (Eq, Show)

type Env    =  Map.Map Name Value    -- from names to values

type Eval1 alpha  =   Identity alpha

runEval1          ::  Eval1 alpha -> alpha
runEval1 ev       =   runIdentity ev

eval1                 :: Env -> Exp -> Eval1 Value
eval1 env (Lit  i)     = return $ IntVal i
eval1 env (Var  n)     = return $ fromJust (Map.lookup n env)
eval1 env (Plus e1 e2) = do  IntVal i1  <- eval1 env e1
                             IntVal i2  <- eval1 env e2
                             return $ IntVal (i1 + i2)
eval1 env (Abs  n  e)  = return $ FunVal env n e
eval1 env (App  e1 e2) = do  val1  <- eval1 env e1
                             val2  <- eval1 env e2
                             case val1 of
                                 FunVal env' n body ->
                                     eval1 (Map.insert n val2 env') body


exampleExp = Plus (Lit 12) (App (Abs "x" (Var "x")) (Plus (Lit 4) (Lit 2)))


-- show
main = do
    putStrLn $ show $ runEval1 (eval1 Map.empty exampleExp)
    -- ==> IntVal 18
    putStrLn $ show $ runEval1 (eval1 Map.empty (Var "x"))
    -- results in error : Maybe.fromJust: Nothing
-- /show
```

(Note: `runEval1` is used to get the result of `eval1` "out" of the monad.)

The following sections will show how to leverage the monadic structure
of `eval1` to fix problems with the evaluator by changing the type of
the evaluator to use more monads &#x2014; all the while using the same
top-level structure of `eval1`.

---

# adding error handling

## unbound variables

Unbound variables are handled with `Either` (as in `eval0e`) but now
the wrapping/unwrapping of `Left` / `Right` is hidden "inside" the
monad definition of `Either` (not shown) rather than cluttering the
program.

`eval2a` is *exactly* the same as `eval1` except for `Var` handling
and the type signature.  That's the point, the evaluator has become
more powerful without extensive rewriting.  Instead, more monads are
used (i.e., `ErrorT`).

``` haskell
-- String is the type arg to ErrorT : the type of exceptions in example
type Eval2 alpha = ErrorT String Identity alpha

runEval2     :: Eval2 alpha -> Either String alpha
runEval2 ev  = runIdentity (runErrorT ev)

eval2a                 :: Env -> Exp -> Eval2 Value
eval2a env (Lit  i)     = return $ IntVal i

-- eval1 / eval2a diff:
eval2a env (Var  n)     = case Map.lookup n env of
                              Nothing -> fail $ "unbound var: " ++ n
                              Just v  -> return v

eval2a env (Plus e1 e2) = do  IntVal i1  <- eval2a env e1
                              IntVal i2  <- eval2a env e2
                              return $ IntVal (i1 + i2)
eval2a env (Abs  n  e)  = return $ FunVal env n e
eval2a env (App  e1 e2) = do  val1  <- eval2a env e1
                              val2  <- eval2a env e2
                              case val1 of
                                  FunVal env' n body -> eval2a (Map.insert n val2 env') body
```

`eval2a` handles normal evaluation as before but also handles unbound
variables in a more useful manner:


``` active haskell
{-# LANGUAGE PackageImports #-}

import           "mtl" Control.Monad.Identity
import           "mtl" Control.Monad.Error
import           "mtl" Control.Monad.Reader
import           "mtl" Control.Monad.State
import           "mtl" Control.Monad.Writer

import                 Data.Maybe
import qualified       Data.Map as Map

type Name   =  String                -- variable names

data Exp    =  Lit  Integer          -- expressions
            |  Var  Name
            |  Plus Exp  Exp
            |  Abs  Name Exp
            |  App  Exp  Exp
            deriving (Eq, Show)

data Value  =  IntVal Integer        -- values
            |  FunVal Env Name Exp
            deriving (Eq, Show)

type Env    =  Map.Map Name Value    -- from names to values

-- String is the type arg to ErrorT : the type of exceptions in example
type Eval2 alpha = ErrorT String Identity alpha

runEval2     :: Eval2 alpha -> Either String alpha
runEval2 ev  = runIdentity (runErrorT ev)

eval2a                 :: Env -> Exp -> Eval2 Value
eval2a env (Lit  i)     = return $ IntVal i

-- eval1 / eval2a diff:
eval2a env (Var  n)     = case Map.lookup n env of
                              Nothing -> fail $ "unbound var: " ++ n
                              Just v  -> return v

eval2a env (Plus e1 e2) = do  IntVal i1  <- eval2a env e1
                              IntVal i2  <- eval2a env e2
                              return $ IntVal (i1 + i2)
eval2a env (Abs  n  e)  = return $ FunVal env n e
eval2a env (App  e1 e2) = do  val1  <- eval2a env e1
                              val2  <- eval2a env e2
                              case val1 of
                                  FunVal env' n body -> eval2a (Map.insert n val2 env') body

exampleExp = Plus (Lit 12) (App (Abs "x" (Var "x")) (Plus (Lit 4) (Lit 2)))

-- show
main = do
    putStrLn $ show $ runEval2 (eval2a Map.empty exampleExp)
    -- ==> Right (IntVal 18)
    putStrLn $ show $ runEval2 (eval2a Map.empty (Var "no-way"))
    -- ==> Left "unbound var: no-way"
-- /show
```

## dynamic type errors

An improvement.  But all the evaluators above still give poor error
messages for incorrect dynamic typing:

``` active haskell
{-# LANGUAGE PackageImports #-}

import           "mtl" Control.Monad.Identity
import           "mtl" Control.Monad.Error
import           "mtl" Control.Monad.Reader
import           "mtl" Control.Monad.State
import           "mtl" Control.Monad.Writer

import                 Data.Maybe
import qualified       Data.Map as Map

type Name   =  String                -- variable names

data Exp    =  Lit  Integer          -- expressions
            |  Var  Name
            |  Plus Exp  Exp
            |  Abs  Name Exp
            |  App  Exp  Exp
            deriving (Eq, Show)

data Value  =  IntVal Integer        -- values
            |  FunVal Env Name Exp
            deriving (Eq, Show)

type Env    =  Map.Map Name Value    -- from names to values

-- String is the type arg to ErrorT : the type of exceptions in example
type Eval2 alpha = ErrorT String Identity alpha

runEval2     :: Eval2 alpha -> Either String alpha
runEval2 ev  = runIdentity (runErrorT ev)

eval2a                 :: Env -> Exp -> Eval2 Value
eval2a env (Lit  i)     = return $ IntVal i

-- eval1 / eval2a diff:
eval2a env (Var  n)     = case Map.lookup n env of
                              Nothing -> fail $ "unbound var: " ++ n
                              Just v  -> return v

eval2a env (Plus e1 e2) = do  IntVal i1  <- eval2a env e1
                              IntVal i2  <- eval2a env e2
                              return $ IntVal (i1 + i2)
eval2a env (Abs  n  e)  = return $ FunVal env n e
eval2a env (App  e1 e2) = do  val1  <- eval2a env e1
                              val2  <- eval2a env e2
                              case val1 of
                                  FunVal env' n body -> eval2a (Map.insert n val2 env') body

exampleExp = Plus (Lit 12) (App (Abs "x" (Var "x")) (Plus (Lit 4) (Lit 2)))


-- show
                                                    -- 12 + (\x -> x)
main = putStrLn $ show $ runEval2 (eval2a Map.empty (Plus (Lit 12) (Abs "x" (Var "x"))))
-- ==> Left "Pattern match failure in do expression at /home/app/isolation-runner-work/projects/24798/src.205/Main.hs:42:31-39"
-- /show
```



That is fixed by pattern matching in `Plus` and `App` handling and
explicitly throwing an appropriate error:

``` haskell
eval2b                 :: Env -> Exp -> Eval2 Value
eval2b env (Lit  i)     = return $ IntVal i
eval2b env (Var  n)     = case Map.lookup n env of
                              Nothing -> fail $ "unbound var: " ++ n
                              Just v  -> return v
eval2b env (Plus e1 e2) = do  e1'  <- eval2b env e1
                              e2'  <- eval2b env e2
                              -- eval2a / eval2b diff:
                              case (e1', e2') of
                                  (IntVal i1, IntVal i2) -> return $ IntVal (i1 + i2)
                                  _                      -> throwError "type error in Plus"
eval2b env (Abs  n  e)  = return $ FunVal env n e
eval2b env (App  e1 e2) = do  val1  <- eval2b env e1
                              val2  <- eval2b env e2
                              -- eval2a / eval2b diff:
                              case val1 of
                                  FunVal env' n body -> eval2b (Map.insert n val2 env') body
                                  _                  -> throwError "type error in App"
```

The monadic structure enabled "throwing" the error without the need to
thread that error return throughout the code.  Instead, it is hidden
and handled by the `ErrorT` monad.

``` active haskell
{-# LANGUAGE PackageImports #-}

import           "mtl" Control.Monad.Identity
import           "mtl" Control.Monad.Error
import           "mtl" Control.Monad.Reader
import           "mtl" Control.Monad.State
import           "mtl" Control.Monad.Writer

import                 Data.Maybe
import qualified       Data.Map as Map

type Name   =  String                -- variable names

data Exp    =  Lit  Integer          -- expressions
            |  Var  Name
            |  Plus Exp  Exp
            |  Abs  Name Exp
            |  App  Exp  Exp
            deriving (Eq, Show)

data Value  =  IntVal Integer        -- values
            |  FunVal Env Name Exp
            deriving (Eq, Show)

type Env    =  Map.Map Name Value    -- from names to values

-- String is the type arg to ErrorT : the type of exceptions in example
type Eval2 alpha = ErrorT String Identity alpha

runEval2     :: Eval2 alpha -> Either String alpha
runEval2 ev  = runIdentity (runErrorT ev)

eval2b                 :: Env -> Exp -> Eval2 Value
eval2b env (Lit  i)     = return $ IntVal i
eval2b env (Var  n)     = case Map.lookup n env of
                              Nothing -> fail $ "unbound var: " ++ n
                              Just v  -> return v
eval2b env (Plus e1 e2) = do  e1'  <- eval2b env e1
                              e2'  <- eval2b env e2
                              -- eval2a / eval2b diff:
                              case (e1', e2') of
                                  (IntVal i1, IntVal i2) -> return $ IntVal (i1 + i2)
                                  _                      -> throwError "type error in Plus"
eval2b env (Abs  n  e)  = return $ FunVal env n e
eval2b env (App  e1 e2) = do  val1  <- eval2b env e1
                              val2  <- eval2b env e2
                              -- eval2a / eval2b diff:
                              case val1 of
                                  FunVal env' n body -> eval2b (Map.insert n val2 env') body
                                  _                  -> throwError "type error in App"

-- show
main = do
    putStrLn $ show $ runEval2 (eval2b Map.empty (Plus (Lit 12) (Abs "x" (Var "x"))))
    -- ==> Left "type error in Plus"
    putStrLn $ show $ runEval2 (eval2b Map.empty (App (Lit 12) (Lit 0)))
    -- ==> Left "type error in App"
-- /show
```

---

# hiding the environment

The next change hides `Env` (via the `ReaderT` monad) since `Env` is
only extended in `App` and used in `Var` and `Abs`.

Notice how, for each successive evaluator (i.e., `eval1`, `eval2`,
`eval3`), an additional monad is pushed onto the front of the "monad
stack" used in the type of the evaluator.  Likewise, the final value
expression evaluation is obtained by removing each monad layer via
`runIdentity`, `runErrorT`, `runReaderT`.

``` active haskell
{-# LANGUAGE PackageImports #-}

import           "mtl" Control.Monad.Identity
import           "mtl" Control.Monad.Error
import           "mtl" Control.Monad.Reader
import           "mtl" Control.Monad.State
import           "mtl" Control.Monad.Writer

import                 Data.Maybe
import qualified       Data.Map as Map

type Name   =  String                -- variable names

data Exp    =  Lit  Integer          -- expressions
            |  Var  Name
            |  Plus Exp  Exp
            |  Abs  Name Exp
            |  App  Exp  Exp
            deriving (Eq, Show)

data Value  =  IntVal Integer        -- values
            |  FunVal Env Name Exp
            deriving (Eq, Show)

type Env    =  Map.Map Name Value    -- from names to values

exampleExp = Plus (Lit 12) (App (Abs "x" (Var "x")) (Plus (Lit 4) (Lit 2)))

-- show
type Eval3 alpha = ReaderT Env (ErrorT String Identity) alpha

runEval3     :: Env -> Eval3 alpha -> Either String alpha
runEval3 env ev  = runIdentity (runErrorT (runReaderT ev env))

eval3             :: Exp -> Eval3 Value
eval3 (Lit  i)     = return $ IntVal i
eval3 (Var  n)     = do env <- ask                -- eval2b / eval3 diff
                        case Map.lookup n env of
                            Nothing  -> throwError ("unbound variable: " ++ n)
                            Just val -> return val
eval3 (Plus e1 e2) = do e1'  <- eval3 e1
                        e2'  <- eval3 e2
                        case (e1', e2') of
                            (IntVal i1, IntVal i2) -> return $ IntVal (i1 + i2)
                            _                      -> throwError "type error in Plus"
eval3 (Abs  n  e)  = do env <- ask
                        return $ FunVal env n e
eval3 (App  e1 e2) = do val1  <- eval3 e1
                        val2  <- eval3 e2
                        case val1 of
                                                  -- eval2b / eval3 diff
                            FunVal env' n body -> local (const (Map.insert n val2 env')) (eval3 body)
                            _                  -> throwError "type error in App"

main = putStrLn $ show $ runEval3 Map.empty (eval3 exampleExp)
       -- ==> Right (IntVal 18)
-- /show
```

In `eval3`, the `ReaderT` `ask` function is used to obtain `Env` in
`Var` and `Abs`, and `local` is used to extend `Env` for the recursive
call to `eval3` in `App`.  (Note: the `local` environment, in this
case, does not depend on the current environment, so `const` is used.)

Again, understanding the exact details mentioned here is not
necessary.  Instead, notice how the code only changed where `Env` is
used.  Nothing else changed (other than the type signature and not
giving `Env` as an explicit parameter to `eval3`).


---

# adding state

As an example of state, the evaluator is extended with "profiling" :
an integer counting calls to the evaluator.  The state added is *not*
state like a mutable location in imperative languages.  It is
"effectful" &#x2014; meaning updated values are seen after updating but no
locations are mutated.  How that happens is not covered in this
article.

The `StateT` monad is wrapped around the innermost monad `Identity`
(order of `State` and `Error` matters).

``` haskell
type Eval4 alpha = ReaderT Env (ErrorT String (StateT Integer Identity)) alpha

-- returns evaluation result (error or value) and state
-- give initial state arg for flexibility
runEval4            ::  Env -> Integer -> Eval4 alpha -> (Either String alpha, Integer)
runEval4 env st ev  =   runIdentity (runStateT (runErrorT (runReaderT ev env)) st)

-- tick type not same as =Eval4= so it can reused elsewhere.
tick :: (Num s, MonadState s m) => m ()
tick = do  st <- get
           put (st + 1)

-- eval4          :: Exp -> Eval4 Value
eval4 (Lit i)      = do tick
                        return $ IntVal i
eval4 (Var n)      = do tick
                        env <- ask
                        case Map.lookup n env of
                            Nothing -> throwError ("unbound variable: " ++ n)
                            Just val -> return val
eval4 (Plus e1 e2) = do tick
                        e1'  <- eval4 e1
                        e2'  <- eval4 e2
                        case (e1', e2') of
                            (IntVal i1, IntVal i2) ->
                                return $ IntVal (i1 + i2)
                            _ -> throwError "type error in addition"
eval4 (Abs n e)    = do tick
                        env <- ask
                        return $ FunVal env n e
eval4 (App e1 e2)  = do tick
                        val1  <- eval4 e1
                        val2  <- eval4 e2
                        case val1 of
                            FunVal env' n body -> local (const (Map.insert n val2 env')) (eval4 body)
                            _ -> throwError "type error in application"
```

`eval4` is identical to `eval3` (other than the change in type
signature) except each case starts by calling `tick` (and `do` is
added to `Lit`).

``` active haskell
{-# LANGUAGE PackageImports #-}

import           "mtl" Control.Monad.Identity
import           "mtl" Control.Monad.Error
import           "mtl" Control.Monad.Reader
import           "mtl" Control.Monad.State
import           "mtl" Control.Monad.Writer

import                 Data.Maybe
import qualified       Data.Map as Map

type Name   =  String                -- variable names

data Exp    =  Lit  Integer          -- expressions
            |  Var  Name
            |  Plus Exp  Exp
            |  Abs  Name Exp
            |  App  Exp  Exp
            deriving (Eq, Show)

data Value  =  IntVal Integer        -- values
            |  FunVal Env Name Exp
            deriving (Eq, Show)

type Env    =  Map.Map Name Value    -- from names to values

type Eval4 alpha = ReaderT Env (ErrorT String (StateT Integer Identity)) alpha

-- returns evaluation result (error or value) and state
-- give initial state arg for flexibility
runEval4            ::  Env -> Integer -> Eval4 alpha -> (Either String alpha, Integer)
runEval4 env st ev  =   runIdentity (runStateT (runErrorT (runReaderT ev env)) st)

-- tick type not same as =Eval4= so it can reused elsewhere.
tick :: (Num s, MonadState s m) => m ()
tick = do  st <- get
           put (st + 1)

-- eval4          :: Exp -> Eval4 Value
eval4 (Lit i)      = do tick
                        return $ IntVal i
eval4 (Var n)      = do tick
                        env <- ask
                        case Map.lookup n env of
                            Nothing -> throwError ("unbound variable: " ++ n)
                            Just val -> return val
eval4 (Plus e1 e2) = do tick
                        e1'  <- eval4 e1
                        e2'  <- eval4 e2
                        case (e1', e2') of
                            (IntVal i1, IntVal i2) ->
                                return $ IntVal (i1 + i2)
                            _ -> throwError "type error in addition"
eval4 (Abs n e)    = do tick
                        env <- ask
                        return $ FunVal env n e
eval4 (App e1 e2)  = do tick
                        val1  <- eval4 e1
                        val2  <- eval4 e2
                        case val1 of
                            FunVal env' n body -> local (const (Map.insert n val2 env')) (eval4 body)
                            _ -> throwError "type error in application"

exampleExp = Plus (Lit 12) (App (Abs "x" (Var "x")) (Plus (Lit 4) (Lit 2)))

-- show
main = putStrLn $ show $ runEval4 Map.empty 0 (eval4 exampleExp)
       -- (Right (IntVal 18),8) -- 8 reduction steps
-- /show
```

---

# adding logging

The evaluator is now extended to collect the name of each variable
encountered during evaluation and return the collection when
evaluation is done.

That is done via the `WriterT` monad.

(`WriterT` is a kind of a dual to `ReaderT`: `WriterT` can add (e.g.,
"write") values to result of computation, whereas `ReaderT` can only
use (e.g., "read") values passed in.)

``` haskell
type Eval5 alpha = ReaderT Env  (ErrorT String (WriterT [String] (StateT Integer Identity))) alpha

runEval5            ::  Env -> Integer -> Eval5 alpha -> ((Either String alpha, [String]), Integer)
runEval5 env st ev  =   runIdentity (runStateT (runWriterT (runErrorT (runReaderT ev env))) st)

eval5             :: Exp -> Eval5 Value
eval5 (Lit i)      = do tick
                        return $ IntVal i
eval5 (Var n)      = do tick
                        -- eval4 / eval5 diff
                        tell [n] -- collect name of each var encountered during evaluation
                        env <- ask
                        case Map.lookup n env of
                            Nothing  -> throwError ("unbound variable: " ++ n)
                            Just val -> return val
eval5 (Plus e1 e2) = do tick
                        e1'  <- eval5 e1
                        e2'  <- eval5 e2
                        case (e1', e2') of
                            (IntVal i1, IntVal i2) -> return $ IntVal (i1 + i2)
                            _                      -> throwError "type error in addition"
eval5 (Abs n e)     = do tick
                         env <- ask
                         return $ FunVal env n e
eval5 (App e1 e2)   = do tick
                         val1  <- eval5 e1
                         val2  <- eval5 e2
                         case val1 of
                             FunVal env' n body -> local (const (Map.insert n val2 env')) (eval5 body)
                             _                  -> throwError "type error in application"
```

The only change from `eval4` to `eval5` (besides type signature) is
the usage of `tell` in `Var` handling.

``` active haskell
{-# LANGUAGE PackageImports #-}

import           "mtl" Control.Monad.Identity
import           "mtl" Control.Monad.Error
import           "mtl" Control.Monad.Reader
import           "mtl" Control.Monad.State
import           "mtl" Control.Monad.Writer

import                 Data.Maybe
import qualified       Data.Map as Map

type Name   =  String                -- variable names

data Exp    =  Lit  Integer          -- expressions
            |  Var  Name
            |  Plus Exp  Exp
            |  Abs  Name Exp
            |  App  Exp  Exp
            deriving (Eq, Show)

data Value  =  IntVal Integer        -- values
            |  FunVal Env Name Exp
            deriving (Eq, Show)

type Env    =  Map.Map Name Value    -- from names to values

type Eval5 alpha = ReaderT Env  (ErrorT String (WriterT [String] (StateT Integer Identity))) alpha

runEval5            ::  Env -> Integer -> Eval5 alpha -> ((Either String alpha, [String]), Integer)
runEval5 env st ev  =   runIdentity (runStateT (runWriterT (runErrorT (runReaderT ev env))) st)

-- tick type not same as =Eval4= so it can reused elsewhere.
tick :: (Num s, MonadState s m) => m ()
tick = do  st <- get
           put (st + 1)


eval5             :: Exp -> Eval5 Value
eval5 (Lit i)      = do tick
                        return $ IntVal i
eval5 (Var n)      = do tick
                        -- eval4 / eval5 diff
                        tell [n] -- collect name of each var encountered during evaluation
                        env <- ask
                        case Map.lookup n env of
                            Nothing  -> throwError ("unbound variable: " ++ n)
                            Just val -> return val
eval5 (Plus e1 e2) = do tick
                        e1'  <- eval5 e1
                        e2'  <- eval5 e2
                        case (e1', e2') of
                            (IntVal i1, IntVal i2) -> return $ IntVal (i1 + i2)
                            _                      -> throwError "type error in addition"
eval5 (Abs n e)     = do tick
                         env <- ask
                         return $ FunVal env n e
eval5 (App e1 e2)   = do tick
                         val1  <- eval5 e1
                         val2  <- eval5 e2
                         case val1 of
                             FunVal env' n body -> local (const (Map.insert n val2 env')) (eval5 body)
                             _                  -> throwError "type error in application"

exampleExp = Plus (Lit 12) (App (Abs "x" (Var "x")) (Plus (Lit 4) (Lit 2)))

-- show
main = putStrLn $ show $ runEval5 Map.empty 0 (eval5 exampleExp)
       -- ==> ((Right (IntVal 18),["x"]),8)
-- /show
```

At first, it may seem like magic that state, logging, etc., can
suddenly be accessed even though they do not seem to appear as
explicit parameters.  The magic is in eval's type signature.  It is a
monad stack that is essentially a data structure (and more) being
passed throughout eval.  Therefore `ask`, `tell`, etc., can access the
appropriate part of the stack when needed.

(Aside: There is some "utility" magic in the monad transformers ([mtl](http://hackage.haskell.org/package/mtl-2.1.2)).
Even though there is a stack of monads, and a function such as `ask`
needs to operate on a specific monad in the stack (i.e., `ReaderT`),
the monad transformer implementation "automatically" applies the
function to the appropriate monad in the stack, rather than the main
line code needing to explicitly access the right level.)


---

# IO

The final extension is to add IO to the evaluator: `eval6` will print
the value of each `Lit` encountered during evaluation.

``` haskell
type Eval6 alpha = ReaderT Env  (ErrorT String (WriterT [String] (StateT Integer IO))) alpha

runEval6           ::  Env -> Integer -> Eval6 alpha -> IO ((Either String alpha, [String]), Integer)
runEval6 env st ev  =  runStateT (runWriterT (runErrorT (runReaderT ev env))) st

eval6             :: Exp -> Eval6 Value
eval6 (Lit  i)     = do tick
                        -- eval5 / eval 6 diff
                        -- must use =liftIO= to lift into the currently running monad
                        liftIO $ print i -- print each int when evaluated
                        return $ IntVal i
eval6 (Var  n)     = do tick
                        tell [n]
                        env <- ask
                        case Map.lookup n env of
                            Nothing  -> throwError ("unbound variable: " ++ n)
                            Just val -> return val
eval6 (Plus e1 e2) = do tick
                        e1'  <- eval6 e1
                        e2'  <- eval6 e2
                        case (e1', e2') of
                            (IntVal i1, IntVal i2) -> return $ IntVal (i1 + i2)
                            _                      -> throwError "type error in addition"
eval6 (Abs  n  e)  = do tick
                        env <- ask
                        return $ FunVal env n e
eval6 (App  e1 e2) = do tick
                        val1  <- eval6 e1
                        val2  <- eval6 e2
                        case val1 of
                            FunVal env' n body -> local (const (Map.insert n val2 env')) (eval6 body)
                            _                  -> throwError "type error in application"
```

The only change from `eval5` to `eval6` (besides type signature) is
the usage of `liftIO ...` in `Lit` handling.

``` active haskell
{-# LANGUAGE PackageImports #-}

import           "mtl" Control.Monad.Identity
import           "mtl" Control.Monad.Error
import           "mtl" Control.Monad.Reader
import           "mtl" Control.Monad.State
import           "mtl" Control.Monad.Writer

import                 Data.Maybe
import qualified       Data.Map as Map

type Name   =  String                -- variable names

data Exp    =  Lit  Integer          -- expressions
            |  Var  Name
            |  Plus Exp  Exp
            |  Abs  Name Exp
            |  App  Exp  Exp
            deriving (Eq, Show)

data Value  =  IntVal Integer        -- values
            |  FunVal Env Name Exp
            deriving (Eq, Show)

type Env    =  Map.Map Name Value    -- from names to values

type Eval6 alpha = ReaderT Env  (ErrorT String (WriterT [String] (StateT Integer IO))) alpha

runEval6           ::  Env -> Integer -> Eval6 alpha -> IO ((Either String alpha, [String]), Integer)
runEval6 env st ev  =  runStateT (runWriterT (runErrorT (runReaderT ev env))) st

-- tick type not same as =Eval4= so it can reused elsewhere.
tick :: (Num s, MonadState s m) => m ()
tick = do  st <- get
           put (st + 1)


eval6             :: Exp -> Eval6 Value
eval6 (Lit  i)     = do tick
                        -- eval5 / eval 6 diff
                        -- must use =liftIO= to lift into the currently running monad
                        liftIO $ print i -- print each int when evaluated
                        return $ IntVal i
eval6 (Var  n)     = do tick
                        tell [n]
                        env <- ask
                        case Map.lookup n env of
                            Nothing  -> throwError ("unbound variable: " ++ n)
                            Just val -> return val
eval6 (Plus e1 e2) = do tick
                        e1'  <- eval6 e1
                        e2'  <- eval6 e2
                        case (e1', e2') of
                            (IntVal i1, IntVal i2) -> return $ IntVal (i1 + i2)
                            _                      -> throwError "type error in addition"
eval6 (Abs  n  e)  = do tick
                        env <- ask
                        return $ FunVal env n e
eval6 (App  e1 e2) = do tick
                        val1  <- eval6 e1
                        val2  <- eval6 e2
                        case val1 of
                            FunVal env' n body -> local (const (Map.insert n val2 env')) (eval6 body)
                            _                  -> throwError "type error in application"

exampleExp = Plus (Lit 12) (App (Abs "x" (Var "x")) (Plus (Lit 4) (Lit 2)))

-- show
main = runEval6 Map.empty 0 (eval6 exampleExp) >>= putStrLn . show
       -- prints 12 4 2 on separate lines and returns:
       -- ==> ((Right (IntVal 18),["x"]),8)
-- /show
```

---

# summary

The important point to see is that evaluators `eval1` through `eval6`
all have the same structure.  The only change between them is in the
type signature and the usage of specific monad functions (e.g., `ask`,
`tell`) to access data "in" the monad stack.

The mechanics of how state, logging, environment hiding, handling
errors, etc., are weaved through that structure are hidden inside the
monad implementations (rather than cluttering the main program).

Hopefully this article provides a glimpse into the power and
usefulness of monads.

---

# source code

The emacs org-mode literate source code of this article is available at:

-   <https://github.com/haroldcarr/learn-haskell-coq-ml-etc/blob/master/haskell/paper/haroldcarr/2014-02-23-concrete-monads-0-motivating-example/concrete-monads-0-motivating-example.org>

# feedback

Join the discussion at [reddit](http://www.reddit.com/r/haskell/comments/1yriwf/example_of_why_to_use_monads_and_what_they_can_do/)


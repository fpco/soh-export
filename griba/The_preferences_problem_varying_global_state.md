# The Preferences problem. Varying global state

## The Preferences problem (or runtime configuration changes)

### Global state in Standard ML

```
$ sml
Standard ML of New Jersey v110.78 [built: Sat Dec 27 20:53:42 2014]

- val v = ref 5 ;
val v = ref 5 : int ref

- val addToV = fn w => !v + w ;
val addToV = fn : int -> int

- addToV 5 ;
val it = 10 : int

- fun incrV () = v := !v +1 ;
val incrV = fn : unit -> unit

- incrV () ;
val it = () : unit

- addToV 5 ;
val it = 11 : int

```

The use of global variables in functions breaks the [Referential transparency](https://en.wikipedia.org/wiki/Referential_transparency) principle that states that if a function call with the same arguments gives you always the same result (it depends only on the arguments) is a **pure** function and your program will be more predictable and easier to debug.


### Proper global variables in Haskell

You may create Haskell global variables with Data.IORef.newIORef as part of IO effects in the IO Monad. Then, to use them elsewhere, you will have to pass them as parameters, keeping the [Referential transparency](https://en.wikipedia.org/wiki/Referential_transparency) of the functions that use them:

``` active haskell
{-# LANGUAGE PackageImports #-}
import Data.IORef
import Control.Monad (when)
import System.IO (hFlush, stdout)
import "safe" Safe (readMay)

default (Int)  -- literal desambiguation

main :: IO ()
main = do
         ref <- newIORef 0
         loop ref

loop :: IORef Int -> IO ()
loop ref = do
        w <- readIORef ref
        putStrLn $ "the variable is: " ++ show w
        putStr "input a positive value to add (else the program will end): "
        hFlush stdout
        str <- getLine

        case (readMay str :: Maybe Int) of
          Nothing -> do
            putStrLn "unreadable entry!!"
            loop ref
          Just v -> when (v > 0) $ do   -- repeat if v > 0, modifying the variable with (v+)
                          modifyIORef ref (v+)
                          loop ref
                    -- else finish the loop
  
```


### Encoding the global state in a Monad - The Reader Monad

Here the global state will be called "the environment".

Since the monad result depends on the environment, the monad data should host this relation as a function.

``` haskell
newtype Reader env a = Reader { runReader :: (env -> a) }  

class MonadReader env m | m -> env where
    ask   :: m env                            -- get the environment
    local :: (env -> env) -> m a -> m a       -- evaluate an action with a modified environment

asks :: MonadReader env m => (env -> a) -> m a   -- retrieve a projection on the environment
asks env_selector = ask >>= (\env -> return $ env_selector env)
```

We will use the ReaderT monad transformer

#### With a simplified environment


``` active haskell
{-# LANGUAGE PackageImports #-}
import "mtl" Control.Monad.Reader (ReaderT( runReaderT), MonadReader( ask, local))  
import "mtl" Control.Monad.Trans (liftIO)
import Control.Monad (when)
import System.IO (hFlush, stdout)
import "safe" Safe (readMay)
default (Int)  -- literal desambiguation

type Env = Int  -- simplified environment
initial_env = 0

main :: IO ()
main = (runReaderT loop) initial_env   -- apply the monad content (a function) to the initial environment

loop :: ReaderT Env IO ()
loop = do
    env <- ask            -- get the actual environment
    str <- liftIO $ do    
               putStrLn $ "the environment is: " ++ show env  
               putStr "input a positive value to add (else the program will end): "
               hFlush stdout
               getLine

    case (readMay str :: Maybe Int) of
        Nothing -> do
            liftIO $ putStrLn "unreadable entry!!"
            loop
        Just v -> -- repeat only if v > 0, modifying the environment with (v+)
                  when (v > 0) $ local (v+) loop    
                                                    
```

#### With a multifield environment and lenses

I propose [profunctor lenses](/user/griba/easier_lenses_profunctor_based_with_mezzolens), as an easy way to build component updaters.


``` active haskell
{-# LANGUAGE PackageImports #-}
import "mtl" Control.Monad.Reader (ReaderT( runReaderT), MonadReader( ask, local), asks)
import "mtl" Control.Monad.Trans (liftIO)
import Control.Monad (when)
import System.IO (hFlush, stdout)
import "safe" Safe (readMay)
import "mezzolens" Mezzolens (Lens', get, set)
import "mezzolens" Mezzolens.Unchecked (lens)
default (Int)  -- literal desambiguation

data MyEnv = MyEnv {_fld1::Int, _fld2::String}

lensFld1 :: Lens' MyEnv Int
lensFld1 = lens _fld1 (\v env -> env {_fld1 = v})

type Env = MyEnv
initial_env = MyEnv 0 ""

main :: IO ()
main = (runReaderT loop) initial_env   -- apply the monad content (a function) to the initial environment

fld1Sel :: MyEnv -> Int
fld1Sel = get lensFld1

envUpdaterOnFld1 :: (Int -> Int) -> MyEnv -> MyEnv
envUpdaterOnFld1 = lensFld1

loop :: ReaderT Env IO ()
loop = do
    w <- asks fld1Sel         -- get field _fld1 of the environment
    str <- liftIO $ do
               putStrLn $ "the first part of the environment is: " ++ show w
               putStr "input a positive value to add (else the program will end): "
               hFlush stdout
               getLine

    case (readMay str :: Maybe Int) of
        Nothing -> do
            liftIO $ putStrLn "unreadable entry!!"
            loop
        Just v -> -- repeat only if v > 0, modifying the environment with (v+)
                  when (v > 0) $ local (envUpdaterOnFld1 (v+)) loop    
```
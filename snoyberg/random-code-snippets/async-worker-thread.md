# Async exceptions sent to worker thread

Regarding: [http://www.reddit.com/r/haskell/comments/1i5coe/catching_all_exceptions_school_of_haskell/cb15baw](http://www.reddit.com/r/haskell/comments/1i5coe/catching_all_exceptions_school_of_haskell/cb15baw)

```haskell active
{-# LANGUAGE FlexibleContexts #-}
import Control.Exception
import Control.Concurrent
import Control.Concurrent.STM
import GHC.Magic (inline)
import Prelude hiding (catch)
import Control.DeepSeq
import Control.Monad.Trans.Control
import Control.Monad.Trans.Reader
import Control.Monad.IO.Class
import Control.Monad (liftM)
import Control.Concurrent.MVar

-- show
main :: IO ()
main = do
    mtid <- newEmptyMVar
    forkIO $ do
        tid <- takeMVar mtid
        throwTo tid HeapOverflow
    eres <- tryAny $ do
        myThreadId >>= putMVar mtid
        threadDelay 50000
        return "Done"
    print eres
-- /show

tryAnyIO :: IO a -> IO (Either SomeException a)
tryAnyIO action = withAsync action waitCatch

tryAny :: MonadBaseControl IO m => m a -> m (Either SomeException a)
tryAny action =
    -- MAGIC!
    liftBaseWith (\runInIO -> tryAnyIO (runInIO action)) >>=
    either (return . Left) (liftM Right . restoreM)

catchAny :: MonadBaseControl IO m => m a -> (SomeException -> m a) -> m a
catchAny action onE = tryAny action >>= either onE return

tryAnyDeep :: (MonadBaseControl IO m, NFData a)
           => m a
           -> m (Either SomeException a)
tryAnyDeep action = tryAny $ do
    res <- action
    return $!! res -- here's the magic

catchAnyDeep :: (MonadBaseControl IO m, NFData a)
             => m a
             -> (SomeException -> m a)
             -> m a
catchAnyDeep action onE = tryAnyDeep action >>= either onE return

dangerous :: Monad m => m Int
dangerous = return $ error "Unevaluated!"

-- From async
data Async a = Async { asyncThreadId :: {-# UNPACK #-} !ThreadId
                       -- ^ Returns the 'ThreadId' of the thread running the given 'Async'.
                     , _asyncWait    :: STM (Either SomeException a) }

withAsync :: IO a -> (Async a -> IO b) -> IO b
withAsync = inline withAsyncUsing forkIO

withAsyncUsing :: (IO () -> IO ThreadId)
               -> IO a -> (Async a -> IO b) -> IO b
-- The bracket version works, but is slow.  We can do better by
-- hand-coding it:
withAsyncUsing doFork = \action inner -> do
  var <- newEmptyTMVarIO
  mask $ \restore -> do
    t <- doFork $ try (restore action) >>= atomically . putTMVar var
    let a = Async t (readTMVar var)
    r <- restore (inner a) `catchAll` \e -> do cancel a; throwIO e
    cancel a
    return r

catchAll :: IO a -> (SomeException -> IO a) -> IO a
catchAll = catch

cancel :: Async a -> IO ()
cancel (Async t _) = throwTo t ThreadKilled

waitCatch :: Async a -> IO (Either SomeException a)
waitCatch = atomically . waitCatchSTM

waitCatchSTM :: Async a -> STM (Either SomeException a)
waitCatchSTM (Async _ w) = w
```
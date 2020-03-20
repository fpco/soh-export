# ResourceT and deallocation variations

```haskell active
{-# LANGUAGE FlexibleContexts #-}
import Control.Monad.Trans.Resource
import Control.Exception.Lifted (mask)
import Control.Monad.IO.Class
import Control.Monad.Trans.Control

data ReleaseKeyV a = ReleaseKeyV !ReleaseKey !(a -> IO ())

registerV :: MonadResource m => (a -> IO ()) -> a -> m (ReleaseKeyV a)
registerV cleanup onExc = do
    key <- register (cleanup onExc)
    return $! ReleaseKeyV key cleanup

releaseV :: (MonadBaseControl IO m, MonadIO m) => ReleaseKeyV a -> a -> m ()
releaseV (ReleaseKeyV key cleanup) value = mask $ \restore -> do
    res <- unprotect key
    case res of
        Just _ -> restore $ liftIO $ cleanup value
        Nothing -> return () -- should probably throw an error, this key was already released
        
main :: IO ()
main = do
    runResourceT $ do
        key <- registerV putStrLn "exceptional"
        releaseV key "unexceptional"
    runResourceT $ do
        key <- registerV putStrLn "exceptional"
        error "foo"
        releaseV key "unexceptional"
```
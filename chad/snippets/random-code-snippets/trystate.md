# tryState

```haskell active
import Control.Exception
import Control.Monad.Trans.State
import Control.Monad.IO.Class

tryIO :: IO a -> IO (Either IOException a)
tryIO = try

foo :: StateT String IO Int
foo = do
    s <- get
    liftIO $ readIO s

tryState :: StateT String IO a -> StateT String IO (Either IOException a)
tryState (StateT f) = StateT $ \s0 -> do
    eres <- tryIO $ f s0
    return $ case eres of
        Left e -> (Left e, s0)
        Right (x, s1) -> (Right x, s1)

main :: IO ()
main = flip evalStateT "123" $ do
    let go = tryState foo >>= liftIO . print

    go
    put "456"
    go
    put "hello"
    go
    put "789"
    go
```
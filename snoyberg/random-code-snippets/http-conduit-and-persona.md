# http-conduit and Persona

```active haskell
import Network.HTTP.Conduit
import Control.Concurrent (threadDelay)
import Control.Monad.IO.Class

main = do
    req' <- parseUrl "https://verifier.login.persona.org/verify"
    let req = req' { checkStatus = \_ _ _ -> Nothing }
    withManager $ \man -> do
        httpLbs req man >>= liftIO . print
        httpLbs req man >>= liftIO . print
        httpLbs req man >>= liftIO . print

        liftIO $ threadDelay 10000000
        httpLbs req man >>= liftIO . print
        httpLbs req man >>= liftIO . print
```
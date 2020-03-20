# http-conduit buffering (tutorial)

```haskell active
{-# LANGUAGE OverloadedStrings #-}

import           Blaze.ByteString.Builder (Builder, flush, fromByteString)
import           Control.Concurrent       (threadDelay)
import           Control.Concurrent.Async (withAsync)
import           Control.Monad.IO.Class   (MonadIO, liftIO)
import           Data.ByteString          (ByteString)
import           Data.Conduit             (ResourceT, Source, yield, ($$))
import qualified Data.Conduit.List        as CL
import           Data.Monoid              ((<>))
import           Network.HTTP.Conduit     (RequestBody (RequestBodySourceChunked),
                                           httpLbs, method, parseUrl,
                                           requestBody, withManager)
import           Network.HTTP.Types       (status200)
import qualified Network.Wai              as Wai
import           Network.Wai.Handler.Warp (run)

source :: Source (ResourceT IO) Builder
source = do say "a"; wait; say "b"; wait; say "c"

wait :: MonadIO m => m ()
wait = liftIO $ threadDelay 1000000

say :: Monad m => ByteString -> Source m Builder
say bs = yield (fromByteString bs <> flush)

app :: Wai.Application
app req = do
    Wai.requestBody req $$ CL.mapM_ (liftIO . print)
    return $ Wai.responseLBS status200 [] "done"

main :: IO ()
main = withAsync (run 12345 app) $ const $ do
    threadDelay 100000
    putStrLn "Making HTTP request"
    res <- do
        req <- parseUrl "http://localhost:12345/"
        withManager $ httpLbs req
            { method = "POST"
            , requestBody = RequestBodySourceChunked source
            }
    print res
```
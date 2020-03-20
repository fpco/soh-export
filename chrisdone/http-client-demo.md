# HTTP client demo

``` haskell active autorun
import Network.HTTP.Client
import Network.HTTP.Client.TLS
import qualified Data.ByteString.Lazy.Char8 as L8
main = do req <- parseUrl "https://reddit.com/"
          resp <- withManager tlsManagerSettings (httpLbs req)
          L8.putStrLn (responseBody resp)
```
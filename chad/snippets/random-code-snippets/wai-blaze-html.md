# WAI and blaze-html

```haskell active web
{-# LANGUAGE OverloadedStrings #-}

module Main where 

import Network.Wai (Application, Response (ResponseBuilder)) 
import Network.HTTP.Types (status200) 
import Network.HTTP.Types.Header (hContentType, hContentLength, hConnection) 
import Network.Wai.Handler.Warp (run) 
import Blaze.ByteString.Builder (fromByteString, fromLazyByteString) 
import qualified Data.ByteString.Char8 as BS (pack, length) 

import qualified Text.Blaze.Html5 as H
import qualified Text.Blaze.Html5.Attributes as A
import Text.Blaze.Html.Renderer.Utf8

import qualified Data.ByteString.Lazy as LB
import System.Environment (getEnv)

application:: Application 
application _ = return $   
    ResponseBuilder status200 [(hContentType, "text/html"),  
                               (hConnection, "keep-alive")]  
                    $ renderHtmlBuilder rooth

rooth :: H.Html
rooth = H.docTypeHtml $ do
    H.body $ do
        H.h1 "Hello"

main:: IO ()  
main = getEnv "PORT" >>= flip run application . read
```
# Simplest FastCGI-server

We want to build simplest FastCGI-server. Assumed that we already have a web-server (for example, Apache with `mod_fastcgi` module).

### Solution

We have to install [`direct-fastcgi`](http://hackage.haskell.org/package/direct-fastcgi) package, and after that:


```haskell
module Main where

import Control.Concurrent (forkIO)
import Network.FastCGI

handleRequest :: FastCGI ()
handleRequest = do
    setResponseHeaderSource HttpContentType "application/json" 
    fPutStr "{ \"Hello from\": \"Haskell world!\" }"

main :: IO ()
main = acceptLoop forkIO handleRequest
```

After building, copy the executable in your `DocumentRoot` and go to the server (via your configured alias). The result in the browser will be:


```bash
{ "Hello from": "Haskell world!" }
```

Header info:

```bash
Date: Mon, 12 May 2014 19:28:05 GMT
Server: Apache/2.2.22 (Ubuntu)
Vary: Accept-Encoding
Content-Encoding: gzip
Content-Length: 43
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: application/json

200 OK
```

That's all.
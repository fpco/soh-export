# http-conduit




  



Most of Yesod is about serving content over HTTP. But that's only half the story: someone has to receive it. And even when you're writing a web app, sometimes that someone will be you. If you want to consume content from other services or interact with RESTful APIs, you'll need to write client code. And the recommended approach for that is [http-conduit](http://hackage.haskell.org/package/http-conduit).




  



This chapter is not directly connected to Yesod, and will be generally useful for anyone wanting to make HTTP requests.




 



## Synopsis



  



```haskell active
{-# LANGUAGE OverloadedStrings #-}
import Network.HTTP.Conduit -- the main module

-- The streaming interface uses conduits
import Data.Conduit
import Data.Conduit.Binary (sinkFile)

import qualified Data.ByteString.Lazy as L
import Control.Monad.IO.Class (liftIO)

main :: IO ()
main = do
    -- Simplest query: just download the information from the given URL as a
    -- lazy ByteString.
    simpleHttp "http://www.example.com/foo.txt" >>= L.writeFile "foo.txt"

    -- Use the streaming interface instead. We need to run all of this inside a
    -- ResourceT, to ensure that all our connections get properly cleaned up in
    -- the case of an exception.
    runResourceT $ do
        -- We need a Manager, which keeps track of open connections. simpleHttp
        -- creates a new manager on each run (i.e., it never reuses
        -- connections).
        manager <- liftIO $ newManager def

        -- A more efficient version of the simpleHttp query above. First we
        -- parse the URL to a request.
        req <- liftIO $ parseUrl "http://www.example.com/foo.txt"

        -- Now get the response
        res <- http req manager

        -- And finally stream the value to a file
        responseBody res $$+- sinkFile "foo.txt"

        -- Make it a POST request, don't follow redirects, and accept any
        -- status code.
        let req2 = req
                { method = "POST"
                , redirectCount = 0
                , checkStatus = \_ _ _ -> Nothing
                }
        res2 <- http req2 manager
        responseBody res2 $$+- sinkFile "post-foo.txt"

```




 



## Concepts



  



The simplest way to make a request in <code>http-conduit</code> is with the <code>simpleHttp</code> function. This function takes a <code>String</code> giving a URL and returns a <code>ByteString</code> with the contents of that URL. But under the surface, there are a few more steps:




  



* A new connection <code>Manager</code> is allocated.



* The URL is parsed to a <code>Request</code>. If the URL is invalid, then an exception is thrown.



* The HTTP request is made, following any redirects from the server.



* If the response has a status code outside the 200-range, an exception is thrown.



* The response body is read into memory and returned.



* <code>runResourceT</code> is called, which will free up any resources (e.g., the open socket to the server).




  



If you want more control of what's going on, then you can configure any of the steps above (plus a few more) by explicitly creating a <code>Request</code> value, allocating your <code>Manager</code> manually, and using the <code>http</code> and <code>httpLbs</code> functions.




 



## Request



  



The easiest way to creating a <code>Request</code> is with the <code>parseUrl</code> function. This function will return a value in any <code>Failure</code> monad, such as <code>Maybe</code> or <code>IO</code>. The last of those is the most commonly used, and results in a runtime exception whenever an invalid URL is provided. However, you can use a different monad if, for example, you want to validate user input.




  



```haskell active
import Network.HTTP.Conduit
import System.Environment (getArgs)
import qualified Data.ByteString.Lazy as L
import Control.Monad.IO.Class (liftIO)

main :: IO ()
main = do
    args <- getArgs
    case args of
        [urlString] ->
            case parseUrl urlString of
                Nothing -> putStrLn "Sorry, invalid URL"
                Just req -> withManager $ \manager -> do
                    res <- httpLbs req manager
                    liftIO $ L.putStr $ responseBody res
        _ -> putStrLn "Sorry, please provide exactly one URL"

```




  



The <code>Request</code> type is abstract so that <code>http-conduit</code> can add new settings in the future without breaking the API (see the Settings Type chapter for more information). In order to make changes to individual records, you use record notation. For example, a modification to our program that issues <code>HEAD</code> requests and prints the response headers would be:




  



```haskell active
{-# LANGUAGE OverloadedStrings #-}
import Network.HTTP.Conduit
import System.Environment (getArgs)
import qualified Data.ByteString.Lazy as L
import Control.Monad.IO.Class (liftIO)

main :: IO ()
main = do
    args <- getArgs
    case args of
        [urlString] ->
            case parseUrl urlString of
                Nothing -> putStrLn "Sorry, invalid URL"
                Just req -> withManager $ \manager -> do
                    let reqHead = req { method = "HEAD" }
                    res <- http reqHead manager
                    liftIO $ do
                        print $ responseStatus res
                        mapM_ print $ responseHeaders res
        _ -> putStrLn "Sorry, please provide example one URL"

```




  



There are a number of different configuration settings in the API, some noteworthy ones are:




  



<dl>




   




    



<dt>



proxy



</dt>




    



<dd>



Allows you to pass the request through the given proxy server.



</dd>




   




   




    



<dt>



redirectCount



</dt>




    



<dd>



Indicate how many redirects to follow. Default is 10.



</dd>




   




   




    



<dt>



checkStatus



</dt>




    



<dd>



Check the status code of the return value. By default, gives an exception for any non-2XX response.



</dd>




   




   




    



<dt>



requestBody



</dt>




    



<dd>



The request body to be sent. Be sure to also update the <code>method</code>. For the common case of url-encoded data, you can use the <code>urlEncodedBody</code> function.



</dd>




   




  



</dl>




 



## Manager



  



The connection manager allows you to reuse connections. When making multiple queries to a single server (e.g., accessing Amazon S3), this can be critical for creating efficient code. A manager will keep track of multiple connections to a given server (taking into account port and SSL as well), automatically reaping unused connections as needed. When you make a request, <code>http-conduit</code> first tries to check out an existing connection. When you're finished with the connection (if the server allows keep-alive), the connection is returned to the manager. If anything goes wrong, the connection is closed.




  



To keep our code exception-safe, we use the <code>ResourceT</code> monad transformer. All this means for you is that your code needs to be wrapped inside a call to <code>runResourceT</code>, either implicitly or explicitly, and that code inside that block will need to <code>liftIO</code> to perform normal IO actions.




  



There are two ways you can get ahold of a manager. <code>newManager</code> will return a manager that will not be automatically closed (you can use <code>closeManager</code> to do so manually), while <code>withManager</code> will start a new <code>ResourceT</code> block, allow you to use the manager, and then automatically close the <code>ResourceT</code> when you're done. If you want to use a <code>ResourceT</code> for an entire application, and have no need to close it, you should probably use <code>newManager</code>.




  



One other thing to point out: you obviously don't want to create a new manager for each and every request; that would defeat the whole purpose. You should create your <code>Manager</code> early and then share it.




 



## Response



  



The <code>Response</code> datatype has three pieces of information: the status code, the response headers, and the response body. The first two are straight-forward; let's discuss the body.




  



The <code>Response</code> type has a type variable to allow the response body to be of multiple types. If you want to use <code>http-conduit</code>'s streaming interface, you want this to be a <code>Source</code>. For the simple interface, it will be a lazy <code>ByteString</code>. One thing to note is that, even though we use a lazy <code>ByteString</code>, *the entire response is held in memory*. In other words, we perform no lazy I/O in this package.




  



The <code>conduit</code> package does provide a lazy module which would allow you to read this value in lazily, but like any lazy I/O, it's a bit unsafe, and definitely non-deterministic. If you need it though, you can use it.




 



## http and httpLbs



  



So let's tie it together. The <code>http</code> function gives you access to the streaming interface (i.e., it returns a <code>Response</code> using a <code>BufferedSource</code>) while <code>httpLbs</code> returns a lazy <code>ByteString</code>. Both of these return values in the <code>ResourceT</code> transformer so that they can access the <code>Manager</code> and have connections handled properly in the case of exceptions.




  



If you want to ignore the remainder of a large response body, you can connect to the <code>sinkNull</code> sink. The underlying connection will automatically be closed, preventing you from having to read a large response body over the network.




 


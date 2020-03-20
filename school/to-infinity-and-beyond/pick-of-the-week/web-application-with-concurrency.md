# Web application with concurrency

The best way to learn is to learn from example, therefore I came up with one that combines small bits from web development, networking, foreign interfaces, concurrency and crypto.

The goal is to write a simple web application that would download an arbitary file from the requested url and compute its md5 hash sum concurrently, while also updating download progress status and displaying it in the browser.

First let's define our app
``` haskell
    data App = App { progress :: Ptr Double }
```

It has just one variable in the state that will contain our download progress in percenteges. The reason why we use pointer is because we gonna use this state also as a state for the foreign function call.


Next we define our paths. Basically we need three of them:

1. Home page will contain an interface with the submition form
2. Path to the download controller  that will do all the job and return a JSON object with the hash
3. Path to the status controller that will simply return the current progress also as a JSON object
``` haskell
    mkYesod "App" [parseRoutes|
        / HomeR GET
        /download DownloadR GET
        /status StatusR GET
    |]
```
Our main code lies inside download controller

``` haskell
    getDownloadR = do
```

First we extract a required url field from get parameters

``` haskell
      url <- runInputGet $ ireq urlField "url"
```

and our pointer from the Yesod environment

``` haskell
      progress <- progress <$> getYesod
```

In order to be able concurrently feed the function that computes the hash with the downloaded chunks of data, we need a special concurrent queue.
An `stm-chans` hackage provides exactly the thing we need inside of [Control.Concurrent.STM.TQueue](http://hackage.haskell.org/package/stm-2.4.2/docs/Control-Concurrent-STM-TQueue.html) module. It defines a bunch of queues, we need an unbounded finite (closable) queue aka TMQueue.
Unbounded because we don't need to know the size of the file we download, finite because the file is finite, and closable means that we can tell when the queue stops (file finished downloading).

So let's make a new queue.
``` haskell
      queue <- lift newTMQueueIO 
```
We need to lift it from `IO` to a `Handler` monad for our app.
      
Next we should start our download process. For this purpose we use [download-curl](http://hackage.haskell.org/package/download-curl-0.1.4/docs/Network-Curl-Download.html) hackage which is just a convenient binding to the `curlib` library. 
``` haskell
       openURIWithOpts 
          [ CurlProgressFunction progressFunction, 
            CurlProgressData $ castPtr progress, 
            CurlNoProgress False,
            CurlWriteFunction $ writeFunction queue
          ] $ unpack url
```
The options tell curl to use external functions to manage download progress and incoming data. `CurlProgressData` requires address of the state variable, that's why we cast it first from `Ptr Double` to `Ptr ()`. Also we launch this process parallerly in another thread and close the queue after it ends 
``` haskell
      lift $ forkIO $ do 
        openURIWithOpts 
          [ CurlProgressFunction progressFunction, 
            CurlProgressData $ castPtr progress, 
            CurlNoProgress False,
            CurlWriteFunction $ writeFunction queue
          ] $ unpack url
        atomically $ closeTMQueue queue
        return ()
```
Next we convert our queue to a source using `Conduit` and direct it to a sink that gradually computes the hash as the data flows into it   
``` haskell
      h :: MD5Digest <- sourceTMQueue queue $$ sinkHash
```
And finally we return a JSON object with our hash in it
``` haskell
      return $ object ["hash" .= show h]
```
Now let's define our external function calls.

The progress function just writes the percentage to our variable and returns a correct status code
``` haskell
    progressFunction p total now _ _ = do
      poke (castPtr p) $ 100*now/total
      return 0
```
And write function extracts an array from a pointer, converts it to a lazy bytestring and adds it to the queue
``` haskell
    writeFunction queue p s n _ = do
      let size = s*n
      a <- peekArray (fromIntegral size) p
      atomically $ writeTMQueue queue $ pack $ map fromIntegral a
      return size
```
The code for the status controller is also very simple
``` haskell
    getStatusR = do
      progress <- getYesod >>= lift . peek . progress
      return $ object ["progress" .= progress]
```
Read the variable and return JSON.

# Web Sockets

The current code isn't very efficient, because it needs client to request the progress of the download from the server.
The better way would be to make server tell client when to update progress. That's why we gonna use [WebSockets](http://hackage.haskell.org/package/websockets-0.7.4.1/docs/Network-WebSockets.html#t:WebSocketsData) to establish communication between client and server.

First we add another variable to Yesod state
``` haskell
data App = App { progress :: Ptr Double, sink :: MVar (Sink Hybi00) }
```
It is a sink for a web socket connection, that acts as some special handler, so that other threads could send data into websocket through this sink. `Hybi00` is a protocol name.

Inside our controller code we pass this sink to our progress function 

``` haskell
sink <- sink <$> getYesod
...
  CurlProgressFunction $ progressFunction sink,
...
```

Then we update the progress function to send progress data to this sink every time progress changes at least by one percent.
``` haskell
progressFunction sink (castPtr -> p) total now _ _ = do
  let progress = 100*now/total
  lastProgress <- peek p
  if progress - lastProgress > 1 then do
    poke p progress
    sink <- readMVar sink
    sendSink sink $ textData $ tshow progress 
  else
    return ()
  return 0
```

Our main function should initialize the state and start http and websocket servers.
``` haskell
main = do
  progress <- new 0
  sink <- newEmptyMVar
  forkIO $ warpEnv $ App progress sink
  runServer "0.0.0.0" 8000 $ websocketApp sink
```

WebSocket app accepts any request, updates the state and waits forever doing nothing.
``` haskell
websocketApp sink request = do
  acceptRequest request
  getSink >>= liftIO . putMVar sink 
  forever $ do
    _ :: Text <- receiveData
    return ()
```
# code

That's it! And here is the complete code for the app with the web page layout (hamlet + coffescript + angular-js + bootstrap). It contains a form for url submission and two progress bars for both implemented methods, first one updates progress every second and another one that updates when the message comes from the web socket.

You need to cabal install some packages first

    cabal install classy-prelude yesod crypto-conduit download-curl websockets stm-conduit

and follow the installation instructions for [CoffeeScript](http://coffeescript.org/) to be able to run this code.

``` haskell
{-# LANGUAGE NoImplicitPrelude, QuasiQuotes, TemplateHaskell, TypeFamilies, MultiParamTypeClasses, OverloadedStrings, ScopedTypeVariables, ViewPatterns #-}

import ClassyPrelude
import Yesod
import Text.Coffee
import Network.Curl.Download
import Network.Curl.Opts
import Network.WebSockets
import Data.Conduit hiding (Sink)
import Crypto.Conduit (sinkHash)
import Data.Digest.Pure.MD5 (MD5Digest)
import Foreign
import Control.Concurrent (forkIO)
import Control.Concurrent.STM.TMQueue
import Control.Monad.STM
import Data.Conduit.TQueue

data App = App { progress :: Ptr Double, sink :: MVar (Sink Hybi00) }

mkYesod "App" [parseRoutes|
/ HomeR GET
/download DownloadR GET
/status StatusR GET
|]

instance Yesod App where
  defaultLayout widget = do
    pc <- widgetToPageContent $ do
      setTitle "Hash from URL"
      addStylesheetRemote "//netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap.min.css"
      addScriptRemote "//ajax.googleapis.com/ajax/libs/angularjs/1.0.8/angular.min.js"
      toWidget [coffee|
        DownloadCtrl = ($scope, $http, $timeout) ->
          updateProgress = () -> 
            $scope.stop = $timeout(() ->
              $http.get("@{StatusR}")
                .success(
                  (data) ->
                    $scope.progress = data.progress
                    updateProgress()
                )
            , 1000)

          $scope.startDownload = () ->
            ws = new WebSocket("ws://localhost:8000/")
            ws.onmessage = (message) -> 
              $scope.$apply(() -> $scope.progress2 = message.data)
            updateProgress()
            $http.get("@{DownloadR}?url=" + $scope.url)
              .success(
                (data) ->
                  $timeout.cancel($scope.stop)
                  $scope.progress = 0
                  $scope.progress2 = 0
                  $scope.hash = data.hash
              )              
                  |]
      widget
    giveUrlRenderer [hamlet|
      <html ng-app>
        <head>
          <title>#{pageTitle pc}
          ^{pageHead pc}
        <body>
          ^{pageBody pc}
    |]

instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage

renderForm = [whamlet|
<div ng-controller=DownloadCtrl>
  <form ng-submit=startDownload()>
    <div .form-group>
      <label for=url>URL
      <input .form-control required ng-model=url type=url>
      <button .btn type=submit>Download
  <div ."progress progress-striped active">
    <div .progress-bar  role="progressbar" aria-valuenow={{progress}} aria-valuemin=0 aria-valuemax=100 style="width: {{progress}}%">
      <span .sr-only>{{progress}}% Complete
  <div ."progress progress-striped active">
      <div .progress-bar  role="progressbar" aria-valuenow={{progress2}} aria-valuemin=0 aria-valuemax=100 style="width: {{progress2}}%">
        <span .sr-only>{{progress2}}% Complete
  <div>{{hash}}
  <div>{{progress2}}
|]

getHomeR :: Handler Html
getHomeR = defaultLayout renderForm

progressFunction sink (castPtr -> p) total now _ _ = do
  let progress = 100*now/total
  lastProgress <- peek p
  if progress - lastProgress > 1 then do
    poke p progress
    sink <- readMVar sink
    sendSink sink $ textData $ tshow progress 
  else
    return ()
  return 0

writeFunction queue p s n _ = do
  let size = s*n
  a <- peekArray (fromIntegral size) $ castPtr p
  atomically $ writeTMQueue queue $ pack a
  return size

getDownloadR = do
  url <- runInputGet $ ireq urlField "url"
  progress <- progress <$> getYesod
  sink <- sink <$> getYesod
  queue <- lift newTMQueueIO 
  lift $ forkIO $ do 
    openURIWithOpts 
      [ CurlFollowLocation True, 
        CurlAutoReferer True, 
        CurlProgressFunction $ progressFunction sink,
        CurlProgressData $ castPtr progress, 
        CurlNoProgress False,
        CurlWriteFunction $ writeFunction queue
      ] $ unpack url
    atomically $ closeTMQueue queue
    sink <- readMVar sink
    sendSink sink $ close $ asText "Connection closed"
    return ()
  h :: MD5Digest <- sourceTMQueue queue $$ sinkHash
  return $ object ["hash" .= show h]

getStatusR = do
  progress <- getYesod >>= lift . peek . progress
  return $ object ["progress" .= progress]

websocketApp sink request = do
  acceptRequest request
  getSink >>= liftIO . putMVar sink 
  forever $ do
    _ :: Text <- receiveData
    return ()

main = do
  progress <- new 0
  sink <- newEmptyMVar
  forkIO $ warpEnv $ App progress sink
  runServer "0.0.0.0" 8000 $ websocketApp sink
```
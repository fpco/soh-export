# Part 3

This is part 3 of a series where we develop a file hosting web application. We
ended the previous entry with the ability to upload files, but only their
names were stored. Today we are going to store file contents as well. A
summary page for each file will display the contents if possible. The ability
to download files from the server will be added next time.

# Storing File Contents

We need a place to store file contents. One way to go about doing this would
be to create real files on the server's hard drive. We would recommend this
approach if we were storing many files, keeping them for a long period of
time, or wanted to have access to them from outside of the application. To
keep things simple we'll just keep the file contents in memory along with
information about them. This means that our application will reset it's data
each time it runs.

The application state is currently just a list of filenames. Let's extend this
to an association list of file names to contents. Lazy bytestrings are the
most convenient type for our use. Other types such as strict bytestrings or
vectors of bytes would have worked as well.

``` active haskell web
{-# START_FILE Dispatch.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Dispatch where

import Yesod

import Foundation
import Handler.Home

mkYesodDispatch "App" resourcesApp
{-# START_FILE Foundation.hs #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
module Foundation where

import Control.Concurrent.STM
{-hi-}import Data.ByteString.Lazy (ByteString){-/hi-}
import Data.Text (Text)

import Yesod

-- data App = App (TVar [Text])
{-hi-}data App = App (TVar [(Text, ByteString)]){-/hi-}
instance Yesod App

instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage

mkYesodData "App" $(parseRoutesFile "config/routes")

getList :: Handler [Text]
getList = do
    App tstate <- getYesod
    -- liftIO $ readTVarIO tstate
{-hi-}    state <- liftIO $ readTVarIO tstate
    return $ map fst state{-/hi-}

-- addFile :: App -> Text -> Handler ()
{-hi-}addFile :: App -> (Text, ByteString) -> Handler (){-/hi-}
addFile (App tstore) op =
    liftIO . atomically $ do
        modifyTVar tstore $ \ ops -> op : ops
-- /show
{-# START_FILE Main.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Main where

import Control.Concurrent.STM

import Yesod

import Dispatch ()
import Foundation

main :: IO ()
main = do
    tfilenames <- atomically $ newTVar []
    warpEnv $ App tfilenames
{-# START_FILE Handler/Home.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
module Handler.Home where

{-hi-}import Data.Conduit
import Data.Conduit.Binary{-/hi-}
import Data.Default
import Yesod
import Yesod.Default.Util
import Control.Monad.Trans.Resource (runResourceT)

import Foundation

getHomeR :: Handler Html
getHomeR = do
    (formWidget, formEncType) <- generateFormPost uploadForm
    filenames <- getList
    defaultLayout $ do
        setTitle "File Processor"
        $(widgetFileNoReload def "home")

postHomeR :: Handler Html
postHomeR = do
    ((result, _), _) <- runFormPost uploadForm
    case result of
      FormSuccess fi -> do
        app <- getYesod
        -- addFile app $ fileName fi
{-hi-}        fileBytes <- runResourceT $ fileSource fi $$ sinkLbs
        addFile app (fileName fi, fileBytes){-/hi-}
      _ -> return ()
    redirect HomeR

uploadForm = renderDivs $ fileAFormReq "file"
-- /show
{-# START_FILE templates/home.hamlet #-}
-- show
-- /show
<h2>Previously submitted files
$if null filenames
    <p>No files have been uploaded yet.
$else
    <ul>
        $forall filename <- filenames
            <li>#{filename}
<h2>Submit new file
<form method=post action=@{HomeR} enctype=#{formEncType}>
  ^{formWidget}
  <input type="submit" value="Upload">
{-# START_FILE config/routes #-}
-- show
-- /show
/ HomeR GET POST
```

The foundation type, `App`, has been changed to `data App = App (TVar [(Text,
ByteString)])`. This prompted a few trivial changes to the collect of accessor
functions in "Foundation.hs". The type signature for `addFile` was updated,
but its body remained the same. The `getList` action was changed to discard
file contents. Keeping these accessor functions in "Foundation.hs" allows us
to update them without touching code in other modules for the most part.

The `postHomeR` handler is where we extract file contents into a lazy
bytestring. Yesod achieves high performance by allowing our handler to work
with the incoming data stream as it arrives. The line assigning `fileBytes`
converts from Yesod's native byte stream to a lazy `ByteString`.

# File Summary Page

Until now we've been using a single page. We'd like to add a second route for
displaying details about files that have been uploaded. Each file will be
identified by its file name in the URL. The key concept here is that route
handlers are able to accept arguments which Yesod parses from the URL.

``` active haskell web
{-# START_FILE Dispatch.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
module Dispatch where

import Yesod

import Foundation
import Handler.Home
{-hi-}import Handler.Preview{-/hi-}

mkYesodDispatch "App" resourcesApp
-- /show
{-# START_FILE Foundation.hs #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
module Foundation where

import Control.Concurrent.STM
import Data.ByteString.Lazy (ByteString)
import Data.Text (Text)
{-hi-}import qualified Data.Text as Text{-/hi-}

import Yesod

data App = App (TVar [(Text, ByteString)])
instance Yesod App

instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage

mkYesodData "App" $(parseRoutesFile "config/routes")

getList :: Handler [Text]
getList = do
    App tstate <- getYesod
    state <- liftIO $ readTVarIO tstate
    return $ map fst state

addFile :: App -> (Text, ByteString) -> Handler ()
addFile (App tstore) op =
    liftIO . atomically $ do
        modifyTVar tstore $ \ ops -> op : ops

{-hi-}getById :: Text -> Handler ByteString
getById ident = do
    App tstore <- getYesod
    operations <- liftIO $ readTVarIO tstore
    case lookup ident operations of
      Nothing -> notFound
      Just bytes -> return bytes{-/hi-}
-- /show
{-# START_FILE Main.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Main where

import Control.Concurrent.STM

import Yesod

import Dispatch ()
import Foundation

main :: IO ()
main = do
    tfilenames <- atomically $ newTVar []
    warpEnv $ App tfilenames
{-# START_FILE Handler/Home.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Handler.Home where

import qualified Data.ByteString.Lazy as L
import Data.Conduit
import Data.Conduit.Binary
import Data.Default
import Yesod
import Yesod.Default.Util
import Control.Monad.Trans.Resource (runResourceT)

import Foundation

getHomeR :: Handler Html
getHomeR = do
    (formWidget, formEncType) <- generateFormPost uploadForm
    filenames <- getList
    defaultLayout $ do
        setTitle "File Processor"
        $(widgetFileNoReload def "home")

postHomeR :: Handler Html
postHomeR = do
    ((result, _), _) <- runFormPost uploadForm
    case result of
      FormSuccess fi -> do
        app <- getYesod
        fileBytes <- runResourceT $ fileSource fi $$ sinkLbs
        addFile app (fileName fi, fileBytes)
      _ -> return ()
    redirect HomeR

uploadForm = renderDivs $ fileAFormReq "file"
{-# START_FILE Handler/Preview.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}

-- show
{-hi-}module Handler.Preview where

import Control.Exception hiding (Handler)
import qualified Data.ByteString.Lazy as LB
import Data.Default
import Data.Text (Text)
import qualified Data.Text as Text
import qualified Data.Text.Lazy as LT
import qualified Data.Text.Lazy.Encoding as LT
import Text.Blaze
import Yesod
import Yesod.Default.Util

import Foundation

getPreviewR :: Text -> Handler Html
getPreviewR filename = do
    bytes <- getById filename
    defaultLayout $ do
        setTitle . toMarkup $ "File Processor - " `Text.append` filename
        previewBlock <- liftIO $ preview bytes
        $(widgetFileNoReload def "preview")

preview :: LB.ByteString -> IO Widget
preview bytes = do
    eText <- try . evaluate $ LT.decodeUtf8 bytes :: IO (Either SomeException LT.Text)
    return $ case eText of
      Left _ -> errorMessage
      Right text -> [whamlet|<pre>#{text}|]
  where
    errorMessage = [whamlet|<pre>Unable to display file contents.|]{-/hi-}
-- /show
{-# START_FILE templates/home.hamlet #-}
<h2>Previously submitted files
$if null filenames
    <p>No files have been uploaded yet.
$else
    <ul>
        $forall filename <- filenames
$#            <li>#{filename}
{-hi-}            <li>
                <a href=@{PreviewR filename}>#{filename}{-/hi-}
<h2>Submit new file
<form method=post action=@{HomeR} enctype=#{formEncType}>
  ^{formWidget}
  <input type="submit" value="Upload">
{-# START_FILE templates/preview.hamlet #-}
{-hi-}<a href=@{HomeR}>home
<h1>#{filename}
<p>
  ^{previewBlock}{-/hi-}
{-# START_FILE config/routes #-}
/           HomeR    GET POST
{-hi-}/file/#Text PreviewR GET{-/hi-}
```

Two new files were added. One is a Hamlet template for the preview page. The
other is a route handler for it. The "config/routes" file gained an entry to
define the new route. The part reading `#Text` is a placeholder which becomes
an argument to any handler functions. Hooks to link our handlers to route
paths are written for us in "Dispatch.hs", which is why that module imports
the "Handler.Preview" module.

The handler for `PreviewR`, `getPreviewR` should be easy to understand. It
takes an argument which Yesod parses from the supplied URL. It's possible that
no file exists by the given name. In that case `getById` will abort the
handler by sending an error response.

The `preview` action may be difficult to follow. Depending on the type of
file, it would be nice to generate a block of HTML to display a preview of
it. For now we've written support for text documents. The
<hoogle>decodeUtf8</hoogle> function does the conversion, but throws an
exception in the IO monad if the bytestring fails to parse. The reason for
calling <hoogle>evaluate</hoogle> is that we want to ensure the error is caught
immediately.

Our template for the main page was updated so that file names are linked to
their summary page. Try running the example to see how everything works. See
if you can modify the home template so that it includes a hyperlink to a
nonexistant file. We'll include a solution below.

@@@
``` active haskell web
{-# START_FILE Dispatch.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Dispatch where

import Yesod

import Foundation
import Handler.Home
import Handler.Preview

mkYesodDispatch "App" resourcesApp
{-# START_FILE Foundation.hs #-}
-- show
-- /show
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Foundation where

import Control.Concurrent.STM
import Data.ByteString.Lazy (ByteString)
import Data.Text (Text)
import qualified Data.Text as Text

import Yesod

data App = App (TVar [(Text, ByteString)])
instance Yesod App

instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage

mkYesodData "App" $(parseRoutesFile "config/routes")

getList :: Handler [Text]
getList = do
    App tstate <- getYesod
    state <- liftIO $ readTVarIO tstate
    return $ map fst state

addFile :: App -> (Text, ByteString) -> Handler ()
addFile (App tstore) op =
    liftIO . atomically $ do
        modifyTVar tstore $ \ ops -> op : ops

getById :: Text -> Handler ByteString
getById ident = do
    App tstore <- getYesod
    operations <- liftIO $ readTVarIO tstore
    case lookup ident operations of
      Nothing -> notFound
      Just bytes -> return bytes
{-# START_FILE Main.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Main where

import Control.Concurrent.STM

import Yesod

import Dispatch ()
import Foundation

main :: IO ()
main = do
    tfilenames <- atomically $ newTVar []
    warpEnv $ App tfilenames
{-# START_FILE Handler/Home.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Handler.Home where

import qualified Data.ByteString.Lazy as L
import Data.Conduit
import Data.Conduit.Binary
import Data.Default
import Yesod
import Yesod.Default.Util
import Control.Monad.Trans.Resource (runResourceT)

import Foundation

getHomeR :: Handler Html
getHomeR = do
    (formWidget, formEncType) <- generateFormPost uploadForm
    filenames <- getList
    defaultLayout $ do
        setTitle "File Processor"
        $(widgetFileNoReload def "home")

postHomeR :: Handler Html
postHomeR = do
    ((result, _), _) <- runFormPost uploadForm
    case result of
      FormSuccess fi -> do
        app <- getYesod
        fileBytes <- runResourceT $ fileSource fi $$ sinkLbs
        addFile app (fileName fi, fileBytes)
      _ -> return ()
    redirect HomeR

uploadForm = renderDivs $ fileAFormReq "file"
{-# START_FILE Handler/Preview.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}

module Handler.Preview where

import Control.Exception hiding (Handler)
import qualified Data.ByteString.Lazy as LB
import Data.Default
import Data.Text (Text)
import qualified Data.Text as Text
import qualified Data.Text.Lazy as LT
import qualified Data.Text.Lazy.Encoding as LT
import Text.Blaze
import Yesod
import Yesod.Default.Util

import Foundation

getPreviewR :: Text -> Handler Html
getPreviewR filename = do
    bytes <- getById filename
    defaultLayout $ do
        setTitle . toMarkup $ "File Processor - " `Text.append` filename
        previewBlock <- liftIO $ preview bytes
        $(widgetFileNoReload def "preview")

preview :: LB.ByteString -> IO Widget
preview bytes = do
    eText <- try . evaluate $ LT.decodeUtf8 bytes :: IO (Either SomeException LT.Text)
    return $ case eText of
      Left _ -> errorMessage
      Right text -> [whamlet|<pre>#{text}|]
  where
    errorMessage = [whamlet|<pre>Unable to display file contents.|]
{-# START_FILE templates/home.hamlet #-}
<h2>Previously submitted files
<ul>
{-hi-}  <li>
    <a href=@{PreviewR "missing.txt"}>missing.txt{-/hi-}
  $forall filename <- filenames
    <li>
      <a href=@{PreviewR filename}>#{filename}
<h2>Submit new file
<form method=post action=@{HomeR} enctype=#{formEncType}>
  ^{formWidget}
  <input type="submit" value="Upload">
{-# START_FILE templates/preview.hamlet #-}
-- show
-- /show
<a href=@{HomeR}>home
<h1>#{filename}
<p>
  ^{previewBlock}
{-# START_FILE config/routes #-}
-- show
-- /show
/           HomeR    GET POST
/file/#Text PreviewR GET
```
@@@

# Improved Styling

The application is really coming along. We're able to upload files to a
server, and if they are text files we can read the contents within the
browser. In the next part of this series we'll show you how to turn things
around and let users download files as well. Before this we think it's time to
improve the appearance of our application with CSS styling.

## Local Style Sheets

Yesod supports a template language for writing CSS called Cassius. We've
been using the <hoogle>widgetFileNoReload</hoogle> template haskell function
to load Hamlet templates. As long as you give your Cassius files the same name
they will be added as well. This is also true of JavaScript. As an example,
The following additional file would add styling to our `<pre>` tag on the
summary page.

Go ahead and upload a text file, and then take a look at its summary page to
see the new styling.

``` active haskell web
{-# START_FILE Dispatch.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Dispatch where

import Yesod

import Foundation
import Handler.Home
import Handler.Preview

mkYesodDispatch "App" resourcesApp
{-# START_FILE Foundation.hs #-}
-- show
-- /show
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Foundation where

import Control.Concurrent.STM
import Data.ByteString.Lazy (ByteString)
import Data.Text (Text)
import qualified Data.Text as Text

import Yesod

data App = App (TVar [(Text, ByteString)])
instance Yesod App

instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage

mkYesodData "App" $(parseRoutesFile "config/routes")

getList :: Handler [Text]
getList = do
    App tstate <- getYesod
    state <- liftIO $ readTVarIO tstate
    return $ map fst state

addFile :: App -> (Text, ByteString) -> Handler ()
addFile (App tstore) op =
    liftIO . atomically $ do
        modifyTVar tstore $ \ ops -> op : ops

getById :: Text -> Handler ByteString
getById ident = do
    App tstore <- getYesod
    operations <- liftIO $ readTVarIO tstore
    case lookup ident operations of
      Nothing -> notFound
      Just bytes -> return bytes
{-# START_FILE Main.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Main where

import Control.Concurrent.STM

import Yesod

import Dispatch ()
import Foundation

main :: IO ()
main = do
    tfilenames <- atomically $ newTVar []
    warpEnv $ App tfilenames
{-# START_FILE Handler/Home.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Handler.Home where

import qualified Data.ByteString.Lazy as L
import Data.Conduit
import Data.Conduit.Binary
import Data.Default
import Yesod
import Yesod.Default.Util
import Control.Monad.Trans.Resource (runResourceT)

import Foundation

getHomeR :: Handler Html
getHomeR = do
    (formWidget, formEncType) <- generateFormPost uploadForm
    filenames <- getList
    defaultLayout $ do
        setTitle "File Processor"
        $(widgetFileNoReload def "home")

postHomeR :: Handler Html
postHomeR = do
    ((result, _), _) <- runFormPost uploadForm
    case result of
      FormSuccess fi -> do
        app <- getYesod
        fileBytes <- runResourceT $ fileSource fi $$ sinkLbs
        addFile app (fileName fi, fileBytes)
      _ -> return ()
    redirect HomeR

uploadForm = renderDivs $ fileAFormReq "file"
{-# START_FILE Handler/Preview.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}

module Handler.Preview where

import Control.Exception hiding (Handler)
import qualified Data.ByteString.Lazy as LB
import Data.Default
import Data.Text (Text)
import qualified Data.Text as Text
import qualified Data.Text.Lazy as LT
import qualified Data.Text.Lazy.Encoding as LT
import Text.Blaze
import Yesod
import Yesod.Default.Util

import Foundation

getPreviewR :: Text -> Handler Html
getPreviewR filename = do
    bytes <- getById filename
    defaultLayout $ do
        setTitle . toMarkup $ "File Processor - " `Text.append` filename
        previewBlock <- liftIO $ preview bytes
        $(widgetFileNoReload def "preview")

preview :: LB.ByteString -> IO Widget
preview bytes = do
    eText <- try . evaluate $ LT.decodeUtf8 bytes :: IO (Either SomeException LT.Text)
    return $ case eText of
      Left _ -> errorMessage
      Right text -> [whamlet|<pre>#{text}|]
  where
    errorMessage = [whamlet|<pre>Unable to display file contents.|]
{-# START_FILE templates/home.hamlet #-}
-- show
-- /show
<h2>Previously submitted files
$if null filenames
    <p>No files have been uploaded yet.
$else
    <ul>
        $forall filename <- filenames
            <li>
                <a href=@{PreviewR filename}>#{filename}
<h2>Submit new file
<form method=post action=@{HomeR} enctype=#{formEncType}>
  ^{formWidget}
  <input type="submit" value="Upload">
{-# START_FILE templates/preview.hamlet #-}
-- show
-- /show
<a href=@{HomeR}>home
<h1>#{filename}
<p>
  ^{previewBlock}
{-# START_FILE templates/preview.cassius #-}
{-hi-}/* add rounded grey box around text */
pre
    -webkit-border-radius: 5pt
    -moz-border-radius: 5pt
    border-radius: 5pt
    border: 1pt solid #999
    background: #DDD
    margin: 1em
    padding: 1em
    white-space: pre-wrap{-/hi-}
{-# START_FILE config/routes #-}
-- show
-- /show
/           HomeR    GET POST
/file/#Text PreviewR GET
```

## Global Style Sheets

For small projects like this one a single, global style sheet is
preferred. We'll delete the "templates/preview.cassius" file. Even in larger
projects it's likely that you will have style sheets which should be applied
to every page. To do this we will customize the behavior of
<hoogle>defaultLayout</hoogle>. This function does just what it's name
implies. It takes whatever widget content you define, and adds it in addition
to it's own collection of Hamlet, Cassius, and Julius templates.

``` active haskell web
{-# START_FILE Dispatch.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Dispatch where

import Yesod

import Foundation
import Handler.Home
import Handler.Preview

mkYesodDispatch "App" resourcesApp
{-# START_FILE Foundation.hs #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
module Foundation where

import Control.Concurrent.STM
import Data.ByteString.Lazy (ByteString)
{-hi-}import Data.Default{-/hi-}
import Data.Text (Text)
import qualified Data.Text as Text
{-hi-}import Text.Hamlet{-/hi-}
import Yesod
{-hi-}import Yesod.Default.Util{-/hi-}

data App = App (TVar [(Text, ByteString)])

-- instance Yesod App
{-hi-}instance Yesod App where
  defaultLayout widget = do
    pc <- widgetToPageContent $ $(widgetFileNoReload def "default-layout")
    withUrlRenderer $(hamletFile "templates/default-layout-wrapper.hamlet"){-/hi-}

instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage

mkYesodData "App" $(parseRoutesFile "config/routes")

getList :: Handler [Text]
getList = do
    App tstate <- getYesod
    state <- liftIO $ readTVarIO tstate
    return $ map fst state

addFile :: App -> (Text, ByteString) -> Handler ()
addFile (App tstore) op =
    liftIO . atomically $ do
        modifyTVar tstore $ \ ops -> op : ops

getById :: Text -> Handler ByteString
getById ident = do
    App tstore <- getYesod
    operations <- liftIO $ readTVarIO tstore
    case lookup ident operations of
      Nothing -> notFound
      Just bytes -> return bytes
-- /show
{-# START_FILE Main.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Main where

import Control.Concurrent.STM

import Yesod

import Dispatch ()
import Foundation

main :: IO ()
main = do
    tfilenames <- atomically $ newTVar []
    warpEnv $ App tfilenames
{-# START_FILE Handler/Home.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Handler.Home where

import qualified Data.ByteString.Lazy as L
import Data.Conduit
import Data.Conduit.Binary
import Data.Default
import Yesod
import Yesod.Default.Util
import Control.Monad.Trans.Resource (runResourceT)

import Foundation

getHomeR :: Handler Html
getHomeR = do
    (formWidget, formEncType) <- generateFormPost uploadForm
    filenames <- getList
    defaultLayout $ do
        setTitle "File Processor"
        $(widgetFileNoReload def "home")

postHomeR :: Handler Html
postHomeR = do
    ((result, _), _) <- runFormPost uploadForm
    case result of
      FormSuccess fi -> do
        app <- getYesod
        fileBytes <- runResourceT $ fileSource fi $$ sinkLbs
        addFile app (fileName fi, fileBytes)
      _ -> return ()
    redirect HomeR

uploadForm = renderDivs $ fileAFormReq "file"
{-# START_FILE Handler/Preview.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}

module Handler.Preview where

import Control.Exception hiding (Handler)
import qualified Data.ByteString.Lazy as LB
import Data.Default
import Data.Text (Text)
import qualified Data.Text as Text
import qualified Data.Text.Lazy as LT
import qualified Data.Text.Lazy.Encoding as LT
import Text.Blaze
import Yesod
import Yesod.Default.Util

import Foundation

getPreviewR :: Text -> Handler Html
getPreviewR filename = do
    bytes <- getById filename
    defaultLayout $ do
        setTitle . toMarkup $ "File Processor - " `Text.append` filename
        previewBlock <- liftIO $ preview bytes
        $(widgetFileNoReload def "preview")

preview :: LB.ByteString -> IO Widget
preview bytes = do
    eText <- try . evaluate $ LT.decodeUtf8 bytes :: IO (Either SomeException LT.Text)
    return $ case eText of
      Left _ -> errorMessage
      Right text -> [whamlet|<pre>#{text}|]
  where
    errorMessage = [whamlet|<pre>Unable to display file contents.|]
{-# START_FILE templates/default-layout.hamlet #-}
{-hi-}^{widget}{-/hi-}
{-# START_FILE templates/default-layout-wrapper.hamlet #-}
{-hi-}$newline never
$doctype 5
<html>
    <head>
        <title>#{pageTitle pc}
        ^{pageHead pc}
    <body>
        ^{pageBody pc}{-/hi-}
{-# START_FILE templates/home.hamlet #-}
-- show
-- /show
<h2>Previously submitted files
$if null filenames
    <p>No files have been uploaded yet.
$else
    <ul>
        $forall filename <- filenames
            <li>
                <a href=@{PreviewR filename}>#{filename}
<h2>Submit new file
<form method=post action=@{HomeR} enctype=#{formEncType}>
  ^{formWidget}
  <input type="submit" value="Upload">
{-# START_FILE templates/preview.hamlet #-}
-- show
-- /show
<a href=@{HomeR}>home
<h1>#{filename}
<p>
  ^{previewBlock}
{-# START_FILE config/routes #-}
-- show
-- /show
/           HomeR    GET POST
/file/#Text PreviewR GET
```

If you run the previous example you will see that nothing has changed. We've
redefined <hoogle>defaultLayout</hoogle> to do essentially the same thing as
it normally does. The function takes a widget as an argument to be placed in
the output HTML's `<body>` tag.

Two template files are used because of the way widgets work. A widget is a
collection of HTML, CSS, and JavaScript. You will mostly be working with
widgets when using Yesod, but the <hoogle>defaultLayout</hoogle> is one place
where widgets cannot be used. By calling <hoogle>widgetToPageContent</hoogle>,
and then inserting that into a non-widget hamlet file we will be able to treat
our "default-layout" template as a widget.

Now that we've done this we can simply add a file named
"default-layout.cassius" to apply global style sheets:

``` active haskell web
{-# START_FILE Dispatch.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Dispatch where

import Yesod

import Foundation
import Handler.Home
import Handler.Preview

mkYesodDispatch "App" resourcesApp
{-# START_FILE Foundation.hs #-}
-- show
-- /show
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Foundation where

import Control.Concurrent.STM
import Data.ByteString.Lazy (ByteString)
import Data.Default
import Data.Text (Text)
import qualified Data.Text as Text
import Text.Hamlet
import Yesod
import Yesod.Default.Util

data App = App (TVar [(Text, ByteString)])

-- instance Yesod App
instance Yesod App where
  defaultLayout widget = do
    pc <- widgetToPageContent $ $(widgetFileNoReload def "default-layout")
    withUrlRenderer $(hamletFile "templates/default-layout-wrapper.hamlet")

instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage

mkYesodData "App" $(parseRoutesFile "config/routes")

getList :: Handler [Text]
getList = do
    App tstate <- getYesod
    state <- liftIO $ readTVarIO tstate
    return $ map fst state

addFile :: App -> (Text, ByteString) -> Handler ()
addFile (App tstore) op =
    liftIO . atomically $ do
        modifyTVar tstore $ \ ops -> op : ops

getById :: Text -> Handler ByteString
getById ident = do
    App tstore <- getYesod
    operations <- liftIO $ readTVarIO tstore
    case lookup ident operations of
      Nothing -> notFound
      Just bytes -> return bytes
{-# START_FILE Main.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Main where

import Control.Concurrent.STM

import Yesod

import Dispatch ()
import Foundation

main :: IO ()
main = do
    tfilenames <- atomically $ newTVar []
    warpEnv $ App tfilenames
{-# START_FILE Handler/Home.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Handler.Home where

import qualified Data.ByteString.Lazy as L
import Data.Conduit
import Data.Conduit.Binary
import Data.Default
import Yesod
import Yesod.Default.Util
import Control.Monad.Trans.Resource (runResourceT)

import Foundation

getHomeR :: Handler Html
getHomeR = do
    (formWidget, formEncType) <- generateFormPost uploadForm
    filenames <- getList
    defaultLayout $ do
        setTitle "File Processor"
        $(widgetFileNoReload def "home")

postHomeR :: Handler Html
postHomeR = do
    ((result, _), _) <- runFormPost uploadForm
    case result of
      FormSuccess fi -> do
        app <- getYesod
        fileBytes <- runResourceT $ fileSource fi $$ sinkLbs
        addFile app (fileName fi, fileBytes)
      _ -> return ()
    redirect HomeR

uploadForm = renderDivs $ fileAFormReq "file"
{-# START_FILE Handler/Preview.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}

module Handler.Preview where

import Control.Exception hiding (Handler)
import qualified Data.ByteString.Lazy as LB
import Data.Default
import Data.Text (Text)
import qualified Data.Text as Text
import qualified Data.Text.Lazy as LT
import qualified Data.Text.Lazy.Encoding as LT
import Text.Blaze
import Yesod
import Yesod.Default.Util

import Foundation

getPreviewR :: Text -> Handler Html
getPreviewR filename = do
    bytes <- getById filename
    defaultLayout $ do
        setTitle . toMarkup $ "File Processor - " `Text.append` filename
        previewBlock <- liftIO $ preview bytes
        $(widgetFileNoReload def "preview")

preview :: LB.ByteString -> IO Widget
preview bytes = do
    eText <- try . evaluate $ LT.decodeUtf8 bytes :: IO (Either SomeException LT.Text)
    return $ case eText of
      Left _ -> errorMessage
      Right text -> [whamlet|<pre>#{text}|]
  where
    errorMessage = [whamlet|<pre>Unable to display file contents.|]
{-# START_FILE templates/default-layout.cassius #-}
{-hi-}body
    font-family: Tahoma, Geneva, sans-serif
    font-size: 1pc
form
    clear: both
    margin:auto
    position:relative
    text-decoration: none
    -webkit-border-radius: 5pt
    -moz-border-radius: 5pt
    border-radius: 5pt
    padding:1em
    border: 1pt solid #999
    border: inset 1pt solid #333
    /* Force form elements to be consistent with each other */
    input, textarea, select, button
        margin: 1pt
        -webkit-box-sizing: border-box
        -moz-box-sizing: border-box
        box-sizing: border-box
    select
        width: 100%
    input
        display:block
        border: 1pt solid #999
    input[type=submit]
        float: right
        background: #09C
        color: #fff
        -webkit-border-radius: 5pt
        -moz-border-radius: 5pt
        border-radius: 5pt
        border: 1pt solid #999
    /* Change color on mouseover */
    input[type=submit]:hover
        background:#fff
        color:#09c

/* force bottom border to extend below floating elements */
form::after
    content: ""
    display: block
    visibility: hidden
    clear: both{-/hi-}

/* add rounded grey box around text */
pre
    -webkit-border-radius: 5pt
    -moz-border-radius: 5pt
    border-radius: 5pt
    border: 1pt solid #999
    background: #DDD
    margin: 1em
    padding: 1em
    white-space: pre-wrap
{-# START_FILE templates/default-layout.hamlet #-}
-- show
-- /show
^{widget}
{-# START_FILE templates/default-layout-wrapper.hamlet #-}
-- show
-- /show
$newline never
$doctype 5
<html>
    <head>
        <title>#{pageTitle pc}
        ^{pageHead pc}
    <body>
        ^{pageBody pc}
{-# START_FILE templates/home.hamlet #-}
-- show
-- /show
<h2>Previously submitted files
$if null filenames
    <p>No files have been uploaded yet.
$else
    <ul>
        $forall filename <- filenames
            <li>
                <a href=@{PreviewR filename}>#{filename}
<h2>Submit new file
<form method=post action=@{HomeR} enctype=#{formEncType}>
  ^{formWidget}
  <input type="submit" value="Upload">
{-# START_FILE templates/preview.hamlet #-}
-- show
-- /show
<a href=@{HomeR}>home
<h1>#{filename}
<p>
  ^{previewBlock}
{-# START_FILE config/routes #-}
-- show
-- /show
/           HomeR    GET POST
/file/#Text PreviewR GET
```

The main purpose of this styling is to improve the appearance of the upload
form. The best resource on Cassius templates is Michael Snoyman's book. If you
are familiar with CSS, this example should give you an idea of what Cassius
Syntax looks like.

# Summary

What do you think would happen if 2 files having the same name were uploaded?
As things stand they would both appear in the list, but only one would be
accessible. In the next and final entry of this series we will fix this
problem by keying on unique identifiers rather than filenames. We'll also add
a link to the summary page for downloading files. Finally, we'll expand on the
`preview` action so that images can be displayed before being downloaded.
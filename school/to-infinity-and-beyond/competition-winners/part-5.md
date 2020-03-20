# Building a File Hosting Service in Yesod - Part 5

# Introduction

The School of Haskell published a [4-part tutorial series](https://www.fpcomplete.com/school/ide-tutorials/building-a-file-hosting-service-in-yesod) showing how to write a simple self-hosted file server application. Consider this article the 5th entry in that series. People have been asking how the example might be extended in various ways. The most common request is for uploaded files to be kept in a database so that they will be available after restarting the application.

The most straightforward approach will be to use the [persistent](http://hackage.haskell.org/package/persistent) family of libraries, and interface with an [SQLite](http://www.sqlite.org/) database. Persistent is the natural choice for a data access layer. It presents the database as a simple [ORM](http://en.wikipedia.org/wiki/Object-relational_mapping) (Object-relational mapping). This will allow us to use regular Haskell objects and a simple SQL-like query language to keep track of uploaded files.

SQLite was chosen because it is the easiest to start using. Persistent supports other database backends as well. We'll make switching to another one as easy as possible by constraining SQLite-specific code to a single module.

## Following Along

[Part 1](https://www.fpcomplete.com/school/advanced-haskell/building-a-file-hosting-service-in-yesod/part%201) of this series contains a section explaining many of the conventions used when presenting examples. We will adopt the same style here.

There are a number of ways to follow along. One option is to work off of your own copy of this project, implementing changes as we go. To do this I suggest cloning the [File Server template](https://www.fpcomplete.com/page/project-build) from within FP Haskell Center. All of the active code samples are ready to run. Simply click on the "Open in IDE" link, select "Main.hs" from the drop down list towards the top of your screen, and click on the play button.

For those who prefer to work from their local computer, there is a companion [GitHub project](https://github.com/mikesteele81/soh-file-server-tutorial-project) which can be cloned. Most sections will contain a link pointing to what the project should look like after changes have been made. The final active code sample will contain full commenting for readers to use as a basis for further work.

## Reviewing Current Design

[github branch: 05-00](https://github.com/mikesteele81/soh-file-server-tutorial-project/tree/05-00)

Let's start by reviewing how file storage is currently handled. The server always starts with an empty data set. An in-memory data structure is populated as users upload files. We use an <hoogle>IntMap</hoogle> to hold information about files. The `App` type holds a reference to the file store and an `Int` value to be used as the next key for any future files that are uploaded. The value is initialized to 0 whenever the server starts, and is incremented each time it is used. The following is an abbreviated version of what can be found in "Foundation.hs":

``` haskell
-- | Extend this record to hold any information about uploaded files that you
-- need. Examples might be the time at which a file was uploaded, or the
-- identifier of the user account that added it.
--
-- All of these fields are initialized in the HomeR route's POST handler.
data StoredFile = StoredFile !Text !Text !ByteString

-- | A collection of uploaded files keyed on a unique 'Int' identifier.
type Store = IntMap StoredFile

-- | This is the application\'s "foundation" type. The first argument to 'App'
-- is the next identifier to be used when a new file is uploaded. The second
-- is a mapping from identifiers to files.
data App = App (TVar Int) (TVar Store)
```

The fields of `App` represent all of the information our application needs to perform its task of serving files. Both are reference types so that we can modify them during runtime. The `App` type being an instance of `Yesod` gives us access to it within any handler action when we call `getYesod`.

Our use of `IntMap` for storage is partially hidden. There is a collection of accessors that handler actions use whenever they need to look up or update information. Here are their type signatures:

``` haskell
getList :: Handler [(Int, StoredFile)]
addFile :: App -> StoredFile -> Handler ()
getById :: Int -> Handler StoredFile
```

Using these helper functions reduces the number of modules that need to be imported by our route handlers. They will also be helpful when we switch to using a database.

# Exchanging In-Memory Storage for a Database

We are about to replace one of the core parts of our application. Having a plan for how to proceed should allow us to maintain a working system as we make progress. The new data access layer will be built up alongside our current one, but will be left unused until the final step.

The first phase will be to refactor some of our existing code. By making a few trivial changes now we can avoid having them mixed in with significant changes later. We will also switch a few of our lazy data types to strict variants which are required by Persistent.

The second phase will be to add support for SQLite. There is a certain amount of preliminary work before we are able to execute database queries. We need to tell the application how to open a connection to the database, what our data model should look like, and how to integrate it with Yesod. All of this can be done without disrupting the application's existing behavior.

The third phase will be to remove support for our current data access layer. This will amount to rewriting the three accessor functions being used, changing type signatures in a few places and deleting the old code. 

## Phase 1: Refactoring

It's best to rework existing code in small steps when possible. There are a few things we can do now that will make later changes easier to follow.

### Updating addFile accessor

Among the three accessors we have, `addFile` is a little different from the others. It takes our `App` as an argument. This is uneccessary because we can always retrieve the foundation with <hoogle search="Yesod.Core.Handler.getYesod" results="1">getYesod</hoogle>. Removing the argument will allow us to simplify the `HomeR` route's POST handler and possibly avoid trivial changes later.

[github branch: 05-01](https://github.com/mikesteele81/soh-file-server-tutorial-project/tree/05-01)

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
import Handler.Download
import Handler.Home
import Handler.Preview

mkYesodDispatch "App" resourcesApp
{-# START_FILE Foundation.hs #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Foundation where

import Control.Concurrent.STM
import Data.ByteString.Lazy (ByteString)
import Data.Default
import Data.IntMap (IntMap)
import qualified Data.IntMap as IntMap
import Data.Text (Text)
import qualified Data.Text as Text
import Text.Hamlet
import Yesod
import Yesod.Default.Util

data StoredFile = StoredFile !Text !Text !ByteString
type Store = IntMap StoredFile
data App = App (TVar Int) (TVar Store)

instance Yesod App where
  defaultLayout widget = do
    pc <- widgetToPageContent $ $(widgetFileNoReload def "default-layout")
    withUrlRenderer $(hamletFile "templates/default-layout-wrapper.hamlet")

instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage

mkYesodData "App" $(parseRoutesFile "config/routes")

getNextId :: App -> STM Int
getNextId (App tnextId _) = do
    nextId <- readTVar tnextId
    writeTVar tnextId $ nextId + 1
    return nextId

getList :: Handler [(Int, StoredFile)]
getList = do
    App _ tstore <- getYesod
    store <- liftIO $ readTVarIO tstore
    return $ IntMap.toList store

-- show
-- addFile :: App -> StoredFile -> Handler ()
-- addFile app@(App _ tstore) file =
{-hi-}addFile :: StoredFile -> Handler ()
addFile file = do
    app@(App _ tstore) <- getYesod{-/hi-}
    liftIO . atomically $ do
        ident <- getNextId app
        modifyTVar tstore $ IntMap.insert ident file
-- /show

getById :: Int -> Handler StoredFile
getById ident = do
    App _ tstore <- getYesod
    store <- liftIO $ readTVarIO tstore
    case IntMap.lookup ident store of
      Nothing -> notFound
      Just file -> return file
{-# START_FILE Main.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Main where

import Control.Concurrent.STM
import Data.IntMap
import Yesod

import Dispatch ()
import Foundation

main :: IO ()
main = do
    tstore <- atomically $ newTVar empty
    tident <- atomically $ newTVar 0
    warpEnv $ App tident tstore
{-# START_FILE Handler/Download.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Handler.Download where

import qualified Data.Text as Text
import qualified Data.Text.Encoding as Text

import Yesod

import Foundation

getDownloadR :: Int -> Handler TypedContent
getDownloadR ident = do
    StoredFile filename contentType bytes <- getById ident
    addHeader "Content-Disposition" $ Text.concat
        [ "attachment; filename=\"", filename, "\""]
    sendResponse (Text.encodeUtf8 contentType, toContent bytes)
{-# START_FILE Handler/Home.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Handler.Home where

import Control.Monad.Trans.Resource
import Data.Conduit
import Data.Conduit.Binary
import Data.Default
import Yesod
import Yesod.Default.Util

import Foundation

getHomeR :: Handler Html
getHomeR = do
    (formWidget, formEncType) <- generateFormPost uploadForm
    storedFiles <- getList
    defaultLayout $ do
        setTitle "File Processor"
        $(widgetFileNoReload def "home")
-- show
postHomeR :: Handler Html
postHomeR = do
    ((result, _), _) <- runFormPost uploadForm
    case result of
      FormSuccess fi -> do
--         app <- getYesod
        fileBytes <- runResourceT $ fileSource fi $$ sinkLbs
--         addFile app $ StoredFile (fileName fi) (fileContentType fi) fileBytes
{-hi-}        addFile $ StoredFile (fileName fi) (fileContentType fi)
                             fileBytes{-/hi-}
      _ -> return ()
    redirect HomeR
-- /show

uploadForm :: Html -> MForm Handler (FormResult FileInfo, Widget)
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

getPreviewR :: Int -> Handler Html
getPreviewR ident = do
    StoredFile filename contentType bytes <- getById ident
    defaultLayout $ do
        setTitle . toMarkup $ "File Processor - " `Text.append` filename
        previewBlock <- liftIO $ preview ident contentType bytes
        $(widgetFileNoReload def "preview")

preview :: Int -> Text -> LB.ByteString -> IO Widget
preview ident contentType bytes
  | "image/" `Text.isPrefixOf` contentType =
    return [whamlet|<img src=@{DownloadR ident}>|]
  | otherwise = do
    eText <- try . evaluate $ LT.decodeUtf8 bytes :: IO (Either SomeException LT.Text)
    return $ case eText of
      Left _ -> errorMessage
      Right text -> [whamlet|<pre>#{text}|]
  where
    errorMessage = [whamlet|<pre>Unable to display file contents.|]
{-# START_FILE templates/default-layout.cassius #-}
body
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
    clear: both

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
^{widget}
{-# START_FILE templates/default-layout-wrapper.hamlet #-}
$newline never
$doctype 5
<html>
  <head>
    <title>#{pageTitle pc}
    ^{pageHead pc}
  <body>
    ^{pageBody pc}
{-# START_FILE templates/home.hamlet #-}
<h2>Previously submitted files
$if null storedFiles
  <p>No files have been uploaded yet.
$else
  <ul>
    $forall (ident, StoredFile filename _ _) <- storedFiles
      <li>
        <a href=@{PreviewR ident}>#{filename}
<h2>Submit new file
<form method=post action=@{HomeR} enctype=#{formEncType}>
  ^{formWidget}
  <input type="submit" value="Upload">
{-# START_FILE templates/preview.hamlet #-}
<a href=@{HomeR}>home
<h1>#{filename}
<a href=@{DownloadR ident}>download
<p>
  ^{previewBlock}
{-# START_FILE config/routes #-}
/                   HomeR     GET POST
/file/#Int          PreviewR  GET
/file/#Int/download DownloadR GET
```


### Using record syntax in App

All of the helper functions in "Foundation.hs" currently access fields of the `App` type by pattern matching. There is nothing wrong with this, but it means we will need to adjust every function when a new field is added. Using record syntax will allow us to leave most clients of the `App` data type untouched when changes are made.

[github branch: 05-02](https://github.com/mikesteele81/soh-file-server-tutorial-project/tree/05-02)

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
import Handler.Download
import Handler.Home
import Handler.Preview

mkYesodDispatch "App" resourcesApp
{-# START_FILE Foundation.hs #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Foundation where

import Control.Concurrent.STM
import Data.ByteString.Lazy (ByteString)
import Data.Default
import Data.IntMap (IntMap)
import qualified Data.IntMap as IntMap
import Data.Text (Text)
import qualified Data.Text as Text
import Text.Hamlet
import Yesod
import Yesod.Default.Util

-- show
data StoredFile = StoredFile !Text !Text !ByteString
type Store = IntMap StoredFile
-- data App = App (TVar Int) (TVar Store)
{-hi-}data App = App
    { tnextId :: TVar Int
    , tstore :: TVar Store
    }{-/hi-}
-- /show

instance Yesod App where
  defaultLayout widget = do
    pc <- widgetToPageContent $ $(widgetFileNoReload def "default-layout")
    withUrlRenderer $(hamletFile "templates/default-layout-wrapper.hamlet")

instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage

mkYesodData "App" $(parseRoutesFile "config/routes")

-- show
getNextId :: App -> STM Int
-- getNextId (App tnextId _) = do
--     nextId <- readTVar tnextId
--     writeTVar tnextId $ nextId + 1
{-hi-}getNextId app = do
    nextId <- readTVar $ tnextId app
    writeTVar (tnextId app) $ nextId + 1{-/hi-}
    return nextId

getList :: Handler [(Int, StoredFile)]
getList = do
--     App _ tstore <- getYesod
--     store <- liftIO $ readTVarIO tstore
{-hi-}    app <- getYesod
    store <- liftIO . readTVarIO $ tstore app{-/hi-}
    return $ IntMap.toList store

addFile :: StoredFile -> Handler ()
addFile file = do
--     app@(App _ tstore) <- getYesod
{-hi-}    app <- getYesod{-/hi-}
    liftIO . atomically $ do
        ident <- getNextId app
--         modifyTVar tstore $ IntMap.insert ident file
{-hi-}        modifyTVar (tstore app) $ IntMap.insert ident file{-/hi-}

getById :: Int -> Handler StoredFile
getById ident = do
--     App _ tstore <- getYesod
--     store <- liftIO $ readTVarIO tstore
{-hi-}    app <- getYesod
    store <- liftIO . readTVarIO $ tstore app{-/hi-}
    case IntMap.lookup ident store of
      Nothing -> notFound
      Just file -> return file
-- /show
{-# START_FILE Main.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Main where

import Control.Concurrent.STM
import Data.IntMap
import Yesod

import Dispatch ()
import Foundation

main :: IO ()
main = do
    tstore <- atomically $ newTVar empty
    tident <- atomically $ newTVar 0
    warpEnv $ App tident tstore
{-# START_FILE Handler/Download.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Handler.Download where

import qualified Data.Text as Text
import qualified Data.Text.Encoding as Text

import Yesod

import Foundation

getDownloadR :: Int -> Handler TypedContent
getDownloadR ident = do
    StoredFile filename contentType bytes <- getById ident
    addHeader "Content-Disposition" $ Text.concat
        [ "attachment; filename=\"", filename, "\""]
    sendResponse (Text.encodeUtf8 contentType, toContent bytes)
{-# START_FILE Handler/Home.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Handler.Home where

import Control.Monad.Trans.Resource
import Data.Conduit
import Data.Conduit.Binary
import Data.Default
import Yesod
import Yesod.Default.Util

import Foundation

getHomeR :: Handler Html
getHomeR = do
    (formWidget, formEncType) <- generateFormPost uploadForm
    storedFiles <- getList
    defaultLayout $ do
        setTitle "File Processor"
        $(widgetFileNoReload def "home")

postHomeR :: Handler Html
postHomeR = do
    ((result, _), _) <- runFormPost uploadForm
    case result of
      FormSuccess fi -> do
        fileBytes <- runResourceT $ fileSource fi $$ sinkLbs
        addFile $ StoredFile (fileName fi) (fileContentType fi)
                             fileBytes
      _ -> return ()
    redirect HomeR

uploadForm :: Html -> MForm Handler (FormResult FileInfo, Widget)
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

getPreviewR :: Int -> Handler Html
getPreviewR ident = do
    StoredFile filename contentType bytes <- getById ident
    defaultLayout $ do
        setTitle . toMarkup $ "File Processor - " `Text.append` filename
        previewBlock <- liftIO $ preview ident contentType bytes
        $(widgetFileNoReload def "preview")

preview :: Int -> Text -> LB.ByteString -> IO Widget
preview ident contentType bytes
  | "image/" `Text.isPrefixOf` contentType =
    return [whamlet|<img src=@{DownloadR ident}>|]
  | otherwise = do
    eText <- try . evaluate $ LT.decodeUtf8 bytes :: IO (Either SomeException LT.Text)
    return $ case eText of
      Left _ -> errorMessage
      Right text -> [whamlet|<pre>#{text}|]
  where
    errorMessage = [whamlet|<pre>Unable to display file contents.|]
{-# START_FILE templates/default-layout.cassius #-}
body
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
    clear: both

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
^{widget}
{-# START_FILE templates/default-layout-wrapper.hamlet #-}
$newline never
$doctype 5
<html>
  <head>
    <title>#{pageTitle pc}
    ^{pageHead pc}
  <body>
    ^{pageBody pc}
{-# START_FILE templates/home.hamlet #-}
<h2>Previously submitted files
$if null storedFiles
  <p>No files have been uploaded yet.
$else
  <ul>
    $forall (ident, StoredFile filename _ _) <- storedFiles
      <li>
        <a href=@{PreviewR ident}>#{filename}
<h2>Submit new file
<form method=post action=@{HomeR} enctype=#{formEncType}>
  ^{formWidget}
  <input type="submit" value="Upload">
{-# START_FILE templates/preview.hamlet #-}
<a href=@{HomeR}>home
<h1>#{filename}
<a href=@{DownloadR ident}>download
<p>
  ^{previewBlock}
{-# START_FILE config/routes #-}
/                   HomeR     GET POST
/file/#Int          PreviewR  GET
/file/#Int/download DownloadR GET
```

The names for `tstore` and `tnextId` were chosen simply to match a few `let` bindings. By the end of our work they will both be removed, so having a good name is not important in this case. Our purpose for doing this is to reduce the number of modifications that must be made in the next step.

### Switching to strict bytestrings

When writing the original application we chose to use lazy variants of the <hoogle search="Data.Text.Lazy">Text</hoogle> and <hoogle search="Data.ByteString.Lazy">ByteString</hoogle> data types. They were more convenient to use because the web framework uses lazy bytestrings to handle file uploads.

As a review, here is the POST handler used to process the file upload form. The line reading `runResourceT $ fileSource fi $$ sinkLbs` may be difficult to follow, but it produces a lazy bytestring from binary form data:

``` haskell
postHomeR :: Handler Html
postHomeR = do
    ((result, _), _) <- runFormPost uploadForm
    case result of
      FormSuccess fi -> do
        fileBytes <- runResourceT $ fileSource fi $$ sinkLbs
        addFile $ StoredFile (fileName fi) (fileContentType fi)
                             fileBytes
      _ -> return ()
    redirect HomeR
```

The situation has changed now that we will be storing our files in a database. The Persistent library does not support lazy bytestrings, so we should switch to using the strict variant.

[github branch: 05-03](https://github.com/mikesteele81/soh-file-server-tutorial-project/tree/05-03)

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
import Handler.Download
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
-- import Data.ByteString.Lazy (ByteString)
{-hi-}import Data.ByteString (ByteString){-/hi-}
import Data.Default
import Data.IntMap (IntMap)
import qualified Data.IntMap as IntMap
import Data.Text (Text)
import qualified Data.Text as Text
import Text.Hamlet
import Yesod
import Yesod.Default.Util

data StoredFile = StoredFile !Text !Text !ByteString
type Store = IntMap StoredFile
data App = App
    { tnextId :: TVar Int
    , tstore :: TVar Store
    }

instance Yesod App where
  defaultLayout widget = do
    pc <- widgetToPageContent $ $(widgetFileNoReload def "default-layout")
    withUrlRenderer $(hamletFile "templates/default-layout-wrapper.hamlet")

instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage

mkYesodData "App" $(parseRoutesFile "config/routes")

getNextId :: App -> STM Int
getNextId app = do
    nextId <- readTVar $ tnextId app
    writeTVar (tnextId app) $ nextId + 1
    return nextId

getList :: Handler [(Int, StoredFile)]
getList = do
    app <- getYesod
    store <- liftIO . readTVarIO $ tstore app
    return $ IntMap.toList store

addFile :: StoredFile -> Handler ()
addFile file = do
    app <- getYesod
    liftIO . atomically $ do
        ident <- getNextId app
        modifyTVar (tstore app) $ IntMap.insert ident file

getById :: Int -> Handler StoredFile
getById ident = do
    app <- getYesod
    store <- liftIO . readTVarIO $ tstore app
    case IntMap.lookup ident store of
      Nothing -> notFound
      Just file -> return file
-- /show
{-# START_FILE Main.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Main where

import Control.Concurrent.STM
import Data.IntMap
import Yesod

import Dispatch ()
import Foundation

main :: IO ()
main = do
    tstore <- atomically $ newTVar empty
    tident <- atomically $ newTVar 0
    warpEnv $ App tident tstore
{-# START_FILE Handler/Download.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Handler.Download where

import qualified Data.Text as Text
import qualified Data.Text.Encoding as Text

import Yesod

import Foundation

getDownloadR :: Int -> Handler TypedContent
getDownloadR ident = do
    StoredFile filename contentType bytes <- getById ident
    addHeader "Content-Disposition" $ Text.concat
        [ "attachment; filename=\"", filename, "\""]
    sendResponse (Text.encodeUtf8 contentType, toContent bytes)
{-# START_FILE Handler/Home.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
module Handler.Home where

import Control.Monad.Trans.Resource
{-hi-}import qualified Data.ByteString as S
import qualified Data.ByteString.Lazy as L{-/hi-}
import Data.Conduit
import Data.Conduit.Binary
import Data.Default
import Yesod
import Yesod.Default.Util

import Foundation

getHomeR :: Handler Html
getHomeR = do
    (formWidget, formEncType) <- generateFormPost uploadForm
    storedFiles <- getList
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
        addFile $ StoredFile (fileName fi) (fileContentType fi)
--                              fileBytes
{-hi-}                             (S.pack . L.unpack $ fileBytes){-/hi-}
      _ -> return ()
    redirect HomeR

uploadForm :: Html -> MForm Handler (FormResult FileInfo, Widget)
uploadForm = renderDivs $ fileAFormReq "file"
-- /show
{-# START_FILE Handler/Preview.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}

-- show
module Handler.Preview where

import Control.Exception hiding (Handler)
-- import qualified Data.ByteString.Lazy as LB
{-hi-}import qualified Data.ByteString as SB{-/hi-}
import Data.Default
import Data.Text (Text)
import qualified Data.Text as Text
-- import qualified Data.Text.Lazy as LT
-- import qualified Data.Text.Lazy.Encoding as LT
{-hi-}import qualified Data.Text.Encoding as Text{-/hi-}
import Text.Blaze
import Yesod
import Yesod.Default.Util

import Foundation

getPreviewR :: Int -> Handler Html
getPreviewR ident = do
    StoredFile filename contentType bytes <- getById ident
    defaultLayout $ do
        setTitle . toMarkup $ "File Processor - " `Text.append` filename
        previewBlock <- liftIO $ preview ident contentType bytes
        $(widgetFileNoReload def "preview")

-- preview :: Int -> Text -> LB.ByteString -> IO Widget
{-hi-}preview :: Int -> Text -> SB.ByteString -> IO Widget{-/hi-}
preview ident contentType bytes
  | "image/" `Text.isPrefixOf` contentType =
    return [whamlet|<img src=@{DownloadR ident}>|]
  | otherwise = do
--     eText <- try . evaluate $ LT.decodeUtf8 bytes :: IO (Either SomeException LT.Text)
{-hi-}    eText <- try . evaluate $ Text.decodeUtf8 bytes :: IO (Either SomeException Text){-/hi-}
    return $ case eText of
      Left _ -> errorMessage
      Right text -> [whamlet|<pre>#{text}|]
  where
    errorMessage = [whamlet|<pre>Unable to display file contents.|]
-- /show
{-# START_FILE templates/default-layout.cassius #-}
body
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
    clear: both

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
^{widget}
{-# START_FILE templates/default-layout-wrapper.hamlet #-}
$newline never
$doctype 5
<html>
  <head>
    <title>#{pageTitle pc}
    ^{pageHead pc}
  <body>
    ^{pageBody pc}
{-# START_FILE templates/home.hamlet #-}
<h2>Previously submitted files
$if null storedFiles
  <p>No files have been uploaded yet.
$else
  <ul>
    $forall (ident, StoredFile filename _ _) <- storedFiles
      <li>
        <a href=@{PreviewR ident}>#{filename}
<h2>Submit new file
<form method=post action=@{HomeR} enctype=#{formEncType}>
  ^{formWidget}
  <input type="submit" value="Upload">
{-# START_FILE templates/preview.hamlet #-}
<a href=@{HomeR}>home
<h1>#{filename}
<a href=@{DownloadR ident}>download
<p>
  ^{previewBlock}
{-# START_FILE config/routes #-}
/                   HomeR     GET POST
/file/#Int          PreviewR  GET
/file/#Int/download DownloadR GET
```

## Phase 2: Adding Support for Persistent

With refactoring out of the way we are ready to begin adding support for a database. The "persistent" family of libraries provides an easy-to-use database layer that will work well for our purposes. We will write to an [SQLite](http://www.sqlite.org/) database, but other backends are could have been chosen. Switching to a different one such as MySQL should be a matter of changing a single file.

These are the libraries we will be using:

* [persistent](http://hackage.haskell.org/package/persistent)

* [persistent-sqlite](http://hackage.haskell.org/package/persistent-sqlite)

* [yesod-persistent](http://hackage.haskell.org/package/yesod-persistent)

We have three steps to perform before we can start manipulating database tables. First, we need to create a pool of connections for Persistent to use. Second, we need to tell Yesod how to access that pool. Finally, we need to specify our database schema.

### Opening and closing the database file

The first thing we need to do is define how to open a database connection. The Persistent library works with <hoogle search="Database.Persist.Sql
.ConnectionPool" results="1">ConnectionPool</hoogle> objects that backend libraries such as persistent-sqlite initialize. It should be simple to exchange one backend for another as long we restrict ourselves to the basic interface. Within this project the only place where we say anything about SQLite specifically will be in a new module named "Config.hs". The rest of the application will not be concerned with which backend is used.

``` haskell
{-hi-}module Config where

import Database.Persist.Sqlite

persistConfig :: SqliteConf
persistConfig = SqliteConf "database" 100{-/hi-}
```

The type of `persistConfig` is <hoogle search="Database.Persist.Sqlite
.SqliteConf" results="1">SqliteConf</hoogle>, but other modules will only be interested in its <hoogle search="Database.Persist.Class.PersistConfig" results="1">PersistConfig</hoogle> instance. The following example creates a <hoogle search="Database.Persist.Sql
.ConnectionPool" results="1">ConnectionPool</hoogle> as the application starts, and stores this in the foundation type. Yesod will need to have access to it in the next section.

[github branch: 05-04](https://github.com/mikesteele81/soh-file-server-tutorial-project/tree/05-04)

``` active haskell web
{-# START_FILE Config.hs #-}
{-# LANGUAGE OverloadedStrings #-}

-- show
{-hi-}module Config where

import Database.Persist.Sqlite

persistConfig :: SqliteConf
persistConfig = SqliteConf "database" 100{-/hi-}
-- /show
{-# START_FILE Dispatch.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Dispatch where

import Yesod

import Foundation
import Handler.Download
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
import Data.ByteString (ByteString)
import Data.Default
import Data.IntMap (IntMap)
import qualified Data.IntMap as IntMap
import Data.Text (Text)
import qualified Data.Text as Text
{-hi-}import Database.Persist.Sql{-/hi-}
import Text.Hamlet
import Yesod
import Yesod.Default.Util

data StoredFile = StoredFile !Text !Text !ByteString
type Store = IntMap StoredFile
data App = App
    { tnextId :: TVar Int
    , tstore :: TVar Store
{-hi-}    , connPool :: ConnectionPool{-/hi-}
    }

instance Yesod App where
  defaultLayout widget = do
    pc <- widgetToPageContent $ $(widgetFileNoReload def "default-layout")
    withUrlRenderer $(hamletFile "templates/default-layout-wrapper.hamlet")

instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage

mkYesodData "App" $(parseRoutesFile "config/routes")

getNextId :: App -> STM Int
getNextId app = do
    nextId <- readTVar $ tnextId app
    writeTVar (tnextId app) $ nextId + 1
    return nextId

getList :: Handler [(Int, StoredFile)]
getList = do
    app <- getYesod
    store <- liftIO . readTVarIO $ tstore app
    return $ IntMap.toList store

addFile :: StoredFile -> Handler ()
addFile file = do
    app <- getYesod
    liftIO . atomically $ do
        ident <- getNextId app
        modifyTVar (tstore app) $ IntMap.insert ident file

getById :: Int -> Handler StoredFile
getById ident = do
    app <- getYesod
    store <- liftIO . readTVarIO $ tstore app
    case IntMap.lookup ident store of
      Nothing -> notFound
      Just file -> return file
-- /show
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}

-- show
module Main where

import Control.Concurrent.STM
import Data.IntMap
{-hi-}import Database.Persist.Sql{-/hi-}
import Yesod

{-hi-}import Config{-/hi-}
import Dispatch ()
import Foundation

main :: IO ()
main = do
{-hi-}    pool <- createPoolConfig persistConfig{-/hi-}
    tstore <- atomically $ newTVar empty
    tident <- atomically $ newTVar 0
--     warpEnv $ App tident tstore
{-hi-}    warpEnv $ App tident tstore pool{-/hi-}
-- /show
{-# START_FILE Handler/Download.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Handler.Download where

import qualified Data.Text as Text
import qualified Data.Text.Encoding as Text

import Yesod

import Foundation

getDownloadR :: Int -> Handler TypedContent
getDownloadR ident = do
    StoredFile filename contentType bytes <- getById ident
    addHeader "Content-Disposition" $ Text.concat
        [ "attachment; filename=\"", filename, "\""]
    sendResponse (Text.encodeUtf8 contentType, toContent bytes)
{-# START_FILE Handler/Home.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Handler.Home where

import Control.Monad.Trans.Resource
import qualified Data.ByteString as S
import qualified Data.ByteString.Lazy as L
import Data.Conduit
import Data.Conduit.Binary
import Data.Default
import Yesod
import Yesod.Default.Util

import Foundation

getHomeR :: Handler Html
getHomeR = do
    (formWidget, formEncType) <- generateFormPost uploadForm
    storedFiles <- getList
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
        addFile $ StoredFile (fileName fi) (fileContentType fi)
                             (S.pack . L.unpack $ fileBytes)
      _ -> return ()
    redirect HomeR

uploadForm :: Html -> MForm Handler (FormResult FileInfo, Widget)
uploadForm = renderDivs $ fileAFormReq "file"
{-# START_FILE Handler/Preview.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}

module Handler.Preview where

import Control.Exception hiding (Handler)
import qualified Data.ByteString as SB
import Data.Default
import Data.Text (Text)
import qualified Data.Text as Text
import qualified Data.Text.Encoding as Text
import Text.Blaze
import Yesod
import Yesod.Default.Util

import Foundation

getPreviewR :: Int -> Handler Html
getPreviewR ident = do
    StoredFile filename contentType bytes <- getById ident
    defaultLayout $ do
        setTitle . toMarkup $ "File Processor - " `Text.append` filename
        previewBlock <- liftIO $ preview ident contentType bytes
        $(widgetFileNoReload def "preview")

preview :: Int -> Text -> SB.ByteString -> IO Widget
preview ident contentType bytes
  | "image/" `Text.isPrefixOf` contentType =
    return [whamlet|<img src=@{DownloadR ident}>|]
  | otherwise = do
    eText <- try . evaluate $ Text.decodeUtf8 bytes :: IO (Either SomeException Text)
    return $ case eText of
      Left _ -> errorMessage
      Right text -> [whamlet|<pre>#{text}|]
  where
    errorMessage = [whamlet|<pre>Unable to display file contents.|]
{-# START_FILE templates/default-layout.cassius #-}
body
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
    clear: both

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
^{widget}
{-# START_FILE templates/default-layout-wrapper.hamlet #-}
$newline never
$doctype 5
<html>
  <head>
    <title>#{pageTitle pc}
    ^{pageHead pc}
  <body>
    ^{pageBody pc}
{-# START_FILE templates/home.hamlet #-}
<h2>Previously submitted files
$if null storedFiles
  <p>No files have been uploaded yet.
$else
  <ul>
    $forall (ident, StoredFile filename _ _) <- storedFiles
      <li>
        <a href=@{PreviewR ident}>#{filename}
<h2>Submit new file
<form method=post action=@{HomeR} enctype=#{formEncType}>
  ^{formWidget}
  <input type="submit" value="Upload">
{-# START_FILE templates/preview.hamlet #-}
<a href=@{HomeR}>home
<h1>#{filename}
<a href=@{DownloadR ident}>download
<p>
  ^{previewBlock}
{-# START_FILE config/routes #-}
/                   HomeR     GET POST
/file/#Int          PreviewR  GET
/file/#Int/download DownloadR GET
```

### Integrating with Yesod

In the last section we specified how the to open and close the database. Now we need to tell Yesod where to find our database configuration and connection pool. The goal is for us to be able to call <hoogle search="Yesod.Persist.Core.runDB" results="1">runDB</hoogle> to perform a database update or query from one of our handler actions.

[github branch: 05-05](https://github.com/mikesteele81/soh-file-server-tutorial-project/tree/05-05)

``` active haskell web
{-# START_FILE Config.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Config where

import Database.Persist.Sqlite

persistConfig :: SqliteConf
persistConfig = SqliteConf "database" 100
{-# START_FILE Dispatch.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Dispatch where

import Yesod

import Foundation
import Handler.Download
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
import Data.ByteString (ByteString)
import Data.Default
import Data.IntMap (IntMap)
import qualified Data.IntMap as IntMap
import Data.Text (Text)
import qualified Data.Text as Text
import Database.Persist.Sql
import Text.Hamlet
import Yesod
import Yesod.Default.Util

{-hi-}import Config{-/hi-}

data StoredFile = StoredFile !Text !Text !ByteString
type Store = IntMap StoredFile
data App = App
    { tnextId :: TVar Int
    , tstore :: TVar Store
    , connPool :: ConnectionPool
    }

instance Yesod App where
  defaultLayout widget = do
    pc <- widgetToPageContent $ $(widgetFileNoReload def "default-layout")
    withUrlRenderer $(hamletFile "templates/default-layout-wrapper.hamlet")

instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage

{-hi-}instance YesodPersist App where
  type YesodPersistBackend App = SqlBackend
  runDB action = defaultRunDB (const persistConfig) connPool action

instance YesodPersistRunner App where
  getDBRunner = defaultGetDBRunner connPool{-/hi-}

mkYesodData "App" $(parseRoutesFile "config/routes")

getNextId :: App -> STM Int
getNextId app = do
    nextId <- readTVar $ tnextId app
    writeTVar (tnextId app) $ nextId + 1
    return nextId

getList :: Handler [(Int, StoredFile)]
getList = do
    app <- getYesod
    store <- liftIO . readTVarIO $ tstore app
    return $ IntMap.toList store

addFile :: StoredFile -> Handler ()
addFile file = do
    app <- getYesod
    liftIO . atomically $ do
        ident <- getNextId app
        modifyTVar (tstore app) $ IntMap.insert ident file

getById :: Int -> Handler StoredFile
getById ident = do
    app <- getYesod
    store <- liftIO . readTVarIO $ tstore app
    case IntMap.lookup ident store of
      Nothing -> notFound
      Just file -> return file
-- /show
{-# START_FILE Main.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Main where

import Control.Concurrent.STM
import Data.IntMap
import Database.Persist
import Yesod

import Config
import Dispatch ()
import Foundation

main :: IO ()
main = do
    pool <- createPoolConfig persistConfig
    tstore <- atomically $ newTVar empty
    tident <- atomically $ newTVar 0
    warpEnv $ App tident tstore pool
{-# START_FILE Handler/Download.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Handler.Download where

import qualified Data.Text as Text
import qualified Data.Text.Encoding as Text

import Yesod

import Foundation

getDownloadR :: Int -> Handler TypedContent
getDownloadR ident = do
    StoredFile filename contentType bytes <- getById ident
    addHeader "Content-Disposition" $ Text.concat
        [ "attachment; filename=\"", filename, "\""]
    sendResponse (Text.encodeUtf8 contentType, toContent bytes)
{-# START_FILE Handler/Home.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Handler.Home where

import Control.Monad.Trans.Resource
import qualified Data.ByteString as S
import qualified Data.ByteString.Lazy as L
import Data.Conduit
import Data.Conduit.Binary
import Data.Default
import Yesod
import Yesod.Default.Util

import Foundation

getHomeR :: Handler Html
getHomeR = do
    (formWidget, formEncType) <- generateFormPost uploadForm
    storedFiles <- getList
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
        addFile $ StoredFile (fileName fi) (fileContentType fi)
                             (S.pack . L.unpack $ fileBytes)
      _ -> return ()
    redirect HomeR

uploadForm :: Html -> MForm Handler (FormResult FileInfo, Widget)
uploadForm = renderDivs $ fileAFormReq "file"
{-# START_FILE Handler/Preview.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}

module Handler.Preview where

import Control.Exception hiding (Handler)
import qualified Data.ByteString as SB
import Data.Default
import Data.Text (Text)
import qualified Data.Text as Text
import qualified Data.Text.Encoding as Text
import Text.Blaze
import Yesod
import Yesod.Default.Util

import Foundation

getPreviewR :: Int -> Handler Html
getPreviewR ident = do
    StoredFile filename contentType bytes <- getById ident
    defaultLayout $ do
        setTitle . toMarkup $ "File Processor - " `Text.append` filename
        previewBlock <- liftIO $ preview ident contentType bytes
        $(widgetFileNoReload def "preview")

preview :: Int -> Text -> SB.ByteString -> IO Widget
preview ident contentType bytes
  | "image/" `Text.isPrefixOf` contentType =
    return [whamlet|<img src=@{DownloadR ident}>|]
  | otherwise = do
    eText <- try . evaluate $ Text.decodeUtf8 bytes :: IO (Either SomeException Text)
    return $ case eText of
      Left _ -> errorMessage
      Right text -> [whamlet|<pre>#{text}|]
  where
    errorMessage = [whamlet|<pre>Unable to display file contents.|]
{-# START_FILE templates/default-layout.cassius #-}
body
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
    clear: both

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
^{widget}
{-# START_FILE templates/default-layout-wrapper.hamlet #-}
$newline never
$doctype 5
<html>
  <head>
    <title>#{pageTitle pc}
    ^{pageHead pc}
  <body>
    ^{pageBody pc}
{-# START_FILE templates/home.hamlet #-}
<h2>Previously submitted files
$if null storedFiles
  <p>No files have been uploaded yet.
$else
  <ul>
    $forall (ident, StoredFile filename _ _) <- storedFiles
      <li>
        <a href=@{PreviewR ident}>#{filename}
<h2>Submit new file
<form method=post action=@{HomeR} enctype=#{formEncType}>
  ^{formWidget}
  <input type="submit" value="Upload">
{-# START_FILE templates/preview.hamlet #-}
<a href=@{HomeR}>home
<h1>#{filename}
<a href=@{DownloadR ident}>download
<p>
  ^{previewBlock}
{-# START_FILE config/routes #-}
/                   HomeR     GET POST
/file/#Int          PreviewR  GET
/file/#Int/download DownloadR GET
```

Most projects' <hoogle search="Yesod.Persist.Core.YesodPersist" results="1">YesodPersist</hoogle> instance will be nearly the same, so the <hoogle search="Yesod.Persist.Core.defaultRunDB" results="1">defaultRunDB</hoogle> helper function is available to do most of the work for us. We only need to tell it where to find a database configuration and a connection pool. 

### Defining the data model

Think of the [Persistent](http://hackage.haskell.org/package/persistent) library as a simple [ORM](http://en.wikipedia.org/wiki/Object-relational_mapping) (object-relational mapping). We use a miniature language to define a collection of objects that our project will use. A call to <hoogle results="1" search="Database.Persist.TH.mkPersist">mkPersist</hoogle> translates this into Haskell data types and a few migration functions. The data types will be simple records that instantiate all of the type classes Persistent needs to work with. Migration functions are used to update the underlying database schema to match what we've defined.

Our application is small enough that we only need a single data type. We'll give it the same name that is currently being used. Here is the model definition:

```
StoredFile
    name Text
    contentType Text
    content ByteString
```

Each Line beginning at column 0 becomes a Haskell data type and a table within the database. Indented lines become fields of the data type in Haskell and fields of a table in the database. After each field name comes a type. Any type that is an instance of <hoogle results="1" search="Database.Persist.Class.PersistField">PersistField</hoogle> can be used. These are translated to SQL types specific to the backend being used.

As an example, the persist definition given above will produce a Haskell type and database schema similar to the following:

``` haskell
data StoredFile
    { name        :: Text
    , contentType :: Text
    , content     :: ByteString
    }
```

```
CREATE TABLE "stored_file"(
  "id" INTEGER PRIMARY KEY,
  "name" VARCHAR NOT NULL,
  "content_type" VARCHAR NOT NULL,
  "content" BLOB NOT NULL
);
```

As you will see, persistence and routing are treated similarly. Reviewing the section on routing in [part 1](https://www.fpcomplete.com/school/advanced-haskell/building-a-file-hosting-service-in-yesod/part%201) may help you understand how persistence is handled.

We're going to add 2 files to the project. The model definition will be stored in "config/models" alongside "config/routes". Whereas with routing we wrote a short "Dispatch.hs" module to generate Haskell boilerplate code from our definition, we will now write a short "Model.hs" module to generate Haskell code from our model definition. Going forward, we shouldn't have to update our "Model.hs" file very frequently. It will mostly contain import statements.

[github branch: 05-06](https://github.com/mikesteele81/soh-file-server-tutorial-project/tree/05-06)

``` active haskell web
{-# START_FILE Config.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Config where

import Database.Persist.Sqlite

persistConfig :: SqliteConf
persistConfig = SqliteConf "database" 100
{-# START_FILE Dispatch.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Dispatch where

import Yesod

import Foundation
import Handler.Download
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
import Data.ByteString (ByteString)
import Data.Default
import Data.IntMap (IntMap)
import qualified Data.IntMap as IntMap
import Data.Text (Text)
import qualified Data.Text as Text
import Database.Persist.Sql
import Text.Hamlet
import Yesod
import Yesod.Default.Util

import Config

data StoredFile = StoredFile !Text !Text !ByteString
type Store = IntMap StoredFile
data App = App
    { tnextId :: TVar Int
    , tstore :: TVar Store
    , connPool :: ConnectionPool
    }

instance Yesod App where
  defaultLayout widget = do
    pc <- widgetToPageContent $ $(widgetFileNoReload def "default-layout")
    withUrlRenderer $(hamletFile "templates/default-layout-wrapper.hamlet")

instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage

instance YesodPersist App where
  type YesodPersistBackend App = SqlBackend
  runDB action = defaultRunDB (const persistConfig) connPool action

instance YesodPersistRunner App where
  getDBRunner = defaultGetDBRunner connPool

mkYesodData "App" $(parseRoutesFile "config/routes")

getNextId :: App -> STM Int
getNextId app = do
    nextId <- readTVar $ tnextId app
    writeTVar (tnextId app) $ nextId + 1
    return nextId

getList :: Handler [(Int, StoredFile)]
getList = do
    app <- getYesod
    store <- liftIO . readTVarIO $ tstore app
    return $ IntMap.toList store

addFile :: StoredFile -> Handler ()
addFile file = do
    app <- getYesod
    liftIO . atomically $ do
        ident <- getNextId app
        modifyTVar (tstore app) $ IntMap.insert ident file

getById :: Int -> Handler StoredFile
getById ident = do
    app <- getYesod
    store <- liftIO . readTVarIO $ tstore app
    case IntMap.lookup ident store of
      Nothing -> notFound
      Just file -> return file
{-# START_FILE Main.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Main where

import Control.Concurrent.STM
import Data.IntMap
import Database.Persist.Sql
import Yesod

import Config
import Dispatch ()
import Foundation

main :: IO ()
main = do
    pool <- createPoolConfig persistConfig
    tstore <- atomically $ newTVar empty
    tident <- atomically $ newTVar 0
    warpEnv $ App tident tstore pool
{-# START_FILE Model.hs #-}
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE GADTs #-}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
{-hi-}module Model where

import Data.ByteString (ByteString)
import Data.Text (Text)
import Database.Persist.Quasi
import Yesod
import Data.Typeable (Typeable)

share [mkPersist sqlSettings, mkMigrate "migrateAll"]
    $(persistFileWith lowerCaseSettings "config/models"){-/hi-}
-- /show
{-# START_FILE Handler/Download.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Handler.Download where

import qualified Data.Text as Text
import qualified Data.Text.Encoding as Text

import Yesod

import Foundation

getDownloadR :: Int -> Handler TypedContent
getDownloadR ident = do
    StoredFile filename contentType bytes <- getById ident
    addHeader "Content-Disposition" $ Text.concat
        [ "attachment; filename=\"", filename, "\""]
    sendResponse (Text.encodeUtf8 contentType, toContent bytes)
{-# START_FILE Handler/Home.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Handler.Home where

import Control.Monad.Trans.Resource
import qualified Data.ByteString as S
import qualified Data.ByteString.Lazy as L
import Data.Conduit
import Data.Conduit.Binary
import Data.Default
import Yesod
import Yesod.Default.Util

import Foundation

getHomeR :: Handler Html
getHomeR = do
    (formWidget, formEncType) <- generateFormPost uploadForm
    storedFiles <- getList
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
        addFile $ StoredFile (fileName fi) (fileContentType fi)
                             (S.pack . L.unpack $ fileBytes)
      _ -> return ()
    redirect HomeR

uploadForm :: Html -> MForm Handler (FormResult FileInfo, Widget)
uploadForm = renderDivs $ fileAFormReq "file"
{-# START_FILE Handler/Preview.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}

module Handler.Preview where

import Control.Exception hiding (Handler)
import qualified Data.ByteString as SB
import Data.Default
import Data.Text (Text)
import qualified Data.Text as Text
import qualified Data.Text.Encoding as Text
import Text.Blaze
import Yesod
import Yesod.Default.Util

import Foundation

getPreviewR :: Int -> Handler Html
getPreviewR ident = do
    StoredFile filename contentType bytes <- getById ident
    defaultLayout $ do
        setTitle . toMarkup $ "File Processor - " `Text.append` filename
        previewBlock <- liftIO $ preview ident contentType bytes
        $(widgetFileNoReload def "preview")

preview :: Int -> Text -> SB.ByteString -> IO Widget
preview ident contentType bytes
  | "image/" `Text.isPrefixOf` contentType =
    return [whamlet|<img src=@{DownloadR ident}>|]
  | otherwise = do
    eText <- try . evaluate $ Text.decodeUtf8 bytes :: IO (Either SomeException Text)
    return $ case eText of
      Left _ -> errorMessage
      Right text -> [whamlet|<pre>#{text}|]
  where
    errorMessage = [whamlet|<pre>Unable to display file contents.|]
{-# START_FILE templates/default-layout.cassius #-}
body
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
    clear: both

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
^{widget}
{-# START_FILE templates/default-layout-wrapper.hamlet #-}
$newline never
$doctype 5
<html>
  <head>
    <title>#{pageTitle pc}
    ^{pageHead pc}
  <body>
    ^{pageBody pc}
{-# START_FILE templates/home.hamlet #-}
<h2>Previously submitted files
$if null storedFiles
  <p>No files have been uploaded yet.
$else
  <ul>
    $forall (ident, StoredFile filename _ _) <- storedFiles
      <li>
        <a href=@{PreviewR ident}>#{filename}
<h2>Submit new file
<form method=post action=@{HomeR} enctype=#{formEncType}>
  ^{formWidget}
  <input type="submit" value="Upload">
{-# START_FILE templates/preview.hamlet #-}
<a href=@{HomeR}>home
<h1>#{filename}
<a href=@{DownloadR ident}>download
<p>
  ^{previewBlock}
{-# START_FILE config/models #-}
{-hi-}StoredFile
    name Text
    contentType Text
    content ByteString{-/hi-}
{-# START_FILE config/routes #-}
/                   HomeR     GET POST
/file/#Int          PreviewR  GET
/file/#Int/download DownloadR GET
```


There's much more to writing model definitions than we covered here. The best place to learn more is Michael Snoyman's book, [Developing Web Applications with Haskell and Yesod](http://www.yesodweb.com/book). There is a [chapter](http://www.yesodweb.com/book/persistent) dedicated to Persistent covering all of the details.

### Migrating the database schema

What should happen when we try to open a database that either doesn't exist or does not match the specified model? A few functions have been generated for us to perform "migrations", but it's up to us to use them. Triggering a migration will cause the database schema to be compared with our definition. The database will be updated whenever it's clear what the right thing to do is.

[github branch: 05-07](https://github.com/mikesteele81/soh-file-server-tutorial-project/tree/05-07)

``` active haskell web
{-# START_FILE Config.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Config where

import Database.Persist.Sqlite

persistConfig :: SqliteConf
persistConfig = SqliteConf "database" 100
{-# START_FILE Dispatch.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Dispatch where

import Yesod

import Foundation
import Handler.Download
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
import Data.ByteString (ByteString)
import Data.Default
import Data.IntMap (IntMap)
import qualified Data.IntMap as IntMap
import Data.Text (Text)
import qualified Data.Text as Text
import Database.Persist.Sql
import Text.Hamlet
import Yesod
import Yesod.Default.Util

import Config

data StoredFile = StoredFile !Text !Text !ByteString
type Store = IntMap StoredFile
data App = App
    { tnextId :: TVar Int
    , tstore :: TVar Store
    , connPool :: ConnectionPool
    }

instance Yesod App where
  defaultLayout widget = do
    pc <- widgetToPageContent $ $(widgetFileNoReload def "default-layout")
    withUrlRenderer $(hamletFile "templates/default-layout-wrapper.hamlet")

instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage

instance YesodPersist App where
  type YesodPersistBackend App = SqlBackend
  runDB action = defaultRunDB (const persistConfig) connPool action

instance YesodPersistRunner App where
  getDBRunner = defaultGetDBRunner connPool

mkYesodData "App" $(parseRoutesFile "config/routes")

getNextId :: App -> STM Int
getNextId app = do
    nextId <- readTVar $ tnextId app
    writeTVar (tnextId app) $ nextId + 1
    return nextId

getList :: Handler [(Int, StoredFile)]
getList = do
    app <- getYesod
    store <- liftIO . readTVarIO $ tstore app
    return $ IntMap.toList store

addFile :: StoredFile -> Handler ()
addFile file = do
    app <- getYesod
    liftIO . atomically $ do
        ident <- getNextId app
        modifyTVar (tstore app) $ IntMap.insert ident file

getById :: Int -> Handler StoredFile
getById ident = do
    app <- getYesod
    store <- liftIO . readTVarIO $ tstore app
    case IntMap.lookup ident store of
      Nothing -> notFound
      Just file -> return file
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}

-- show
module Main where

import Control.Concurrent.STM
{-hi-}import Control.Monad.Logger
import Control.Monad.Trans.Resource{-/hi-}
import Data.IntMap
import Database.Persist.Sql
import Yesod

import Config
import Dispatch ()
import Foundation
{-hi-}import Model (migrateAll){-/hi-}

main :: IO ()
main = do
    pool <- createPoolConfig persistConfig
{-hi-}    runResourceT $ runStderrLoggingT $ flip runSqlPool pool
        $ runMigration migrateAll{-/hi-}
    tstore <- atomically $ newTVar empty
    tident <- atomically $ newTVar 0
    warpEnv $ App tident tstore pool
-- /show
{-# START_FILE Model.hs #-}
-- show
-- /show
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE GADTs #-}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Model where

import Data.ByteString (ByteString)
import Data.Text (Text)
import Database.Persist.Quasi
import Yesod
import Data.Typeable (Typeable)

share [mkPersist sqlSettings, mkMigrate "migrateAll"]
    $(persistFileWith lowerCaseSettings "config/models")
{-# START_FILE Handler/Download.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Handler.Download where

import qualified Data.Text as Text
import qualified Data.Text.Encoding as Text

import Yesod

import Foundation

getDownloadR :: Int -> Handler TypedContent
getDownloadR ident = do
    StoredFile filename contentType bytes <- getById ident
    addHeader "Content-Disposition" $ Text.concat
        [ "attachment; filename=\"", filename, "\""]
    sendResponse (Text.encodeUtf8 contentType, toContent bytes)
{-# START_FILE Handler/Home.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Handler.Home where

import Control.Monad.Trans.Resource
import qualified Data.ByteString as S
import qualified Data.ByteString.Lazy as L
import Data.Conduit
import Data.Conduit.Binary
import Data.Default
import Yesod
import Yesod.Default.Util

import Foundation

getHomeR :: Handler Html
getHomeR = do
    (formWidget, formEncType) <- generateFormPost uploadForm
    storedFiles <- getList
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
        addFile $ StoredFile (fileName fi) (fileContentType fi)
                             (S.pack . L.unpack $ fileBytes)
      _ -> return ()
    redirect HomeR

uploadForm :: Html -> MForm Handler (FormResult FileInfo, Widget)
uploadForm = renderDivs $ fileAFormReq "file"
{-# START_FILE Handler/Preview.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}

module Handler.Preview where

import Control.Exception hiding (Handler)
import qualified Data.ByteString as SB
import Data.Default
import Data.Text (Text)
import qualified Data.Text as Text
import qualified Data.Text.Encoding as Text
import Text.Blaze
import Yesod
import Yesod.Default.Util

import Foundation

getPreviewR :: Int -> Handler Html
getPreviewR ident = do
    StoredFile filename contentType bytes <- getById ident
    defaultLayout $ do
        setTitle . toMarkup $ "File Processor - " `Text.append` filename
        previewBlock <- liftIO $ preview ident contentType bytes
        $(widgetFileNoReload def "preview")

preview :: Int -> Text -> SB.ByteString -> IO Widget
preview ident contentType bytes
  | "image/" `Text.isPrefixOf` contentType =
    return [whamlet|<img src=@{DownloadR ident}>|]
  | otherwise = do
    eText <- try . evaluate $ Text.decodeUtf8 bytes :: IO (Either SomeException Text)
    return $ case eText of
      Left _ -> errorMessage
      Right text -> [whamlet|<pre>#{text}|]
  where
    errorMessage = [whamlet|<pre>Unable to display file contents.|]
{-# START_FILE templates/default-layout.cassius #-}
body
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
    clear: both

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
^{widget}
{-# START_FILE templates/default-layout-wrapper.hamlet #-}
$newline never
$doctype 5
<html>
  <head>
    <title>#{pageTitle pc}
    ^{pageHead pc}
  <body>
    ^{pageBody pc}
{-# START_FILE templates/home.hamlet #-}
<h2>Previously submitted files
$if null storedFiles
  <p>No files have been uploaded yet.
$else
  <ul>
    $forall (ident, StoredFile filename _ _) <- storedFiles
      <li>
        <a href=@{PreviewR ident}>#{filename}
<h2>Submit new file
<form method=post action=@{HomeR} enctype=#{formEncType}>
  ^{formWidget}
  <input type="submit" value="Upload">
{-# START_FILE templates/preview.hamlet #-}
<a href=@{HomeR}>home
<h1>#{filename}
<a href=@{DownloadR ident}>download
<p>
  ^{previewBlock}
{-# START_FILE config/models #-}
StoredFile
    name Text
    contentType Text
    content ByteString
{-# START_FILE config/routes #-}
/                   HomeR     GET POST
/file/#Int          PreviewR  GET
/file/#Int/download DownloadR GET
```

The call to `migrateAll` in the "Main.hs" will cause the following to be logged if no file named "database" exists yet.

```
Migrating: CREATE TABLE "stored_file"("id" INTEGER PRIMARY KEY,"name" VARCHAR NOT NULL,"content_type" VARCHAR NOT NULL,"content" BLOB NOT NULL)
[Debug#SQL] "CREATE TABLE \"stored_file\"(\"id\" INTEGER PRIMARY KEY,\"name\" VARCHAR NOT NULL,\"content_type\" VARCHAR NOT NULL,\"content\" BLOB NOT NULL)" [] @(persistent-1.2.3.0:Database.Persist.Sql.Raw ./Database/Persist/Sql/Raw.hs:37:5)
```

## Phase 3: Switching to Persistent Storage

All of the steps needed to access a database have been dealt with. Settings for the backend have been defined in "Config.hs". These are accessed for initialization in "Main.hs" and in calls to <hoogle search="Yesod.Persist.Core.runDB" results="1">runDB</hoogle> through the foundation type's <hoogle search="Yesod.Persist.Core.YesodPersist" results="1">YesodPersist</hoogle> instance. As the database is opened, a model definition defined in "config/models" is used to update the database schema if necessary.

The time has come to exchange our old data access layer for the new one. They are currently running side by side. None of our handlers are importing "Model.hs", so only the old code is accessible. We're going to do this all at once, but there are actually 3 small steps to perform:

* Update routing system to identify stored files by a Persistent identifier rather than an `Int`.
* Update accessor functions to perform database operations.
* Update templates to work with the new `StoredFile` type.

### Updating routing system

We need to update the routing definition specified in "config/routes". It currently uses `Int` values to identify files. Persistent uses abstracted types for this purpose. In the case of our application this type is either `StoredFileId` or `Key StoredFile`.

``` haskell
{-# START_FILE config/routes #-}
-- /                   HomeR     GET POST
-- /file/#Int          PreviewR  GET
-- /file/#Int/download DownloadR GET
{-hi-}/                            HomeR     GET POST
/file/#StoredFileId          PreviewR  GET
/file/#StoredFileId/download DownloadR GET{-/hi-}
```

Our `getPreviewR` and `getDownloadR` handler actions will need to be updated to use the new types:

``` haskell
{-# START_FILE Handler/Download.hs #-}
module Handler.Download where

import qualified Data.Text as Text
import qualified Data.Text.Encoding as Text

import Yesod

import Foundation
{-hi-}import Model{-/hi-}

-- getDownloadR :: Int -> Handler TypedContent
{-hi-}getDownloadR :: Key StoredFile -> Handler TypedContent{-/hi-}
getDownloadR ident = do
    StoredFile filename contentType bytes <- getById ident
    addHeader "Content-Disposition" $ Text.concat
        [ "attachment; filename=\"", filename, "\""]
    sendResponse (Text.encodeUtf8 contentType, toContent bytes)
{-# START_FILE Handler/Preview.hs #-}
module Handler.Preview where

import Control.Exception hiding (Handler)
import qualified Data.ByteString as SB
import Data.Default
import Data.Text (Text)
import qualified Data.Text as Text
import qualified Data.Text.Encoding as Text
import Text.Blaze
import Yesod
import Yesod.Default.Util

import Foundation
{-hi-}import Model{-/hi-}

-- getPreviewR :: Int -> Handler Html
{-hi-}getPreviewR :: Key StoredFile -> Handler Html{-/hi-}
getPreviewR ident = do
    StoredFile filename contentType bytes <- getById ident
    defaultLayout $ do
        setTitle . toMarkup $ "File Processor - " `Text.append` filename
        previewBlock <- liftIO $ preview ident contentType bytes
        $(widgetFileNoReload def "preview")

-- preview :: Int -> Text -> SB.ByteString -> IO Widget
{-hi-}preview :: Key StoredFile -> Text -> SB.ByteString -> IO Widget{-/hi-}
preview ident contentType bytes
  | "image/" `Text.isPrefixOf` contentType =
    return [whamlet|<img src=@{DownloadR ident}>|]
  | otherwise = do
    eText <- try . evaluate $ Text.decodeUtf8 bytes :: IO (Either SomeException Text)
    return $ case eText of
      Left _ -> errorMessage
      Right text -> [whamlet|<pre>#{text}|]
  where
    errorMessage = [whamlet|<pre>Unable to display file contents.|]
```

### Updating accessor functions

Starting with [part 2](https://www.fpcomplete.com/school/advanced-haskell/building-a-file-hosting-service-in-yesod/part%202) of this series we began manipulating our collection of files through a few accessor functions. Every piece of code having to do with storage was contained in the Foundation module. Accessors run in the `Handler` monad for convenient access to the foundation type and so they can short circuit with an HTTP error response if needed.

As you will see, most of these will shrink down to a single line. Having given our foundation type a 
<hoogle search="Yesod.Persist.Core.YesodPersist" results="1">YesodPersist</hoogle> instance, we can execute a database query in handler actions with <hoogle search="Yesod.Persist.Core.runDB" results="1">runDB</hoogle>. Read the <hoogle search="Database.Persist.Class" results="1">Database.Persist.Class</hoogle> module's documentation to see which operations can be performed.

#### addFile

We'll start with the `addFile` accessor. Previously this was the most complicated of them, but is now the most simple. Persistent manages automatically incrementing identifiers for us:

``` haskell
addFile :: StoredFile -> Handler ()
-- addFile file = do
--     app <- getYesod
--     liftIO . atomically $ do
--         ident <- getNextId app
--         modifyTVar (tstore app) $ IntMap.insert ident file
{-hi-}addFile file = runDB $ insert_ file{-/hi-}
```

This was the only place where we were calling `getNextId`, so it can be deleted:

``` haskell
-- getNextId :: App -> STM Int
-- getNextId app = do
--     nextId <- readTVar $ tnextId app
--     writeTVar (tnextId app) $ nextId + 1
--     return nextId
```

#### getById

The `getById` accessor is used in `getPreviewR` and `getDownloadR` to retrieve information about a specific file. As with elsewhere, we are now identifying files with a `Key StoredFile` rather than an `Int`.

The interesting thing is that we are able to short circuit with an HTTP 404 response if visitors click on a broken link such as "http://myhost.com/file/987654321".

``` haskell
-- getById :: Int -> Handler StoredFile
-- getById ident = do
--     app <- getYesod
--     store <- liftIO . readTVarIO $ tstore app
--     case IntMap.lookup ident store of
--       Nothing -> notFound
--       Just file -> return file
{-hi-}getById :: Key StoredFile -> Handler StoredFile
getById ident = do
    mfile <- runDB $ get ident
    case mfile of
      Nothing -> notFound
      Just file -> return file{-/hi-}
```

#### getList

The `getList` accessor is used by the `getHomeR` route handler to generate a list of hyperlinks to preview pages. The key pieces of information needed are the internal identifier for a file and its name. Previously we returned a tuple, but Persistent has a specific type for this purpose:

``` haskell
-- getList :: Handler [(Int, StoredFile)]
-- getList = do
--     app <- getYesod
--     store <- liftIO . readTVarIO $ tstore app
--     return $ IntMap.toList store
{-hi-}getList :: Handler [Entity StoredFile]
getList = runDB $ selectList [] []{-/hi-}
```

The empty lists given to <hoogle search="Database.Persist.Class.selectList" results="1">selectList</hoogle> are used for sorting and filtering. You might sort alphabetically by name with `selectList [] [Asc StoredFileName]`. Use combinators found in the <hoogle search="Database.Persist" results="1">Database.Persist</hoogle> module to specify filter criteria. You can filter out all but .png files with `selectList [StoredFileContentType ==. "image/png"] []`. I recommend experimenting with this.

### Updating templates

Our "Handler/Home.hs" handler module and associated Hamlet template will need to be updated slightly because of changes we made to `getList`:

``` haskell
{-# START_FILE Handler/Home.hs #-}
module Handler.Home where

import Control.Monad.Trans.Resource
import qualified Data.ByteString as S
import qualified Data.ByteString.Lazy as L
import Data.Conduit
import Data.Conduit.Binary
import Data.Default
import Yesod
import Yesod.Default.Util

import Foundation
{-hi-}import Model{-/hi-}

getHomeR :: Handler Html
getHomeR = do
    (formWidget, formEncType) <- generateFormPost uploadForm
    storedFiles <- getList
    defaultLayout $ do
        setTitle "File Processor"
        $(widgetFileNoReload def "home")

postHomeR :: Handler Html
postHomeR = do
    ((result, _), _) <- runFormPost uploadForm
    case result of
      FormSuccess fi -> do
        fileBytes <- runResourceT $ fileSource fi $$ sinkLbs
        addFile $ StoredFile (fileName fi) (fileContentType fi)
                             (S.pack . L.unpack $ fileBytes)
      _ -> return ()
    redirect HomeR

uploadForm :: Html -> MForm Handler (FormResult FileInfo, Widget)
uploadForm = renderDivs $ fileAFormReq "file"
{-# START_FILE templates/home.hamlet #-}
<h2>Previously submitted files
$if null storedFiles
  <p>No files have been uploaded yet.
$else
  <ul>
$#     $forall (ident, StoredFile filename _ _) <- storedFiles
{-hi-}    $forall (Entity ident (StoredFile filename _ _)) <- storedFiles{-/hi-}
      <li>
        <a href=@{PreviewR ident}>#{filename}
<h2>Submit new file
<form method=post action=@{HomeR} enctype=#{formEncType}>
  ^{formWidget}
  <input type="submit" value="Upload">
```

### Combined example

Here is a combined look at all the changes we've made in this section.

[github branch: 05-08](https://github.com/mikesteele81/soh-file-server-tutorial-project/tree/05-08)

``` active haskell web
{-# START_FILE Config.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}

module Config where

import Database.Persist.Sqlite

persistConfig :: SqliteConf
persistConfig = SqliteConf "database" 100
{-# START_FILE Dispatch.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Dispatch where

import Yesod

import Foundation
import Handler.Download
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

-- import Control.Concurrent.STM
-- import Data.ByteString (ByteString)
import Data.Default
-- import Data.IntMap (IntMap)
-- import qualified Data.IntMap as IntMap
-- import Data.Text (Text)
-- import qualified Data.Text as Text
import Database.Persist.Sql
import Text.Hamlet
import Yesod
import Yesod.Default.Util

import Config
{-hi-}import Model{-/hi-}

-- data StoredFile = StoredFile !Text !Text !ByteString
-- type Store = IntMap StoredFile
-- data App = App
--     { tnextId :: TVar Int
--     , tstore :: TVar Store
--     , connPool :: ConnectionPool
--     }
{-hi-}data App = App
    { connPool :: ConnectionPool
    }{-/hi-}

instance Yesod App where
  defaultLayout widget = do
    pc <- widgetToPageContent $ $(widgetFileNoReload def "default-layout")
    withUrlRenderer $(hamletFile "templates/default-layout-wrapper.hamlet")

instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage

instance YesodPersist App where
  type YesodPersistBackend App = SqlBackend
  runDB action = defaultRunDB (const persistConfig) connPool action

instance YesodPersistRunner App where
  getDBRunner = defaultGetDBRunner connPool

mkYesodData "App" $(parseRoutesFile "config/routes")

-- getNextId :: App -> STM Int
-- getNextId app = do
--     nextId <- readTVar $ tnextId app
--     writeTVar (tnextId app) $ nextId + 1
--     return nextId

-- getList :: Handler [(Int, StoredFile)]
-- getList = do
--     app <- getYesod
--     store <- liftIO . readTVarIO $ tstore app
--     return $ IntMap.toList store
{-hi-}getList :: Handler [Entity StoredFile]
getList = runDB $ selectList [] []{-/hi-}

addFile :: StoredFile -> Handler ()
-- addFile file = do
--     app <- getYesod
--     liftIO . atomically $ do
--         ident <- getNextId app
--         modifyTVar (tstore app) $ IntMap.insert ident file
{-hi-}addFile file = runDB $ insert_ file{-/hi-}

-- getById :: Int -> Handler StoredFile
-- getById ident = do
--     app <- getYesod
--     store <- liftIO . readTVarIO $ tstore app
--     case IntMap.lookup ident store of
--       Nothing -> notFound
--       Just file -> return file
{-hi-}getById :: Key StoredFile -> Handler StoredFile
getById ident = do
    mfile <- runDB $ get ident
    case mfile of
      Nothing -> notFound
      Just file -> return file{-/hi-}
-- /show
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}

-- show
module Main where

-- import Control.Concurrent.STM
import Control.Monad.Logger
import Control.Monad.Trans.Resource
-- import Data.IntMap
import Database.Persist.Sql
import Yesod

import Config
import Dispatch ()
import Foundation
import Model (migrateAll)

main :: IO ()
main = do
    pool <- createPoolConfig persistConfig
    runResourceT $ runStderrLoggingT $ flip runSqlPool pool
        $ runMigration migrateAll
--     tstore <- atomically $ newTVar empty
--     tident <- atomically $ newTVar 0
--     warpEnv $ App tident tstore pool
{-hi-}    warpEnv $ App pool{-/hi-}
-- /show
{-# START_FILE Model.hs #-}
-- show
-- /show
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE GADTs #-}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Model where

import Data.ByteString (ByteString)
import Data.Text (Text)
import Database.Persist.Quasi
import Yesod
import Data.Typeable (Typeable)

share [mkPersist sqlSettings, mkMigrate "migrateAll"]
    $(persistFileWith lowerCaseSettings "config/models")
{-# START_FILE Handler/Download.hs #-}
{-# LANGUAGE OverloadedStrings #-}

-- show
module Handler.Download where

import qualified Data.Text as Text
import qualified Data.Text.Encoding as Text

import Yesod

import Foundation
{-hi-}import Model{-/hi-}

-- getDownloadR :: Int -> Handler TypedContent
{-hi-}getDownloadR :: Key StoredFile -> Handler TypedContent{-/hi-}
getDownloadR ident = do
    StoredFile filename contentType bytes <- getById ident
    addHeader "Content-Disposition" $ Text.concat
        [ "attachment; filename=\"", filename, "\""]
    sendResponse (Text.encodeUtf8 contentType, toContent bytes)
-- /show
{-# START_FILE Handler/Home.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
module Handler.Home where

import Control.Monad.Trans.Resource
import qualified Data.ByteString as S
import qualified Data.ByteString.Lazy as L
import Data.Conduit
import Data.Conduit.Binary
import Data.Default
import Yesod
import Yesod.Default.Util

import Foundation
{-hi-}import Model{-/hi-}

getHomeR :: Handler Html
getHomeR = do
    (formWidget, formEncType) <- generateFormPost uploadForm
    storedFiles <- getList
    defaultLayout $ do
        setTitle "File Processor"
        $(widgetFileNoReload def "home")

postHomeR :: Handler Html
postHomeR = do
    ((result, _), _) <- runFormPost uploadForm
    case result of
      FormSuccess fi -> do
        fileBytes <- runResourceT $ fileSource fi $$ sinkLbs
        addFile $ StoredFile (fileName fi) (fileContentType fi)
                             (S.pack . L.unpack $ fileBytes)
      _ -> return ()
    redirect HomeR

uploadForm :: Html -> MForm Handler (FormResult FileInfo, Widget)
uploadForm = renderDivs $ fileAFormReq "file"
-- /show
{-# START_FILE Handler/Preview.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}

-- show
module Handler.Preview where

import Control.Exception hiding (Handler)
import qualified Data.ByteString as SB
import Data.Default
import Data.Text (Text)
import qualified Data.Text as Text
import qualified Data.Text.Encoding as Text
import Text.Blaze
import Yesod
import Yesod.Default.Util

import Foundation
{-hi-}import Model{-/hi-}

-- getPreviewR :: Int -> Handler Html
{-hi-}getPreviewR :: Key StoredFile -> Handler Html{-/hi-}
getPreviewR ident = do
    StoredFile filename contentType bytes <- getById ident
    defaultLayout $ do
        setTitle . toMarkup $ "File Processor - " `Text.append` filename
        previewBlock <- liftIO $ preview ident contentType bytes
        $(widgetFileNoReload def "preview")

-- preview :: Int -> Text -> SB.ByteString -> IO Widget
{-hi-}preview :: Key StoredFile -> Text -> SB.ByteString -> IO Widget{-/hi-}
preview ident contentType bytes
  | "image/" `Text.isPrefixOf` contentType =
    return [whamlet|<img src=@{DownloadR ident}>|]
  | otherwise = do
    eText <- try . evaluate $ Text.decodeUtf8 bytes :: IO (Either SomeException Text)
    return $ case eText of
      Left _ -> errorMessage
      Right text -> [whamlet|<pre>#{text}|]
  where
    errorMessage = [whamlet|<pre>Unable to display file contents.|]
-- /show
{-# START_FILE templates/default-layout.cassius #-}
body
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
    clear: both

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
^{widget}
{-# START_FILE templates/default-layout-wrapper.hamlet #-}
$newline never
$doctype 5
<html>
  <head>
    <title>#{pageTitle pc}
    ^{pageHead pc}
  <body>
    ^{pageBody pc}
{-# START_FILE templates/home.hamlet #-}
<h2>Previously submitted files
$if null storedFiles
  <p>No files have been uploaded yet.
$else
  <ul>
$#     $forall (ident, StoredFile filename _ _) <- storedFiles
{-hi-}    $forall (Entity ident (StoredFile filename _ _)) <- storedFiles{-/hi-}
      <li>
        <a href=@{PreviewR ident}>#{filename}
<h2>Submit new file
<form method=post action=@{HomeR} enctype=#{formEncType}>
  ^{formWidget}
  <input type="submit" value="Upload">
{-# START_FILE templates/preview.hamlet #-}
<a href=@{HomeR}>home
<h1>#{filename}
<a href=@{DownloadR ident}>download
<p>
  ^{previewBlock}
{-# START_FILE config/models #-}
StoredFile
    name Text
    contentType Text
    content ByteString
{-# START_FILE config/routes #-}
-- /                   HomeR     GET POST
-- /file/#Int          PreviewR  GET
-- /file/#Int/download DownloadR GET
{-hi-}/                            HomeR     GET POST
/file/#StoredFileId          PreviewR  GET
/file/#StoredFileId/download DownloadR GET{-/hi-}
```

# Summary

We've taken the "File Server" project template available from the FP Haskell Center, and modified it so that application state is preserved across restarts. You are welcome to modify and extend this application for your own use. Complete source code with a step-by-step history and additional documentation is available on GitHub at [https://github.com/mikesteele81/soh-file-server-tutorial-project](https://github.com/mikesteele81/soh-file-server-tutorial-project).

With the changes we've made in this entry, there are a few features that would be nice to have. One obvious shortcoming is that there is no way to delete individual files. Support for file deletion would require adding a new accessor function, route, POST handler, optional GET handler, and a form. I challenge readers to review what we've covered up to this point by implementing file deletion themselves. I may demonstrate how to do this in a future entry.

Thank you for following along. Please direct questions, corrections, and other feedback to the [tutorial author feedback](https://plus.google.com/105779205729062195299/posts/Uh9dzjbV6qs) section of FP Complete's Google Plus [community page](https://plus.google.com/communities/108451803388539942499).

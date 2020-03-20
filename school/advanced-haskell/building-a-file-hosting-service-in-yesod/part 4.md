# Part 4

This is the last entry in a series where we write a file serving web
application from start to finish. Today we're going to add two user-facing
features. One of these will be an addition to the preview page so that images
can be viewed. The other will be to place a download link on the preview page.

# Using Unique Identifiers

Before adding any features, I want to improve the way files are stored in
memory. Right now, each file's name and contents are stored in an association
list, `[(Text, ByteString)]`. It's possible for multiple entries to share the
same name. This is a problem because `getById` operates on the assumption that
files can be uniquely looked up by name.

If we were to continue expanding this application there would be other
problems with our list of files as well. We might at some point want to add
the ability to rename files, for example. File names are part of each
URL. Renaming them would break web links.

## Step 1: Access Files by Int

What we need is a pool of unique numbers, and the best place to store it is in
the foundation type. Let's go a step further and define a data type for
uploaded files as well. We'll continue using an association list, but file
names and contents will now be keyed on their unique number as a group.

``` haskell
-- type StoredFile = (Text, ByteString)
-- type Store = [StoredFile]
-- data App = App (TVar Store)
{-hi-}data StoredFile = StoredFile !Text !ByteString
type Store = [(Int, StoredFile)]
data App = App (TVar Int) (TVar Store){-/hi-}
```

We'll need a way to generate these unique numbers. The following STM action will be
put in "Foundation.hs" along with other accessor functions.

``` haskell
getNextId :: App -> STM Int
getNextId (App tnextId _) = do
    nextId <- readTVar tnextId
    writeTVar tnextId $ nextId + 1
    return nextId
```

These are not very invasive changes. We're still using an association list to
keep track of files. The only difference is that they are keyed on an `Int`
rather than a `Text`. In cases like this I like to update obvious things such
as type signatures first, and then work through compiler errors to make all
the needed adjustments in function bodies.

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

module Foundation where

import Control.Concurrent.STM
import Data.ByteString.Lazy (ByteString)
import Data.Default
import Data.Text (Text)
import qualified Data.Text as Text
import Text.Hamlet
import Yesod
import Yesod.Default.Util

-- show
-- type StoredFile = (Text, ByteString)
-- type Store = [StoredFile]
-- data App = App (TVar Store)
{-hi-}data StoredFile = StoredFile !Text !ByteString
type Store = [(Int, StoredFile)]
data App = App (TVar Int) (TVar Store){-/hi-}

instance Yesod App where
  defaultLayout widget = do
    pc <- widgetToPageContent $ $(widgetFileNoReload def "default-layout")
    withUrlRenderer $(hamletFile "templates/default-layout-wrapper.hamlet")

instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage

mkYesodData "App" $(parseRoutesFile "config/routes")

{-hi-}getNextId :: App -> STM Int
getNextId (App tnextId _) = do
    nextId <- readTVar tnextId
    writeTVar tnextId $ nextId + 1
    return nextId{-/hi-}

-- getList :: Handler [StoredFile]
{-hi-}getList :: Handler [(Int, StoredFile)]{-/hi-}
getList = do
    App _ tstore <- getYesod
    liftIO $ readTVarIO tstore

addFile :: App -> StoredFile -> Handler ()
addFile app@(App _ tstore) file =
    liftIO . atomically $ do
        nextId <- getNextId app
        modifyTVar tstore $ \ files -> (nextId, file) : files

-- getById :: Text -> Handler ByteString
{-hi-}getById :: Int -> Handler StoredFile{-/hi-}
getById ident = do
    App _ tstore <- getYesod
    store <- liftIO $ readTVarIO tstore
    case lookup ident store of
      Nothing -> notFound
      Just file -> return file
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
    tnextid <- newTVarIO 1
    tstore <- newTVarIO []
    warpEnv $ App tnextid tstore
{-# START_FILE Handler/Home.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Handler.Home where

import Control.Monad.Trans.Resource
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
        addFile app $ StoredFile (fileName fi) fileBytes
      _ -> return ()
    redirect HomeR

uploadForm = renderDivs $ fileAFormReq "file"
{-# START_FILE Handler/Preview.hs #-}
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

-- show
-- getPreviewR :: Text -> Handler Html
{-hi-}getPreviewR :: Int -> Handler Html{-/hi-}
getPreviewR fileid = do
    StoredFile filename bytes <- getById fileid
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
-- /show
{-# START_FILE templates/default-layout.cassius #-}
-- show
-- /show
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
$if null storedFiles
  <p>No files have been uploaded yet.
$else
  <ul>
    $forall (fileid, StoredFile filename _) <- storedFiles
      <li>
        <a href=@{PreviewR fileid}>#{filename}
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
/          HomeR    GET POST
{-hi-}/file/#Int PreviewR GET{-/hi-}
```

The example listed above is an active code sample so that you can work through
the function bodies with me as we get them to compile.

### getList

The `getList` action is used on the main page to display a list of links to
file summary pages. Generating these links will require an additional piece of
information from now on. We'll need the unique identifier of each file to
generate link targets.

``` haskell
-- getList :: Handler [StoredFile]
{-hi-}getList :: Handler [(Int, StoredFile)]{-/hi-}
getList = do
--    App tstore <- getYesod
{-hi-}    App _ tstore <- getYesod{-/hi-}
    liftIO $ readTVarIO tstore
```

### addFile

The `addFile` action needs to generate a unique identifier before adding
anything to the store. The `getNextId` STM action operates within the same
transaction used to modify our file store.

``` haskell
addFile :: App -> StoredFile -> Handler ()
-- addFile (App tstore) file =
{-hi-}addFile app@(App _ tstore) file ={-/hi-}
    liftIO . atomically $ do
--        modifyTVar tstore $ \files -> file : files
{-hi-}        ident <- getNextId app
        modifyTVar tstore $ \files -> (ident, file) : files{-/hi-}
```

### routing configuration

Our routing file needs to be updated. The preview route now takes an `Int`
rather than a `Text` to designate files. Making this change will mean that our
`getPreviewR` must be updated as well.

``` haskell
{-# START_FILE config/routes #-}
/          HomeR    GET POST
{-hi-}/file/#Int PreviewR GET{-/hi-}
```

### getPreviewR

The `getPreviewR` handler takes an identifier instead of a file name. The name
of the file is now returned by `getById` along with its contents.

``` haskell
-- getPreviewR :: Text -> Handler Html
{-hi-}getPreviewR :: Int -> Handler Html{-/hi-}
-- getPreviewR filename = do
--    bytes <- getById filename
{-hi-}getPreviewR ident = do
    StoredFile filename bytes <- getById ident{-/hi-}
    defaultLayout $ do
        setTitle . toMarkup $ "File Processor - " `Text.append` filename
        previewBlock <- liftIO $ preview bytes
        $(widgetFileNoReload def "preview")
```

### home page template

The home page template needs to be updated. Our `storedFiles` binding used to
hold a `[Text]` representing all file names. It now contains a `[(Int,
StoredFile)]` representing the mapping between unique identifiers and files.

``` haskell
{-# START_FILE templates/home.hamlet #-}
<h2>Previously submitted files
$if null storedFiles
  <p>No files have been uploaded yet.
$else
  <ul>
$#    $forall filename <- storedFiles
$#      <li>
$#        <a href=@{PreviewR filename}>#{filename}
{-hi-}    $forall (ident, StoredFile filename _) <- storedFiles
      <li>
        <a href=@{PreviewR ident}>#{filename}{-/hi-}
<h2>Submit new file
<form method=post action=@{HomeR} enctype=#{formEncType}>
  ^{formWidget}
  <input type="submit" value="Upload">
```

### postHomeR

A minor update needs to be made to `postHomeR`. The `addFile` action takes a
`StoredFile` rather than a `(Text, ByteString)`.

``` haskell
postHomeR :: Handler Html
postHomeR = do
    ((result, _), _) <- runFormPost uploadForm
    case result of
      FormSuccess fi -> do
        app <- getYesod
        fileBytes <- runResourceT $ fileSource fi $$ sinkLbs
--        addFile app (fileName fi, fileBytes)
{-hi-}        addFile app $ StoredFile (fileName fi) fileBytes{-/hi-}
      _ -> return ()
    redirect HomeR
```

## Step 2: Store Files in an IntMap

There is a further improvement that can be made now that our application
accesses files by `Int`. Right now all of the files are stored in an
association list. I would rather use a data type that prevents the possibility
of duplicate keys.

I split these changes into 2 steps to reduce the chance of making a
mistake. Switching from `[(Int, StoredFile)]` to `IntMap StoredFile` should
not require any changes outside of accessor functions and `main` where the state is initialized.

``` haskell
{-# START_FILE Foundation.hs #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
module Foundation where

import Control.Concurrent.STM
import Data.ByteString.Lazy (ByteString)
import Data.Default
{-hi-}import Data.IntMap (IntMap)
import qualified Data.IntMap as IntMap{-/hi-}
import Data.Text (Text)
import qualified Data.Text as Text
import Text.Hamlet
import Yesod
import Yesod.Default.Util

data StoredFile = StoredFile !Text !ByteString
-- type Store = [(Int, StoredFile)]
{-hi-}type Store = IntMap StoredFile{-/hi-}
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
--    liftIO $ readTVarIO tstore
{-hi-}    store <- liftIO $ readTVarIO tstore
    return $ IntMap.toList store{-/hi-}

addFile :: App -> StoredFile -> Handler ()
addFile app@(App _ tstore) file =
    liftIO . atomically $ do
        ident <- getNextId app
--        modifyTVar tstore $ \files -> (ident, file) : files
{-hi-}        modifyTVar tstore $ IntMap.insert ident file{-/hi-}

getById :: Int -> Handler StoredFile
getById ident = do
    App _ tstore <- getYesod
    store <- liftIO $ readTVarIO tstore
--    case lookup ident store of
{-hi-}    case IntMap.lookup ident store of{-/hi-}
      Nothing -> notFound
      Just file -> return file
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}

-- show
module Main where

import Control.Concurrent.STM
import Data.IntMap

import Yesod

import Dispatch ()
import Foundation

main :: IO ()
main = do
--    tstore <- atomically $ newTVar []
{-hi-}    tstore <- atomically $ newTVar empty{-/hi-}
    tident <- atomically $ newTVar 0
    warpEnv $ App tident tstore
-- /show
```

## An Overview

Here is a listing of all the changes that have been made in this section. We're now
ready to resume adding user-visible features.

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
import Data.Default
{-hi-}import Data.IntMap (IntMap)
import qualified Data.IntMap as IntMap{-/hi-}
import Data.Text (Text)
import qualified Data.Text as Text
import Text.Hamlet
import Yesod
import Yesod.Default.Util

-- type StoredFile = (Text, ByteString)
-- type Store = [StoredFile]
-- data App = App (TVar Store)
{-hi-}data StoredFile = StoredFile !Text !ByteString
type Store = IntMap StoredFile
data App = App (TVar Int) (TVar Store){-/hi-}

instance Yesod App where
  defaultLayout widget = do
    pc <- widgetToPageContent $ $(widgetFileNoReload def "default-layout")
    withUrlRenderer $(hamletFile "templates/default-layout-wrapper.hamlet")

instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage

mkYesodData "App" $(parseRoutesFile "config/routes")

{-hi-}getNextId :: App -> STM Int
getNextId (App tnextId _) = do
    nextId <- readTVar tnextId
    writeTVar tnextId $ nextId + 1
    return nextId{-/hi-}

-- getList :: Handler [StoredFile]
{-hi-}getList :: Handler [(Int, StoredFile)]{-/hi-}
getList = do
--    App tstore <- getYesod
--    liftIO $ readTVarIO tstore
{-hi-}    App _ tstore <- getYesod
    store <- liftIO $ readTVarIO tstore
    return $ IntMap.toList store{-/hi-}

addFile :: App -> StoredFile -> Handler ()
-- addFile (App tstore) file =
{-hi-}addFile app@(App _ tstore) file ={-/hi-}
    liftIO . atomically $ do
--        modifyTVar tstore $ \files -> file : files
{-hi-}        ident <- getNextId app
        modifyTVar tstore $ IntMap.insert ident file{-/hi-}

-- getBd :: Text -> Handler ByteString
{-hi-}getById :: Int -> Handler StoredFile{-/hi-}
getById ident = do
    App _ tstore <- getYesod
    store <- liftIO $ readTVarIO tstore
--    case lookup ident store of
{-hi-}    case IntMap.lookup ident store of{-/hi-}
      Nothing -> notFound
      Just file -> return file
-- /show
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}

-- show
module Main where

import Control.Concurrent.STM
import Data.IntMap
import Yesod

import Dispatch ()
import Foundation

main :: IO ()
main = do
--    tstore <- atomically $ newTVar []
{-hi-}    tstore <- atomically $ newTVar empty{-/hi-}
    tident <- atomically $ newTVar 0
    warpEnv $ App tident tstore
-- /show
{-# START_FILE Handler/Home.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
module Handler.Home where

import Control.Monad.Trans.Resource
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
--        addFile app (fileName fi, fileBytes)
{-hi-}        addFile app $ StoredFile (fileName fi) fileBytes{-/hi-}
      _ -> return ()
    redirect HomeR

uploadForm = renderDivs $ fileAFormReq "file"
-- /show
{-# START_FILE Handler/Preview.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}

-- show
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

-- getPreviewR :: Text -> Handler Html
{-hi-}getPreviewR :: Int -> Handler Html{-/hi-}
-- getPreviewR filename = do
--    bytes <- getById filename
{-hi-}getPreviewR ident = do
    StoredFile filename bytes <- getById ident{-/hi-}
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
-- /show
{-# START_FILE templates/default-layout.cassius #-}
-- show
-- /show
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
<h2>Previously submitted files
$if null storedFiles
  <p>No files have been uploaded yet.
$else
  <ul>
$#    $forall filename <- storedFiles
$#      <li>
$#        <a href=@{PreviewR filename}>#{filename}
{-hi-}    $forall (ident, StoredFile filename _) <- storedFiles
      <li>
        <a href=@{PreviewR ident}>#{filename}{-/hi-}
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
/          HomeR    GET POST
{-hi-}/file/#Int PreviewR GET{-/hi-}
```

# Serving Files

One of the primary features which I've delayed adding until now is the ability
to download files. My reason for doing this is that we have to consider the
reality that competing web browsers behave differently from each other.

Most operating systems decide how to interpret a file based on the last part
of its name. It's natural to assume that this logic would carry over to files
accessed through a URL, but not all devices that can run web browsers store files with names. When our
application sends a file, it should include a header designating the file's
type. The browser, whether it be Internet Explorer, a mobile browser, or
some other HTTP-enabled application, will use this to decide what to do with
it. It won't matter whether the URL ends in something that looks like a
well-known file type.

## Storing Content Types

What should our files' type designations, their MIME types, be? There is a
list of common MIME types, but all of the popular web browser and operating
system combinations expect different strings. For this application we will use
whichever MIME type was supplied when the file was uploaded. As with HTTP
response headers, requests also supply a header to declare file types. Yesod
makes this available to us as the file's `fileContentType`. We will store this
along with the file name and content. The following example will hide all but
the few blocks of text that change.


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
-- data StoredFile = StoredFile !Text !ByteString
{-hi-}data StoredFile = StoredFile !Text !Text !ByteString{-/hi-}
type Store = IntMap StoredFile
data App = App (TVar Int) (TVar Store)
-- /show

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

addFile :: App -> StoredFile -> Handler ()
addFile app@(App _ tstore) file =
    liftIO . atomically $ do
        ident <- getNextId app
        modifyTVar tstore $ IntMap.insert ident file

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
{-# START_FILE Handler/Home.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Handler.Home where

import Control.Monad.Trans.Resource
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

-- show
postHomeR :: Handler Html
postHomeR = do
    ((result, _), _) <- runFormPost uploadForm
    case result of
      FormSuccess fi -> do
        app <- getYesod
        fileBytes <- runResourceT $ fileSource fi $$ sinkLbs
--        addFile app $ StoredFile (fileName fi) fileBytes
{-hi-}        addFile app $ StoredFile (fileName fi) (fileContentType fi) fileBytes{-/hi-}
      _ -> return ()
    redirect HomeR
-- /show

uploadForm = renderDivs $ fileAFormReq "file"
{-# START_FILE Handler/Preview.hs #-}
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

-- show
getPreviewR :: Int -> Handler Html
getPreviewR ident = do
--    StoredFile filename bytes <- getById ident
{-hi-}    StoredFile filename _ bytes <- getById ident{-/hi-}
    defaultLayout $ do
        setTitle . toMarkup $ "File Processor - " `Text.append` filename
        previewBlock <- liftIO $ preview bytes
        $(widgetFileNoReload def "preview")
-- /show

preview :: LB.ByteString -> IO Widget
preview bytes = do
    eText <- try . evaluate $ LT.decodeUtf8 bytes :: IO (Either SomeException LT.Text)
    return $ case eText of
      Left _ -> errorMessage
      Right text -> [whamlet|<pre>#{text}|]
  where
    errorMessage = [whamlet|<pre>Unable to display file contents.|]
{-# START_FILE templates/default-layout.cassius #-}
-- show
-- /show
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
<h2>Previously submitted files
$if null storedFiles
  <p>No files have been uploaded yet.
$else
  <ul>
$#    $forall (ident, StoredFile filename _) <- storedFiles
{-hi-}    $forall (ident, StoredFile filename _ _) <- storedFiles{-/hi-}
      <li>
        <a href=@{PreviewR ident}>#{filename}
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
/          HomeR    GET POST
/file/#Int PreviewR GET
```

## File Download Links

In order to serve files we will create a new route along with a custom
handler. The process will be much the same as when we added the preview
page. We'll create a new entry in the "config/routes" file, a new module
within the "Handlers" directory which the "Dispatch" module will import, and
a new handler for when the route is accessed.

``` active haskell web
{-# START_FILE Dispatch.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
module Dispatch where

import Yesod

import Foundation
{-hi-}import Handler.Download{-/hi-}
import Handler.Home
import Handler.Preview

mkYesodDispatch "App" resourcesApp
-- /show
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

addFile :: App -> StoredFile -> Handler ()
addFile app@(App _ tstore) file =
    liftIO . atomically $ do
        ident <- getNextId app
        modifyTVar tstore $ IntMap.insert ident file

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
{-# LANGUAGE OverloadedStrings #-}

-- show
{-hi-}module Handler.Download where

import qualified Data.Text as Text
import qualified Data.Text.Encoding as Text

import Yesod

import Foundation

getDownloadR :: Int -> Handler TypedContent
getDownloadR ident = do
    StoredFile filename contentType bytes <- getById ident
    addHeader "Content-Disposition" $ Text.concat
        [ "attachment; filename=\"", filename, "\""]
    sendResponse (Text.encodeUtf8 contentType, toContent bytes){-/hi-}
-- /show
{-# START_FILE Handler/Home.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Handler.Home where

import Control.Monad.Trans.Resource
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
        addFile app $ StoredFile (fileName fi) (fileContentType fi) fileBytes
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

getPreviewR :: Int -> Handler Html
getPreviewR ident = do
    StoredFile filename _ bytes <- getById ident
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
-- show
-- /show
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
{-hi-}<a href=@{DownloadR ident}>download{-/hi-}
<p>
  ^{previewBlock}
{-# START_FILE config/routes #-}
/                   HomeR     GET POST
/file/#Int          PreviewR  GET
{-hi-}/file/#Int/download DownloadR GET{-/hi-}
```

Try executing the previous example to see how it works. Upload a file, and
then download it using the link on the preview page. Your web browser should
prompt you to download a file with the original name even though the URL may
be something like "http://something.fpcomplete.com/file/0/download". Now
delete the two lines starting with `addHeader` in `getDownloadR`, and try
running through the same test to see whether anything changes.

The "Content-Disposition" header hints to your web browser that it should
download the resource as a file rather than trying to display it within the
browser window. Yesod's <hoogle>addHeader</hoogle> is able to add any custom
header you need. Browsers are not required to observe the
"Content-Disposition" header, so we included the content type of our file as
well.

If the file does end up being saved on visitors' computers, their operating
system will determine what to do with the file based on browser settings and
file associations that may be in place.

Consider a PDF file as an example. If the "Content-Disposition" header is set
and acknowledged by the browser, then it will be downloaded as a file. Some
web browsers will prompt the user to open or save it. It will then be up to
the operating system to decide what to do. It's likely that the file will
be opened with Adobe Reader or some other .pdf handler.

In the case where the "Content-Disposition" header is ignored or not set, the
browser will be supplied a content type. This string will probably be
"application/pdf", but in the case of our application the content type will be
whatever was supplied when the file was uploaded. In any case, the browser
will determine what do based on "application/pdf". Google Chrome has a
built-in .pdf reader. Internet Explorer will rely on the Adobe Reader plugin
if that is available. Any web browser may choose open the file externally
or fail to display it at their option.

# Image Previews

In part 3 we wrote a somewhat complicated function to preview the contents of
text files. We did this by attempting to parse the byte stream as text, and
failing if an exception is raised. I mentioned that we would later expand it
to support image previews as well.

When files are uploaded, the web browser supplies an extra bit of information
which hints at the file's contents. Yesod makes this available to us as the
file's content type. Content type designations vary depending on which web
browser is being used. In the case of image files, the content type can
generally be relied on to start with "image/".

Our strategy will be to use the content type that was supplied when the file
was uploaded. Our `preview` action will then return different blocks of HTML
depending on what the content type is. Using this method we could create
sensible previews of a variety of file types.

The `preview` action blindly tries to convert any file to a text
block. Instead of this, let's make it run through a list of alternative
possibilities. Attempting to parse the file as text should be a last resort.

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

addFile :: App -> StoredFile -> Handler ()
addFile app@(App _ tstore) file =
    liftIO . atomically $ do
        ident <- getNextId app
        modifyTVar tstore $ IntMap.insert ident file

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
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Handler.Home where

import Control.Monad.Trans.Resource
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
        addFile app $ StoredFile (fileName fi) (fileContentType fi) fileBytes
      _ -> return ()
    redirect HomeR

uploadForm = renderDivs $ fileAFormReq "file"
{-# START_FILE Handler/Preview.hs #-}
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

-- show
getPreviewR :: Int -> Handler Html
getPreviewR ident = do
--    StoredFile filename _ bytes <- getById ident
{-hi-}    StoredFile filename contentType bytes <- getById ident{-/hi-}
    defaultLayout $ do
        setTitle . toMarkup $ "File Processor - " `Text.append` filename
--        previewBlock <- liftIO $ preview bytes
{-hi-}        previewBlock <- liftIO $ preview ident contentType bytes{-/hi-}
        $(widgetFileNoReload def "preview")

-- preview :: LB.ByteString -> IO Widget
-- preview bytes = do
{-hi-}preview :: Int -> Text -> LB.ByteString -> IO Widget
preview ident contentType bytes
--    = do
  | "image/" `Text.isPrefixOf` contentType =
    return [whamlet|<img src=@{DownloadR ident}>|]
  | otherwise = do{-/hi-}
    eText <- try . evaluate $ LT.decodeUtf8 bytes :: IO (Either SomeException LT.Text)
    return $ case eText of
      Left _ -> errorMessage
      Right text -> [whamlet|<pre>#{text}|]
  where
    errorMessage = [whamlet|<pre>Unable to display file contents.|]
-- /show
{-# START_FILE templates/default-layout.cassius #-}
-- show
-- /show
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
-- /show
<a href=@{HomeR}>home
<h1>#{filename}
<a href=@{DownloadR ident}>download
<p>
  ^{previewBlock}
{-# START_FILE config/routes #-}
-- show
-- /show
/                   HomeR     GET POST
/file/#Int          PreviewR  GET
/file/#Int/download DownloadR GET
```

Try uploading a file ending in ".png", ".jpg", ".bmp", or some other common
image format. You should see the image displayed.

# Summary

We've written a web application that lets us upload files to a server, and
then download those files using a generated URL. Uploaded files are only
temporarily stored in memory, and disappear when the application
restarts. You are welcome to modify and extend this application for your own
use. A project template is available within the [FP Haskell Center](https://www.fpcomplete.com/school/project-templates/file-server).

There are a few obvious ways in which this project could be improved. You may
wish to have the application state persist when the server restarts. For this,
I would recommend using the "persistent" library. You would then want to add a
way to delete files from the main screen or preview page.

Another feature that would be nice to have is the ability to restrict files to
authenticated users. There are many login systems available, and Yesod makes
it easy to integrate them. When it comes to serving files there are a few
things to be aware of. Some web browsers pass off file download actions to a
separate process. Since authentication is handled with cookies, the download
request comes from a separate, unauthenticated client agent. Android's default
web browser works this way. Certain media players which have built-in web
streaming capabilities will also cause this to happen. The way around this
would be to generate temporary download links that the official download link
redirects to.

Thank you for following along. I hope you found this tutorial useful, and will
be able to integrate some part of it in your own projects.

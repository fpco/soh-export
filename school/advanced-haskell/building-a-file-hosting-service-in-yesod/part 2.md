# Part 2

This is part 2 of a series where we develop a file hosting web application. In
the previous entry everything was contained within a single source file, but
the final application will be too large for this. We will also want the
flexibility of separating the web content from the service itself.

# Branching Out

Let's start by splitting our source code into separate files. Our final
application will have 3 routes and 2 pages. It will be natural to place the
program logic for handling these in their own source files. Our directory
structure will look as follows:

```
Main.hs
Foundation.hs
Dispatch.hs
Handler/Home.hs

config/routes
templates/home.hamlet
```

All of the markup we create will go into the "templates" folder. The "Handler"
folder will contain a single file for each of our 3 routes. Our application
will need to perform a few initializations as it starts, and these will go in
"Main.hs". The "Foundation.hs" module will house a few data structures and
functions to keep track of uploaded files.

## Separate Markup from Source Code

I want us to be in the position where we can make changes to web content
without touching any source code. Each page will have its own templates for HTML,
CSS, and JavaScript. We'll group these together by giving them the same name,
but ending with different file extensions. There are helper functions in the
<hoogle>Yesod.Default.Util</hoogle> module that will know how to load and
combine them correctly. Place your templates in a folder named "templates".

``` active haskell web
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Main where

{-hi-}import Data.Default{-/hi-}
import Yesod
{-hi-}import Yesod.Default.Util{-/hi-}

data App = App
instance Yesod App

-- mkYesod "App" [parseRoutes|
-- / HomeR GET
-- |]
{-hi-}mkYesod "App" $(parseRoutesFile "config/routes"){-/hi-}

getHomeR :: Handler Html
getHomeR = defaultLayout $ do
    let filenames = ["readme.txt", "report.pdf", "music.wav"] :: [String]
    setTitle "File Processor"
--     toWidget [whamlet|
-- <h2>Previously submitted files
-- $if null filenames
--     <p>No files have been uploaded yet.
-- $else
--     <ul>
--         $forall filename <- filenames
--             <li>#{filename}
-- |]
{-hi-}    $(widgetFileNoReload def "home"){-/hi-}

main :: IO ()
main = warpEnv App
{-# START_FILE templates/home.hamlet #-}
{-hi-}$if null filenames
    <p>No files have been uploaded yet.
$else
    <h2>Previously submitted files
    <ul>
        $forall filename <- filenames
            <li>#{filename}{-/hi-}
{-# START_FILE config/routes #-}
{-hi-}/ HomeR GET{-/hi-}
```

Two new files have been added to the project. Our routing definition is now
kept in "config/routes". Our Hamlet template has been placed in
"templates/home.hamlet". If we wanted to define any CSS to be used
specifically with this template, we would have named it
"templates/home.cassius". Our "home.hamlet" template is still able to contain
a reference to `filenames` even though it is in an external file.

## Breaking Apart Dependencies

We want logic for handling our pages to each be in its own module. In order
to achieve this goal we'll first need to break apart some dependencies in the
source code. Replacing <hoogle>mkYesod</hoogle> with calls to
<hoogle>mkYesodData</hoogle> and <hoogle>mkYesodDispatch</hoogle> will
accomplish this. The former generates glue code between our program and the
Yesod Web Framework. The latter generates code to handle routing between URL
strings and Haskell data types.

``` active haskell web
{-# START_FILE Dispatch.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
module Dispatch where

import Data.Default
import Yesod
import Yesod.Default.Util

import Foundation

{-hi-}mkYesodDispatch "App" resourcesApp{-/hi-}

getHomeR :: Handler Html
getHomeR = do
    let filenames = ["readme.txt", "report.pdf", "music.wav"] :: [String]
    defaultLayout $ do
        setTitle "File Processor"
        $(widgetFileNoReload def "home")
-- /show
{-# START_FILE Foundation.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
module Foundation where

import Yesod

data App = App
instance Yesod App

mkYesodData "App" $(parseRoutesFile "config/routes")
-- /show
{-# START_FILE Main.hs #-}
module Main where

import Yesod

{-hi-}import Dispatch (){-/hi-}
import Foundation

main :: IO ()
main = warpEnv App
{-# START_FILE templates/home.hamlet #-}
-- show
-- /show
$if null filenames
    <p>No files have been uploaded yet.
$else
    <h2>Previously submitted files
    <ul>
        $forall filename <- filenames
            <li>#{filename}
{-# START_FILE config/routes #-}
-- show
-- /show
/ HomeR GET
```

Take a look at the call to <hoogle>mkYesodDispatch</hoogle> in
"Dispatch.hs". You may be wondering where `resourcesApp` came from. It was
generated by <hoogle>mkYesodData</hoogle> in "Foundation.hs". Now look at
"Main.hs", and notice the line reading, `import Dispatch ()`. This means that
the only thing we want to import from "Dispatch.hs" is its
<hoogle>Yesod</hoogle> class instance.

## Separate Route Handlers

The only handler we have so far is `getHomeR`. Let's create a new module for
it. Later on we'll add a handler for POST requests to the root route in the
same module. The "Dispatch.hs" file will never need to be touched again. It
will only contain automatically generated code.

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

mkYesodDispatch "App" resourcesApp

-- getHomeR :: Handler Html
-- getHomeR = do
--     let filenames = ["readme.txt", "report.pdf", "music.wav"] :: [String]
--     defaultLayout $ do
--         setTitle "File Processor"
--         $(widgetFileNoReload def "home")
-- /show
{-# START_FILE Foundation.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Foundation where

import Yesod

data App = App
instance Yesod App

mkYesodData "App" $(parseRoutesFile "config/routes")
{-# START_FILE Main.hs #-}
-- show
-- /show
module Main where

import Yesod

import Dispatch ()
import Foundation

main :: IO ()
main = warpEnv App
{-# START_FILE Handler/Home.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
{-hi-}module Handler.Home where

import Data.Default
import Yesod
import Yesod.Default.Util

import Foundation

getHomeR :: Handler Html
getHomeR = do
    let filenames = ["readme.txt", "report.pdf", "music.wav"] :: [String]
    defaultLayout $ do
        setTitle "File Processor"
        $(widgetFileNoReload def "home"){-/hi-}
-- /show
{-# START_FILE templates/home.hamlet #-}
-- show
-- /show
$if null filenames
    <p>No files have been uploaded yet.
$else
    <h2>Previously submitted files
    <ul>
        $forall filename <- filenames
            <li>#{filename}
{-# START_FILE config/routes #-}
-- show
-- /show
/ HomeR GET
```

# Storing Filenames in Memory

The main page should display a list of filenames which changes over time. This
list will be held in memory. We will access it through the `App` data
type. It's currently an empty record, but we can put something inside it by
changing its declaration. Here is how we might store a list of file names.

``` haskell
data App = App [Text]
```

The foundation type is always available within methods of the
<hoogle>Yesod</hoogle> class and handler actions by calling
<hoogle>getYesod</hoogle>. The following example shows how to store the list
of filenames within global application state instead of a local `let` binding.

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
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
module Foundation where

{-hi-}import Data.Text (Text){-/hi-}

import Yesod

-- data App = App
{-hi-}data App = App [Text]{-/hi-}
instance Yesod App

mkYesodData "App" $(parseRoutesFile "config/routes")

{-hi-}getList :: Handler [Text]
getList = do
    App state <- getYesod
    return state{-/hi-}
-- /show
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}

-- show
module Main where

import Yesod

import Dispatch ()
import Foundation

main :: IO ()
-- main = warpEnv App
{-hi-}main = warpEnv $ App ["readme.txt", "report.pdf", "music.wav"]{-/hi-}
-- /show
{-# START_FILE Handler/Home.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
module Handler.Home where

import Data.Default
import Yesod
import Yesod.Default.Util

import Foundation

getHomeR :: Handler Html
getHomeR = do
--     let filenames = ["readme.txt", "report.pdf", "music.wav"] :: [String]
{-hi-}    filenames <- getList{-/hi-}
    defaultLayout $ do
        setTitle "File Processor"
        $(widgetFileNoReload def "home")
-- /show
{-# START_FILE templates/home.hamlet #-}
-- show
-- /show
$if null filenames
    <p>No files have been uploaded yet.
$else
    <h2>Previously submitted files
    <ul>
        $forall filename <- filenames
            <li>#{filename}
{-# START_FILE config/routes #-}
-- show
-- /show
/ HomeR GET
```

Notice that our list of filenames is now being initialized in the `main`
action. From there the list will remain in memory, and will always be
available. Filenames are accessed through the new `getList` handler
action. Other functions to work with the list of files will be added to
"Foundation.hs" as we need them.

## Modifying the Filename List at Runtime

There is one problem with our application state, however. There's no way to
change it at runtime. There is no way to alter a value once it has been bound
to a name or used in a data constructor. The Yesod framework takes our
initialized foundation type in the call to <hoogle>warpEnv</hoogle> and passes
it around to handler actions, but does not provide us with any way to replace
it.

The way around this is to store reference types or database connections in the
foundation type. In the following example our foundation type's read-only
list, a `[Text]`, will be changed to a `TVar [Text]`. This marks it as a
transactional variable. Any time we want to modify it, we will have to do so
within an STM block by calling <hoogle>atomically</hoogle>. When all we want
to do is see what data is stored, we can do so with
<hoogle>readTVarIO</hoogle>.

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
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
module Foundation where

{-hi-}import Control.Concurrent.STM{-/hi-}
import Data.Text (Text)

import Yesod

-- data App = App [Text]
{-hi-}data App = App (TVar [Text]){-/hi-}
instance Yesod App

mkYesodData "App" $(parseRoutesFile "config/routes")

getList :: Handler [Text]
getList = do
--     App state <- getYesod
--     return state
{-hi-}    App tstate <- getYesod
    liftIO $ readTVarIO tstate{-/hi-}

{-hi-}addFile :: App -> Text -> Handler ()
addFile (App tstore) op =
    liftIO . atomically $ do
        modifyTVar tstore $ \ ops -> op : ops{-/hi-}
-- /show
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}

-- show
module Main where

{-hi-}import Control.Concurrent.STM{-/hi-}

import Yesod

import Dispatch ()
import Foundation

main :: IO ()
-- main = warpEnv $ App ["readme.txt", "report.pdf", "music.wav"]
{-hi-}main = do
    tfilenames <- atomically $ newTVar ["readme.txt", "report.pdf", "music.wav"]
    warpEnv $ App tfilenames{-/hi-}
-- /show
{-# START_FILE Handler/Home.hs #-}
-- show
-- /show
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}
module Handler.Home where

import Data.Default
import Yesod
import Yesod.Default.Util

import Foundation

getHomeR :: Handler Html
getHomeR = do
    filenames <- getList
    defaultLayout $ do
        setTitle "File Processor"
        $(widgetFileNoReload def "home")
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
{-# START_FILE config/routes #-}
-- show
-- /show
/ HomeR GET
```

I added a handler action named `addFile` even though we aren't using it
yet. The next section will include a form to upload files, and we will need it
then.

# Uploading Files

Let's add a form to the main page so that visitors can upload files to the
server. Yesod makes it easy to process HTML forms without having to perform
any bookkeeping to map HTML inputs to Haskell.

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
-- show
{-hi-}{-# LANGUAGE MultiParamTypeClasses #-}{-/hi-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Foundation where

import Control.Concurrent.STM
import Data.Text (Text)

import Yesod

data App = App (TVar [Text])
instance Yesod App

{-hi-}instance RenderMessage App FormMessage where
  renderMessage _ _ = defaultFormMessage{-/hi-}

mkYesodData "App" $(parseRoutesFile "config/routes")

getList :: Handler [Text]
getList = do
    App tstate <- getYesod
    liftIO $ readTVarIO tstate

addFile :: App -> Text -> Handler ()
addFile (App tstore) op =
    liftIO . atomically $ do
        modifyTVar tstore $ \ ops -> op : ops
-- /show
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}

-- show
module Main where

import Control.Concurrent.STM

import Yesod

import Dispatch ()
import Foundation

main :: IO ()
main = do
--     tfilenames <- atomically $ newTVar ["readme.txt", "report.pdf", "music.wav"]
{-hi-}    tfilenames <- atomically $ newTVar []{-/hi-}
    warpEnv $ App tfilenames
-- /show
{-# START_FILE Handler/Home.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

-- show
module Handler.Home where

import Data.Default
import Yesod
import Yesod.Default.Util

import Foundation

getHomeR :: Handler Html
getHomeR = do
    {-hi-}(formWidget, formEncType) <- generateFormPost uploadForm{-/hi-}
    filenames <- getList
    defaultLayout $ do
        setTitle "File Processor"
        $(widgetFileNoReload def "home")

{-hi-}postHomeR :: Handler Html
postHomeR = do
    ((result, _), _) <- runFormPost uploadForm
    case result of
      FormSuccess fi -> do
        app <- getYesod
        addFile app $ fileName fi
      _ -> return ()
    redirect HomeR

uploadForm = renderDivs $ fileAFormReq "file"{-/hi-}
-- /show
{-# START_FILE templates/home.hamlet #-}
$if null filenames
    <p>No files have been uploaded yet.
$else
    <h2>Previously submitted files
    <ul>
        $forall filename <- filenames
            <li>#{filename}
{-hi-}<h2>Submit new file
<form method=post action=@{HomeR} enctype=#{formEncType}>
    ^{formWidget}
    <input type="submit" value="Upload">{-/hi-}
{-# START_FILE config/routes #-}
{-hi-}/ HomeR GET POST{-/hi-}
```

The file list is now initialized to an empty list in `main`. Locate
`uploadForm` in the `Handler.Home` module. It defines a form with a single
required field labeled "file". Calling <hoogle>generateFormPost</hoogle>
produces a widget and an encoding type to include in our page. The widget, a
block of HTML, CSS, and JavaScript, is bound to `formWidget`. It is embedded
in our template with `^{formWidget}`.

Our routing definition now contains "POST" in addition to "GET". This means
that the root route will accept POST requests. We'll need to write a
`postHomeR` route handler. The upload form's definition is accessed a second
time to process any request data that arrives.

The final piece of new syntax in the Hamlet template has to do with
routing. Including `@{HomeR}` where a URL would normally go causes Yesod to
generate a correct URL snippet to include in the final HTML. Our route
definition causes a separate Haskell data type to be created for every
possible route. So far our only route is `HomeR`. There will eventually be two
more, one for a file summary page, and one for a download link.

# Summary

Try running the last example. Upload a file or two. Observe that the list of
files updates dynamically as soon as the upload form redirects you back to the
main page. In the next part of this series I will show you how to store a
file's contents in addition to its name. We'll create a preview page that will
show some useful information.

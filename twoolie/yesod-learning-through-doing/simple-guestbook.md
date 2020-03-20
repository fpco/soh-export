# Simple Guestbook

Here i'm going to show you how to build a super simple web app using haskell and yesod. For the purposes of the tutorial, this is a single piece of code, however you would split the code among multiple filesif you were to build on this project.

Everyone knows that the internet peaked in the 90s and so today we're going to build somthing that no 90s website can be without, the Guest Book.

``` active haskell web
-- show Imports and Pragmas
{-# LANGUAGE OverloadedStrings, FlexibleContexts, ScopedTypeVariables,
             GeneralizedNewtypeDeriving, TemplateHaskell, GADTs,
             TypeFamilies, QuasiQuotes, MultiParamTypeClasses #-}

-- As usual, a generous helping of pragmas and imports.
-- These will need to be placed at the top of your source files.

-- Everything you need to work with Yesod, exported from one package
import Yesod

-- Standard platform datatypes & functions
import Data.Text (Text)
import Data.Time (UTCTime, getCurrentTime)
import Data.Time.Format (formatTime)
import System.Locale (defaultTimeLocale)

-- Using the `persistent` database layer
import Database.Persist
import Database.Persist.TH
import Database.Persist.Sqlite

-- Monad helpers
import Control.Applicative ((<$>),(<*>))
import Control.Monad.IO.Class (liftIO)
import Control.Monad.Logger (runNoLoggingT)
import Control.Monad.Trans.Resource (runResourceT)

-- show Types.hs

-- Our datatype that represents our App
-- the fields of this type represent pieces of the
-- global environment in which it runs.
data GuestBook = GuestBook { _db :: Connection }

-- Declare that our App is a Yesod site
-- This will provide you with sensible defaults
-- and your website's defaultLayout
instance Yesod GuestBook where

  defaultLayout contents = do
    PageContent title headTags bodyTags <- widgetToPageContent $ do  
      addStylesheetRemote "//netdna.bootstrapcdn.com/twitter-bootstrap/2.3.2/css/bootstrap-combined.min.css"
      toWidget [cassius|
        body
          padding: 0px 10px
      |]
      contents
    hamletToRepHtml [hamlet|
      $doctype 5
      <html>
        <head>
          <title>#{title}
          ^{headTags}
        <body>^{bodyTags}
    |]

-- Tell yesod to use the SqlPersist backend by default
instance YesodPersist GuestBook where
    type YesodPersistBackend GuestBook = SqlPersist
    
    -- We use runDB to more easily thread our database
    -- connection into functions in the Handler monad.
    runDB action = getYesod >>= runSqlConn action . _db

-- Provide a default pattern for the internationalisation machinery.
instance RenderMessage GuestBook FormMessage where
    renderMessage _ _ = defaultFormMessage

-- show Models.hs

-- Define our data models & create a function to setup our tables.
-- (Uses a generous helping of Template Haskell)
share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
Signature
  name     Text
  message  Text
  date     UTCTime
    deriving Show
|]

-- show Routes.hs

-- More TH, this time generating Type-Safe Routes
-- This will generate a sum datatype that describes
-- your app's routes and datatypes.
mkYesod "GuestBook" [parseRoutes|
  /      HomeR   GET
  /sign  SubmitR GET POST
|]

-- show Forms.hs

-- Simple form for creating Signatures
entryForm = renderBootstrap $ Signature
  <$> areq textField "Name" Nothing
  <*> areq textField "Message" Nothing
  <*> lift (liftIO getCurrentTime)

-- show Views.hs

-- Our homepage view, querying and displaying signatures
-- from our all-important guestbook
getHomeR = do
  -- Query from the db, annotating type so the compiler knows what we're querying
  signatures :: [Entity Signature] <- runDB $ selectList [] [Asc SignatureDate]
  -- Render out the template
  defaultLayout [whamlet|
    <h1>Welcome to the Guestbook!
    <table .table.table-bordered>
      <thead>
        <tr>
          <th> Name
          <th> Message
          <th> When
      <tbody>
        <!-- $forall loops over all the values -->
        $forall Entity _ signature <- signatures
          <tr>
            <td> #{signatureName    signature}
            <td> #{signatureMessage signature}
            <td> #{fTime $ signatureDate signature}
    <!-- @{SubmitR} fills in the rendered route to our signing url -->
    <p><a href=@{SubmitR}>Sign Guestbook.
  |] where
       -- A quick and dirty time formatter
       fTime :: UTCTime -> String
       fTime = formatTime defaultTimeLocale "%F %X %Z"

renderEntryForm widget enctype = [whamlet|
  <form method=POST action=@{SubmitR} enctype=#{enctype}>
    <table>^{widget}
    <div .form-actions>
      <button .btn.btn-primary type=submit> Sign Guestbook
|]

-- The GET handler for our signing page
getSubmitR = do
  (formWidget, enctype) <- generateFormPost entryForm
  defaultLayout [whamlet|
    <h1> Sign The Guestbook!
    ^{renderEntryForm formWidget enctype}
    <p><a href=@{HomeR}>Back Home.
  |]

-- The POST handler for our signing page
postSubmitR = do
  -- run the Posted form, and retrieve the values
  ((result, formWidget), enctype) <- runFormPost entryForm
  case result of
    FormSuccess signature@(Signature name message _) -> do
      -- Insert our Signature object into the Database
      runDB $ insert signature
      -- Render out the success template
      defaultLayout [whamlet|
        <h2> Successfully Submitted!
        <blockquote>
          #{message}
          <br>-- #{name}
        <p><a href=@{HomeR}>Back to Guestbook.
      |]
    FormFailure _ -> defaultLayout [whamlet|
        <h2> OOPS! Please, Try again.
        ^{renderEntryForm formWidget enctype}
      |]
    FormMissing -> redirect SubmitR

-- show Main.hs

-- We'll use a temporary in-memory database for this example
main = withSqliteConn ":memory:" $ \conn -> do
  time <- liftIO getCurrentTime
  
  runNoLoggingT $ runResourceT $ flip runSqlConn conn $ do
    -- Create the db tables (running our auto-generated migration)
    runMigration migrateAll
  
    -- Create some example data
    insert $ Signature "Johnny" "Hello, Haskeller" time
    insert $ Signature "James" "Guestbook? What is this, Geocities?" time
    insert $ Signature "Judy" "Wow, totally Retro" time
  
  -- Initialise our webserver and begin serving content
  -- Here we bind our database connection into our application state.
  warpEnv $ GuestBook { _db = conn }

```

And there you go, A simple application to curate your very own piece of internet nostalgia.

You can find out more about yesod by reading the docs at [YesodWeb](http://www.yesodweb.com/book)
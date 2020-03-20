# Yesod Routing

# Routing Fundamentals

If you're familiar with other web frameworks, you will have a pretty good idea of the basic paradigm of URL routing, but for the uninitiated I will explain the basics here.

When you visit your favorite website the first thing your browser will request from the server a specific resource. The very first line of the connection to the server will often look like this 

    GET /blog/archive/2005/10/02 HTTP/1.1

The first part of the header, namely `GET` is known as the Method. Here' we're performing a simple GET request to retrieve the contents of the resource located at `/blog/archive/2005/10/02` and using the `HTTP/1.1` version of the protocol to do so. The location of the resource is known as Path which the framework maps to a Route.

Many web frameworks present some programmable way to specify a link between routes and the specific function that they should run in order to serve that route.

    # Ruby (Using a DSL)
    match "/patients/:id" => "patients#show"

    # Flask (Using a DSL)
    @app.route('/patients/<id>')
    def patients_detail(id): ...
    
    # Django (Using regex)
    url(r'^patients/(?P<id>\d+)/$', views.PatientsDetailView.as_view())
    
When a request comes into the server, the framework interprets the path and maps it to the appropriate route and hands off control the the function associated with that URL.

The result is a function called with string arguments.

In yesod, the fundamental unit

``` haskell
type PatientId = Int

-- This is the URL routing definition
mkYesod "BusinessDirectory" [parseRoutes|
  /patients/#PatientId PatientDetailR GET
|]

-- The views that can be called
getPatientDetailR :: Handler RepHTML
getPatientDetailR id = undefined

```

This defines a single route that takes a single parameter of type `PatientId` and can service the `GET` HTTP method.




``` active haskell web

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

-- Persist
import Database.Persist
import Database.Persist.Sqlite
import Database.Persist.TH

-- Monad helpers
import Control.Applicative ((<$>),(<*>))
import Control.Monad.IO.Class (liftIO)
import Control.Monad.Logger (runNoLoggingT)
import Control.Monad.Trans.Resource (runResourceT)

data BusinessDirectory = BusinessDirectory { _db :: Connection }

instance Yesod BusinessDirectory

instance YesodPersist BusinessDirectory where
    type YesodPersistBackend BusinessDirectory = SqlPersist

    runDB action = getYesod >>= runSqlConn action . _db

share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persist|
Business
    name String
    deriving Show
Contact
    businessId BusinessId
    name String
    deriving Show
|]

mkYesod "BusinessDirectory" [parseRoutes|
  /                                         HomeR     GET
  /business/#BusinessId                     BusinessR GET
  /business/#BusinessId/contact/#ContactId  ContactR  GET
|]

blankR :: Handler RepHtml
blankR = defaultLayout [whamlet|
  <h1> This page intentionally left blank. |]

getHomeR :: Handler RepHtml
getHomeR = do
  businesses <- runDB $ selectList [] [Asc BusinessName]
  defaultLayout [whamlet|
  <h2> Businesses
  <ul>
    $forall Entity business_id business <- businesses
      <li><a href=@{ BusinessR business_id }>
        #{ businessName business } |]

getBusinessR :: BusinessId  -> Handler RepHtml
getBusinessR business_id = do
  Just business <- runDB $ get business_id
  contacts <- runDB $ selectList [ContactBusinessId ==. business_id] [Asc ContactName]
  defaultLayout [whamlet|
  <h2> Business
  <h3> Name
  <p> #{ businessName business }
  <h3> Contacts
  <ul>
    $forall Entity contact_id contact <- contacts
      <li><a href=@{ContactR business_id contact_id}>
        #{ contactName contact }
  |]

getContactR :: BusinessId -> ContactId -> Handler RepHtml
getContactR business_id contact_id = do
  Just business <- runDB $ get business_id
  Just contact  <- runDB $ get contact_id
  let bus_id_from_contact = contactBusinessId contact
  if bus_id_from_contact == business_id then
    defaultLayout [whamlet|
      <h2> Contact
      <h3> Name
      <p>  #{ contactName contact }
      <h3> Business
      <p>  #{ businessName business }
    |]
  else
    redirect $ ContactR bus_id_from_contact contact_id

main = withSqliteConn ":memory:" $ \conn -> do

  runNoLoggingT $ runResourceT $ flip runSqlConn conn $ do
    runMigration migrateAll
    b1 <- insert $ Business "Sales Direct Corp. 1"
    insert $ Contact b1 "Contact Person 1"
    insert $ Contact b1 "Contact Person 2"
    b2 <- insert $ Business "Sales Direct Corp. 2"
    insert $ Contact b2 "Contact Person 1"
    insert $ Contact b2 "Contact Person 2"

  warpEnv BusinessDirectory { _db=conn }

```

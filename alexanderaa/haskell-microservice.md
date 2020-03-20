# Web app/Microservice example in Haskell

> "The term 'Microservice Architecture' has sprung up over the last few years to describe a particular way of designing software applications as suites of independently deployable services. While there is **no precise definition** of this architectural style, there are certain common characteristics around organization around business capability, automated deployment, intelligence in the endpoints, and decentralized control of languages and data."
[(Martin Fowler - Microservices)](http://martinfowler.com/articles/microservices.html)


## Introduction

The intention of this tutorial is to build a complete web application in Haskell and to highlight a variety of related topics - providing a starting point for Haskell beginners and discussion topics for everyone else.


### Tools

Haskell is a more complex language than many others. Therefore all code examples run in online FPComplete IDE - to allow more time to experiment with the code rather than build and deployment.

The IDE does not have all libraries preinstalled, so a minor configuration change will be required to run the examples - the lines below should be included into the 'extra packages' section of the IDE config.

```
Hackage: digestive-functors 0.8.0.0
Hackage: hspec-discover 2.1.8
Hackage: hspec-expectations 0.7.0
Hackage: hspec-core 2.1.8
Hackage: hspec 2.1.8
Hackage: hspec-snap 0.3.3.0
Hackage: snaplet-sqlite-simple 0.4.8.3
```

Haskell has a variety of tools to develop web applications. Snap, Yesod, Happstack and other quality frameworks are available for immediate use and it is easy to find their documentation online. Cabal and its recently created sibling, Stack, are used for application deployment and packaging.

The general approach used in Haskell web frameworks is to have a framework-specific monad, which enforces certain kind of API - allows to read request data, send responses, interact with routes, cookies, application state, etc. in a predefined way.


### Starting a project

Normally we would use a sequence of commands like below to start our project:

```
$ mkdir project && cd project  # create project directory
$ cabal sandbox init           # create development sandbox

<Add actual source files, e.g. Main.hs; add dependencies to project.cabal>

$ cabal install                # compile the application

```
Cabal is the name for two different things in Haskell. One of them is a library and another one is a package manager, based on the forementioned library. There is also a new tool, which is compatible with Cabal and is called Stack.

Note, that most frameworks provide project templates, e.g.:

```
$ snap init barebones          # creates a predefined directory/file structure for a new project
```

We will start with a very simplistic application, written using Snap framework. The application does not do anything except that it returns "hello" to every request.

```active haskell
{-# LANGUAGE OverloadedStrings #-}

{- | Simple application.

    No cookies, no logging, no routes,
    no database connections,
    no templates and no forms.
-}

module Main where

import Control.Applicative
import Snap
import qualified System.Environment as E

app :: Snap ()
app = do
    writeText "hello"

main :: IO ()
main = do

    -- FPComplete-specific: we need to get a port number to serve at
    port <- read <$> E.getEnv "PORT"

    -- Serve the app
    httpServe (setPort port defaultConfig) app

```
The application should be pretty self-explanatory at this stage. Try to run the app in the IDE and note the link in the "Console" tab (you might need to scroll up a little bit).

To gain understanding of how things work, it is very useful to review type signatures. For example, using [Hoogle](https://www.haskell.org/hoogle/) or [Hoogle at FPComplete](http://www.stackage.org/lts-2.19/hoogle) or [Hayoo](http://hayoo.fh-wedel.de/) - try to find out what is the type signature for writeText. Both Hoogle and Hayoo could be installed on your development machine.

Snap does not make too many choices in terms of how the application should be structured and which libraries to use. There are multiple choices of libraries for interaction with databases, templating, etc.


## Building a microservice

In this tutorial we will be building an app which provides simple postal code lookup functionality. It returns zip codes for addresses it knows, nothing else.
Lets add a QuickCheck test to the application to see how it works.

```active haskell
{-# LANGUAGE OverloadedStrings #-}

{- | Simple application.

    No cookies, no logging, no routes,
    no database connections,
    no templates and no forms.
-}

module Main where

import Control.Applicative
import Snap
import Test.QuickCheck
import qualified System.Environment as E

app :: Snap ()
app = do
    writeText "hello?"

prop_plus :: Int -> Int -> Bool
prop_plus a b = (a+b) == (b+a)

main :: IO ()
main = do

    -- FPComplete-specific: we need to get a port number to serve at
    port <- read <$> E.getEnv "PORT"

    -- Run tests
    quickCheck prop_plus
    -- Serve the app
    httpServe (setPort port defaultConfig) app

```

There are more specialised libraries, which help to run more framework-specific tests. For example, [Hspec](http://hspec.github.io/writing-specs.html).
Hspec includes integration both with Snap framework and with QuickCheck.

Lets try to use Hspec in our project - describe an application, which responds with "hello", when we visit /hello, and implement the application.

```active haskell
{-# LANGUAGE OverloadedStrings #-}

{- | Simple application. -}

module Main where

import Data.ByteString
import Control.Applicative
import Snap
import Test.Hspec
import qualified Test.Hspec.Snap as T
import qualified System.Environment as E

-- | Application database pool, state, etc. can be added here
data App = App {}

-- | Simple handler
sayHello :: Handler App App ()
sayHello = do
    writeText "hello"

-- | Application routes
routes :: [(ByteString, Handler App App ())]
routes = [("/hello", method GET $ sayHello)]

-- | Initialisation
appInit :: SnapletInit App App
appInit = makeSnaplet "microservice" "My example microservice" Nothing $ do
    addRoutes routes
    return $ App

-- | Tests
tests :: Spec
tests = do
    T.snap (route routes) appInit $ do
        describe "Application" $ do
            it "Has /hello route" $ do
                T.get "/hello" >>= T.should200
            it "Says 'hello'" $ do
                T.get "/hello" >>= T.shouldHaveText "hello"

main :: IO ()
main = do

    -- FPComplete-specific: we need to get a port number to serve at
    port <- read <$> E.getEnv "PORT"

    -- Run tests
    hspec tests

    -- Serve the app
    serveSnaplet (setPort port defaultConfig) appInit
```


Now, it is time to extend the application even further - add database logic.

Things to look at:

* To add database connection, or cookie handling or templating or any global state to the application, we reuse or create a snaplet - basically a module, which provides certain functionality using framework tools and API.

    In this particular case it was a bit challenging to connect database, because FPComplete does not provide database hosting. However, there is still a possibility to create SQLite databases, which we do in this example.

    I also do not know how to create a proper directory structure for Snap configuration files. As a result, to make this example self-contained, I wrote a small Snaplet, which creates in-memory SQLite database without looking into configuration files - sqliteInit.

    As we use in-memory database, we need a way to re-create database on application start. initDb does just that.
    
* To include a snaplet into our application, we add it to the App type. 
    makeLenses is a Template Haskell construct. The idea is pretty simple - it automatically generates accessors (lenses) for the App type. We probably can do the same explicitly if needed (try to remove makeLenses!).

* appInit constains initialization logic for the application - can be used to initialize snaplets, add routes, etc. In this case it only adds routes, creates an in-memory SQLite database and provides a database connection handle.

* MVar is used to manage the connection to our database. MVar is a primitive from Control.Concurrent, which provides a useful API for concurrent programming. putMVar puts value into MVar, when it is empty and blocks otherwise. takeMVar empties the MVar if it is not empty, otherwise blocks.

* hAdd, hFind are handlers. Their responsibility is to take requests and return responses

* query and execute are taken from Database.SQLite.Simple, which is a pretty low-level library (there are quite a few ORMs for Haskell. In contrast, sqlite-simple provides almost direct access to the database, which might be good for certain applications. It is also easier for understanding and looks like a good starting option. 
    * Check if it is possible to get SQL injection vulnerability in the example code. A good starting point will be to look at the Query type.

* Show, Readable, ToField, FromRow type classes provide an interface for data type/format conversion. It is possible to automatically generate type class instances for certain types - look at Address and Postcode for example. "deriving Show" generates Show class instance automatically. However, it is also possible to create instances explicitly - look at ToField instances in the code.
    

```active haskell
{-# LANGUAGE FlexibleInstances   #-}
{-# LANGUAGE OverloadedStrings   #-}
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE TemplateHaskell     #-}

module Main where

import           Control.Applicative
import           Control.Concurrent
import           Control.Lens
import           Data.ByteString
import           Data.Maybe
import qualified Data.Text                        as TX
import qualified Data.Text.Encoding               as TE
import qualified Database.SQLite.Simple           as S
import           Database.SQLite.Simple.ToField
import           Snap
import           Snap.Snaplet.SqliteSimple
import           Snap.Util.Readable
import qualified System.Environment               as E
import           Test.Hspec
import qualified Test.Hspec.Snap                  as T

-----------------------------------------------------------------------------

-- | Application database pool, state, etc. can be added here
data App = App { _db :: Snaplet Sqlite }
makeLenses ''App

instance HasSqlite (Handler b App) where
  getSqliteState = with db get

-----------------------------------------------------------------------------

data Address = Address TX.Text   deriving Show
data Postcode = Postcode TX.Text deriving Show

instance Readable Address where
    fromBS bs = return $ Address (TE.decodeUtf8 bs)
instance Readable Postcode where
    fromBS bs = return $ Postcode (TE.decodeUtf8 bs)


instance ToField Address where
    toField (Address x) = toField x
instance ToField Postcode where
    toField (Postcode x) = toField x

instance FromRow Postcode where
    fromRow = Postcode <$> field
instance ToRow Address where
    toRow (Address x) = [toField x]

-----------------------------------------------------------------------------

-- | Create database schema - we use in-memory database
initDb :: S.Connection -> IO S.Connection
initDb c = do
    S.execute_ c "create table post_code (address text, code int)"
    S.execute_ c "insert into post_code (address, code) values ('Grafton','x1010')"
    return c

-- | Initialise the snaplet
sqliteInit :: SnapletInit b Sqlite
sqliteInit = makeSnaplet "sqlite-simple" description datadir $ do
    c <- liftIO $ S.open ":memory:" >>= initDb >>= newMVar
    return $ Sqlite c
  where
    description = "Sqlite abstraction"
    datadir = Nothing

-- | Initialise app
appInit :: SnapletInit App App
appInit = makeSnaplet "microservice" "My example microservice" Nothing $ do
    addRoutes routes
    d <- nestSnaplet "in-memory db" db Main.sqliteInit
    return $ App d

-----------------------------------------------------------------------------

-- | Handler - finds postcode for known addresses
hFind :: Handler App App ()
hFind = do
    address <- fromJust <$> getParam "address"
    addr <- fromBS $ address
    r <- query "select cast(code as text) from post_code where address = ?" (addr :: Address) :: Handler App App [Postcode]
    writeText $ (TX.pack . show) r

-- | Handler - adds new (address, postcode) pairs
hAdd :: Handler App App ()
hAdd = do
    address  <- fromJust <$> getParam "address"
    postcode <- fromJust <$> getParam "postcode"
    addr :: Address <- fromBS address
    code :: Postcode <- fromBS postcode
    execute "insert into post_code (address, code) values (?,?)" (addr, code)  :: Handler App App ()

-----------------------------------------------------------------------------

-- | Application routes
routes :: [(ByteString, Handler App App ())]
routes = [("/find", hFind),
          ("/add",  hAdd )]

-- | Tests
tests :: Spec
tests =
    T.snap (route routes) appInit $ do
        describe "Application" $ do
            it "Has /add route" $ do
                T.get' "/add" (T.params [("address", "Grafton"), ("postcode", "1010")]) >>= T.should200
            it "Adds (postcode,address) pair" $ do
                T.post "/add" (T.params [("address", "Grafton"), ("postcode", "1010")]) >>= T.should200
            it "Returns postcode for known address" $ do
                T.get' "/find" (T.params [("address", "Grafton")]) >>= T.shouldHaveText "1010"

main :: IO ()
main = do
    -- FPComplete-specific: we need to get a port number to serve at
    port <- read <$> E.getEnv "PORT"

    -- Run tests
    hspec tests

    -- Serve the app
    serveSnaplet (setPort port defaultConfig) appInit
```

At this stage we have a working application, which provides a very simplistic postal code lookup service. There are obviously many things, which could be improved. Some of them are:

* Support for a variety of input/output formats
    * Could be (easily) accomplished with Aeson and similar libraries - that would require us to e.g. define ToJSON and FromJSON type class instances
* Database interactions/ORM
    * There are several quality implementations of ORM in Haskell. For example, [Esqueleto](https://hackage.haskell.org/package/esqueleto) provides a DSL for database manipulations:
```haskell
do people <- select (from $ \person -> return person)
   liftIO $ mapM_ (putStrLn . personName . entityVal) people
```
* Concurrency and multithreading
    * Haskell has really advanced support for writing both concurrent and parallel software. Good starting points will be:
        * [Control.Parallel](https://hackage.haskell.org/package/parallel)
    * and for concurrency primitives:
        * [Control.Concurrent.STM](https://hackage.haskell.org/package/stm)
        * [Control.Concurrent.MVar](http://hackage.haskell.org/package/base-4.8.0.0/docs/Control-Concurrent-MVar.html)
        * [Data.IORef](http://hackage.haskell.org/package/base-4.8.0.0/docs/Data-IORef.html)
* Automated API generation, based on table structure (or any equivalent description)
    * [rest](http://silkapp.github.io/rest/)
    * [postgrest](https://hackage.haskell.org/package/postgrest)
    * .. and many others

## Summary

Haskell language and implementations include a variety of concepts, which assist us to write reliable software. Haskell makes the task of code verification easier, provides practical concurrency primitives, allows to enforce APIs (e.g. remember Snap monad) and combine pieces of software easier. As a side effect, these properties also make average quality of Haskell libraries much higher.

The tutorial highlights several things which might be useful for people who consider Haskell as a language for their next web application - the expectation is that many things mentioned in the tutorial will be not quite clear for beginners.

[Haskell wiki](https://wiki.haskell.org/Haskell) and [School of Haskell](https://www.fpcomplete.com/school) are really helpful and highly recommended resources to explore both theoretical and practical sides of Haskell.

    
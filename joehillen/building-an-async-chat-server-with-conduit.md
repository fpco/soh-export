# Building an Async Chat Server with Conduit

# Introduction

As a more advanced and practical exercise for learning
how to use the data streaming library
[conduit](https://hackage.haskell.org/package/conduit),
I thought I would recreate the asynchronous chat server
from Simon Marlow's book,
[Parallel and Concurrent Programming in Haskell](http://chimera.labs.oreilly.com/books/1230000000929/ch12.html#sec_chat).
If you haven't read it yet,
it's required reading for anyone who wants to build
performant, real-world Haskell applications.

This tutorial assumes you that already know Haskell,
and that you've read these other conduit tutorials:

 * [Conduit Overview](https://www.fpcomplete.com/school/to-infinity-and-beyond/pick-of-the-week/conduit-overview)
 * [network-conduit, async, and conduit-combinators](http://www.yesodweb.com/blog/2014/03/network-conduit-async)

Now, let's get started.

# Setting Up

The first thing we will do is set up a new cabal sandbox
and install some dependencies:

```bash
mkdir conduit-chat
cd conduit-chat
cabal sandbox init
cabal install conduit conduit-combinators network-conduit
```

I ran into a [bug](https://github.com/haskell/cabal/issues/1875)
with cabal 1.20, so you might have to run this in order to install
conduit-combinators:

```bash
cabal install conduit-combinators --constraint 'vector-instances==3.3' --max-backjumps=-1
```

# Simple Server

Let's start with the most basic TCP server.

```haskell
import           Conduit
import           Data.Conduit.Network
import qualified Data.ByteString.Char8 as BS

main :: IO ()
main = runTCPServer (serverSettings 4000 "*") $ \appData ->
    appSource appData $$ awaitForever $ liftIO . BS.putStr
```

Build and run the server using:

```bash
cabal exec runhaskell server.hs
```

And connect to it using telnet:

```bash
telnet 127.0.0.1 4000
```

Not a very interesting server,
but as you might have noticed with the original async chat server
there is a lot of boilerplate when working with async and network Handles.
[network-conduit](hackage.haskell.org/package/network-conduit) relieves most of that boilerplate.


# Logging In

The first thing we want to do is allow users to log in. 

The original login function is called `readName`.
The only login requirement is a username
that doesn't conflict with other users' names.
We are going to have the same function, but we're going to use conduits.

We might want to do something like this with `readName` as a `Conduit`:

```haskell
main :: IO ()
main = do
    server <- newServer
    runTCPServer (serverSettings 4000 "*") $ \appData ->
        appSource appData $$ readName server =$ appSink appData
```

However, we need a way to get the `Client` after `readName` finishes,
but the type of `appSink` is `Sink ByteString IO ()`.

We can use [fuseUpstream](http://hackage.haskell.org/package/conduit-1.1.6/docs/Data-Conduit.html#v:fuseUpstream)
for this.

Here is the result:

```haskell
{-# LANGUAGE OverloadedStrings, RecordWildCards, LambdaCase #-}

import           Conduit
import           Data.Conduit
import           Data.Conduit.Network
import qualified Data.ByteString.Char8 as BS
import           Text.Printf              (printf)
import           Control.Concurrent.STM
import qualified Data.Map as Map
import           Data.Word8               (_cr)
import           Control.Monad
import           Control.Concurrent.Async (concurrently)
import           Control.Exception        (finally)

type ClientName = BS.ByteString

data Client = Client
  { clientName     :: ClientName
  , clientApp      :: AppData
  }


instance Show Client where
    show client =
        BS.unpack (clientName client) ++ "@"
            ++ show (appSockAddr $ clientApp client)


data Server = Server {
    clients :: TVar (Map.Map ClientName Client)
}


data Message = Notice BS.ByteString
             | Tell ClientName BS.ByteString
             | Broadcast ClientName BS.ByteString
             | Command BS.ByteString
             deriving Show


newServer :: IO Server
newServer = do
  c <- newTVarIO Map.empty
  return Server { clients = c }


newClient :: ClientName -> AppData -> STM Client
newClient name app = do
    return Client { clientName = name
                  , clientApp  = app
                  }

checkAddClient :: Server -> ClientName -> AppData -> IO (Maybe Client)
checkAddClient server@Server{..} name app = atomically $ do
    clientmap <- readTVar clients
    if Map.member name clientmap then
        return Nothing
    else do
        client <- newClient name app
        writeTVar clients $ Map.insert name client clientmap
        return (Just client)

-- show
readName :: Server -> AppData -> ConduitM BS.ByteString BS.ByteString IO Client
readName server app = go
  where
  go = do
    yield "What is your name? " $$ appSink app
    name <- lineAsciiC $ takeCE 80 =$= filterCE (/= _cr) =$= foldC
    if BS.null name then
        go
    else do
        ok <- liftIO $ checkAddClient server name app
        case ok of
            Nothing -> do
                respond "The name '%s' is in use, please choose another\n" name
                go
            Just client -> do
                respond "Welcome, %s!\n" name
                return client
  respond msg name = yield $ BS.pack $ printf msg $ BS.unpack name


main :: IO ()
main = do
    server <- newServer
    runTCPServer (serverSettings 4000 "*") $ \app -> do
        client <-
            appSource app $$ readName server app `fuseUpstream` appSink app
        printf "%s has connected" $ BS.unpack $ clientName client
```

# Let's Chat!

Now that we have a client connected and logged in, let's start chatting!

In the original chat server,
a client has a `TChan` to receive messages. 
We can use conduits for this.

First, install [stm-conduit](http://hackage.haskell.org/package/stm-conduit):

```bash
cabal install stm-conduit
```

and we will use it like this:

```haskell
import           Data.Conduit.TMChan

data Client = Client
  { clientName     :: ClientName
  , clientChan     :: TMChan Message
  , clientApp      :: AppData
  }


newClient :: ClientName -> AppData -> STM Client
newClient name app = do
    chan <- newTMChan
    return Client { clientName     = name
                  , clientApp      = app
                  , clientChan     = chan
                  }

main :: IO ()
main = do
    server <- newServer
    runTCPServer (serverSettings 4000 "*") $ \app -> do
        (fromClient, client) <-
            appSource app $$+ readName server app `fuseUpstream` appSink app
        void $ concurrently
            (fromClient $$+- linesUnboundedAsciiC =$= mapC Command =$ sinkTMChan (clientChan client) True)
            (sourceTMChan (clientChan client) $$ handleMessage server client =$ appSink app)

```

We must be sure to use a `ResumableSource` with `$$+` or else there could be data loss.

With this, all lines sent to the client's socket are turned into `Command`s
and written to the client's `TMChan`.
Concurrently, `Message`s sent to the client's `TMChan` are processed by
the conduit `handleMessage`.

This is very similar to the original `handleMessage`
but with two simple changes.

First, instead of `readTVar` at the beginning of the message handling loop,
we will use `awaitForever`. 
Second, instead of `output` writing to a socket `Handle`,
we write to the client's `Sink` using `yield`. 

Thus `output` becomes:

```haskell
output s = yield $ s `BS.append` "\n"
```

Here is the result:    

```haskell
sendMessage :: Client -> Message -> STM ()
sendMessage Client{..} msg = writeTMChan clientChan msg

sendToName :: Server -> ClientName -> Message -> STM Bool
sendToName server@Server{..} name msg = do
    clientmap <- readTVar clients
    case Map.lookup name clientmap of
        Nothing -> return False
        Just client -> sendMessage client msg >> return True

(<++>) = BS.append

handleMessage :: Server -> Client -> Conduit Message IO BS.ByteString
handleMessage server client@Client{..} = awaitForever $ \case
    Notice msg         -> output $ "*** " <++> msg
    Tell name msg      -> output $ "*" <++> name <++> "*: " <++> msg
    Broadcast name msg -> output $ "<" <++> name <++> ">: " <++> msg
    Command msg        -> case BS.words msg of
        ["/tell", who, what] -> do
            ok <- liftIO $ atomically $
                sendToName server who $ Tell clientName what
            unless ok $ output $ who <++> " is not connected."

        -- ignore empty strings
        [""] -> return ()
        [] -> return ()

        -- broadcasts
        ws ->
            if BS.head (head ws) == '/' then
                output $ "Unrecognized command: " <++> msg
            else
                liftIO $ atomically $
                    broadcast server $ Broadcast clientName msg
  where
    output s = yield $ s <++> "\n"
```

Most other functions are just about the same as the original.
Once you put it all together, you should have a working chat server.


# Quitting

One last feature we haven't implemented yet is quitting.
There are multiple ways a user can quit.
First is using the '/quit' command.
Another might be to just drop the TCP connection.

There are several ways to implement this,
but it seems the simplest method would be to raise an exception
to exit out of the conduits
and then notify the other users in the exception handler.

Let's do it!

First we will copy over the `removeClient`
from the original async chat server (with a minor change):

```haskell
removeClient :: Server -> Client -> IO ()
removeClient server@Server{..} client@Client{..} = atomically $ do
    modifyTVar' clients $ Map.delete clientName
    broadcast server $ Notice (clientName <++> " has disconnected")
```

Next we will use it in `main`
(The conduits have been moved to a separate function called `runClient`):

```haskell
runClient :: ResumableSource IO BS.ByteString -> Server -> Client -> IO ()
runClient clientSource server client@Client{..} =
    void $ concurrently
        (clientSource $$+- linesUnboundedAsciiC =$ clientSink client)
        (sourceTMChan clientChan $$ handleMessage server client =$ appSink clientApp)

removeClient :: Server -> Client -> IO ()
removeClient server@Server{..} client@Client{..} = atomically $ do
    modifyTVar' clients $ Map.delete clientName
    broadcast server $ Notice (clientName <++> " has disconnected")

-- show
main :: IO ()
main = do
    server <- newServer
    runTCPServer (serverSettings 4000 "*") $ \app -> do
        (fromClient, client) <-
            appSource app $$+ readName server app =$ appSink app
        (runClient fromClient server client)
            `finally` (removeClient server client)
```

Lastly, we implement the '/quit' command:

```haskell
handleMessage :: Server -> Client -> Conduit Message IO BS.ByteString
handleMessage server client@Client{..} = awaitForever $ \case
    Notice msg -> output $ "*** " <++> msg
    Tell name msg      -> output $ "*" <++> name <++> "*: " <++> msg
    Broadcast name msg -> output $ "<" <++> name <++> ">: " <++> msg
    Command msg        -> case BS.words msg of
        ["/tell", who, what] -> do
            ok <- liftIO $ atomically $
                sendToName server who $ Tell clientName what
            unless ok $ output $ who <++> " is not connected."

-- show
        ["/quit"] -> do
            error . BS.unpack $ clientName <++> " has quit"
-- /show

        -- ignore empty strings
        [""] -> return ()
        [] -> return ()

        -- broadcasts
        ws ->
            if BS.head (head ws) == '/' then
                output $ "Unrecognized command: " <++> msg
            else
                liftIO $ atomically $
                    broadcast server $ Broadcast clientName msg
  where
    output s = yield $ s <++> "\n"
```

# Conclusion

[Here is the whole server put together.](https://gist.github.com/joehillen/b6cc59285d50fd67c120)

There are many more features that could be added,
like more commands (such as '/kick', '/list', and '/help')
or support for rooms,
but those are exercises for the reader.

If you are looking for more conduit practice, 
then I recommend implementing a better chat client than telnet.

I hope this has helped you understand conduits in a more applied way.

Happy Hacking!

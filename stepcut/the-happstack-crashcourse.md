# The Happstack Crashcourse

This is a work in progress. In the meantime you can read the complete version of *The Happstack Crashcourse* [here](http://happstack.com/docs/crashcourse/index.html).

Hello World
===========

Your first app!
---------------

Our first Happstack application is a simple server that responds to all
requests with the string, `Hello, World!`.

``` haskell web active
module Main where

import Happstack.Server.Env (nullConf, simpleHTTP, toResponse, ok)

main :: IO ()
main = simpleHTTP nullConf $ ok "Hello, World!"
```

If you are reading this on [School of
Haskell](https://www.fpcomplete.com/), the examples import the module
`Happstack.Server.Env` instead of `Happstack.Server`. This is a
(hopefully) temporary hack so that the interactive code examples work on
School of Haskell. To run code from School of Haskell locally simply
replace `Happstack.Server.Env` with `Happstack.Server`.

If you are reading this on School of Haskell, you can run the examples
interactively with out installing anything.

If you want to run the code locally, and you have not already installed
Happstack -- you will need to do that first. You can find instructions
on how to install Happstack at
<http://happstack.com/page/view-page-slug/2/download>.

To build the application run:

    $ ghc -threaded HelloWorld.hs -o helloworld

The executable will be named `helloworld`.

Alternatively, you can use `runhaskell` and avoid the compilation step.

    $ runhaskell HelloWorld.hs

Run this app and point your browser at
[`http://localhost:8000/`](http://localhost:8000/). (assuming you are
building the program on your local machine.) The page should load and
say `"Hello, World!"`.

Alternatively, we can use `curl`:

     $ curl http://localhost:8000/
    Hello, World!

`curl` is a command-line utility which can be used to create many types
of HTTP requests. Unlike a browser, it does not attempt to render the
results, it just prints the body of the response to the console.

If you run `curl` with the `-v` option it will provide verbose output
which includes the headers sent back and forth between curl and the
server:

     $ curl -v http://localhost:8000/
     * About to connect() to localhost port 8000 (#0)
     *   Trying 127.0.0.1... connected
     > GET / HTTP/1.1
     > User-Agent: curl/7.22.0 (x86_64-pc-linux-gnu)
     > Host: localhost:8000
     > Accept: */*
     >
     < HTTP/1.1 200 OK
     < Transfer-Encoding: chunked
     < Connection: Keep-Alive
     < Content-Type: text/plain; charset=UTF-8
     < Date: Thu, 13 Dec 2012 00:19:01 GMT
     < Server: Happstack/7.0.7
     <
     * Connection #0 to host localhost left intact
     * Closing connection #0
     Hello, World!

This can sometimes be useful for debugging why your site is not working
as you expect.

`curl` is not required by Happstack or this book, but it is a useful
tool for web development. `curl` is not part of Happstack. The official
curl website is <http://curl.haxx.se>.

The parts of `Hello World`
--------------------------

### Listening for HTTP requests

The `simpleHTTP` function is what actually starts the program listening
for incoming HTTP requests:

``` haskell
simpleHTTP :: (ToMessage a) => Conf -> ServerPartT IO a -> IO ()
```

We'll examine the various parts of this type signature in the following
sections.

### Configuring the HTTP listener

The first argument is some simple server configuration information. It
is defined as:

``` haskell
data Conf = Conf
    { port       :: Int
    , validator  :: Maybe (Response -> IO Response)
    , logAccess  :: forall t. FormatTime t => Maybe (LogAccess t)
    , timeout    :: Int
    }
```

The fields can be described as:

`port`  
the TCP port to listen on for incoming connection

`validator`  
on-the-fly validation of output during development

`logAccess`  
logging function for HTTP requests

`timeout`  
number of seconds to wait before killing an inactive connection

The default config is `nullConf` which is simply defined as:

``` haskell
-- | Default configuration contains no validator and the port is set to 8000
nullConf :: Conf
nullConf = Conf
    { port      = 8000
    , validator = Nothing
    , logAccess = Just logMAccess
    , timeout   = 30
    }
```

### Processing a `Request`

The second argument is a bit more interesting. It is the handler which
processes an incoming HTTP `Request` and generates a `Response`.
`ServerPartT IO a` is essentially a fancy way of writing a function with
the type:

``` haskell
Request -> IO a
```

`simpleHTTP` processes each incoming request in its own thread. It will
parse the `Request`, call your `ServerPartT` handler, and then return
the `Response` to the client. When developing your handler, it is
natural to think about things as if you are writing a program which
processes a single `Request`, generates a `Response`, and exits. However
it is important when doing I/O, such as writing files to disk, or
talking to a database to remember that there may be other threads
running simultaneously.

### Setting the HTTP response code

In this example, our handler is simply:

``` haskell
ok "Hello, World!" :: ServerPartT IO String
```

`ok` is one of several combinators which can be used to set the HTTP
response code. In this case, it will set the response code to `200 OK`.
The type signature for `ok` can be simplified to:

``` haskell
ok :: a -> ServerPartT IO a
```

`ok` acts much like `return` except it also sets the HTTP response code
for a `Response`.

`Happstack.Server.SimpleHTTP` contains similar functions for the common
HTTP response codes including, `notFound`, `seeOther`, `badRequest` and
more.

### Creating a `Response`

The `ToMessage` class is used to turn values of different types into
HTTP responses. It contains three methods:

``` haskell
class ToMessage a where
  toContentType :: a -> ByteString
  toMessage     :: a -> Lazy.ByteString
  toResponse    :: a -> Response
```

A vast majority of the time we only call the `toResponse` method.

`simpleHTTP` automatically calls `toResponse` to convert the value
returned by the handler into a `Response` -- so we did not have to call
`toResponse` explicitly. It converted the `String` `"Hello, World!"`
into a `Response` with the content-type `"text/plain"` and the message
body `"Hello, World!"`

Often times we will opt to explicitly call `toResponse`. For example:

``` haskell active web
-- / show
module Main where

import Happstack.Server.Env (nullConf, simpleHTTP, toResponse, ok)
-- show
main :: IO ()
main = simpleHTTP nullConf $ ok (toResponse "Hello, World!")
```

Happstack comes with pre-defined `ToMessage` instances for many types
such as `Text.Html.Html`, `Text.XHtml.Html`, `String`, the types from
HSP, and more.

Choosing between multiple `ServerPartTs`
----------------------------------------

In the first example, we had only one `ServerPartT`. All `Request`s were
handled by the same part and returned the same `Response`.

In general, our applications will have many `ServerPartT`s. We combine
them into a single `ServerPartT` by using `MonadPlus`. Typically via the
`msum` function:

``` haskell
msum :: (MonadPlus m) => [m a] -> m a
```

In the following example we combine three `ServerPartT`s together.

``` haskell web active
module Main where
```

``` haskell web active
import Control.Monad
import Happstack.Server.Env (nullConf, simpleHTTP, ok, dir)
```

``` haskell web active
main :: IO ()
main = simpleHTTP nullConf $ msum [ mzero
                                  , ok "Hello, World!"
                                  , ok "Unreachable ServerPartT"
                                  ]
```

The behaviour of `MonadPlus` is to try each `ServerPartT` in succession,
until one succeeds.

In the example above, the first part is `mzero`, so it will always fail.
The second part will always succeed. That means the third part will
never be reachable.

Alas, that means this application will appear to behave exactly like the
first application. What we need are some ways to have parts match or
fail depending on the contents of the HTTP `Request`.

Route Filters
=============

*a.k.a Responding to different url paths*

Happstack provides a variety of ways to match on parts of the `Request`
(such as the path or request method) and respond appropriately.

Happstack provides two different systems for mapping the request path to
a handler. In this section we will cover a simple, untyped routing
system. Later we will look at fancier, type-safe routing sytem known as
`web-routes`.

Using `dir` to match on static path components
----------------------------------------------

We can use `dir` to handle components of the URI path which are static.
For example, we might have a site with the two URLs: `hello` and
`goodbye`.

``` haskell web active
module Main where

import Control.Monad
import Happstack.Server.Env (nullConf, simpleHTTP, ok, dir, seeOther)

main :: IO ()
main = simpleHTTP nullConf $ msum
    [ dir "hello"    $ ok "Hello, World!"
    , dir "goodbye"  $ ok "Goodbye, World!"
    , seeOther "/hello" "/hello"
    ]
```

If we start the app and point our browser at
<http://localhost:8000/hello> we get the `hello` message, and if we
point it at <http://localhost:8000/goodbye>, we get the `goodbye`
message.

Using `dir` to match on multiple components
-------------------------------------------

We can match on multiple components by chaining calls to `dir` together:

``` haskell web active
module Main where

import Control.Monad (msum)
import Happstack.Server.Env (nullConf, simpleHTTP, ok, dir)

main :: IO ()
main = simpleHTTP nullConf $ msum [ dir "hello"    $ dir "world" $ ok "Hello, World!"
                                  , dir "goodbye"  $ dir "moon"  $ ok "Goodbye, Moon!"
                                  ]
```

If we start the app and point our browser at
<http://localhost:8000/hello/world> we get the hello message, and if we
point it at <http://localhost:8000/goodbye/moon>, we get the goodbye
message.

Using `dirs` as shorthand to match on multiple components
---------------------------------------------------------

As a shorthand, we can also use `dirs` to handle multiple static patch
components.

``` haskell web active
module Main where

import Control.Monad (msum)
import Happstack.Server.Env (nullConf, simpleHTTP, ok, dirs)

main :: IO ()
main = simpleHTTP nullConf $ msum [ dirs "hello/world"  $ ok "Hello, World!"
                                  , dirs "goodbye/moon" $ ok "Goodbye, Moon!"
                                  ]
```

If we start the app and point our browser at
<http://localhost:8000/hello/world> we get the hello message, and if we
point it at <http://localhost:8000/goodbye/moon>, we get the goodbye
message.

Matching on variable path segments
----------------------------------

Often times a path segment will contain a variable value we want to
extract and use, such as a number or a string. We can use the `path`
combinator to do that.

``` haskell
path :: (FromReqURI a, MonadPlus m, ServerMonad m) => (a -> m b) -> m b
```

You may find that type to be a little hard to follow because it is
pretty abstract looking. Fortunately, we can look at it in an easier
way. A `ServerPart` is a valid instance of, `ServerMonad m`, so we can
just replace the `m` with `ServerPart`. You can do this anywhere you see
type signatures with `(ServerMonad m) =>` in them. In this case, the
final result would look like:

``` haskell
path :: (FromReqURI a) => (a -> ServerPart b) -> ServerPart b
```

`path` will attempt to extract and decode a path segment, and if it
succeeds, it will pass the decode value to the nested server part.

Let's start with the most basic example, extracting a `String` value. We
will extend the Hello World server so that we can say hello to anyone.

``` haskell web active
module Main where

import Control.Monad (msum)
import Happstack.Server.Env (nullConf, simpleHTTP, ok, dir, path)

main :: IO ()
main = simpleHTTP nullConf $ msum [ dir "hello" $ path $ \s -> ok $ "Hello, " ++ s
                                  ]
```

Now, if we start the app and point our browser at:
<http://localhost:8000/hello/World> we get the response
`"Hello, World"`. if we point it at
<http://localhost:8000/hello/Haskell>, we get `"Hello, Haskell"`.

`FromReqURI:` extending `path`
------------------------------

We can extend path so that we can extract our own types from the path
components as well. We simply add an instance to the FromReqURI class:

``` haskell
class FromReqURI a where
    fromReqURI :: String -> Maybe a
```

For example, let's say that we want to create a type to represent
subjects we can greet.

``` haskell web active
 -- show
 module Main where

 import Control.Monad (msum)
 import Data.Char (toLower)
 import Happstack.Server.Env (FromReqURI(..), nullConf, simpleHTTP, ok, dir, path)

-- show Subject data-type

 data Subject = World | Haskell

 sayHello :: Subject -> String
 sayHello World   = "Hello, World!"
 sayHello Haskell = "Greetings, Haskell!"

 -- show We simply add an instance such as:

 instance FromReqURI Subject where
     fromReqURI sub =
         case map toLower sub of
           "haskell" -> Just Haskell
           "world"   -> Just World
           _         -> Nothing

 -- show Now when we use `path` it will extract a value of type `Subject`.

 main :: IO ()
 main = simpleHTTP nullConf $ dir "hello" $ path $ \subject -> ok $ (sayHello subject)
```

Now, if we start the app and point our browser at:
<http://localhost:8000/hello/World> we get the response
`"Hello, World"`. if we point it at
<http://localhost:8000/hello/Haskell>, we get `"Greetings, Haskell!"`.

Matching on request method `(GET, POST, etc)`
---------------------------------------------

We can specify that a route is only valid for specific HTTP request
methods by using the `method` guard:

``` haskell
method :: (ServerMonad m, MonadPlus m, MatchMethod method) => method -> m ()
```

Here is a simple demo app:

``` haskell web active
module Main where

import Control.Monad (msum)
import Happstack.Server.Env (Method(GET, POST), dir, method, nullConf, ok, simpleHTTP)

main :: IO ()
main = simpleHTTP nullConf $ msum
       [ do method GET
            ok $ "You did a GET request.\n"
       , do method POST
            ok $ "You did a POST request.\n"
       , dir "foo" $ do method GET
                        ok $ "You did a GET request on /foo\n"
       ]
```

Using `curl` we can see the expected results for normal `GET` and `POST`
requests to `/`:

     $ curl http://localhost:8000/
    You did a GET request.
     $ curl -d '' http://localhost:8000/
    You did a POST request.

Note that `method` does not require that all the segments of request
path have been consumed. We can see in here that `/foo` is accepted, and
so is `/foo/bar`.

<pre>
 $ curl http://localhost:8000/foo
You did a GET request on /foo
 $ curl http://localhost:8000/foo/bar
You did a GET request on /foo
</pre>


You can use `nullDir` to assert that all the path segments have been
consumed:

``` haskell
nullDir :: (ServerMonad m, MonadPlus m) => m ()
```

Advanced method matching with `MatchMethod`
-------------------------------------------

The method routing functions use a class `(MatchMethod method)` instead
of the concrete type `Method`. The `MatchMethod` class looks like this:

``` haskell
class MatchMethod m where
    matchMethod :: m -> Method -> Bool

instance MatchMethod Method           where ...
instance MatchMethod [Method]         where ...
instance MatchMethod (Method -> Bool) where ...
instance MatchMethod ()               where ...
```

This allows us to easily match on more than one method by either
providing a list of acceptable matches, or by providing a function which
returns a boolean value. We can use this feature to support the `HEAD`
method. When the client does a `HEAD` request, the server is supposed to
return the same headers it would for a GET request, but with an empty
response body. Happstack includes special support for handling this
automatically in most cases.

``` haskell web active
module Main where

import Control.Monad (msum)
import Happstack.Server.Env (Method(GET, HEAD), dir, methodM, nullConf, ok, simpleHTTP)

main :: IO ()
main = simpleHTTP nullConf $ msum
       [ do methodM [GET, HEAD]
            ok $ "Hello, World\n"
       ]
```

We can now use curl to do a normal `GET` request, or we can use the `-I`
flag which does a `HEAD` request:

     $ curl http://localhost:8000/
    Hello, World
     $ curl -I http://localhost:8000/
    HTTP/1.1 200 OK
    Connection: Keep-Alive
    Content-Length: 13
    Content-Type: text/plain; charset=UTF-8
    Date: Tue, 15 Jun 2010 19:56:07 GMT
    Server: Happstack/0.5.0

Happstack automatically notices that it is a `HEAD` request, and does
not send the body.

Other Routing Filters
---------------------

SimpleHTTP includes a number of other useful routing filters, such as:

`nullDir :: (ServerMonad m, MonadPlus m) => m ()`  
check that there are no path segments remaining

`host :: (ServerMonad m, MonadPlus m) => String -> m a -> m a`  
match on a specific host name in the Request

`withHost :: (ServerMonad m, MonadPlus m) => (String -> m a) -> m a`  
Lookup the host header and pass it to the handler.

`uriRest :: (ServerMonad m) => (String -> m a) -> m a`  
Grab the rest of the URL (dirs + query) and passes it to your handler

`anyPath :: (ServerMonad m, MonadPlus m) => m r -> m r`  
Pop any path element and ignore when choosing a 'ServerPartT' to handle
the request.

`trailingSlash :: (ServerMonad m, MonadPlus m) => m ()`  
Guard which checks that the Request URI ends in `/`. Useful for
distinguishing between `foo` and `foo/`


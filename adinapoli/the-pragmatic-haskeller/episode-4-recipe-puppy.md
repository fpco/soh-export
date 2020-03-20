# Episode 4 - Recipe Puppy

## Foreword

This is part of [The Pragmatic Haskeller](https://github.com/cakesolutions/the-pragmatic-haskeller) series.


## Speaking to the world

Now we have our webapp that can read json from the outside world and store them
inside MongoDB. But during my daily job what I usually need to do is to talk
to some REST service and get, manipulate and store some arbitrary JSON. 
Fortunately for us, Haskell and its rich, high-quality libraries ecosystem
makes the process a breeze.

## The "hard" way
Remember what I told you about this series? We have to reason as pragmatic
programmers, choosing the tools which seem more appropriate for the task at
stake. During my initial exploration of the library space, I landed on
[HTTP](http://hackage.haskell.org/package/HTTP-4000.2.8).
It makes the process of making GET requests as simple as:


``` active haskell
module Main where

import Network.HTTP
import Control.Applicative

get :: String -> IO String
get url = do
    response <- simpleHTTP $ getRequest url
    getResponseBody response

main = take 204 <$> get "http://www.alfredodinapoli.com" >>= print
```

Alas, `HTTP` does not support SSL out of the box (a quick glimpse to the
dependency list for this library would be enough):

``` active haskell
module Main where

import Network.HTTP
import Control.Applicative

get :: String -> IO String
get url = do
    response <- simpleHTTP $ getRequest url
    getResponseBody response

-- show
main = take 20 <$> get "https://www.google.it" >>= print
-- /show
```

Now, here is where pragmatism comes into play; if you are planning to do just
"plain" http requests and don't need SSL at all, you can stick with `HTTP` and
live long and happy. In case you have more demanding use case scenarios, you
have two options:

1. Use an Haskell library like [HsOpenSSL](http://hackage.haskell.org/package/HsOpenSSL)
   or [tsl](http://hackage.haskell.org/package/tls) to integrate `HTTP` with SSL.

2. Use a library which supports SSL out of the box, like [http-conduit](http://hackage.haskell.org/package/http-conduit)

### Fred asks: Why not use something like `http-streams`?

Answer: even though I'm a big fan of [io-streams](http://hackage.haskell.org/package/io-streams-1.0.2.1)
per se, I decided *not* to use [http-streams](http://hackage.haskell.org/package/http-streams).
It looks very promising, but I've found one little wart: it depends on `HsOpenSSL`.
This brings two drawbacks on the table:

1. HsOpenSSL's author "wants to encourage you to use and improve the tls 
   package instead as long as possible. The only problem is that the tls 
   package has not received as much review as OpenSSL from cryptography 
   specialists yet, thus we can't assume it's secure enough."

2. Using `HsOpenSSL` makes the process of writing secured http connection a bit
   too verbose, in my humble opinion. This is an excerpt from `http-streams`
   documentation:

``` haskell
import OpenSSL (withOpenSSL)

main :: IO ()
main = withOpenSSL $ do
    ctx <- baselineContextSSL
    c <- openConnectionSSL ctx "api.github.com" 443
    ...
    closeConnection c
```

As we'll see, using `http-conduit` will make the process a no-brainer. Said that,
don't take my words as a judgment over the library quality. I like it very much
and I hope to see a bit of boilerplate scrapped in the future.

## http-conduit

This library took some design choices for you, using `tsl` under the hood.
It's extremely easy to use, so easy that I'll include here an extra snippet,
non included in the Github lesson, but that I hope will be useful as a real
world example. After all, I couldn't be in peace of mind publishing an episode
which claims to be some sort of "solution for real world problems" if I was
limiting the scope to GET requests. Hardly in production you'll do just plain
GET requests.

### Getting some JSON from Foursquare
Imagine the scenario; you need to fetch some JSON from Foursquare (which talks
through SSL) and manipulate this JSON to extract some information. To be honest,
during the [first episode](https://www.fpcomplete.com/school/pick-of-the-week/episode-1-json)
I didn't tell you all the truth about `aeson`.
There are two features which are simply outstanding:

* Generic derivation. Using GHC's Generics you can let `aeson` derive automatically
  the marshalling/unmarshalling boilerplate code. Try this at home:

``` active haskell
{-# LANGUAGE DeriveGeneric #-}
{-# LANGUAGE OverloadedStrings #-}
module Main where

import Data.Aeson
import qualified Data.ByteString.Lazy.Char8 as BL

import GHC.Generics (Generic)

data Person = Person { name :: String,
                       age  :: Int} deriving (Show, Generic)

instance FromJSON Person
instance ToJSON Person

rawData = BL.pack "{\"name\": \"John\", \"age\": 35}"
main = print $ (decode rawData  :: Maybe Person)
```

Pretty impressive, isn't it?

* Manipulation of the "raw" AST. As written in `aeson`'s documentation, 
  "sometimes you want to work with JSON data directly, without first converting
  it to a custom data type. The Value type, which is an instance of `FromJSON`,
  is used to represent an arbitrary JSON AST (abstract syntax tree)."

We can use this awesome functionality to extract the number of checkins for
a FS' venue, with the following code:

``` active haskell
{-# LANGUAGE OverloadedStrings #-}
module Main where

import Data.Monoid
import Data.Aeson
import Network.HTTP.Conduit

type FsVenueId = String

apiUrl :: String
apiUrl = "https://api.foursquare.com/v2/venues/"

requestUrl :: String
requestUrl = "?oauth_token=" <>
             "FGCUCQ0II3HEFFEZYI24U4FBTAP4AUSDHAJWOUX1FIE5QIY5" <> "&v=" <>
             "20130427"

requestBuilder :: FsVenueId -> String
requestBuilder vid = apiUrl <> vid <> requestUrl


getVenue :: FsVenueId -> IO (Maybe Value)
getVenue vid = do
    rawJson <- simpleHttp $ requestBuilder vid
    return (decode rawJson :: Maybe Value)

main = do
  response <- getVenue "40a55d80f964a52020f31ee3"
  case response of
    (Just v) -> print . take 400 . show $ v
    Nothing  -> print "Failed to fetch venue."
```

Et voilà! Now we have a pretty generic data structure, and the task to write
a simple function to find the field "checkinsCount" inside the `Object` "stat"
is left as an exercise for the reader. Highlighted you can see how easy it was
to talk to FS using SSL; yes, just a one liner, which the function <hoogle>simpleHttp</hoogle>.

## Calling Recipe Puppy from our webapp

[Recipe Puppy](http://www.recipepuppy.com/) is a nice REST service I've found googling for a web service
with minimal authentication overhead and, of course, cooking-related. Let's
suppose we want to retrive recipes based on one ingredient contained within,
well, it turns out that all it takes is to call this REST link:

[Find recipes which contain onion](http://www.recipepuppy.com/api?i=garlic)

All we need, then, is a small wrapper which allows us to call our underlying
web service. We'll be using Recipe Puppy for its simplicity, but at this point
you should know we can make complex calls thanks to `HTTP` or `http-conduit`.
The choice I've made was to segregate our routes into a separate file, and
then appending the routes back to the main routes function (inside Site.hs):

``` haskell
{-# LANGUAGE OverloadedStrings #-}

module Pragmatic.Server.RecipePuppy where

import Pragmatic.Server.Application
import Data.ByteString
import Snap hiding (get)
import Data.Monoid
import Data.Text (Text)
import qualified Data.Text as T
import qualified Data.Text.Encoding as T
import qualified Network.HTTP as H

apiUrl :: Text
apiUrl = "http://www.recipepuppy.com/api/"

puppyRoutes :: [(ByteString, AppHandler ())]
puppyRoutes = [("/puppy/search/:ingredient", searchByIngredient)]

get :: String -> IO String
get url = do
    response <- H.simpleHTTP $ H.getRequest url
    H.getResponseBody response

searchByIngredient :: AppHandler ()
searchByIngredient = do
    i <- getParam "ingredient"
    let ingredient = maybe "" T.decodeUtf8 i
    output <- liftIO $ get (T.unpack $ apiUrl <> "?i=" <> ingredient)
    writeText (T.pack output)
```

Just some comments: 

* I used `HTTP`, but `http-conduit` would have been a good choice too
* I've used the <hoogle>maybe</hoogle> function to yield an empty string in
  case `i` was `Nothing`, or applying the function `decodeUtf8` in case it 
  was not.

Done! Now if we append the `puppyRoutes` list to the "main" one inside `Site.hs`,
we can navigate to [http://localhost:8000/puppy/search/garlic](http://localhost:8000/puppy/search/garlic)
and see some JSON!

## External References

Refer to the official documentations, as always.

## The code

Grab the code [here](https://github.com/cakesolutions/the-pragmatic-haskeller/tree/master/04-recipe-puppy).
The example is self contained, just cabal-install it!

## Next Time

It's time to build our small DSL for describing recipes!
Stay tuned!

A.

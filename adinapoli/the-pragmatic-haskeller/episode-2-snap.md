# Episode 2 - Snap

## Foreword

This is part of [The Pragmatic Haskeller](https://github.com/cakesolutions/the-pragmatic-haskeller) series.

**At the moment, the examples of this post are not runnable.**

## Our barebone web application

Now that we have our barebone Haskell representation of a recipe, it's time
to spread the news to the world. Which better place than Internet? Haskell has
a good numbers of alternatives when it comes to web development. Here are
several libraries which are part of our toolbelt:

* [Yesod](http://www.yesodweb.com/) (which runs School of Haskell)
* [Snap](http://snapframework.com/)  (the one used for this tutorial)
* [Happstack](http://www.happstack.com/page/view-page-slug/1/happstack)
* [Scotty](https://github.com/xich/scotty)
* [MFlow](http://hackage.haskell.org/package/MFlow)

I've added to the list also two "outsiders", projects which are unknown to most
users but that still might be interesting for someone. Said that, the first three
of the list are the most prominent ones. It's not my intention start a comparison,
so let's move further. For The purposes of this episode, I've used Snap, mainly
 because I love its code design, most of its design choices because I'm a 
`forever $ print "tiny"` [contributor](https://github.com/snapframework/snap/blob/master/CONTRIBUTORS).

Snap can be daunting at first, because the documentation on the website is
very slick and it lacks a sort of "tutorial for dummies" which guides you
through every aspect of your app development.

## Different levels of granularity
 
Like most web frameworks out there, Snap has a scaffolding utility to generate
a fully working web application out of the box. Before unravelling the magic
command, let me show you how succinct can be writing a small application
listening on a port and spitting out "Hello World!":

``` haskell
{-# START_FILE Version1.hs #-}
{-# LANGUAGE OverloadedStrings #-}

import Snap

site :: Snap ()
site = writeText "Hello Pragmatic Haskeller!"

main :: IO ()
main = quickHttpServe site
```

If you ignore the pragma and the type signature, it's three lines of code!
You can run it with `runhaskell Version1.hs` and it will do what you expect.

## Printing our recipe from an external file
Right now, our app is not so exciting after all; what if we make it display our
parsed Haskell data structure? That would be a start! This example shows a sort
of Snap's best practice: for each resource we ask, we implement an handler,
responsible to implement the logic. This example might not show immediately
what I'm talking about, but I hope everything will be clarified with the last
version of our server:

``` haskell
{-# START_FILE Version2.hs #-}
{-# LANGUAGE OverloadedStrings #-}

module Pragmatic.Server where

import Prelude hiding (readFile)
import Snap
import Data.Aeson
import Pragmatic.Types
import Pragmatic.JSON.Parser
import Data.Text as T
import Data.ByteString.Lazy

showRecipe :: ByteString -> Snap ()
showRecipe toParse = writeText parseRecipe
  where parseRecipe = case (eitherDecode' toParse :: Either String Recipe) of
                        Left e -> T.pack e
                        Right r -> T.pack . show $ r

app :: ByteString -> Snap ()
app toParse = route [("/recipe", showRecipe toParse)]

main :: IO ()
main = do
    toParse <- readFile "recipe.json"
    quickHttpServe (app toParse)
```

Bonus tip: In this example, we could have make a slightly better design choice;
instead of creating an impure function to live inside the `Snap` monad, we
could have written `showRecipe` in a pure way, using `writeText` directly inside
our "route manager", but for the purposes of this example it doesn't really matter,
but is in general a good suggestion to follow. Whenever you can, always write
pure functions; it will translate in code easier to reason about, reuse and test
but is in general a good suggestion to follow. Whenever you can, always write
pure functions; it will translate in code easier to reason about, reuse and test.

## Build a more structured Snap application
As I told you, Snap already provides a scaffolding utility to scrap some
boilerplate. When you first run `snap init` into an empty directory, it will
create a bunch of files:

* `Main.hs`
* `Application.hs`
* `Site.hs`

While the first one is just to bootstrap the application and you rarely will
need to modify it, I find effective to think about the other two like the place
where your application stack will live (more on that later) and where your
resource handlers will live. So let's take a peek to these two guys.

## Snaplets
Let's take a look to `Application.hs`:

``` haskell
{-# START_FILE Application.hs #-}
{-# LANGUAGE TemplateHaskell #-}

module Pragmatic.Server.Application where

import Control.Lens
import Snap.Snaplet
import Snap.Snaplet.MongoDB.Core

-------------------------------------------------------------------------------
data Pragmatic = Pragmatic 
  { _db :: Snaplet MongoDB }

makeLenses ''Pragmatic


-------------------------------------------------------------------------------
instance HasMongoDB Pragmatic where
    getMongoDB app = view snapletValue (view db app)


-------------------------------------------------------------------------------
type AppHandler = Handler Pragmatic Pragmatic
```

You'll notice a couple of things:

* We have this mysterious monad called `Snaplet`
* We do something with MongoDB
* We have that type synonym called `AppHandler`

Let's go briefly over these three points.

### Snaplets
One of Snap's strengths is represented by Snaplets, auto-contained piece of
functionality we can reuse to build our website. Once you grok it, the entire
mechanism is quite powerful. The classic example is a logger. Suppose you want
to give your website logging capabilities. You could implement logging 
throughout the webapp, or "bundle" it in a series of modules other developers
could use, "jacking-in" your bundle. This is the idea behind Snaplets. The are
a lot of Snaplets coded by the community, from DB to Javascript integration.
I've chosen [snaplet-mongodb-minimalistic](http://hackage.haskell.org/package/snaplet-mongodb-minimalistic): all I have to do is to "plumb" it
to my application `App`, tell Snap how to initialise it (see later) and we have MongoDB in
our app! Last but not least, that curious type synonym is for accessing our
application stack: the outer monad remind us in which environment we are operating
in, so that the inner monad can change according to which Snaplet we are "targeting".
You can think the `AppHandler` as the "root" of our application. If you look
carefully you will note AppHandler has kind `* -> *`, so it's still waiting for
something.. speculations? It's the return type, aka what our handler will yield!


## Putting everything together
What is left? Well, here is `Site.hs` (long file incoming!):

``` haskell
{-# START_FILE Site.hs #-}
{-# LANGUAGE OverloadedStrings #-}

module Pragmatic.Server.Site (app) where

import Data.Aeson
import Data.AesonBson
import Data.ByteString (ByteString)
import qualified Data.ByteString.Lazy as BL
import Data.Text as T
import Database.MongoDB
import Pragmatic.JSON.Parser
import Pragmatic.Server.Application
import Pragmatic.Types
import Snap
import Snap.Snaplet.MongoDB

-------------------------------------------------------------------------------
handleIndex :: AppHandler ()
handleIndex = writeText "Welcome to the pragmatic bakery!"


-------------------------------------------------------------------------------
-- Show the underlying Haskell data structure of recipe.json
handleShow :: AppHandler ()
handleShow = do
    toParse <- liftIO $ BL.readFile "recipe.json"
    writeText $ eitherParse toParse
  where eitherParse tp = case (eitherDecode' tp :: Either String Object) of
                           Left e -> T.pack e
                           Right r -> T.pack . show $ r


-------------------------------------------------------------------------------
-- Here we try to store the recipe.json with the new Data.AesonBson
handleStore :: AppHandler ()
handleStore = do
    toParse <- liftIO $ BL.readFile "recipe.json"
    result <- storeRecipe toParse
    writeText $ T.pack . show $ result


-------------------------------------------------------------------------------
parseRecipe :: BL.ByteString -> Either String Object
parseRecipe = eitherDecode'

-------------------------------------------------------------------------------
storeRecipe :: BL.ByteString -> AppHandler (Either String Object)
storeRecipe recipe = case parseRecipe recipe of
      Left f -> return $ Left f
      Right r -> do
        res <- eitherWithDB $ insert "recipes" $ toBson r
        case res of
          Left _ -> return $ Left "Failed to store the recipe."
          Right _ -> return $ Right r


-------------------------------------------------------------------------------
routes :: [(ByteString, Handler Pragmatic Pragmatic ())]
routes = [("/", handleIndex)
         , ("/show", handleShow)
         , ("/store", handleStore)
         ]


-------------------------------------------------------------------------------
app :: SnapletInit Pragmatic Pragmatic
app = makeSnaplet "pragmatic" "Pragmatic web service" Nothing $ do
    d <- nestSnaplet "db" db $ mongoDBInit 10 (host "127.0.0.1") "pragmatic-haskeller"
    addRoutes routes
    return $ Pragmatic d
```

It's a lot of code, but don't be scared about it, you have already seen part
of it. If you look carefully, all those "handler" are like the function we
wrote in Version2, but on steroid, because now we yield an `AppHandler a` rather
than an humble `Snap a`. Apart from that, I want to point your attention on
three things:

* The `app` function does the heavy bootstrapping; this is something you'll
always need to do. For each Snaplet you add to your application, you need to
also specify how to initialise it. Here we are initialising a new db called
"the-pragmatic-haskeller" using the `mongod` running on localhost. 

* `storeRecipe` uses the MongoDB snaplet to save our recipe into a collection
called "recipes". The conversion to BSON is free and is offered by [aeson-bson](https://github.com/adinapoli/aeson-bson)
(many thanks to the original authors for their job, this is my shameless fork).

**Nota bene:** At the moment `aeson-bson` is not on Hackage. It's a shameless
fork, with minimal tweaks from the original work of Niklas Hambüchen and
Andras Slemmer. The difference is mainly in a different name (all lowercase,
dashed) and a slightly different interface. I'm pretty sure you can plug out
my fork and use the original version without any substantial difference.
If you want the very same version I use, install it from github.

## Conclusions

Wow, it has been a long episode, but we accomplished a lot:

* We have a web server listening on port 8000. Try this at home:
    1. http://localhost:8000/
    2. http://localhost:8000/show
    3. http://localhost:8000/store

* We have marshalling/unmarshalling of our recipes to and from JSON
* We have json/bson serialisation/deserialisation and we can store our
  recipes effortlessly.

## External References

Refer to the official documentation and Oliver Charles' post:

* [Haddock Documentation for Snap](http://hackage.haskell.org/package/snap)

* [24 days of Hackage: Snap](http://ocharles.org.uk/blog/posts/2012-12-19-24-days-of-hackage-snap.html)

## The code

Grab the code [here](https://github.com/cakesolutions/the-pragmatic-haskeller/tree/master/02-snap).
The example is self contained, just cabal-install it!

## Next Time

We'll mess with config file.
Stay tuned!

A.
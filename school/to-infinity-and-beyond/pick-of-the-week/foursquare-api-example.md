# Using the Foursquare API to get a list of trending venues

In this tutorial we will explore how to consume JSON web APIs using Haskell. In particular, we will use the [Google Geocoding API](https://developers.google.com/maps/documentation/geocoding/) to convert an address into a lat long, and then plug that information into the [Foursquare API](https://developer.foursquare.com/docs/) to get a list of trending venues. Topics covered include:

- Making HTTP[S] requests.
- Parsing JSON using Data.Aeson.
- Using type classes to make it easy to add support for new endpoints.
- Propagating errors using the IO monad.

This tutorial is aimed at intermediate Haskell programmers.

<div style="border: 1px solid #ccc; background-color: #e7ffc7; margin-bottom: 10px; padding: 5px;">
<strong><em>For those of you unfamiliar with Foursquare:</em></strong> <a href="https://foursquare.com">Foursquare</a> is a local discovery app. They expose a comprehensive API including a rich point-of-interest (POI) database. Powered by millions of checkins, Foursquare can use algorithms to detect "trending" (e.g. unusually popular) venues. This is the endpoint we will be using in this tutorial. (Disclaimer: I work for Foursquare. That'll be the last bit of marketing in here :).
</div>

<div style="border: 1px solid #ccc; background-color: #ffff99; margin-bottom: 10px; padding: 5px;">
<strong><em>Source code</em></strong>: The entire (runnable) source code for this tutorial is available <a href="https://github.com/wcauchois/haskell-foursquare-api-example">on GitHub</a>. Feel free to refer to the source as you follow along.
</div>

Basics: Making an HTTP Request
---

At its core, consuming a web API is a matter of making HTTP requests to some server with the desired parameters (see: [REST](http://en.wikipedia.org/wiki/Representational_state_transfer)). We'll be using the [Network.HTTP.Conduit](http://hackage.haskell.org/packages/archive/http-conduit/1.2.1/doc/html/Network-HTTP-Conduit.html) library for making these requests.

The interface is simple:

``` haskell
simpleHttp :: MonadIO m => String -> m ByteString
```

`MonadIO` is just a type class for monads that can embed IO operations -- including `IO` itself. The first parameter to this function is a URI, and it returns a [ByteString](http://hackage.haskell.org/packages/archive/bytestring/0.9.0.4/doc/html/Data-ByteString-Lazy.html) -- which is a just a buffer.

Here's an example:

``` active haskell
import Network.HTTP.Conduit
main = do -- The GitHub API is convenient as it doesn't require authentication
          response <- simpleHttp "https://api.github.com/repos/yesodweb/yesod" 
          print response
```

You can expect to see output along the lines of:

    Chunk "{\"id\":237904,\"name\":\"yesod\",\"full_name\":\"yesodweb/yesod\",
    \"owner\":{\"login\":\"yesodweb\",\"id\":930379,\"avatar_url\":\"https://sec
    ure.gravatar.com/avatar/c224026a2005e5ce9a0e1a6defb9f893?d=https://a248.e
    .akamai.net/assets.github.com%2Fimages%2Fgravatars%2Fgravatar-org-420.png\",
    \"gravatar_id\":\"c224026a2005e5ce9a0e1a6defb9f893\",\...

It's not very pretty, but this JSON response contains a lot of juicy information about the Yesod repository, including its URL, description, and owner. [Take a look for yourself](https://api.github.com/repos/yesodweb/yesod).

Extracting Structured Data: Parsing with Aeson
---

[Data.Aeson](http://hackage.haskell.org/packages/archive/aeson/0.6.1.0/doc/html/Data-Aeson.html) provides tons of handy utilities for dealing with JSON. For our purposes, we mostly care about `decode` -- which takes a ByteString and returns a kind of JSON AST.

``` active haskell
import Data.Aeson
import Data.ByteString.Lazy.Char8(pack)
main = print $ (decode (pack "{\"key\": \"val\"}") :: Maybe Value)
```

<div style="border: 1px solid #ccc; background-color: #e7ffc7; margin-bottom: 10px; padding: 5px;">
<strong><em>Aside:</em></strong> Why do we have to annotate the return type of <tt>decode</tt>? If you look at the type signature for decode, <tt>FromJSON a => ByteString -> Maybe a</tt>, you can see that it is generic. The <tt>FromJSON</tt> type class (detailed later) represents any value that can be converted from JSON into a Haskell type. <tt>instance Value FromJSON</tt> is provided by Aeson, but you can also implement FromJSON for your own types (as we will soon do).
</div>

So, what's a `Value`?

``` haskell
data Value = Object !Object -- The exclamation mark (!) is a strictness annotation.
           | Array !Array
           | String !Text
           | Number !Number
           | Bool !Bool
           | Null
```

As you can see, it corresponds very nicely to the structure of a JSON document. Using `simpleHTTP` and `decode` together, you should be able to send a request to an endpoint and parse it into a JSON AST. However, in the next section we'll take a step back and start building some utilities to help us build out different endpoints in a generic manner.

Building Out a Framework For Endpoints
---

What is an endpoint (besides a [miserable pile of secrets](https://gist.github.com/wcauchois/63b8c0b12b598102af69))? Since we're dealing with GET requests only, for our purposes it's basically a URI with some query parameters. We would like to encapsulate those parameters in a typed data structure, and provide a way to build a URI using those parameters. This notion is captured by the following type class:

``` haskell
class Endpoint a where
  buildURI :: a -> String
```

For an example, let's build out the geocoding endpoint. As a quick reminder, geocoding is the process of converting an address like "New York City" into a latitude and longitude. We'll need a lat/long for our subsequent call to the Foursquare API.

From the [Google Geocoding API Documentation](https://developers.google.com/maps/documentation/geocoding/), we can see that the form of the geocoding request is fairly simple: `http://maps.googleapis.com/maps/api/geocode/output?parameters`, where the two required parameters (that we care about) are:

- `address`: The address that you want to geocode.
- `sensor`: Boolean indicating whether the request came from a device with a location sensor. For our program this will always be false.

We encapsulate these parameters with the following ADT:

``` haskell
data GeocoderEndpoint =
  GeocodeEndpoint { address :: String, sensor :: Bool }
```

To convert this data structure into a URI, we need to prepend the API endpoint path to a string consisting of the query parameterized fields.

``` haskell
instance Endpoint GeocoderEndpoint where
  buildURI GeocodeEndpoint { address = address, sensor = sensor } =
    let params = [("address", Just address), ("sensor", Just $ map toLower $ show sensor)]
    in "http://maps.googleapis.com/maps/api/geocode/json" ++ renderQuery True params
```

As you can see, its fairly simple. I've glossed over the implementation of `renderQuery :: Bool -> [(String, Maybe String)] -> String` -- it converts a set of parameters into a string like "?key1=value1&key2=value2" (making sure to URL-encode the values). To see the full definition, [check out the source on GitHub](https://github.com/wcauchois/haskell-foursquare-api-example/blob/master/src/Core.hs).

Detour: Aeson Parsers
---

Before we get to decoding the response from the geocoder endpoint, let's take a detour and explore a powerful feature of Aeson: Parsers.

Parsers are closely tied to the `FromJSON` type class we saw earlier. From the [Aeson documentation](http://hackage.haskell.org/packages/archive/aeson/0.6.1.0/doc/html/Data-Aeson.html#t:FromJSON), a `FromJSON` is "A type that can be converted from JSON, with the possibility of failure." Its only method is `parseJSON :: Value -> Parser a`.

A `Parser` is a monad that encapsulates that possibility of failure, as well as a sequence of operations which convert a `Value` (JSON AST) into some type _a_. A naive implementation of `parseJSON` could simply inspect the Value and `return` an _a_ based on that information, but Aeson also provides a few handy combinators operating within the `Parser` Monad. We'll be using `.:`, which allows you to easily access Object fields. For example:

```haskell active
{-# LANGUAGE OverloadedStrings #-}
import Data.Aeson
import Data.Aeson.Types
import Data.Text(Text)
import Data.ByteString.Lazy.Char8

data Venue = Venue { venueId :: Text, name :: Text } deriving Show

instance FromJSON Venue where
  parseJSON val = do let Object obj = val -- Use pattern matching to extract an Object
                     venueId <- obj .: "id"
                     name <- obj .: "name"
                     return $ Venue venueId name

main = print venue
  where json = "{\"id\": \"some venue id\", \"name\": \"bob's venue\"}"
        venue = decode json >>= parseMaybe parseJSON :: Maybe Venue
```

<div style="border: 1px solid #ccc; background-color: #e7ffc7; margin-bottom: 10px; padding: 5px;">
<strong><em>Aside:</em></strong> Different string types in Haskell. If you look at <a href="http://hackage.haskell.org/packages/archive/aeson/0.6.1.0/doc/html/Data-Aeson.html#v:.:">signature of <tt>.:</tt></a> (essentially <tt>Object -> Text -> Parser a</tt>), you can see that for its second argument it takes a "Text". A Text (from <a href="http://hackage.haskell.org/packages/archive/text/0.11.1.5/doc/html/Data-Text.html">Data.Text</a>) is an efficient representation of a unicode string. However, a string literal will always yield an object of type <tt>String</tt>. The <a href="http://www.haskell.org/ghc/docs/6.12.2/html/users_guide/type-class-extensions.html#overloaded-strings">OverloadedStrings</a> GHC extension allows us to have string literals take alternative types. Concretely, it lets us pass an ordinary string literal as the second argument to <tt>.:</tt> rather than using Data.Text.pack.
</div>

<div style="border: 1px solid #ccc; background-color: #e7ffc7; margin-bottom: 10px; padding: 5px;">
<strong><em>Recall:</em></strong> Pattern match failures inside a Monad will invoke that Monad's <tt>fail</tt> method. So when we use pattern matching to extract an Object from the input Value, its not as dangerous as it might seem.
</div>

Now that we have parser basics down, we should be able to model the geocoder result as an ADT, and parse it from a decoded `Value`.

Modeling the Geocoder Result
---

Now that we can construct a URI for calling the geocoder, we need to make sense of the response. I'd suggest [hitting the endpoint in a web browser](http://maps.googleapis.com/maps/api/geocode/json?address=1600+Amphitheatre+Parkway,+Mountain+View,+CA&sensor=false) to get a rough idea of what the geocoder returns.

The response is fairly generic, with support for multiple results and lots of extra metadata. The only thing _we_ care about is the lat/long of the first result. In JavaScript notation: `response.results[0].geometry.location`.

Since this structure is fairly nested, I built a helper method called `navigateJson :: Value -> [Text] -> Parser Value`. This takes a `Value` and a list of field names and walks down the tree -- for example, if you provided {a: {b: 2}} and \['a', 'b'\] as parameters, it would return 2. I'll elide the definition here; check out the full source [on GitHub](https://github.com/wcauchois/haskell-foursquare-api-example/blob/master/src/Core.hs).

With `navigateJSON`, building the `Parser` should be trivial. Access the 'results' field of the response; get the first entry; navigate thru \['geometry', 'location'\]; return the 'lat' and 'lng' parts of that object.

Before we continue, let's define an ADT to encapsulate this type of result.

```haskell
type LatLng = (Double, Double)
data GeocodeResponse = GeocodeResponse LatLng
  deriving Show -- Handy for debugging
```

Now we can write our parser by declaring an instance of the `FromJSON` type class.

```haskell
instance FromJSON GeocodeResponse where
  parseJSON (Object obj) =
    do (Array results) <- obj .: "results"
       (Object location) <- navigateJson (results V.! 0) ["geometry", "location"]
       (Number lat) <- location .: "lat"
       (Number lng) <- location .: "lng"
       -- Use realToFrac to convert from Aeson Numbers into simple Doubles.
       return $ GeocodeResponse (realToFrac lat, realToFrac lng)
```

Putting It All Together (Part 1)
---

So far we have a method to build URIs for endpoints and a way to parse the JSON response from those endpoints into a structure we can use. Since these steps are encapsulated by the generic type classes `FromJSON` and `Endpoint`, its easy to build a general function to call any endpoint. Let's start with a type signature: `callJsonEndpoint :: (FromJSON j, Endpoint e) => e -> IO j`. "Take an endpoint (with data about the call), make some HTTP request, and parse the response into some JSON object." This process must take place in the IO monad, since we're making a network request.

We'll use `simpleHTTP` like earlier. To run our parser, we'll use a variant of `decode` called `eitherDecode` -- using the error message from `Either`, we can provide more helpful errors (via IO's `fail`).

```haskell
callJsonEndpoint :: (FromJSON j, Endpoint e) => e -> IO j
callJsonEndpoint e =
  do responseBody <- simpleHttp (buildURI e)
     case eitherDecode responseBody of
       Left err -> fail err
       Right res -> return res
```

With `callJsonEndpoint`, we could call the geocoder endpoint like so:

```
(GeocodeResponse latLng) <- callJsonEndpoint $ GeocodeEndpoint "568 Broadway, New York, NY" False
```

Hooking Up Foursquare
---

Now that we have a nifty little framework, we can come back to the motivation for this tutorial: retreiving a list of trending venues around an address. Foursquare makes this easy with an endpoint called ["/venues/trending"](https://developer.foursquare.com/docs/venues/trending). The parameter we care about is the lat/long ("ll"), but you can optionally provide a limit on the results and a radius for your search area.

```haskell
data FoursquareEndpoint =
    VenuesTrendingEndpoint { ll :: LatLng, limit :: Maybe Int, radius :: Maybe Double }

instance Endpoint FoursquareEndpoint where
  buildURI VenuesTrendingEndpoint {ll = ll, limit = limit, radius = radius} =
    let params = [("ll", Just $ renderLatLng ll), ("limit", fmap show limit), ("radius", fmap show radius)]
    in "https://api.foursquare.com/v2/venues/trending" ++ renderQuery True params
```

The docs tell us that this returns a list of "venues", which is another JSON object. For our simple example, we choose to care about two fields: the venue ID, and its name.

```haskell
data Venue = Venue { venueId :: String, name :: String } deriving Show
```

The whole response from this endpoint (list of venues) is encompassed by the following structure:

```haskell
data VenuesTrendingResponse = VenuesTrendingResponse { venues :: [Venue] } deriving Show
```

Implementing `FromJSON` instances for these structures is left as an exercise for the reader (with the [source on GitHub](https://github.com/wcauchois/haskell-foursquare-api-example/blob/master/src/FoursquareModel.hs) available for the lazy ;).

Foursquare Authorization
---

Even though we've implemented data structures, parsing logic, and a URI builder for the venues/trending endpoint, there remains one restriction upon using the Foursquare API: you must have access credentials.

The venues/trending endpoint is ["userless"](https://developer.foursquare.com/overview/auth#userless), so we don't need to go through OAuth, but we still need an API key/secret. I've covered the process of obtaining these credentials in an appendix below, so let's assume we have those for the remainder of this tutorial.

To start, let's define a structure to represent the needed credentials:

```haskell
data FoursquareCredentials = FoursquareCredentials { clientId :: String, clientSecret :: String }
```

We can actually build upon the `Endpoint` framework to build a type of "authorized Foursquare endpoint" that wraps another endpoint and appends the necessary access credentials. Let's call this an `AuthorizedFoursquareEndpoint`:

```haskell
data AuthorizedFoursquareEndpoint = AuthorizedFoursquareEndpoint FoursquareCredentials FoursquareEndpoint
```

In order to build an instance of `Endpoint` for `AuthorizedFoursquareEndpoint`, all we have to do is build the URI of the inner endpoint, and then append "client\_id" and "client\_secret" parameters.

```haskell
instance Endpoint AuthorizedFoursquareEndpoint where
  buildURI (AuthorizedFoursquareEndpoint creds e) = appendParams originalUri authorizationParams
    where originalUri = buildURI e
          authorizationParams = [("client_id", Just $ clientId creds),
                                 ("client_secret", Just $ clientSecret creds),
                                 ("v", Just foursquareApiVersion)]
```
<div style="border: 1px solid #ccc; background-color: #e7ffc7; margin-bottom: 10px; padding: 5px;">
<strong><em>Aside:</em></strong> What's that "v" parameter? From the <a href="https://developer.foursquare.com/overview/versioning">Foursquare documentation</a>: "All requests now accept a v=YYYYMMDD param, which indicates that the client is up to date as of the specified date." In <a href="https://github.com/wcauchois/haskell-foursquare-api-example/blob/master/src/FoursquareEndpoint.hs">my code</a>, I defined the constant <tt>foursquareApiVersion = "20130721"</tt>.
</div>

Finally, let's define a handy helper that lets us authorize an endpoint using an infix syntax (e.g. <tt>endpoint \`authorizeWith\` creds</tt>):

```
authorizeWith = flip AuthorizedFoursquareEndpoint
```

Putting It All Together (Part 2)
---

To recap: we've built a system for describing web API endpoints; we learned how to use Aeson's `FromJSON` to parse the results of those endpoints; and we implemented a function to call those endpoints using that functionality.

At the beginning of this tutorial, we wanted to retrieve a list of trending venues about an address. At this point, we can achieve that goal.

1. Call Google's geocoder endpoint. Store the lat/long.
2. Plug that lat/long into Foursquare's trending venues endpoint.
3. Display those trending venues to the user.

For our Main.hs, we'll also use `getLine` to retrieve access credentials. Here goes:

```haskell
targetAddress = "568 Broadway, New York, NY"
main :: IO ()
main =
  do putStrLn "API key?"
     apiKey <- getLine
     putStrLn "API secret?"
     apiSecret <- getLine
     let creds = FoursquareCredentials apiKey apiSecret

     (GeocodeResponse latLng) <- callJsonEndpoint $ GeocodeEndpoint targetAddress False
     let venuesTrendingEndpoint = VenuesTrendingEndpoint latLng Nothing Nothing `authorizeWith` creds
     (VenuesTrendingResponse venues) <- callJsonEndpoint venuesTrendingEndpoint
     let printVenue v = putStrLn $ "- " ++ name v
     mapM_ printVenue venues
```

And that's it! Once again, see [the full source on GitHub](https://github.com/wcauchois/haskell-foursquare-api-example) for a complete implementation of the techniques described in this tutorial.

Appendix: Obtaining Foursquare API Credentials
---

1. [Sign up for Foursquare](https://foursquare.com/signup) if you don't already have an account.
2. Log in.
3. Go to [My Apps](https://foursquare.com/developers/apps) (this is linked to from the developer website).
4. Click "Create A New App".
5. Enter some details. The only 3 required fields here are the app name, the download URL, and the redirect URI. Since we're not using OAuth, the redirect URI doesn't matter. I would recommend adding any path on a domain you own for these two URIs.
6. Once you click "Save Changes", you'll be directed to the details of your app. This includes your client key and secret, which you can use to run this example!




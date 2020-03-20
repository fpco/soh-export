# Episode 1 - JSON

## Foreword

This is part of [The Pragmatic Haskeller](https://github.com/cakesolutions/the-pragmatic-haskeller) series.

## Every recipe needs (bread and) butter

In the beginning was XML, and then JSON. Call it "meta-exchange format", 
"communication language" or whatever, but you need a sort of *lingua franca* 
for making our app communicate with the rest of the world, through what we
call an *API*. JSON got a lot of momentum lately also because of its simplicity.
It's also used in a lot of NO-SQL DB (e.g Couch DB is queryable through JSON,
MongoDB uses BSON, similar in syntax to JSON).

During my everyday job I use JSON massively, mainly for the reasons I've exposed,
but also because it's easy to manipulate JSON in Scala, thanks to the isomorphism
between *case classes* and JSON objects, and thanks to some handy libraries like
[scalad](https://github.com/eigengo/scalad) and [spray-json](https://github.com/spray/spray-json)
you can write something like this:

``` scala
case class User(name: String, height: Int)
```

And have our class to be marshalled and unmarshalled from and to JSON:
 
``` json
{"name": "Alfredo,
 "height": 187
}
```

So it come as no surprise that the first thing I wanted to explore was how to
do the same in Haskell. Fortunately, this was an easy task, mainly for two
reasons:

* There is this awesome library called [aeson](http://hackage.haskell.org/package/aeson-0.6.1.0) from the authoritative
  [bos](http://www.serpentine.com/blog/)
* I already used it so I was sure about the final outcome

## Jumping the school, fooling around

Without further ado, and keeping an eye on the pragmaticity, we want to create
some Haskell records and marshall/unmarshall the result to and from JSON. In
a similar fashion of the Scala example, we want this:

``` haskell
data User = User { name :: String, height :: Int }
```

To yield the correct JSON. Well, `aeson` allows us to do exactly this, and is
a joy to work with. If you missed it, our final aim is to create a mini programming
language, specifically a DSL to describe recipes, and then save this recipe as
JSON. This is how our website will look like, after this journey:

[The Pragmatic Bakery](http://ec2-107-22-56-237.compute-1.amazonaws.com:8000/)

Ok, but where to start? Any seasoned Haskeller will tell you, in same fashion
master Yoda would tell Luke *use the force*, the Haskeller would say *use the types*.
What we want to do is to model our domain at the type level. Ok, but what we 
want to model, exactly? Let's take for example this JSON, which models a [typical
italian cake](http://cucina.alfemminile.com/w/ricetta/r1124/il-ciambellone-piu-soffice-del-mondo.html):

``` json
{
    "name": "Ciambellone Cake",
    "ingredients": [
        {
            "name": "Flour",
            "quantity": 250,
            "measure": "gr"
        },
        {
            "name": "Sugar",
            "quantity": 250,
            "measure": "gr"
        },
        {
            "name": "Sunflower Oil",
            "quantity": 130,
            "measure": "ml"
        },
        {
            "name": "Water",
            "quantity": 130,
            "measure": "ml"
        },
        {
            "name": "Egg",
            "quantity": 3
        },
        {
            "name": "Yeast",
            "quantity": 1
        }
    ],
    "steps": [
        {
            "step": "Mix everything",
            "order": 1
        },
        {
            "step": "Cook in oven at 200 degrees",
            "order": 2,
            "duration": {
                "duration": 30,
                "measure": "minutes"
            }
        }
    ]
}
```

Beside recommending it (the quantities are true ones, even though I suggest
you use olive oil instead of [vegetable oil](http://wellnessmama.com/2193/why-you-should-never-eat-vegetable-oil-or-margarine/)), what we want is an Haskell
representation of this recipe. So a recipe is a list of ingredient and a list
of steps, correct? Uhm, this seems to be a job for Haskell records!

``` active haskell
-- https://github.com/cakesolutions/the-pragmatic-haskeller/blob/master/01-json/src/Pragmatic/Types.hs

data Recipe = Recipe
  { recipeName :: String
  , ingredients :: [Ingredient]
  , steps :: [Step]
  } deriving Show

type Measure = String

data Ingredient = Ingredient 
  { ingredientName :: String
  , quantity :: Int
  , measure :: Maybe Measure
  } deriving Show

data Step = Step
  { stepName :: String
  , order :: Int
  , stepDuration :: Maybe Duration
  } deriving (Eq, Show)

instance Ord Step where
    compare s1 s2 = compare (order s1) (order s2)

data Duration = Duration
  { duration :: Int
  , durationMeasure :: Measure
  } deriving (Eq, Show)

main = print $ Step "Mix everything" 1 Nothing
```

Nothing (no pun intended) thrilling, after all this is not an episode called "Records for the record",
but JSON is the rockstar here, so let's release `aeson` power!

## The father of JSON

Who played with most JSON libraries out there (in every programming language 
you can think of) knows that writing code to "link" JSON to the language 
representation is pretty much boilerplate coding. Even though `aeson` allows
writing this code [almost automatically](http://hackage.haskell.org/packages/archive/aeson/0.6.1.0/doc/html/Data-Aeson.html#g:5) with the "power of the typeable", it's
something I've have not experimented with (patches are welcome!), so we'll write
boilerplate. What we need to do is to instruct JSON how to convert back and forth
from and to the corresponding Haskell record, and we're done!
For teaching purposes, we'll do it just for one record, but the same applies for
the rest:

``` active haskell
{-# START_FILE Types.hs #-}

module Types where

data Recipe = Recipe
  { recipeName :: String
  , ingredients :: [Ingredient]
  , steps :: [Step]
  } deriving Show

type Measure = String

data Ingredient = Ingredient 
  { ingredientName :: String
  , quantity :: Int
  , measure :: Maybe Measure
  } deriving Show

data Step = Step
  { stepName :: String
  , order :: Int
  , stepDuration :: Maybe Duration
  } deriving (Eq, Show)

instance Ord Step where
    compare s1 s2 = compare (order s1) (order s2)

data Duration = Duration
  { duration :: Int
  , durationMeasure :: Measure
  } deriving (Eq, Show)

{-# START_FILE Parser.hs #-}

{-# LANGUAGE OverloadedStrings #-}
module Parser where

import Data.Aeson
import Types
import Control.Applicative
import Control.Monad


instance FromJSON Recipe where
    parseJSON (Object r) = Recipe <$>
                           r .: "name" <*>
                           r .: "ingredients" <*>
                           r .: "steps"
    parseJSON _ = mzero

instance ToJSON Recipe where
    toJSON (Recipe n i s) = object ["name" .= n, "ingredients" .= i, "steps" .= s]

-------------------------------------------------------------------------------

instance FromJSON Step where
    parseJSON (Object s) = Step <$>
                           s .: "step" <*>
                           s .: "order" <*>
                           s .:? "duration"
    parseJSON _ = mzero

instance ToJSON Step where
    toJSON (Step s o d) = object ["step" .= s, "order" .= o, "duration" .= d]

-------------------------------------------------------------------------------

instance FromJSON Ingredient where
    parseJSON (Object i) = Ingredient <$>
                           i .: "name" <*>
                           i .: "quantity" <*>
                           {-hi-}i .:? "measure" {-/hi-}
    parseJSON _ = mzero

instance ToJSON Ingredient where
    toJSON (Ingredient n q m) = object ["name" .= n, "quantity" .= q, "measure" .= m]

-------------------------------------------------------------------------------
instance FromJSON Duration where
    parseJSON (Object d) = Duration <$>
                           d .: "duration" <*>
                           d .: "measure"
    parseJSON _ = mzero

instance ToJSON Duration where
    toJSON (Duration d m) = object ["duration" .= d, "measure" .= m]

-------------------------------------------------------------------------------

{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}

module Main where

import qualified Data.ByteString.Lazy as BL
import qualified Data.ByteString.Lazy.Char8 as C8
import Types
import Parser()
import Data.Aeson
import Data.Monoid

main :: IO ()
main = do
    let toParse = C8.unlines $ map C8.pack [
                                    "{ ",
                                    "    \"name\": \"Ciambellone Cake\", ",
                                    "    \"ingredients\": [ ",
                                    "        { ",
                                    "            \"name\": \"Flour\", ",
                                    "            \"quantity\": 250, ",
                                    "            \"measure\": \"gr\" ",
                                    "        }, ",
                                    "        { ",
                                    "            \"name\": \"Sugar\", ",
                                    "            \"quantity\": 250, ",
                                    "            \"measure\": \"gr\" ",
                                    "        }, ",
                                    "        { ",
                                    "            \"name\": \"Sunflower Oil\", ",
                                    "            \"quantity\": 130, ",
                                    "            \"measure\": \"ml\" ",
                                    "        }, ",
                                    "        { ",
                                    "            \"name\": \"Water\", ",
                                    "            \"quantity\": 130, ",
                                    "            \"measure\": \"ml\" ",
                                    "        }, ",
                                    "        { ",
                                    "            \"name\": \"Egg\", ",
                                    "            \"quantity\": 3 ",
                                    "        }, ",
                                    "        { ",
                                    "            \"name\": \"Yeast\", ",
                                    "            \"quantity\": 1 ",
                                    "        } ",
                                    "    ], ",
                                    "    \"steps\": [ ",
                                    "        { ",
                                    "            \"step\": \"Mix everything\", ",
                                    "            \"order\": 1 ",
                                    "        }, ",
                                    "        { ",
                                    "            \"step\": \"Cook in oven at 200 degrees\", ",
                                    "            \"order\": 2, ",
                                    "            \"duration\": { ",
                                    "                \"duration\": 30, ",
                                    "                \"measure\": \"minutes\" ",
                                    "            } ",
                                    "        } ",
                                    "    ] ",
                                    "} "
                                ]
      in case (eitherDecode' toParse :: Either String Recipe) of
           Right r -> print r
           Left e -> C8.putStrLn $ C8.pack e <> " in " <> toParse
```

Et voilà, our JSON was converted to shining Haskell records! Last example was
slightly adapted from the Github version to work with strings rather than an 
external file. Please notice the highlighted line of code; in `aeson` we can
use this [:?](http://hackage.haskell.org/packages/archive/aeson/0.6.1.0/doc/html/src/Data-Aeson-Types-Class.html#.%3A%3F)
function to tell `aeson` "try to encode this like a `Maybe`".

## Brief discussion about pragmatism

I won't transform this into another "aeson for dummies", we are pragmatic
programmer, remember? We want to quicky skim over a bunch of tutorial or examples,
better if we can access an Haddock API full of those (like the one `aeson` has)
assembling our code along the way. Be wary, this does not mean we must ask like
"monkeys" and blissfully copy and paste third party code, also because in Haskell
acting that way means you'll be soon stuck in some type error.

What I really mean is that the aim of this tutorial is not explaining every
nook and cranny of each library, but showing you how you can get the job done
in Haskell, real world job. Here, I could have told you `aeson` uses an elegant
applicative interface and typeclasses to make convertion a breeze, but I think
the documentation is good enough for you to progress on your own. And hey, you have
an interactive environment to play with!
Said that, whenever I feel there is no sufficient documentation around a library
I will explain some obscure passage along the way, but this won't be the standard.

## External References

I suggest you read this if you are intrigued by `aeson` and by how Haskell can
handle JSON:

* [Haddock Documentation for Aeson](http://hackage.haskell.org/packages/archive/aeson/0.6.1.0/doc/html/Data-Aeson.html)

* [24 days of Hackage: Aeson](http://ocharles.org.uk/blog/posts/2012-12-07-24-days-of-hackage-aeson.html)

## The code

Grab the code [here](https://github.com/cakesolutions/the-pragmatic-haskeller/tree/master/01-json). The example is self contained, just cabal-install it!

## Next Time

We'll mess with Snap and will pave the way to our webapp!
Stay tuned!

A.

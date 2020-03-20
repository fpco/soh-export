# Episode 5 - A simple DSL

## Foreword

This is part of [The Pragmatic Haskeller](https://github.com/cakesolutions/the-pragmatic-haskeller) series.


## A simple DSL for describing recipes

After a break, let's resume our journey to build our next generation, innovative, useless webapp for writing recipes under the form of a DSL and converting them to JSON, shall we? The result we are aiming at is here:

[The Pragmatic Bakery](http://ec2-107-22-56-237.compute-1.amazonaws.com:8000/)

As you can see, we can play around with the syntax, even have some barebone, informative parsing errors if we end up with an invalid syntax (hint: try to replace one of those "gr" with an "a", and you'll get all the possible unit of measure the parser was expecting). Coding that was not hard at all, and once you wrap your head around the basic syntax, you can achieve a lot using relatively few functions and data types.

## Parsec, Attoparsec, Bytestring-lexing. Which one?
Choosing a parsing library is like getting married: sooner or later you'll look around looking for an extraconiugal affair. There are three libraries which are equally attractive and worth trying out, and [the choice is yours](http://www.youtube.com/watch?v=Xa7Ig6elozo). I've ended up using Parsec mainly for two reasons:

* At the time of coding the DSL, Ben Clifford had just given his talk at the London Haskell User Group about [Parsing Stuff in Haskell](http://www.youtube.com/watch?v=r_Enynu_TV0), which I warmly recommend. Ben was also so kind to give me the slides of the talk, which you'll find inside the repository.

* Parsec was, to my understanding, the first parsing library available in Haskell, and seems to be a good starting point to learn about parsing in general. Once you pick the concepts behind combinator parsing, it will be very easy to switch to a different library (Attoparsec or whatever you want). 


## Parser combinators

To quote the HaskellWiki "Parsec lets you construct parsers by combining higher-order Combinators to create larger expressions. Combinator parsers are written and used within the same programming language as the rest of the program. The parsers are first-class citizens of the language [...]". In a nutshell, you assemble your program out of small, selfcontained, reusable block (which lives in the `Parser` monad) to build a big, complex parser. Paraphrasing Brian Beckman's in his popular video [Don't fear the monad](http://www.youtube.com/watch?v=ZhuHCtR3xq8), "Composability is the way to control software complexity", and Haskell is the king of composability, so it comes as no surprise that combinators are just one incarnation of such a concept.

### There is an applicative in my Parsec
Parsec uses the applicative style to build complex parsers out of simpler ones. As probably most of you know, "[...] Applicative functors represent an abstraction lying in between Functor and Monad in expressivity, first described by McBride and Paterson[...]". Parsec can be daunting at first, especially if you don't have a guide or a clue about where to start. The rule of thumb is just "for most of your use cases, use the high level type `Parser`. If you look in the source code, though (which is always a good habit), you'll find that a `Parser` is just a type synonym:

``` haskell
type Parser = Parsec C.ByteString ()
```

If you ask in ghci for the kind of `Parser` you'll find that is `* -> *`, this means we need to feed in another type to yield kind `*`, which is, no surprises, the type of what we want to parse. In fact, we can go one step deeper and look ad how `Parsec` is defined:

``` haskell
type Parsec s u = ParsecT s u Identity
```

Nice! Again, is just a type synonym for the core of the library, the `ParsecT` transformer:

``` haskell
newtype ParsecT s u m a = [...]
```

Where `a` is the result type we were looking at. Looking inside the code (if you don't trust me, just do it!) you'll find that `ParsecT` derive instances of both Applicative and Monad, so techinically this makes `Parser` both an Applicative and a Monad, but here we'll use the applicative style because yield a more elegant code.

### Our first Parser
After this digression, don't be intimidated, writing a parser is not that complicated. As an example, we'll write a VERY simple Parser that takes anything and spit out True, no matter what. This is something that might happend more often than what you may think. Sometimes we just want to "ignore" what we have parsed to just yield some arbitrary value. Ok, let's write some code:

``` active haskell
module Main where

import Text.ParserCombinators.Parsec
import Control.Applicative hiding ((<|>), optional, many)

alwaysTrue :: Parser Bool
alwaysTrue = pure True

main = print $ parse alwaysTrue "I'm just a description string ignore me" "I will be parsed"
```

As I told you, `Parser` is also a Monad, so this is equivalent:

``` active haskell
module Main where

import Text.ParserCombinators.Parsec
import Control.Applicative hiding ((<|>), optional, many)

alwaysTrue :: Parser Bool
alwaysTrue = return True

main = print $ parse alwaysTrue "" "I will be parsed"
```

What we have done here? Well, it's not hard to guess at all: we have simply put our `True` value inside the Parser applicative using `pure`, yielding a parser which can be used to parse any string whatsoever to produce the value `True`. No big deal. `parse` is a neat function we can use to test our parsers. It also takes a description string as second argument, which you can prompty ignore.

### Small useful parsers
Let's continue our journey creating some useful parser we'll use later on to parse an entire recipe:

``` active haskell
module Main where

import Text.ParserCombinators.Parsec
import Control.Applicative hiding ((<|>), optional, many)

ws :: Parser String
ws = many (oneOf " ")


int :: (Integral a, Read a) => Parser a
int = read <$> many1 digit


stringLike :: Parser String
stringLike = char '"' *> many (noneOf ['\"', '\r', '\n']) <* char '"'


-- A parser combinator which skips whitespaces from both sides
lexeme :: Parser a -> Parser a
lexeme p = ws *> p <* ws

main :: IO ()
main = do
    print $ parse int "" "10"
    print $ parse int "" "    10   "
    print $ parse (lexeme int) "" "   10 "
```

This is a perfect example of combining two parser! We first define the `ws` parser which might be read as "takes as many whitespaces as you can" and `int` which says "take at least one digit, and finally "fmap" the read function on the result, to yield a parser (quick reminder, <$> is fmap). Let's ignore for now the `stringLike` (exercise for the reader, understand what id does and play with it) and let's focus on the `lexeme` combinator. It takes a `Parser` and yield another `Parser`, but it performs a trick: it first performs the action of stripping whitespaces, discarding the result of doing that, and then it strips again the whitespaces, retaining the first argument, our number! If this sounds confusing take a look at the definitions of `*>` and `<*`:

``` haskell
(*>) :: f a -> f b -> f b
-- Sequence actions, discarding the value of the first argument.

(<*) :: f a -> f b -> f a
-- Sequence actions, discarding the value of the second argument.
```

Ha! Pretty neat! So using `*>` ensure we discard the parsing result of the first `ws` (just whitespaces) whilst `<*` does the same thing, but discarding the second argument, the trailing whitespaces, living us with our lovely int! I can barely type for the excitement while writing this. 

### Down the parsing hole, parsing recipes
It's time now to wrap up and to write the parsers for our recipe. This is what we are aiming to parse:

```
"Ciambellone" is made with
    250 gr of "Flour"
    250 gr of "Sugar"
    130 ml of "Sunflower Oil"
    130 ml of "Water"
    3 "Eggs"

  preparated by
    "Mixing everything" and
    "Cooking in oven at 200 degrees" for 40 minutes
```

We have our list of ingredients, our steps (everthing after `prepared`) and everything forms our `Recipe` datatype. For the lazy, these are our types:

[https://github.com/cakesolutions/the-pragmatic-haskeller/blob/master/05-dsl/src/Pragmatic/Types.hs](https://github.com/cakesolutions/the-pragmatic-haskeller/blob/master/05-dsl/src/Pragmatic/Types.hs)

Without further ado let's write a parser!

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


{-# START_FILE DSLParser.hs #-}
{-# LANGUAGE OverloadedStrings #-}

module DSLParser where

import Text.ParserCombinators.Parsec
import Types
import Control.Applicative hiding ((<|>), optional, many)


ws :: Parser String
ws = many (oneOf " ")


int :: (Integral a, Read a) => Parser a
int = read <$> many1 digit


stringLike :: Parser String
stringLike = char '"' *> many (noneOf ['\"', '\r', '\n']) <* char '"'


-- A parser combinator which skips whitespaces from both sides
lexeme :: Parser a -> Parser a
lexeme p = ws *> p <* ws


(<||>) :: Parser a -> Parser a -> Parser a
p <||> q = try p <|> q


-- Here we are saying, try to match one between
-- gr and ml, if you can't default to Nothing.
-- The trick is using  pure :: a -> f a
measureP :: Parser (Maybe String)
measureP = (string "gr" *> (pure . Just $ "gr"))
       <|> (string "ml" *> (pure . Just $ "ml"))
       <|> (string "spoon" *> (pure . Just $ "spoon"))
       <|> (string "cup" *> (pure . Just $ "cup"))
       <|> (pure Nothing)


syntacticSugar :: String -> Parser (Maybe String)
syntacticSugar s = (string s *> (pure . Just $ s)) <|> pure Nothing


ingredient :: Parser Ingredient
ingredient = do
    qt <- lexeme int
    ms <- lexeme measureP
    lexeme (syntacticSugar "of")
    name <- lexeme stringLike
    lexeme (syntacticSugar "and")
    string "\r\n"
    return $ Ingredient name qt ms

-- Step
-------------------------------------------------------------------------------
step :: Parser Step
step = do
    sn <- lexeme stringLike
    d <- optionMaybe durationP
    lexeme (syntacticSugar "and")
    string "\r\n" <||> pure ""
    return $ Step sn 1 d

-- Duration
-------------------------------------------------------------------------------
durationP :: Parser Duration
durationP = do
    lexeme (string "for")
    d <- lexeme int
    u <- lexeme durationUnit
    return $ Duration d u
  where durationUnit = string "seconds" <|> string "minutes" <|> string "hours"


-- Recipe
-------------------------------------------------------------------------------
recipe :: Parser Recipe
recipe = do
    rn <- lexeme stringLike
    lexeme (syntacticSugar "is made with") *> string "\r\n"
    i <- many1 ingredient
    many1 (string "\r\n")
    lexeme (string "prepared by") *> string "\r\n"
    s <- many1 step
    return $ Recipe rn i s

{-# START_FILE Main.hs #-}
module Main where

import DSLParser
import Text.ParserCombinators.Parsec

initialDsl = unlines [
              "\"Ciambellone\" is made with\r",
              "    250 gr of \"Flour\"\r",
              "    250 gr of \"Sugar\"\r",
              "    130 ml of \"Sunflower Oil\"\r",
              "    130 ml of \"Water\"\r",
              "    3 \"Eggs\"\r",
              "\r",
              "  prepared by\r",
              "    \"Mixing everything\" and\r",
              "    \"Cooking in oven at 200 degrees\" for 40 minutes"]

main :: IO ()
main = print $ parse recipe "" initialDsl
```

There are a couple of things which are worth expanding on: first of all, just as Ben did in his talk, we defined the function `(<||>)` which basically takes two parsers and try to match the first one on the target; it it fails it tries the second. Second, we are relying on the monadic nature of a Parser in more complex parsers, for example take a look at the `Parser Recipe` one; we are describing in a high-level fashion how our DSL look like, running parsers along the way trying to get the parsed value back. If everything goes smooth as we thought, we then build our `Recipe` object and return it, otherwise we fail, and this will be reflected with a `Left` in our `parse` outcome.

Before saying farewell, let me conclude with a small observation: do you understand now why `Aeson` requires you to yield a `Parser` (an Attoparsec one, but the concept still holds) when writing `FromJSON` instances of your types? Simple: because this way parsing a json is just a matter of running the parsers and getting the final value back. I think it's a smart trick indeed.


## External References

* The [excellent video](http://www.youtube.com/watch?v=r_Enynu_TV0) of the talk given at the London Haskell User Group

For who wants to explore the solution space:

* [Parsec](http://hackage.haskell.org/package/parsec-3.1.3) on Hackage
* [Attoparsec](http://hackage.haskell.org/package/attoparsec-0.10.4.0) on Hackage
* [Bytestring-lexing](http://hackage.haskell.org/package/bytestring-lexing-0.4.3) on Hackage
* [Blogpost](http://variadic.me/posts/2012-02-25-adventures-in-parsec-attoparsec.html) of a log parser using Attoparsec.
* Another interesting [blogpost](http://newartisans.com/2012/08/parsing-with-haskell-and-attoparsec/) where after an initial Parsec solution, the author converts it using Attoparsec.

## The code

Grab the code [here](https://github.com/cakesolutions/the-pragmatic-haskeller/tree/master/05-dsl).
The example is self contained, just cabal-install it!

## Next Time

We built our DSL, but there is a small bug in the data structure it produces (can you spot it?). Next time we'll fix this problems using lens!

Stay tuned!

A.
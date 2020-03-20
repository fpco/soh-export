# Pattern Synonyms for Dates and an IRC Bot

This is intended to be an informal tutorial for pattern synonyms using dates and an IRC bot as motivating examples (we will avoid getting bogged down with details).


# Introduction to Pattern Synonyms

Pattern synonyms appeared in GHC 7.8 and allow users to abstract away from the actual implementation of a data type. Combined with the view patterns extension they allow users to move some of the logic in their guards and case expressions into patterns.

Familiarity with the `ViewPatterns` extension is assumed. Here are some references if you need to get up to speed: [View patterns: lightweight views for Haskell](https://ghc.haskell.org/trac/ghc/wiki/ViewPatterns), [Guide to GHC Extensions: Pattern and Guard Extensions](https://www.fpcomplete.com/school/to-infinity-and-beyond/pick-of-the-week/guide-to-ghc-extensions/pattern-and-guard-extensions#viewpatterns).

## Example with dates
We build on an example from the paper [Abstract Value Constructors: Symbolic Constants for Standard ML](http://people.cs.uchicago.edu/~jhr/papers/1992/tr-sml-const.pdf) (_PDF_). Let's begin by enabling the extension and defining a simple date representation:

``` haskell
{-# LANGUAGE PatternSynonyms #-}
data Date = Date { month :: Int, day :: Int } deriving Show
```

Before 7.8 we just limited how we can match on our type right off the bat! Maybe you want to be able to match on the months, in which case this would have been a better representation

``` haskell
data Date = January Int | February Int | … | December Int
```

But now we can define pattern synonyms! We start by defining synonyms for months and holidays using `pattern NewPattern = OldPattern` and match them in a function like we would a regular pattern

``` haskell
-- Months
pattern January  day = Date { month = 1,  day = day }
pattern February day = Date { month = 2,  day = day }
pattern March    day = Date { month = 3,  day = day }
-- elided
pattern December day = Date { month = 12, day = day }

-- Holidays
pattern Christmas    = Date { month = 12, day = 25  } 

describe :: Date -> String
describe (January 1)  = "First day of year"
describe (February n) = show n ++ "th of February"
describe Christmas    = "Presents!"
describe _            = "meh"
```

The meaning of `describe` should be clear: the first clause matches only January first, the second clause matches any day in February and the third pattern matches Christmas day.

Pattern matching on `Christmas` is the same as matching on `Date { month = 12, day = 25 }` or `Date 12 25` — only a lot clearer. Normally it is only possible to pattern match on the constructors of a data type, but now we can do it independently of how it is represented. Let's see some outputs to verify our intuition:

``` haskell
ghci> describe Date { month = 12, day = 25 }
"Presents!"
ghci> describe Date { month = 2,  day = 5  }
"5th of February"
```

But here's a trick, you can use these patterns as expressions as well:

``` haskell
ghci> describe Christmas
"Presents!"
ghci> describe (February 10)
"10th of February"
ghci> March 5
Date { month = 3, day = 5 }
```

and not only that but we can define `Christmas` in terms of the pattern `December`

``` haskell
pattern Christmas = December 25
```

Quite elegant — we can now use `Christmas`, `December 25` and `Date 12 25` interchangably in our code.

All the patterns we've seen so far have been examples of simply bidirectional patterns because they can be used as both patterns _and_ expressions: this is not true of all patterns though.

## More complicated dates

Let's say that we wanted to match on the days of December leading up to and following Christmas, it's not clear at all how to do this using the constructions we covered earlier since this depends on a predicate and not only pattern matching. To allow this we need to enable the `ViewPatterns` extension and use _uni-directional_ patterns rather than the bidirectional ones we used previously. Uni-directional patterns are defined using `pattern` and an arrow `<-` rather than `=`:

``` haskell
-- /show
{-# LANGUAGE PatternSynonyms, ViewPatterns #-}

data Date = Date { month :: Int, day :: Int } deriving Show

pattern December day = Date { month = 12, day = day }
-- show
pattern BeforeChristmas {-hi-}<-{-/hi-} December (compare 25 -> GT)
pattern Christmas       {-hi-}<-{-/hi-} December (compare 25 -> EQ)
pattern AfterChristmas  {-hi-}<-{-/hi-} December (compare 25 -> LT)

react :: Date -> String
react BeforeChristmas = "Waiting :("
react Christmas       = "Presents!"
react AfterChristmas  = "Have to wait a whole year :("
react _               = "It's not even December..."
```
There are several things to note

- We used a bidirectional pattern (`December`) to define the new patterns
- These uni-directional patterns cannot be used as expressions (it is not obviously what the values of `BeforeChristmas` and `AfterChristmas` ought to be anyway)
- We used view patterns to compare the given day to 25 in `(compare 25 -> ...)`

This is equivalent to:

``` haskell
react' (Date 12 (compare 25 -> GT)) = "Waiting :("
react' (Date 12 (compare 25 -> EQ)) = "Presents!"
react' (Date 12 (compare 25 -> LT)) = "Have to wait a whole year :("
react' _                            = "It's not even December..."
-- ...
```
using view patterns but no pattern synonyms.

We could also have used different predicates such as `December ((< 25) -> True)`, `December 25` and `December ((> 25) -> True)` but this would require running a new predicate for each clause. In the current design, GHC only applies the predicate `compare 25` once (see _Efficiency_ part of the [GHC user guide](http://www.haskell.org/ghc/docs/latest/html/users_guide/syntax-extns.html#view-patterns)) producing something like this:

``` haskell
react date = case date of
  Date 12 day -> case compare 25 day of
    GT -> "Waiting :("
    EQ -> "Presents!"
    LT -> "Have to wait a whole year :("
  _           -> "It's not even December..."
```

which would matter given a more expensive predicate.

## Accessing values
When matching `BeforeChristmas` and `AfterChristmas` we know that the date value is some day in December other than 25 but we don't know _which_ day. We can retrieve the entire record using an as-pattern

``` haskell
days'tilChristmas :: Date -> Int
days'tilChristmas d@BeforeChristmas = 25 - day d
days'tilChristmas   Christmas       = 0
days'tilChristmas d@AfterChristmas  = 365 + 25 - day d
```

but a nicer way might be to write

``` haskell
isItNow :: Int -> (Ordering, Int)
isItNow day = (compare 25 day, day)

pattern BeforeChristmas day <- December (isItNow -> (GT, day))
pattern Christmas           <- December (isItNow -> (EQ, _))
pattern AfterChristmas  day <- December (isItNow -> (LT, day))

days'tilChristmas :: Date -> Int
days'tilChristmas (BeforeChristmas n) = 25 - n
days'tilChristmas Christmas           = 0
days'tilChristmas (AfterChristmas n)  = 365 + 25 - n

```

**Exercise**: Create a pattern where a Unix time can be used to match our `Date` value, something like `Epoch 1419470000` should match Christmas.

**Exercise**: Represent date with a single Unix timestamp and allow matching on it with `March 5` and `ThirdOf` which matches the third day of any month.

# IRC
This is not an introduction to writing a bot, go [here](http://www.haskell.org/haskellwiki/Roll_your_own_IRC_bot) for that.


## Setting the stage
To build the bot we need some basic commands:


``` haskell
import Control.Monad
import Network
import System.IO
-- show
-- Choose a nick
nick :: Handle -> String -> IO ()
nick h name = hPutStrLn h ("NICK " ++ name)

-- Specify username
user :: Handle -> String -> IO ()
user h name = hPutStrLn h ("USER " ++ name ++ " 0 * :" ++ name)

-- Join a channel
joinChan :: Handle -> String -> IO ()
joinChan h chan = hPutStrLn h ("JOIN " ++ chan)
```

Now we can connect to the server and run our action forever

``` haskell
main = do
  h <- connectTo "irc.freenode.org" (PortNumber 6667)
  hSetBuffering   h NoBuffering
  hSetNewlineMode h (NewlineMode CRLF CRLF)

  nick h "PatternBot"
  user h "PatternBot"

  joinChan h "##patternsynonyms"

  forever (action h)

```

For `action = hGetLine >=> putStrLn` the bot should identify itself, join `##patternsynonyms` and output everything it receives.

## “Ping — Pong” Pattern

If the server says `PING` you must say `PONG`!

An example `PING` command may look like

```
PING :orwell.freenode.net
```
meaning that you need to respond with

```
PONG :orwell.freenode.net
```
to let it know we're still there. Here we can use our patterns!


``` haskell
pattern Ping serv <- (words -> ["PING", serv])
```

this pattern only matches two-word `PING` commands and gives us the server we need to include in our `PONG`. Now `action` turns into: 

``` haskell
pong :: Handle -> String -> IO ()
pong h serv = hPutStrLn h ("PONG " ++ serv)

action :: Handle -> IO ()
action h = do
  line <- hGetLine h
  case line of
    {-hi-}PING serv -> pong h serv{-/hi-}
    _         -> return ()
```

and we can treat `line :: String` as if it were a data type of IRC messages.

## Glad you could (μ : M² → M) us

If new people come to our channel we want them to feel welcome so the bot should greet people as they join. The `JOIN` message looks something like this

```
:<nick>!<user>@<host> JOIN <channel>

```
We want to know who joined what channel so let's parse that in an ad-hoc way (there are packages on Hackage that do this properly). The nick goes from the initial colon to the exclamation mark:

``` haskell
-- /show
import Data.List
-- show
getNick :: String -> Maybe String
getNick (':':prefix) = do
  index <- findIndex (== '!') prefix
  return (take index prefix)
getNick _            = Nothing
```

Now we create the pattern for joins (and for our bot) and integrate them into the logic

``` haskell
pattern PBot = "PatternBot"
pattern JOIN nick chan 
   <- (words -> [getNick -> Just nick, "JOIN", chan])

msg :: Handle -> String -> IO ()
msg h chan msg = hPutStrLn h ("PRIVMSG " ++ chan ++ " :" ++ msg)

action :: Handle -> IO ()
action h = do
  line <- hGetLine h
  case line of
    PING serv      -> pong h serv
    -- Greet channel when we join
    {-hi-}JOIN PBot chan -> msg h chan "Halló, heimur!"{-/hi-}
    -- Greet nicks that join
    {-hi-}JOIN nick chan -> msg h chan (nick ++ ": Welcome to " ++ chan){-/hi-}
    _              -> return ()
```

The `JOIN` pattern definition is not very pretty but we're not concerned with that.

## Responding to messages

Messages are either sent to a channel or to a single user (private message) and are either:

```
:<nick>!<user>@<host> PRIVMSG <channel> :<msg>
:<nick>!<user>@<host> PRIVMSG <nick>    :<msg>
```
We would like to pick out the sender, the message and the target channel or nick:

``` haskell
getPriv :: String -> Maybe (String, String, String)
getPriv msg = case words msg of
  sender : "PRIVMSG" : target : (':':_) : _ -> do
    nick <- getNick sender
    return (nick, target, clean msg)
  _ -> Nothing
  where
  clean = tail . dropWhile (/=':') . dropWhile (/= ' ') . tail
```

Now we create two patterns that determine whether something is a nick or a channel:
``` haskell
pattern Nick n <- ((\a -> (head a /= '#', a)) -> (True, n))
pattern Chan c <- ((\a -> (head a == '#', a)) -> (True, c))

```
Yuck. Anyway, we can use these to define the desired patterns

``` haskell
-- Private message to our bot
pattern PM from m <- (getPriv -> Just (from, Nick PBot,  m))

-- Message to channel
pattern MSG from to m <- (getPriv -> Just (from, Chan to, m))

```

This is so nice is almost absolves me of the horrible code above `:)` but the good thing is that we can replace the underlying representation with a data type provided by some IRC parsing library without having to change the actual `action` code! Now let's put `MSG` to use: if anyone mentions “cats” we respond:

``` haskell
-- /show
import Data.Char
-- show
-- Matches any cat
pattern Cat <- (isInfixOf "cat" . map toLower -> True)

-- …
case line of
  MSG _ chan Cat -> msg h chan "Meow!"
```
## Commands
Now we may want to allow users to run commands starting with `> `:


``` haskell
-- /show
{-# LANGUAGE ScopedTypeVariables #-}
import System.Random
-- show
pattern Command cmd = '>':' ':cmd

pattern Roll <- Command (map toLower -> "roll")

-- …
case line of
  MSG from chan Roll -> do
    roll :: Int <- randomRIO (1, 6)
    msg h chan (from ++ ": You rolled " ++ show roll)
```
It's now easy to add additional commands.

## Responding to PMs

Bots are snarky

``` haskell
case line of
  PM from m -> msg h from ("You said \"" ++ m ++ "\" to me?!")
```

and now we can have a complete session:

```
*** PatternBot (~xxxx) has joined channel ##patternsynonyms
<PatternBot> Halló, heimur!
*** SomeNick (~yyyy) has joined channel ##patternsynonyms
<PatternBot> SomeNick: Welcome to ##patternsynonyms
*** SomeNick (~yyyy) has left channel ##patternsynonyms
<Iceland_jack> PatternBot: hey
<Iceland_jack> I should learn category theory
<PatternBot> Meow!
<Iceland_jack> > roll
<PatternBot> Iceland_jack: You rolled 3
```

and the core logic looks something like this

``` haskell
action :: Handle -> IO ()
action h = do
  line <- hGetLine h
  case line of
    PING serv           -> pong h serv
    JOIN PBot chan      -> msg h chan "Halló, heimur!"
    JOIN nick chan      -> msg h chan (nick ++ ": Welcome to " ++ chan)
    PM   from m         -> msg h from ("You said \"" ++ m ++ "\" to me?!")
    MSG  from chan Cat  -> msg h chan "Meow!"
    MSG  from chan Roll -> do
      roll :: Int <- randomRIO (1, 6)
      msg h chan (from ++ ": You rolled " ++ show roll)
    _                   -> return ()
```

This example is absolutely overusing pattern synonyms it but it ends up being quite pleasant.

# Some ideas

## Infix constructors
There are some things that would be nice to have: currently you can't pattern match on infix non-binary operators

``` haskell
-- This works
data Foo = (:⇒) Sender Recipient Message

pattern (:→) a b c = (:⇒) a b c
pattern To   a b c = (:⇒) a b c

-- This works
msg₁ :: Foo
msg₁ = ("Alice" :→ "Bob") "ossifrage"

msg₂ :: Foo
msg₂ = ("Bob" :→ "Alice") "pasta"

-- But {-hi-}this doesn't{-/hi-}
foo msg = case msg of
  ("Alice" :⇒ "Bob"  ) msg → …
  ("Bob"   :→ "Alice") msg → …
  (a      `To` b     ) msg → …

```
Replace Alice and Bob with nicks and channels and it makes sense for the IRC bot if you're into that sort of thing.

## Variadic patterns
More controversial, since patterns are always fully applied they could as well be variadic:

``` haskell
pattern PING             <- (words -> ["PING"])
pattern PING serv        <- (words -> ["PING", serv])
pattern PING serv₁ serv₂ <- (words -> ["PING", serv₁, serv₂])
```

It would also allow us to define the `BeforeChristmas` and `AfterChristmas` both ways

``` haskell
pattern BeforeChristmas     <- December (isItNow -> (GT, _))
pattern BeforeChristmas day <- December (isItNow -> (GT, day))

pattern AfterChristmas      <- December (isItNow -> (LT, day))
pattern AfterChristmas  day <- December (isItNow -> (LT, day))
```

or any other pattern that may be used for pattern matching only or accessing some value as well:


``` haskell
parity :: Int -> (Bool, Int)
parity n = (even n, n `div` 2)

pattern Even   <- (parity -> (True, _))
pattern Even n <- (parity -> (True, n))

pattern Odd    <- (parity -> (False, _))
pattern Odd n  <- (parity -> (False, n))

```
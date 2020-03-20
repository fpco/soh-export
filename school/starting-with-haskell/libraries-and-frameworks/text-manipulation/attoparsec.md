# Parsing Log FIles in Haskell

# Introduction

In this tutorial you will learn how to parse log-like files and how to
render a log to a file. Many applications use logs to keep track of some
useful information to be analysed later on. Parsing a log-like file it
is an easy parsing task in comparison with parsing, say, a programming
language, but it is an useful practice for a Haskell parser beginner.
Most of the code in this tutorial is editable and runnable, so take
advantage and experiment with the code yourself.

While log files do not have a specific format, we are going to output
them as CSV tables. An specification of CSV can be found in the
[RFC 4180](http://tools.ietf.org/html/rfc4180).

Among the many parser libraries in Haskell we have chosen
[_attoparsec_](http://hackage.haskell.org/package/attoparsec)
in this tutorial. Why? Firstly, because it is easy to use and secondly
because it is fast. The other popular choice is
[_parsec_](http://hackage.haskell.org/package/parsec).
_Parsec_ has a similar interface to _attoparsec_, but share also some differences.
For example, a parser in _parsec_ can be used as a monad transformer, allowing
you to add custom states. Also, when a parsing error arises, _parsec_
gives you a lot more information than _attoparsec_. The lack of these
features in _attoparsec_ is precisely what makes it faster.

# Writing a parser

Writing a parser involves _teaching_ our computer how to read something.
If a human see the string `"25"` it will quickly concludes that the string
contains a number. In fact, probably you read it as "twenty five" instead of
"two five". However, for the computer it is just a string of characters.
In Haskell, we would have to write a function from `String` (or `Text` or `ByteString`,
depending on the input type) to `Integer` in order to use it as a number.
This is what parsing means. But, how we accomplish such task?
Well, say that an application has sent to us the following `ByteString`:

``` haskell
"131.45.68.123"
```

It is the IP of a user that just connected to our server! In our code,
we have the following type definition:

``` haskell
import Data.Word

data IP = IP Word8 Word8 Word8 Word8 deriving Show
```

It is a type we defined for IP's. The `Word8` type represents 8-bit unsigned integer values.
Now it would be great if we could parse the input `131.45.68.123` to the value
`IP 131 45 68 123`. The first thing we look is how IP's are written. They follow this pattern:

* An 8-bit integer.
* A _dot_.
* An 8-bit integer.
* A _dot_.
* An 8-bit integer.
* A _dot_.
* An 8-bit integer.

When we write a parser in Haskell, what we actually do is following the pattern of the input format
from left to right. In this case, the function `parseIP` defines a parser for our type `IP` following
the pattern we just described. Note that the `decimal` parser succeeds for any unsigned integral number
(`Word8` in this example).

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

-- This attoparsec module is intended for parsing text that is
-- represented using an 8-bit character set, e.g. ASCII or ISO-8859-15.
import Data.Attoparsec.Char8
import Data.Word

-- | Type for IP's.
data IP = IP Word8 Word8 Word8 Word8 deriving Show

parseIP :: Parser IP
parseIP = do
  d1 <- decimal
  char '.'
  d2 <- decimal
  char '.'
  d3 <- decimal
  char '.'
  d4 <- decimal
  return $ IP d1 d2 d3 d4

main :: IO ()
main = print $ parseOnly parseIP "131.45.68.123"
```

Note that the output of `parseOnly`, the function that applies the parser
`parseIP` to the input `"131.45.68.123"` returns a value of type
`Either String IP`. This is because parsing is not a _total_ function, meaning
that not every input has an output. For example, parsing the string
`"foo"` cannot result in any IP. As a consequence, the parser fails.
Each time the parser fails, it will return `Left str`, where `str`
is a value of type `String` describing the error (in _attoparsec_, not
very descriptive actually). If the parser ends successfuly, it will
return `Right x`, where `x` is the parsed value.

As you can see, the approach to define a parser is to use simpler parsers
and combine them write parsers for more complex expressions. In the following
example, you will see how to parse a log file, including IP's. We will re-use
the recently created parser.

# Parsing logs

In this section, we develop a parser for log files that mixes content of
different types. We use an example to guide the process.

## Step 1: Define types

Say we have an online shop where we sell computer items
like mouses, keyboards, monitors and speakers. Each time a product is sold,
our application saves some information in a log file, containing the time
when the product was sold, the IP of the client and the name of the product.
Each log entry may be represented by the following type:

``` haskell
import Data.Time

data Product = Mouse | Keyboard | Monitor | Speakers deriving Show

data LogEntry =
  LogEntry { -- A local time contains the date and the time of the day.
             -- For example: 2013-06-29 11:16:23.
             entryTime :: LocalTime
           , entryIP   :: IP
           , entryProduct   :: Product
             } deriving Show
```

The log file will therefore contain a list of elements of type `LogEntry`.

``` haskell
-- | Type synonym of a list of log entries.
type Log = [LogEntry]
```

## Step 2: Follow the syntax

The log file, or anything that we can parse, follows a specific syntax.
For example, here is our today log:

```
2013-06-29 11:16:23 124.67.34.60 keyboard
2013-06-29 11:32:12 212.141.23.67 mouse
2013-06-29 11:33:08 212.141.23.67 monitor
2013-06-29 12:12:34 125.80.32.31 speakers
2013-06-29 12:51:50 101.40.50.62 keyboard
2013-06-29 13:10:45 103.29.60.13 mouse
```

Each line contains a log entry. The idea is to write a parser for log entries,
and iterate it line by line to get the list of every log entry.
The elements contained in each entry would be of type `LocalTime`, `IP` and
`Product`. We have to write parsers for each one and combine them.
Fortunately, we already have a parser for IP's that we can re-use.
Let's write a parser for the time stamps.

We notice that the format followed in our log is:

```
yyyy-MM-dd hh:mm:ss
```

Following this specification, we can easily write the parser as follows.

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.Time
import Data.Attoparsec.Char8

timeParser :: Parser LocalTime
timeParser = do
  y  <- count 4 digit
  char '-'
  mm <- count 2 digit
  char '-'
  d  <- count 2 digit
  char ' '
  h  <- count 2 digit
  char ':'
  m  <- count 2 digit
  char ':'
  s  <- count 2 digit
  return $
    LocalTime { localDay = fromGregorian (read y) (read mm) (read d)
              , localTimeOfDay = TimeOfDay (read h) (read m) (read s)
                }

main :: IO ()
main = print $ parseOnly timeParser "2013-06-30 14:33:29"
```

Note the use of `count` and `digit`. The parser `digit` will get the
following character, in case that this character is a digit, and will fail otherwise.
The combinator `count` repeats a parser a certain number of times. Since in our format,
a year is written with 4 characters, we use `count 4 digit` meaning _read 4 digits from the input_.
The same rationale applies to the rest of the code. At the end, we return a value of type
`LocalTime`.

## Parsing alternatives

Lastly, we need a parser for `Product` values. This one is even easier, but it also have something
new. A product is represented by a word. Each word is different, so there is no single syntax to
read. We have different choices. It is either `keyboard` or `mouse` or `monitor` or `speaker`.
This _or_, separating different alternatives, it is represented in attoparsec by the `<|>` combinator.
The `<|>` operator combines two parsers of the _same type_ in one that first tries to use the first
argument parser. If this one ends without failure, it returns its result. If it fails, it tries with
the second one, returning any result it gives.
This would be the `Product` parser:

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.Attoparsec.Char8
import Control.Applicative

data Product = Mouse | Keyboard | Monitor | Speakers deriving Show

productParser :: Parser Product
productParser =
     (string "mouse"    >> return Mouse)
 <|> (string "keyboard" >> return Keyboard)
 <|> (string "monitor"  >> return Monitor)
 <|> (string "speakers" >> return Speakers)

main :: IO ()
main = do
  print $ parseOnly productParser "mouse"
  print $ parseOnly productParser "mouze"
  print $ parseOnly productParser "monitor"
  print $ parseOnly productParser "keyboard"
```

Note that we have to import the `Control.Applicative` module to use the `<|>` combinator.
Also note that when we try to parse `mouze` we get a cryptic error message (_not enough
bytes_) that does not say much about the parsing error. This is one trade-off of attoparsec
in order to get better performance than parsec. The API of parsec is very similar to the
one of attoparsec, but parsec reports much more information when a parsing error arises.

## Step 3: Combine small parsers to build a bigger one

It is time to combine our parsers into one that can read a whole log entry. We only have
to invoke them in order.

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.Word
import Data.Time
import Data.Attoparsec.Char8
import Control.Applicative

-----------------------
-------- TYPES --------
-----------------------

-- | Type for IP's.
data IP = IP Word8 Word8 Word8 Word8 deriving Show

data Product = Mouse | Keyboard | Monitor | Speakers deriving Show

data LogEntry =
  LogEntry { entryTime :: LocalTime
           , entryIP   :: IP
           , entryProduct   :: Product
             } deriving Show

-----------------------
------- PARSING -------
-----------------------

-- | Parser of values of type 'IP'.
parseIP :: Parser IP
parseIP = do
  d1 <- decimal
  char '.'
  d2 <- decimal
  char '.'
  d3 <- decimal
  char '.'
  d4 <- decimal
  return $ IP d1 d2 d3 d4

-- | Parser of values of type 'LocalTime'.
timeParser :: Parser LocalTime
timeParser = do
  y  <- count 4 digit
  char '-'
  mm <- count 2 digit
  char '-'
  d  <- count 2 digit
  char ' '
  h  <- count 2 digit
  char ':'
  m  <- count 2 digit
  char ':'
  s  <- count 2 digit
  return $
    LocalTime { localDay = fromGregorian (read y) (read mm) (read d)
              , localTimeOfDay = TimeOfDay (read h) (read m) (read s)
                }

-- | Parser of values of type 'Product'.
productParser :: Parser Product
productParser =
     (string "mouse"    >> return Mouse)
 <|> (string "keyboard" >> return Keyboard)
 <|> (string "monitor"  >> return Monitor)
 <|> (string "speakers" >> return Speakers)
-- show
-- | Parser of log entries.
logEntryParser :: Parser LogEntry
logEntryParser = do
  -- First, we read the time.
  t <- timeParser
  -- Followed by a space.
  char ' '
  -- And then the IP of the client.
  ip <- parseIP
  -- Followed by another space.
  char ' '
  -- Finally, we read the type of product.
  p <- productParser
  -- And we return the result as a value of type 'LogEntry'.
  return $ LogEntry t ip p

----------------------
-------- TEST --------
----------------------

main :: IO ()
main = print $ parseOnly logEntryParser "2013-06-29 11:16:23 124.67.34.60 keyboard"
-- /show
```

In order to read the entire log file, we just need to iterate `logEntryParser` until
the end of the file is reached. The combinator `many` will perform a parser _zero_
or more times, returning a list of continuous successful parsings. It will stop whenever
the given parser fails. For example, `many digit` applied to the string `"123abc"` will
return `"123"` and will leave `"abc"` as remainding input. Also, `many digit` applied to
the string `"abc"` will return the empty list without consuming any input.

In conclusion, here is our log file parser.

``` haskell
type Log = [LogEntry]

logParser :: Parser Log
logParser = many $ logEntryParser <* endOfLine
```

The `endOfLine` parser succeeds only when the remaining input starts with an end
of line. The `<*` combinator applies the parser from the left, then the parser from
the right, and then returns the result of the first parser. We use it to get the result
from `logEntryParser` instead of `endOfLine`, which returns `()`.

## Full log file parser

``` active haskell
{-# START_FILE sellings.log #-}
2013-06-29 11:16:23 124.67.34.60 keyboard
2013-06-29 11:32:12 212.141.23.67 mouse
2013-06-29 11:33:08 212.141.23.67 monitor
2013-06-29 12:12:34 125.80.32.31 speakers
2013-06-29 12:51:50 101.40.50.62 keyboard
2013-06-29 13:10:45 103.29.60.13 mouse

{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}

import Data.Word
import Data.Time
import Data.Attoparsec.Char8
import Control.Applicative
-- We import ByteString qualified because the function
-- 'Data.ByteString.readFile' would clash with
-- 'Prelude.readFile'.
import qualified Data.ByteString as B

-----------------------
------ SETTINGS -------
-----------------------

-- | File where the log is stored.
logFile :: FilePath
logFile = "sellings.log"

-----------------------
-------- TYPES --------
-----------------------

-- | Type for IP's.
data IP = IP Word8 Word8 Word8 Word8 deriving Show

data Product = Mouse | Keyboard | Monitor | Speakers deriving Show

-- | Type for log entries.
--   Add, remove of modify fields to fit your own log file.
data LogEntry =
  LogEntry { entryTime :: LocalTime
           , entryIP   :: IP
           , entryProduct   :: Product
             } deriving Show

type Log = [LogEntry]

-----------------------
------- PARSING -------
-----------------------

-- | Parser of values of type 'IP'.
parseIP :: Parser IP
parseIP = do
  d1 <- decimal
  char '.'
  d2 <- decimal
  char '.'
  d3 <- decimal
  char '.'
  d4 <- decimal
  return $ IP d1 d2 d3 d4

-- | Parser of values of type 'LocalTime'.
timeParser :: Parser LocalTime
timeParser = do
  y  <- count 4 digit
  char '-'
  mm <- count 2 digit
  char '-'
  d  <- count 2 digit
  char ' '
  h  <- count 2 digit
  char ':'
  m  <- count 2 digit
  char ':'
  s  <- count 2 digit
  return $
    LocalTime { localDay = fromGregorian (read y) (read mm) (read d)
              , localTimeOfDay = TimeOfDay (read h) (read m) (read s)
                }

-- | Parser of values of type 'Product'.
productParser :: Parser Product
productParser =
     (string "mouse"    >> return Mouse)
 <|> (string "keyboard" >> return Keyboard)
 <|> (string "monitor"  >> return Monitor)
 <|> (string "speakers" >> return Speakers)

-- | Parser of log entries.
logEntryParser :: Parser LogEntry
logEntryParser = do
  -- First, we read the time.
  t <- timeParser
  -- Followed by a space.
  char ' '
  -- And then the IP of the client.
  ip <- parseIP
  -- Followed by another space.
  char ' '
  -- Finally, we read the type of product.
  p <- productParser
  -- And we return the result as a value of type 'LogEntry'.
  return $ LogEntry t ip p

logParser :: Parser Log
logParser = many $ logEntryParser <* endOfLine

----------------------
-------- MAIN --------
----------------------

main :: IO ()
main = B.readFile logFile >>= print . parseOnly logParser
```

## Changes in the log

After some time logging our sales, we have the idea of adding a new field
to each log entry. We ask each customer how he/she found about us and keep
this information in our log. We happily update the logger but quickly notice
that the parser does not work anymore. Apart from changing the `LogEntry` type
we have to modify the parser to work with the new values. We allow our users
to specify the following options:

``` haskell
data Source = Internet | Friend | NoAnswer deriving Show
```

We would report `NoAnswer` in the case that our customer did not answered.
Quickly we write a parser very similar to `productParser`.

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.Attoparsec.Char8
import Control.Applicative

data Source = Internet | Friend | NoAnswer deriving Show

sourceParser :: Parser Source
sourceParser =
      (string "internet" >> return Internet)
  <|> (string "friend" >> return Friend)
  <|> (string "noanswer" >> return NoAnswer)
  
main :: IO ()
main = print $ parseOnly sourceParser "internet"
```

After checking that this parser works, we add it to our `logEntryParser`,
upgrading the type definition of `LogEntry` adding the field `source`.

``` active haskell
{-# START_FILE sellings.log #-}
2013-06-29 16:40:15 154.41.32.99 monitor internet
2013-06-29 16:51:12 103.29.60.13 keyboard internet
2013-06-29 17:13:21 121.95.68.21 speakers friend
2013-06-29 18:20:10 190.80.70.60 mouse noanswer
2013-06-29 18:51:23 102.42.52.64 speakers friend
2013-06-29 19:01:11 78.46.64.23 mouse internet

{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}

import Data.Word
import Data.Time
import Data.Attoparsec.Char8
import Control.Applicative
-- We import ByteString qualified because the function
-- 'Data.ByteString.readFile' would clash with
-- 'Prelude.readFile'.
import qualified Data.ByteString as B

-----------------------
------ SETTINGS -------
-----------------------

-- | File where the log is stored.
logFile :: FilePath
logFile = "sellings.log"

-----------------------
-------- TYPES --------
-----------------------

-- | Type for IP's.
data IP = IP Word8 Word8 Word8 Word8 deriving Show

data Product = Mouse | Keyboard | Monitor | Speakers deriving Show

data Source = Internet | Friend | NoAnswer deriving Show

-- show
data LogEntry =
  LogEntry { entryTime :: LocalTime
           , entryIP   :: IP
           , entryProduct   :: Product
             -- Addition of the 'Source' field
           , source    :: Source
             } deriving Show
-- /show

type Log = [LogEntry]

-----------------------
------- PARSING -------
-----------------------

-- | Parser of values of type 'IP'.
parseIP :: Parser IP
parseIP = do
  d1 <- decimal
  char '.'
  d2 <- decimal
  char '.'
  d3 <- decimal
  char '.'
  d4 <- decimal
  return $ IP d1 d2 d3 d4

-- | Parser of values of type 'LocalTime'.
timeParser :: Parser LocalTime
timeParser = do
  y  <- count 4 digit
  char '-'
  mm <- count 2 digit
  char '-'
  d  <- count 2 digit
  char ' '
  h  <- count 2 digit
  char ':'
  m  <- count 2 digit
  char ':'
  s  <- count 2 digit
  return $
    LocalTime { localDay = fromGregorian (read y) (read mm) (read d)
              , localTimeOfDay = TimeOfDay (read h) (read m) (read s)
                }

-- | Parser of values of type 'Product'.
productParser :: Parser Product
productParser =
     (string "mouse"    >> return Mouse)
 <|> (string "keyboard" >> return Keyboard)
 <|> (string "monitor"  >> return Monitor)
 <|> (string "speakers" >> return Speakers)

sourceParser :: Parser Source
sourceParser =
      (string "internet" >> return Internet)
  <|> (string "friend" >> return Friend)
  <|> (string "noanswer" >> return NoAnswer)

-- show
-- | Parser of log entries.
logEntryParser :: Parser LogEntry
logEntryParser = do
  t <- timeParser
  char ' '
  ip <- parseIP
  char ' '
  p <- productParser
  -- Addition of the 'Source' field
  char ' '
  s <- sourceParser
  --
  return $ LogEntry t ip p s
-- /show

logParser :: Parser Log
logParser = many $ logEntryParser <* endOfLine

----------------------
-------- MAIN --------
----------------------

main :: IO ()
main = B.readFile logFile >>= print . parseOnly logParser
```

### Making the changed parser compatible with the old format

However, this parser only works in the new data, and we do not
want to lose the information we gathered before. The solution
is to add an _optional_ field in the parser and, when no value
is found, return a default value (like `NoAnswer`). The `option`
attoparsec combinators has exactly this purpose.

``` active haskell
{-# START_FILE sellings.log #-}
2013-06-29 11:16:23 124.67.34.60 keyboard
2013-06-29 11:32:12 212.141.23.67 mouse
2013-06-29 11:33:08 212.141.23.67 monitor
2013-06-29 12:12:34 125.80.32.31 speakers
2013-06-29 12:51:50 101.40.50.62 keyboard
2013-06-29 13:10:45 103.29.60.13 mouse
2013-06-29 16:40:15 154.41.32.99 monitor internet
2013-06-29 16:51:12 103.29.60.13 keyboard internet
2013-06-29 17:13:21 121.95.68.21 speakers friend
2013-06-29 18:20:10 190.80.70.60 mouse noanswer
2013-06-29 18:51:23 102.42.52.64 speakers friend
2013-06-29 19:01:11 78.46.64.23 mouse internet

{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}

import Data.Word
import Data.Time
import Data.Attoparsec.Char8
import Control.Applicative
-- We import ByteString qualified because the function
-- 'Data.ByteString.readFile' would clash with
-- 'Prelude.readFile'.
import qualified Data.ByteString as B

-----------------------
------ SETTINGS -------
-----------------------

-- | File where the log is stored.
logFile :: FilePath
logFile = "sellings.log"

-----------------------
-------- TYPES --------
-----------------------

-- | Type for IP's.
data IP = IP Word8 Word8 Word8 Word8 deriving Show

data Product = Mouse | Keyboard | Monitor | Speakers deriving Show

data Source = Internet | Friend | NoAnswer deriving Show

data LogEntry =
  LogEntry { entryTime :: LocalTime
           , entryIP   :: IP
           , entryProduct   :: Product
           , source    :: Source
             } deriving Show

type Log = [LogEntry]

-----------------------
------- PARSING -------
-----------------------

-- | Parser of values of type 'IP'.
parseIP :: Parser IP
parseIP = do
  d1 <- decimal
  char '.'
  d2 <- decimal
  char '.'
  d3 <- decimal
  char '.'
  d4 <- decimal
  return $ IP d1 d2 d3 d4

-- | Parser of values of type 'LocalTime'.
timeParser :: Parser LocalTime
timeParser = do
  y  <- count 4 digit
  char '-'
  mm <- count 2 digit
  char '-'
  d  <- count 2 digit
  char ' '
  h  <- count 2 digit
  char ':'
  m  <- count 2 digit
  char ':'
  s  <- count 2 digit
  return $
    LocalTime { localDay = fromGregorian (read y) (read mm) (read d)
              , localTimeOfDay = TimeOfDay (read h) (read m) (read s)
                }

-- | Parser of values of type 'Product'.
productParser :: Parser Product
productParser =
     (string "mouse"    >> return Mouse)
 <|> (string "keyboard" >> return Keyboard)
 <|> (string "monitor"  >> return Monitor)
 <|> (string "speakers" >> return Speakers)

sourceParser :: Parser Source
sourceParser =
      (string "internet" >> return Internet)
  <|> (string "friend" >> return Friend)
  <|> (string "noanswer" >> return NoAnswer)

-- show
-- | Parser of log entries.
logEntryParser :: Parser LogEntry
logEntryParser = do
  t <- timeParser
  char ' '
  ip <- parseIP
  char ' '
  p <- productParser
  -- Look for the field 'Source' and return
  -- a default value ('NoAnswer') when missing.
  -- The arguments of 'option' are default value
  -- followed by the parser to try.
  s <- option NoAnswer $ char ' ' >> sourceParser
  --
  return $ LogEntry t ip p s
-- /show

logParser :: Parser Log
logParser = many $ logEntryParser <* endOfLine

----------------------
-------- MAIN --------
----------------------

main :: IO ()
main = B.readFile logFile >>= print . parseOnly logParser
```

## Merging data from different logs

Our company is growing fast and we decide to open a new online shop based
in French to extend our customer range to Europe. However, after some time,
we note that our engineer in French is using a different log format.

```
154.41.32.99 29/06/2013 15:32:23 4 internet
76.125.44.33 29/06/2013 16:56:45 3 noanswer
123.45.67.89 29/06/2013 18:44:29 4 friend
100.23.32.41 29/06/2013 19:01:09 1 internet
151.123.45.67 29/06/2013 20:30:13 2 internet
```

It seems that each log entry stores the information in the following order:

* IP.
* Date (in a different format).
* A number representing the product sold.
* The "how you knew from us" field that we called Source before.

Therefore, our new `logEntryParser2` must parse the input in that order.
We note that the date is in a different order (in most Europe countries
is usual to write the day before the month) and is separated by the
`/` symbol instead of `-`. Also, they are using ID's to identify products
instead of writing the whole name.

### Step 1: Write the new parser

Firstly, we write functions to get the ID from a `Product` and viceversa.
Deriving an `Enum` instance for `Product` gives us an automatic implementation of
the methods `toEnum` and `fromEnum`. These functions are a correspondence between
a subset of the integers (type `Int`) and our type (`Product` in this case).
The automatic derivation associates the integer `0` to the first constructor, `1` to the
second, `2` to the third, and so on. Therefore, we can define functions `product(To/From)ID`
as follows.

``` active haskell
-- | Different kind of products are numbered from 1 to 4, in the given
--   order.
data Product = Mouse | Keyboard | Monitor | Speakers deriving (Enum,Show)

productFromID :: Int -> Product
productFromID n = toEnum (n-1)

productToID :: Product -> Int
productToID p = fromEnum p + 1

main :: IO ()
main = do
  print $ productFromID 1
  print $ productFromID 3
  print $ productToID Keyboard
  print $ productToID $ productFromID 4
```

A parser of products would accept a single digit and will apply `productFromID`
to get the `Product` result.

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.Attoparsec.Char8
import Control.Applicative

data Product = Mouse | Keyboard | Monitor | Speakers deriving (Enum,Show)

productFromID :: Int -> Product
productFromID n = toEnum (n-1)

-- show
productParser2 :: Parser Product
productParser2 = productFromID . read . (:[]) <$> digit

main :: IO ()
main = print $ parseOnly productParser2 "4"
-- /show
```

The `entryTime` field also needs a new parser. The process, however, is equivalent
to the previous one. We just need to parse the input in a different order and use
the new delimiters.

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.Time
import Data.Attoparsec.Char8

timeParser2 :: Parser LocalTime
timeParser2 = do
  d  <- count 2 digit
  char '/'
  mm <- count 2 digit
  char '/'
  y  <- count 4 digit
  char ' '
  h  <- count 2 digit
  char ':'
  m  <- count 2 digit
  char ':'
  s  <- count 2 digit
  return $
    LocalTime { localDay = fromGregorian (read y) (read mm) (read d)
              , localTimeOfDay = TimeOfDay (read h) (read m) (read s)
                }

main :: IO ()
main = print $ parseOnly timeParser2 "29/06/2013 15:32:23"
```

The rest of the fields are unchanged, so we are ready to write the full parser
of the new log entries. Again, this is just invoking the defined parsers in the
correct order.

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.Word
import Data.Time
import Data.Attoparsec.Char8
import Control.Applicative

-----------------------
-------- TYPES --------
-----------------------

-- | Type for IP's.
data IP = IP Word8 Word8 Word8 Word8 deriving Show

data Product = Mouse | Keyboard | Monitor | Speakers deriving (Show,Enum)

productFromID :: Int -> Product
productFromID n = toEnum (n-1)

data Source = Internet | Friend | NoAnswer deriving Show

data LogEntry =
  LogEntry { entryTime :: LocalTime
           , entryIP   :: IP
           , entryProduct   :: Product
           , source    :: Source
             } deriving Show

type Log = [LogEntry]

-----------------------
------- PARSING -------
-----------------------

-- | Parser of values of type 'IP'.
parseIP :: Parser IP
parseIP = do
  d1 <- decimal
  char '.'
  d2 <- decimal
  char '.'
  d3 <- decimal
  char '.'
  d4 <- decimal
  return $ IP d1 d2 d3 d4

timeParser2 :: Parser LocalTime
timeParser2 = do
  d  <- count 2 digit
  char '/'
  mm <- count 2 digit
  char '/'
  y  <- count 4 digit
  char ' '
  h  <- count 2 digit
  char ':'
  m  <- count 2 digit
  char ':'
  s  <- count 2 digit
  return $
    LocalTime { localDay = fromGregorian (read y) (read mm) (read d)
              , localTimeOfDay = TimeOfDay (read h) (read m) (read s)
                }

productParser2 :: Parser Product
productParser2 = productFromID . read . (:[]) <$> digit

sourceParser :: Parser Source
sourceParser =
      (string "internet" >> return Internet)
  <|> (string "friend" >> return Friend)
  <|> (string "noanswer" >> return NoAnswer)

-- show
logEntryParser2 :: Parser LogEntry
logEntryParser2 = do
  ip <- parseIP
  char ' '
  t <- timeParser2
  char ' '
  p <- productParser2
  char ' ' 
  s <- sourceParser
  return $ LogEntry t ip p s
  
main :: IO ()
main = print $ parseOnly logEntryParser2 "54.41.32.99 29/06/2013 15:32:23 4 internet"
-- /show
```

Once we have a function to read log entries we do the same as above to iterate the
parser line by line through the log file.

``` haskell
logParser2 :: Parser Log
logParser2 = many $ logEntryParser2 <* endOfLine
```

### Step 2: Merge both logs conserving order

Currently we have two log files, but we want all the data together.
The proposed solution is to parse one file, parse the other file, and
merge both of them. The merging can be done since both parsers have
the same _type_ of output (`Log`). A `Log` is a list of log entries, so
we could just append both lists and we will have all the data together.
However, since both files are sorted by `entryTime`, it would be much nicer
if the merged file is also sorted by `entryTime`.

Given two sorted lists, it is easy to merge them into one sorted list
in _linear time_. This is the procedure used to merge in the _mergesort_ algorithm.

``` active haskell
merge :: Ord a => [a] -> [a] -> [a]
merge xs [] = xs
merge [] ys = ys
merge (x:xs) (y:ys) =
  if x <= y
     then x : merge xs (y:ys)
     else y : merge (x:xs) ys

main :: IO ()
main = print $ merge [1,3,5,7] [2,4,6,8]
```

To use `merge`, the elements of the list must be of a type instance of the
`Ord` class. `Log` is a list of `LogEntry`, so we have to write an `Ord`
instance for `LogEntry`. We use `entryTime` as a reference to compare different
log entries, since our interest is to sort log entries by time.

``` haskell
instance Ord LogEntry where
  le1 <= le2 = entryTime le1 <= entryTime le2
```

Now we are ready to merge both log files into one single result of type `Log`.

``` active haskell
{-# START_FILE sellings.log #-}
2013-06-29 11:16:23 124.67.34.60 keyboard
2013-06-29 11:32:12 212.141.23.67 mouse
2013-06-29 11:33:08 212.141.23.67 monitor
2013-06-29 12:12:34 125.80.32.31 speakers
2013-06-29 12:51:50 101.40.50.62 keyboard
2013-06-29 13:10:45 103.29.60.13 mouse
2013-06-29 16:40:15 154.41.32.99 monitor internet
2013-06-29 16:51:12 103.29.60.13 keyboard internet
2013-06-29 17:13:21 121.95.68.21 speakers friend
2013-06-29 18:20:10 190.80.70.60 mouse noanswer
2013-06-29 18:51:23 102.42.52.64 speakers friend
2013-06-29 19:01:11 78.46.64.23 mouse internet

{-# START_FILE sellings2.log #-}
154.41.32.99 29/06/2013 15:32:23 4 internet
76.125.44.33 29/06/2013 16:56:45 3 noanswer
123.45.67.89 29/06/2013 18:44:29 4 friend
100.23.32.41 29/06/2013 19:01:09 1 internet
151.123.45.67 29/06/2013 20:30:13 2 internet

{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}

import Data.Word
import Data.Time
import Data.Attoparsec.Char8
import Control.Applicative
import qualified Data.ByteString as B

-- show
-----------------------
------ SETTINGS -------
-----------------------
-- | File where the log is stored.
logFile :: FilePath
logFile = "sellings.log"

-- | Second file where the log is stored.
logFile2 :: FilePath
logFile2 = "sellings2.log"
-- /show

-----------------------
-------- TYPES --------
-----------------------

-- | Type for IP's.
data IP = IP Word8 Word8 Word8 Word8 deriving (Eq,Show)

-- | Type for products.
data Product = Mouse | Keyboard | Monitor | Speakers deriving (Eq,Show,Enum)

productFromID :: Int -> Product
productFromID n = toEnum (n-1)

data Source = Internet | Friend | NoAnswer deriving (Eq,Show)

data LogEntry =
  LogEntry { entryTime :: LocalTime
           , entryIP   :: IP
           , entryProduct   :: Product
           , source    :: Source
               -- We derive Eq since is needed to be able
               -- to write an instance of Ord.
             } deriving (Eq, Show)

instance Ord LogEntry where
  le1 <= le2 = entryTime le1 <= entryTime le2

type Log = [LogEntry]

-----------------------
------- PARSING -------
-----------------------

-- | Parser of values of type 'IP'.
parseIP :: Parser IP
parseIP = do
  d1 <- decimal
  char '.'
  d2 <- decimal
  char '.'
  d3 <- decimal
  char '.'
  d4 <- decimal
  return $ IP d1 d2 d3 d4

-- | Parser of values of type 'LocalTime'.
timeParser :: Parser LocalTime
timeParser = do
  y  <- count 4 digit
  char '-'
  mm <- count 2 digit
  char '-'
  d  <- count 2 digit
  char ' '
  h  <- count 2 digit
  char ':'
  m  <- count 2 digit
  char ':'
  s  <- count 2 digit
  return $
    LocalTime { localDay = fromGregorian (read y) (read mm) (read d)
              , localTimeOfDay = TimeOfDay (read h) (read m) (read s)
                }

-- | Parser of values of type 'Product'.
productParser :: Parser Product
productParser =
     (string "mouse"    >> return Mouse)
 <|> (string "keyboard" >> return Keyboard)
 <|> (string "monitor"  >> return Monitor)
 <|> (string "speakers" >> return Speakers)

sourceParser :: Parser Source
sourceParser =
      (string "internet" >> return Internet)
  <|> (string "friend" >> return Friend)
  <|> (string "noanswer" >> return NoAnswer)

-- | Parser of log entries.
logEntryParser :: Parser LogEntry
logEntryParser = do
  t <- timeParser
  char ' '
  ip <- parseIP
  char ' '
  p <- productParser
  s <- option NoAnswer $ char ' ' >> sourceParser
  return $ LogEntry t ip p s

logParser :: Parser Log
logParser = many $ logEntryParser <* endOfLine

timeParser2 :: Parser LocalTime
timeParser2 = do
  d  <- count 2 digit
  char '/'
  mm <- count 2 digit
  char '/'
  y  <- count 4 digit
  char ' '
  h  <- count 2 digit
  char ':'
  m  <- count 2 digit
  char ':'
  s  <- count 2 digit
  return $
    LocalTime { localDay = fromGregorian (read y) (read mm) (read d)
              , localTimeOfDay = TimeOfDay (read h) (read m) (read s)
                }

productParser2 :: Parser Product
productParser2 = productFromID . read . (:[]) <$> digit

logEntryParser2 :: Parser LogEntry
logEntryParser2 = do
  ip <- parseIP
  char ' '
  t <- timeParser2
  char ' '
  p <- productParser2
  char ' ' 
  s <- sourceParser
  return $ LogEntry t ip p s

logParser2 :: Parser Log
logParser2 = many $ logEntryParser2 <* endOfLine

-----------------------
------- MERGING -------
-----------------------

merge :: Ord a => [a] -> [a] -> [a]
merge xs [] = xs
merge [] ys = ys
merge (x:xs) (y:ys) =
  if x <= y
     then x : merge xs (y:ys)
     else y : merge (x:xs) ys

-- show
----------------------
-------- MAIN --------
----------------------

main :: IO ()
main = do
  file1 <- B.readFile logFile
  file2 <- B.readFile logFile2
          -- We are using the Either monad here.
  let r = do xs <- parseOnly logParser  file1
             ys <- parseOnly logParser2 file2
             return $ merge xs ys
  case r of
   Left err -> putStrLn $ "A parsing error was found: " ++ err
   Right log -> mapM_ print log
-- /show
```

## Extracting information from the log file

Once the log file is parsed, we can extract information from it.
Following the previous example, we can check what is the product sold with more frequency
or where most users found our webshop.

Let's calculate the product that has been sold more times. We may create an association list containing
pairs (product,number of sales) for each product. It would have the following type:

``` haskell
type Sales = [(Product,Int)]
```

Given a list like this, we can check how many times a product has been sold.

``` haskell
import Data.Maybe (fromMaybe)

salesOf :: Product -> Sales -> Int
salesOf p xs = fromMaybe 0 $ lookup p xs
```

We can also add one sale more to the list.

``` haskell
addSale :: Product -> Sales -> Sales
-- If we have no sales, we add the product with 1 sale.
addSale p [] = [(p,1)]
addSale p ((x,n):xs) = if p == x then (x,n+1):xs
                                 else (x,n) : addSale p xs
```

Calculating the most sold product can be done using `maximumBy` (from
the `Data.List` module) to compare the elements of the list using the
second component of each pair.

``` haskell
import Data.List (maximumBy)

-- | Given a list of sales, returns the most sold product along with
--   its number of sales.
mostSold :: Sales -> Maybe (Product,Int)
mostSold [] = Nothing
mostSold xs = Just $ maximumBy (\x y -> snd x `compare` snd y) xs
```

We need to use `Maybe` to handle the event when nothing has been sold yet.

The last task remainding is to build a list of type `Sales` from a value of
`Log` type. Since each log entry contains one product, we can use a fold in
the log list using `addSale` for each entry product, adding all these items
to the empty list.

``` haskell
sales :: Log -> Sales
sales = foldr (addSales . entryProduct) []
```

Using now the same data as before, we output the product with more sales.

``` active haskell
{-# START_FILE sellings.log #-}
2013-06-29 11:16:23 124.67.34.60 keyboard
2013-06-29 11:32:12 212.141.23.67 mouse
2013-06-29 11:33:08 212.141.23.67 monitor
2013-06-29 12:12:34 125.80.32.31 speakers
2013-06-29 12:51:50 101.40.50.62 keyboard
2013-06-29 13:10:45 103.29.60.13 mouse
2013-06-29 16:40:15 154.41.32.99 monitor internet
2013-06-29 16:51:12 103.29.60.13 keyboard internet
2013-06-29 17:13:21 121.95.68.21 speakers friend
2013-06-29 18:20:10 190.80.70.60 mouse noanswer
2013-06-29 18:51:23 102.42.52.64 speakers friend
2013-06-29 19:01:11 78.46.64.23 mouse internet

{-# START_FILE sellings2.log #-}
154.41.32.99 29/06/2013 15:32:23 4 internet
76.125.44.33 29/06/2013 16:56:45 3 noanswer
123.45.67.89 29/06/2013 18:44:29 4 friend
100.23.32.41 29/06/2013 19:01:09 1 internet
151.123.45.67 29/06/2013 20:30:13 2 internet

{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}

import Data.Word
import Data.Time
import Data.Attoparsec.Char8
import Control.Applicative
import qualified Data.ByteString as B
import Data.List (maximumBy)
import Data.Maybe (fromMaybe)

-----------------------
------ SETTINGS -------
-----------------------

-- | File where the log is stored.
logFile :: FilePath
logFile = "sellings.log"

-- | Second file where the log is stored.
logFile2 :: FilePath
logFile2 = "sellings2.log"

-----------------------
-------- TYPES --------
-----------------------

-- | Type for IP's.
data IP = IP Word8 Word8 Word8 Word8 deriving (Eq,Show)

-- | Type for products.
data Product = Mouse | Keyboard | Monitor | Speakers deriving (Eq,Show,Enum)

productFromID :: Int -> Product
productFromID n = toEnum (n-1)

data Source = Internet | Friend | NoAnswer deriving (Eq,Show)

data LogEntry =
  LogEntry { entryTime :: LocalTime
           , entryIP   :: IP
           , entryProduct   :: Product
           , source    :: Source
               -- We derive Eq since is needed to be able
               -- to write an instance of Ord.
             } deriving (Eq, Show)

instance Ord LogEntry where
  le1 <= le2 = entryTime le1 <= entryTime le2

type Log = [LogEntry]

-----------------------
------- PARSING -------
-----------------------

-- | Parser of values of type 'IP'.
parseIP :: Parser IP
parseIP = do
  d1 <- decimal
  char '.'
  d2 <- decimal
  char '.'
  d3 <- decimal
  char '.'
  d4 <- decimal
  return $ IP d1 d2 d3 d4

-- | Parser of values of type 'LocalTime'.
timeParser :: Parser LocalTime
timeParser = do
  y  <- count 4 digit
  char '-'
  mm <- count 2 digit
  char '-'
  d  <- count 2 digit
  char ' '
  h  <- count 2 digit
  char ':'
  m  <- count 2 digit
  char ':'
  s  <- count 2 digit
  return $
    LocalTime { localDay = fromGregorian (read y) (read mm) (read d)
              , localTimeOfDay = TimeOfDay (read h) (read m) (read s)
                }

-- | Parser of values of type 'Product'.
productParser :: Parser Product
productParser =
     (string "mouse"    >> return Mouse)
 <|> (string "keyboard" >> return Keyboard)
 <|> (string "monitor"  >> return Monitor)
 <|> (string "speakers" >> return Speakers)

sourceParser :: Parser Source
sourceParser =
      (string "internet" >> return Internet)
  <|> (string "friend" >> return Friend)
  <|> (string "noanswer" >> return NoAnswer)

-- | Parser of log entries.
logEntryParser :: Parser LogEntry
logEntryParser = do
  t <- timeParser
  char ' '
  ip <- parseIP
  char ' '
  p <- productParser
  s <- option NoAnswer $ char ' ' >> sourceParser
  return $ LogEntry t ip p s

logParser :: Parser Log
logParser = many $ logEntryParser <* endOfLine

timeParser2 :: Parser LocalTime
timeParser2 = do
  d  <- count 2 digit
  char '/'
  mm <- count 2 digit
  char '/'
  y  <- count 4 digit
  char ' '
  h  <- count 2 digit
  char ':'
  m  <- count 2 digit
  char ':'
  s  <- count 2 digit
  return $
    LocalTime { localDay = fromGregorian (read y) (read mm) (read d)
              , localTimeOfDay = TimeOfDay (read h) (read m) (read s)
                }

productParser2 :: Parser Product
productParser2 = productFromID . read . (:[]) <$> digit

logEntryParser2 :: Parser LogEntry
logEntryParser2 = do
  ip <- parseIP
  char ' '
  t <- timeParser2
  char ' '
  p <- productParser2
  char ' ' 
  s <- sourceParser
  return $ LogEntry t ip p s

logParser2 :: Parser Log
logParser2 = many $ logEntryParser2 <* endOfLine

-----------------------
------- MERGING -------
-----------------------

merge :: Ord a => [a] -> [a] -> [a]
merge xs [] = xs
merge [] ys = ys
merge (x:xs) (y:ys) =
  if x <= y
     then x : merge xs (y:ys)
     else y : merge (x:xs) ys

----------------------
------ COUNTING ------
----------------------

type Sales = [(Product,Int)]

salesOf :: Product -> Sales -> Int
salesOf p xs = fromMaybe 0 $ lookup p xs

addSale :: Product -> Sales -> Sales
addSale p [] = [(p,1)]
addSale p ((x,n):xs) = if p == x then (x,n+1):xs
                                 else (x,n) : addSale p xs
                        
-- | Given a list of sales, returns the most sold product along with
--   its number of sales.
mostSold :: Sales -> Maybe (Product,Int)
mostSold [] = Nothing
mostSold xs = Just $ maximumBy (\x y -> snd x `compare` snd y) xs

sales :: Log -> Sales
sales = foldr (addSale . entryProduct) []

----------------------
-------- MAIN --------
----------------------

-- show
main :: IO ()
main = do
  file1 <- B.readFile logFile
  file2 <- B.readFile logFile2
  let r = do xs <- parseOnly logParser  file1
             ys <- parseOnly logParser2 file2
             return $ merge xs ys
  case r of
   Left err -> putStrLn $ "A parsing error was found: " ++ err
   Right log ->
     case mostSold (sales log) of
       Nothing -> putStrLn "We didn't sell anything yet."
       Just (p,n) -> putStrLn $ "The product with more sales is " ++ show p
                  ++ " with " ++ show n ++ " sales."
-- /show
```

# From log file to CSV

CSV (Comma Separated Values) files store tabular data and can be used from a large number of applications.
In fact, one of the advantages of using the CSV format is that data stored in this format can be imported
and exported from very different programs. After gathering all the log file information, we are going to
render a CSV table containing it. Then, we will develop a parser to get the data back into Haskell.

## Rendering to CSV

The process of rendering to CSV is straightforward. Rendering is in general simpler than parsing, and  CSV rendering is not an exception.

We define rendering methods for each type, as we defined parsers for each type.
Sometimes, the renderer looks similar to the parser (see `renderIP` below).

Some functions useful when rendering:

* `<>`: This operator from `Data.Monoid` appends values of types instance of the `Monoid` class.
  `ByteString` is one of them.
* `foldMap`: Apply a function over the elements of a structure instance of the `Foldable` class to
  values of a type instance of the `Monoid` class then append all the results.
* `fromString`: It takes a String and return it as a value of any type in the `IsString` class,
  defined at `Data.String`.

``` active haskell
{-# START_FILE sellings.log #-}
2013-06-29 11:16:23 124.67.34.60 keyboard
2013-06-29 11:32:12 212.141.23.67 mouse
2013-06-29 11:33:08 212.141.23.67 monitor
2013-06-29 12:12:34 125.80.32.31 speakers
2013-06-29 12:51:50 101.40.50.62 keyboard
2013-06-29 13:10:45 103.29.60.13 mouse
2013-06-29 16:40:15 154.41.32.99 monitor internet
2013-06-29 16:51:12 103.29.60.13 keyboard internet
2013-06-29 17:13:21 121.95.68.21 speakers friend
2013-06-29 18:20:10 190.80.70.60 mouse noanswer
2013-06-29 18:51:23 102.42.52.64 speakers friend
2013-06-29 19:01:11 78.46.64.23 mouse internet

{-# START_FILE sellings2.log #-}
154.41.32.99 29/06/2013 15:32:23 4 internet
76.125.44.33 29/06/2013 16:56:45 3 noanswer
123.45.67.89 29/06/2013 18:44:29 4 friend
100.23.32.41 29/06/2013 19:01:09 1 internet
151.123.45.67 29/06/2013 20:30:13 2 internet

{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}

import Data.Word
import Data.Time
import Data.Attoparsec.Char8
import Control.Applicative
-- show
import Data.ByteString.Char8 (ByteString,singleton)
import qualified Data.ByteString as B
import qualified Data.ByteString.Char8 as BC
import Data.String
import Data.Char (toLower)
import Data.Monoid hiding (Product)
import Data.Foldable (foldMap)
-- /show

-----------------------
------ SETTINGS -------
-----------------------

-- | File where the log is stored.
logFile :: FilePath
logFile = "sellings.log"

-- | Second file where the log is stored.
logFile2 :: FilePath
logFile2 = "sellings2.log"

-----------------------
-------- TYPES --------
-----------------------

-- | Type for IP's.
data IP = IP Word8 Word8 Word8 Word8 deriving (Eq,Show)

-- | Type for products.
data Product = Mouse | Keyboard | Monitor | Speakers deriving (Eq,Show,Enum)

productFromID :: Int -> Product
productFromID n = toEnum (n-1)

data Source = Internet | Friend | NoAnswer deriving (Eq,Show)

data LogEntry =
  LogEntry { entryTime :: LocalTime
           , entryIP   :: IP
           , entryProduct   :: Product
           , source    :: Source
               -- We derive Eq since is needed to be able
               -- to write an instance of Ord.
             } deriving (Eq, Show)

instance Ord LogEntry where
  le1 <= le2 = entryTime le1 <= entryTime le2

type Log = [LogEntry]

-----------------------
------- PARSING -------
-----------------------

-- | Parser of values of type 'IP'.
parseIP :: Parser IP
parseIP = do
  d1 <- decimal
  char '.'
  d2 <- decimal
  char '.'
  d3 <- decimal
  char '.'
  d4 <- decimal
  return $ IP d1 d2 d3 d4

-- | Parser of values of type 'LocalTime'.
timeParser :: Parser LocalTime
timeParser = do
  y  <- count 4 digit
  char '-'
  mm <- count 2 digit
  char '-'
  d  <- count 2 digit
  char ' '
  h  <- count 2 digit
  char ':'
  m  <- count 2 digit
  char ':'
  s  <- count 2 digit
  return $
    LocalTime { localDay = fromGregorian (read y) (read mm) (read d)
              , localTimeOfDay = TimeOfDay (read h) (read m) (read s)
                }

-- | Parser of values of type 'Product'.
productParser :: Parser Product
productParser =
     (string "mouse"    >> return Mouse)
 <|> (string "keyboard" >> return Keyboard)
 <|> (string "monitor"  >> return Monitor)
 <|> (string "speakers" >> return Speakers)

sourceParser :: Parser Source
sourceParser =
      (string "internet" >> return Internet)
  <|> (string "friend" >> return Friend)
  <|> (string "noanswer" >> return NoAnswer)

-- | Parser of log entries.
logEntryParser :: Parser LogEntry
logEntryParser = do
  t <- timeParser
  char ' '
  ip <- parseIP
  char ' '
  p <- productParser
  s <- option NoAnswer $ char ' ' >> sourceParser
  return $ LogEntry t ip p s

logParser :: Parser Log
logParser = many $ logEntryParser <* endOfLine

timeParser2 :: Parser LocalTime
timeParser2 = do
  d  <- count 2 digit
  char '/'
  mm <- count 2 digit
  char '/'
  y  <- count 4 digit
  char ' '
  h  <- count 2 digit
  char ':'
  m  <- count 2 digit
  char ':'
  s  <- count 2 digit
  return $
    LocalTime { localDay = fromGregorian (read y) (read mm) (read d)
              , localTimeOfDay = TimeOfDay (read h) (read m) (read s)
                }

productParser2 :: Parser Product
productParser2 = productFromID . read . (:[]) <$> digit

logEntryParser2 :: Parser LogEntry
logEntryParser2 = do
  ip <- parseIP
  char ' '
  t <- timeParser2
  char ' '
  p <- productParser2
  char ' ' 
  s <- sourceParser
  return $ LogEntry t ip p s

logParser2 :: Parser Log
logParser2 = many $ logEntryParser2 <* endOfLine

-----------------------
------- MERGING -------
-----------------------

merge :: Ord a => [a] -> [a] -> [a]
merge xs [] = xs
merge [] ys = ys
merge (x:xs) (y:ys) =
  if x <= y
     then x : merge xs (y:ys)
     else y : merge (x:xs) ys

-- show
-----------------------
------ RENDERING ------
-----------------------

-- | Character that will serve as field separator.
--   It should not be one of the characters that
--   appear in the fields.
sepChar :: Char
sepChar = ','

-- | Rendering of IP's to ByteString.
renderIP :: IP -> ByteString
renderIP (IP a b c d) =
     -- Function @show@ creates a String and
     -- fromString makes it a ByteString.
     fromString (show a)
  <> singleton '.'
  <> fromString (show b)
  <> singleton '.'
  <> fromString (show c)
  <> singleton '.'
  <> fromString (show d)

-- | Render a log entry to a CSV row as ByteString.
renderEntry :: LogEntry -> ByteString
renderEntry le =
     fromString (show $ entryTime le)
  <> singleton sepChar
  <> renderIP (entryIP le)
  <> singleton sepChar
     -- We use @fmap toLower@ to write the product name
     -- in lowercase letters.
  <> fromString (fmap toLower $ show $ entryProduct le)
  <> singleton sepChar
  <> fromString (fmap toLower $ show $ source le)

-- | Render a log file to CSV as ByteString.
renderLog :: Log -> ByteString
renderLog = foldMap $ \le -> renderEntry le <> singleton '\n'

----------------------
-------- MAIN --------
----------------------

main :: IO ()
main = do
  file1 <- B.readFile logFile
  file2 <- B.readFile logFile2
          -- We are using the Either monad here.
  let r = do xs <- parseOnly logParser  file1
             ys <- parseOnly logParser2 file2
             return $ merge xs ys
  case r of
   Left err -> putStrLn $ "A parsing error was found: " ++ err
   Right log -> BC.putStrLn $ renderLog log
-- /show
```
## Parsing from CSV

Again, as with log files, we use _attoparsec_ for parsing.
Note that the CSV format is similar to the log format, except
in how fields are separated. Therefore, we can re-use our
field parsers.

We start defining a parser for rows, and then we iterate it
using `many` exactly as before.

``` active haskell
{-# START_FILE sellings.csv #-}
2013-06-29 11:16:23 , 124.67.34.60  , keyboard , noanswer
2013-06-29 11:32:12 , 212.141.23.67 , mouse    , noanswer
2013-06-29 11:33:08 , 212.141.23.67 , monitor  , noanswer
2013-06-29 12:12:34 , 125.80.32.31  , speakers , noanswer
2013-06-29 12:51:50 , 101.40.50.62  , keyboard , noanswer
2013-06-29 13:10:45 , 103.29.60.13  , mouse    , noanswer
2013-06-29 15:32:23 , 154.41.32.99  , speakers , internet
2013-06-29 16:40:15 , 154.41.32.99  , monitor  , internet
2013-06-29 16:51:12 , 103.29.60.13  , keyboard , internet
2013-06-29 16:56:45 , 76.125.44.33  , monitor  , noanswer
2013-06-29 17:13:21 , 121.95.68.21  , speakers , friend
2013-06-29 18:20:10 , 190.80.70.60  , mouse    , noanswer
2013-06-29 18:44:29 , 123.45.67.89  , speakers , friend
2013-06-29 18:51:23 , 102.42.52.64  , speakers , friend
2013-06-29 19:01:09 , 100.23.32.41  , mouse    , internet
2013-06-29 19:01:11 , 78.46.64.23   , mouse    , internet
2013-06-29 20:30:13 , 151.123.45.67 , keyboard , internet

{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}

import Data.Word
import Data.Time
import Data.Attoparsec.Char8
import Control.Applicative
import qualified Data.ByteString as B
-- show
-----------------------
------ SETTINGS -------
-----------------------

-- | File where the CSV is stored.
csvFile :: FilePath
csvFile = "sellings.csv"

-- | Character that will serve as field separator.
--   It should not be one of the characters that
--   appear in the fields.
sepChar :: Char
sepChar = ','
-- /show

-----------------------
-------- TYPES --------
-----------------------

-- | Type for IP's.
data IP = IP Word8 Word8 Word8 Word8 deriving (Eq,Show)

-- | Type for products.
data Product = Mouse | Keyboard | Monitor | Speakers deriving (Eq,Show,Enum)

data Source = Internet | Friend | NoAnswer deriving (Eq,Show)

data LogEntry =
  LogEntry { entryTime :: LocalTime
           , entryIP   :: IP
           , entryProduct   :: Product
           , source    :: Source
               -- We derive Eq since is needed to be able
               -- to write an instance of Ord.
             } deriving (Eq, Show)

type Log = [LogEntry]

-- show
-----------------------
------- PARSING -------
-----------------------
-- /show

-- | Parser of values of type 'IP'.
parseIP :: Parser IP
parseIP = do
  d1 <- decimal
  char '.'
  d2 <- decimal
  char '.'
  d3 <- decimal
  char '.'
  d4 <- decimal
  return $ IP d1 d2 d3 d4

-- | Parser of values of type 'LocalTime'.
timeParser :: Parser LocalTime
timeParser = do
  y  <- count 4 digit
  char '-'
  mm <- count 2 digit
  char '-'
  d  <- count 2 digit
  char ' '
  h  <- count 2 digit
  char ':'
  m  <- count 2 digit
  char ':'
  s  <- count 2 digit
  return $
    LocalTime { localDay = fromGregorian (read y) (read mm) (read d)
              , localTimeOfDay = TimeOfDay (read h) (read m) (read s)
                }
                
-- | Parser of values of type 'Product'.
productParser :: Parser Product
productParser =
     (string "mouse"    >> return Mouse)
 <|> (string "keyboard" >> return Keyboard)
 <|> (string "monitor"  >> return Monitor)
 <|> (string "speakers" >> return Speakers)

sourceParser :: Parser Source
sourceParser =
      (string "internet" >> return Internet)
  <|> (string "friend" >> return Friend)
  <|> (string "noanswer" >> return NoAnswer)

-- show
rowParser :: Parser LogEntry
rowParser = do
  -- Parser of field separators. It skips space characters before
  -- and after the CSV separator char.
  -- Characters considered as space are simple whitespaces and tabs.
  let spaceSkip = many $ satisfy $ inClass [ ' ' , '\t' ]
      sepParser = spaceSkip >> char sepChar >> spaceSkip
  -- Skip spaces at the beginning of the line.
  spaceSkip
  t  <- timeParser
  sepParser
  ip <- parseIP
  sepParser
  p  <- productParser
  sepParser
  s  <- sourceParser
  -- Skip remaining spaces at the end of the line
  spaceSkip
  return $ LogEntry t ip p s

csvParser :: Parser Log
csvParser = many $ rowParser <* endOfLine

----------------------
-------- MAIN --------
----------------------

main :: IO ()
main = do
  file <- B.readFile csvFile
  case parseOnly csvParser file of
    Left err -> putStrLn $ "Error while parsing CSV file: " ++ err
    Right log -> mapM_ print log
-- /show
```

## Using CSV across applications

Use `renderLog` and `Data.ByteString.Char8.writeFile` to write a CSV table
using your log information. However, if you are using a character set different
from ASCII or ISO-8859-15, you should consider using the type `Text` instead
of `ByteString`. Almost the only change you have to do
is to change the import of `Data.Attoparsec.Char8` to `Data.Attoparsec.Text`
(both modules export similar interfaces and are interchangeable) and
adapt the types of the renderer.

Once you have written your data in CSV format, import it from another application.
Use the table, make any changes that you may want and modified data back in Haskell
by parsing the CSV output of your application. Make sure your application and the
Haskell parser are using the same column separator.

# Final App: Read several log files, merge data and render it in CSV

We now present a runnable application that read a list of log files, merge them and
return the result as a CSV table. The files may be read from a local file or an URL.

``` active haskell
{-# START_FILE sellings.log #-}
2013-06-29 11:16:23 124.67.34.60 keyboard
2013-06-29 11:32:12 212.141.23.67 mouse
2013-06-29 11:33:08 212.141.23.67 monitor
2013-06-29 12:12:34 125.80.32.31 speakers
2013-06-29 12:51:50 101.40.50.62 keyboard
2013-06-29 13:10:45 103.29.60.13 mouse
2013-06-29 16:40:15 154.41.32.99 monitor internet
2013-06-29 16:51:12 103.29.60.13 keyboard internet
2013-06-29 17:13:21 121.95.68.21 speakers friend
2013-06-29 18:20:10 190.80.70.60 mouse noanswer
2013-06-29 18:51:23 102.42.52.64 speakers friend
2013-06-29 19:01:11 78.46.64.23 mouse internet

{-# START_FILE sellings2.log #-}
2013-06-29 15:32:23 154.41.32.99 speakers internet
2013-06-29 16:56:45 76.125.44.33 monitor noanswer
2013-06-29 18:44:29 123.45.67.89 speakers friend
2013-06-29 19:01:09 100.23.32.41 mouse internet
2013-06-29 20:30:13 151.123.45.67 keyboard internet

{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}

import Data.Word
import Data.Time
import Data.Attoparsec.Char8
import Control.Applicative
import Data.Either (rights)
import Data.Monoid hiding (Product)
import Data.String
import Data.Char (toLower)
import Data.Foldable (foldMap)
-- ByteString stuff
import Data.ByteString.Char8 (ByteString,singleton)
import qualified Data.ByteString as B
import qualified Data.ByteString.Char8 as BC
import Data.ByteString.Lazy (toChunks)
-- HTTP protocol to perform downloads
import Network.HTTP.Conduit

----------------------
------- FILES --------
----------------------

data File = URL String | Local FilePath

-- | Files where the logs are stored.
--   Modify this value to read logs from
--   other sources.
logFiles :: [File]
logFiles =
  [ Local "sellings.log"
  , Local "sellings2.log"
  , URL "http://daniel-diaz.github.io/misc/sellings3.log"
    ]

getFile :: File -> IO ByteString
-- simpleHttp gets a lazy bytestring, while we
-- are using strict bytestrings.
getFile (URL str) = mconcat . toChunks <$> simpleHttp str
getFile (Local fp) = B.readFile fp

-----------------------
-------- TYPES --------
-----------------------

-- | Type for IP's.
data IP = IP Word8 Word8 Word8 Word8 deriving (Eq,Show)

-- | Type for products.
data Product = Mouse | Keyboard | Monitor | Speakers deriving (Eq,Show,Enum)

productFromID :: Int -> Product
productFromID n = toEnum (n-1)

data Source = Internet | Friend | NoAnswer deriving (Eq,Show)

-- | Each log entry in the log file is represented by a value
--   of this type. Modify the fields of 'LogEntry' accordingly
--   to your log file of interest. However, 'entryTime' is a
--   reasonable field and is also used for merging.
data LogEntry =
  LogEntry { entryTime :: LocalTime
           , entryIP   :: IP
           , entryProduct   :: Product
           , source    :: Source
             } deriving (Eq, Show)

instance Ord LogEntry where
  le1 <= le2 = entryTime le1 <= entryTime le2

type Log = [LogEntry]

-----------------------
------- PARSING -------
-----------------------

-- | Parser of values of type 'IP'.
parseIP :: Parser IP
parseIP = do
  d1 <- decimal
  char '.'
  d2 <- decimal
  char '.'
  d3 <- decimal
  char '.'
  d4 <- decimal
  return $ IP d1 d2 d3 d4

-- | Parser of values of type 'LocalTime'.
timeParser :: Parser LocalTime
timeParser = do
  y  <- count 4 digit
  char '-'
  mm <- count 2 digit
  char '-'
  d  <- count 2 digit
  char ' '
  h  <- count 2 digit
  char ':'
  m  <- count 2 digit
  char ':'
  s  <- count 2 digit
  return $
    LocalTime { localDay = fromGregorian (read y) (read mm) (read d)
              , localTimeOfDay = TimeOfDay (read h) (read m) (read s)
                }

-- | Parser of values of type 'Product'.
productParser :: Parser Product
productParser =
     (string "mouse"    >> return Mouse)
 <|> (string "keyboard" >> return Keyboard)
 <|> (string "monitor"  >> return Monitor)
 <|> (string "speakers" >> return Speakers)

sourceParser :: Parser Source
sourceParser =
      (string "internet" >> return Internet)
  <|> (string "friend" >> return Friend)
  <|> (string "noanswer" >> return NoAnswer)

-- | Parser of log entries.
logEntryParser :: Parser LogEntry
logEntryParser = do
  t <- timeParser
  char ' '
  ip <- parseIP
  char ' '
  p <- productParser
  s <- option NoAnswer $ char ' ' >> sourceParser
  return $ LogEntry t ip p s

logParser :: Parser Log
logParser = many $ logEntryParser <* endOfLine

-----------------------
------- MERGING -------
-----------------------

merge :: Ord a => [a] -> [a] -> [a]
merge xs [] = xs
merge [] ys = ys
merge (x:xs) (y:ys) =
  if x <= y
     then x : merge xs (y:ys)
     else y : merge (x:xs) ys

-----------------------
------ RENDERING ------
-----------------------

-- | Character that will serve as field separator.
--   It should not be one of the characters that
--   appear in the fields.
sepChar :: Char
sepChar = ','

-- | Rendering of IP's to ByteString.
renderIP :: IP -> ByteString
renderIP (IP a b c d) =
     fromString (show a)
  <> singleton '.'
  <> fromString (show b)
  <> singleton '.'
  <> fromString (show c)
  <> singleton '.'
  <> fromString (show d)

-- | Render a log entry to a CSV row as ByteString.
renderEntry :: LogEntry -> ByteString
renderEntry le =
     fromString (show $ entryTime le)
  <> singleton sepChar
  <> renderIP (entryIP le)
  <> singleton sepChar
  <> fromString (fmap toLower $ show $ entryProduct le)
  <> singleton sepChar
  <> fromString (fmap toLower $ show $ source le)

-- | Render a log file to CSV as ByteString.
renderLog :: Log -> ByteString
renderLog = foldMap $ \le -> renderEntry le <> singleton '\n'

----------------------
-------- MAIN --------
----------------------

main :: IO ()
main = do
  files <- mapM getFile logFiles
  let -- Parsed logs
      logs :: [Log]
      logs = rights $ fmap (parseOnly logParser) files
      -- Merged log
      mergedLog :: Log
      mergedLog = foldr merge [] logs
  BC.putStrLn $ renderLog mergedLog
```

# Conclusion

Parsing is one of the tasks that Haskell is really good at.
The parser code is much clearer and easier to write than in traditional languages and it may run
[faster than a C++ parser](http://newartisans.com/2012/08/parsing-with-haskell-and-attoparsec).
I invite you to try to parse bigger things. Following the
[API reference](http://hackage.haskell.org/package/attoparsec)
it should not be hard. As an example, Bryan O'Sullivan wrote an HTTP parser
[here](https://bitbucket.org/bos/attoparsec/src/tip/examples/RFC2616.hs).
I think it is easy to read once you know [how HTTP is defined](http://tools.ietf.org/html/rfc4180).
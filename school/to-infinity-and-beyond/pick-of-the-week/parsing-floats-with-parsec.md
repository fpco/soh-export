# Parsing Floats With Parsec

Note: This uses Parsec 3.

## Introduction

In many languages, when we want to write a simple parser the usual choices are either to use regular expressions or write a complete parser ourselves. Regular expressions are convenient for many reasons: they're concise, they're fast, and for simple parsing they are fairly readable for those that understand the syntax. Writing a parser by hand is much less convenient, but allows fine-grain error reporting and can support languages that are not regular.

These approaches have downsides, though. Regular expressions don't support much in the way of error handling, they require learning and understanding a completely different language, the exact syntax and features can vary between implementations, they only support regular languages (or if they do support languages that aren't regular it's through non-standard extensions), and even people with a lot of experience with regular expressions can have trouble understanding complex ones. Writing a parser by hand can alleviate these problems, but often lead to brittle implementations. The biggest problem with writing parsers by hand, however, is that it's simply a lot more work.

Luckily, parsing is a task that Haskell is especially well-suited for. Through libraries such as Parsec it's possible to build up complex parsers by using simple parsers and parser combinators. Not only does this result in code that is short and easy to understand, it provides good performance and helpful error messages for free.

## Parsing Integers

To start things simple, we'll create a parser for integers. Creating a full-blown parser for this might seem like overkill, but this will allow us to create fast, readable code with good error messages, all while remaining short and concise.

For our first go at creating a parser for integers, we'll ignore the optional leading signs. This makes things quite simple:

``` active haskell
import Control.Monad
-- show
import Text.Parsec
import Control.Applicative

integer = rd <$> many1 digit
    where rd = read :: String -> Integer
-- /show
main = forever $ do putStrLn "Enter an integer: "
                    input <- getLine
                    parseTest integer input
```

So what's happening here?

### Parsing

The heart of our parser is in `many1 digit`. `digit` is a parser that matches a single digit character (0-9). `many1` is a parser combinator; it takes a parser p and transforms it into a parser that parses 1 or more occurances of p. So the two together gives us a parser that takes 1 or more digits. If we ran this parser we would either end up with a `String` containing 1 or more digits or get a parse error. You may have noticed that despite having written no error handling code, our parser outputs detailed, helpful error messages when given bad input. If you haven't already, feel free to feed the example above some bad input and see what happens.

### Reading

Now parsing a string representation of an integer is nice, but really what we want is the value of the integer and not the original input string. To do this conversion we will use the `read` function. Notice that we rename the `read` function to `rd` with the type `String -> Integer`. This is necessary because `read` has a polymorphic type (click it in the example above to see). Because we don't have any types anywhere else, Haskell cannot figure out what type `read` is supposed to return. Therefor, we must state explicitly what the type of `read` is supposed to be, and we rename it to keep the clutter away from the rest of the definition for `integer`.

### Combining the Two

That leaves `<$>` as the final piece of the puzzle. If you opened up `ghci`, imported Parsec, and entered `:type many1 digit` you would get the following type back:

``` haskell
many1 digit :: Stream s m Char => ParsecT s u m [Char]
```

That's quite a mouthful, but the important thing to notice is that the resulting `String` (as `[Char]`) is wrapped up in a `ParsecT` with some other types. This means we won't be able to directly access the parsed `String`. Luckily, `ParsecT` is an instance of `Functor`, which means it implements `fmap`. This means we can use `fmap` (as `<$>` here) to lift the `read` function to work within the `ParsecT` functor.

## Parsing Integers With Leading Sign

Now, an integer parser that could only parse positive integers isn't very useful, so let's expand our parser to dealing with a leading sign:

``` active haskell
import Control.Monad
-- show
import Text.Parsec
import Control.Applicative hiding ((<|>))

integer = rd <$> (plus <|> minus <|> number)
    where rd     = read :: String -> Integer
          plus   = char '+' *> number
          minus  = (:) <$> char '-' <*> number
          number = many1 digit
-- /show
main = forever $ do putStrLn "Enter an integer: "
                    input <- getLine
                    parseTest integer input
```

We've added quite a bit here, so let's break it down piece by piece. We've made 3 new parsers that are used in the final `integer` parser. We use the `<|>` operator (from `Text.Parsec`, not `Control.Applicative`) to implement choice. We will first try to parse using the `plus` parser, trying `minus` next if it fails, and finally `number` if both of those fail. If all 3 fail, then the parser returns an error. Note how our error messages change with this new version of `integer` despite the fact that we still do no explicit error handling.

Let's next look at our 3 parsers, starting with the simplest and ending with the most complicated:

### number

`number` is simply our original integer parser, pulled out so that it can be referenced by the other parsers.

### plus

`plus` is the parser for an integer that begins with a plus sign (`+`). It starts with the parser `char '+'`, which parsers the character `+`. We then link that parser to our number parser with `*>`, from `Control.Applicative`. `*>` simply takes two values wrapped in an applicative functor (which `ParsecT` happens to be) and returns the second wrapped value. However, it still performs any actions the first applicative functor may do. This lets us perform the first parse, grab the `+` character, and then drop the parsed value and continue parsing after it with `number`. This is often useful in cases where we need to ensure something exists, but we don't care about its actual value. Since `read` will assume a positive integer when there's no sign, bringing the `+` along is unnecessary.

### minus

`minus` is the parser for an integer that begins with a minus sign (`-`). It's more complicated than `plus` since we can't simply ignore the minus sign like we can with the plus. Again we use the fact that `ParsecT` is an instance of `Applicative` to combine the two parsers of `char '-'` and `number`. Since we need to combine the two values inside the `ParsecT` values, we use the list constructor `(:)` since `String` is just a list of `Char`s and `char '-'` parses a single `Char` while `number` parses a `String`.

We use `<$>` to lift `(:)` to the functor level. We then use `<*>` to feed the second parser value in as the second argument to `(:)`. This lets us combine our two parsers while simultaneously combining their parsed values.

## Parsing Floats

Now that we have a basic understanding of parsing with Parsec, we can move on to parsing floats. This is quite a bit trickier, as floating point values have many different components. In addition to everything we did for integers, floats may also have a decimal component, an exponent component, or both. The exponent component leads with `e` but needs to be case insensitive, as well as potentially starting with a plus or minus sign.

To start, we'll simply take our integer parser above and change the type of `read` to give us a `Float` instead of an `Integer`.

``` active haskell
import Control.Monad
-- show
import Text.Parsec
import Control.Applicative hiding ((<|>))

float = rd <$> (plus <|> minus <|> number)
    where rd     = read :: String -> Float
          plus   = char '+' *> number
          minus  = (:) <$> char '-' <*> number
          number = many1 digit
-- /show
main = forever $ do putStrLn "Enter a float: "
                    input <- getLine
                    parseTest float input
```

Now, our parser will start getting a bit large for a single declaration, so let's move the smaller parsers to the top level:

``` active haskell
import Control.Monad
-- show
import Text.Parsec
import Control.Applicative hiding ((<|>))

number = many1 digit

plus = char '+' *> number

minus = (:) <$> char '-' <*> number

integer = plus <|> minus <|> number

float = rd <$> integer
    where rd = read :: String -> Float
-- /show
main = forever $ do putStrLn "Enter a float: "
                    input <- getLine
                    parseTest float input
```

Next, we will add the decimal component. Since this is optional, we'll need to use the parser combinator `option`. This function simply takes a default value and a parser. If the parser succeeds it will get the parsed value, otherwise it will use the default value.

``` active haskell
import Control.Monad
-- show
import Text.Parsec
import Control.Applicative hiding ((<|>))

number = many1 digit

plus = char '+' *> number

minus = (:) <$> char '-' <*> number

integer = plus <|> minus <|> number

float = fmap rd $ (++) <$> integer <*> decimal
    where rd      = read :: String -> Float
          decimal = option "" $ (:) <$> char '.' <*> number
-- /show
main = forever $ do putStrLn "Enter a float: "
                    input <- getLine
                    parseTest float input
```

We've added quite a bit of stuff, so let's break down the complete list of changes.

1.  We changed `<$>` to regular `fmap` in the `float` parser so we could use the `$` operator to avoid parentheses.
2.  We made a new parser named `decimal`. This is pretty much just our `minus` parser with the `-` character replaced with the `.` character. It also uses the `option` combinator to signify that it is optional. The `""` argument is the parsed result if the parser fails.
3.  We use `<$>` and `<*>` to combine our `integer` and `decimal` parsers much in the same way we combine `char '-'` and `number` in `minus`. We have to lift the string concatenation operator `(++)` instead of the list constructor `(:)` since in this case both `integer` and `decimal` parse `String`s instead of the first just parsing a single `Char`.

Our manually lifting the `(:)` and `(++)` operators to the functor level is pretty ugly, so let's make our own operators to do this for us:

``` active haskell
import Control.Monad
-- show
import Text.Parsec
import Control.Applicative hiding ((<|>))

(<++>) a b = (++) <$> a <*> b
(<:>) a b = (:) <$> a <*> b

number = many1 digit

plus = char '+' *> number

minus = char '-' <:> number

integer = plus <|> minus <|> number

float = fmap rd $ integer <++> decimal
    where rd      = read :: String -> Float
          decimal = option "" $ char '.' <:> number
-- /show
main = forever $ do putStrLn "Enter a float: "
                    input <- getLine
                    parseTest float input
```

Much better. Note how we kept to the convention of wrapping the original operator names with angle brackets to signify that they are simply their base operator lifted to work with applicative functors. If you hadn't already noticed this pattern, compare the types of `$` and `<$>`.

These new operators will help us further keep things clean as we add the final component to our float parser: exponents. The only difficult thing about exponents is that they are case insensitive: "`1e3`" is just as valid as "`1E3`". We could use the `<|>` operator to handle these two cases separately, but since it's just a single character difference it'd be simpler to use the parser combinator `oneOf`. This combinator takes a list of characters as a `String` and attempts to parse each of them in turn.

``` active haskell
import Control.Monad
-- show
import Text.Parsec
import Control.Applicative hiding ((<|>))

(<++>) a b = (++) <$> a <*> b
(<:>) a b = (:) <$> a <*> b

number = many1 digit

plus = char '+' *> number

minus = char '-' <:> number

integer = plus <|> minus <|> number

float = fmap rd $ integer <++> decimal <++> exponent
    where rd       = read :: String -> Float
          decimal  = option "" $ char '.' <:> number
          exponent = option "" $ oneOf "eE" <:> integer
-- /show
main = forever $ do putStrLn "Enter a float: "
                    input <- getLine
                    parseTest float input
```

And with that, we have created a parser for floating point values. It's short, it's easy to read, it provides good error messages, and it's fast. All thanks to Haskell and the Parsec library.

## Running the Parser

We glossed over how to actually run the parser so far. If you checked the full source of the examples you would have seen that we use the `parseTest` function. This function simply runs the given parser and prints out either the parsed result or the error if the parser failed. This is the ideal tool when testing parsers with `ghci`, as you can simply feed it test strings to make sure everything is working correctly.

The functions for running parsers and getting their output all ultimately result in the same return value: "`Either ParseError a`". `ParseError` is a type in the Parsec library for storing information about parse errors. You can either look up its specifications on Hackage or use `show` to generate the standard error messages as a `String`. `a` is the result type of the parser. For our `float` parser, that type would be `Float`.

There are basically two functions to execute parsers: `parse` and `parseFromFile`.

``` haskell
parse :: Stream s Identity t => Parsec s () a -> SourceName -> s -> Either ParseError a
```

This big type signature is actually fairly simple: it takes a parser, a source name (a `String` used as a part of error messages), a `Stream`, and either results in a `ParseError` or the result type of the parser. A `Stream` is simply a data type suitable for being used as a data source. `ByteString` and `Text` (both strict and lazy) have instances of `Stream`, as well as regular lists and `Strings`.

``` haskell
parseFromFile :: Parser a -> String -> IO (Either ParseError a)
```

This function is pretty simple, taking a parser, a `String` containing the path to the file to parse, and produces either a `ParseError` or the result type of the parser. This is wrapped up in the `IO` monad, as we need to use `IO` to open and read the file.

The trick with `parseFromFile` is the fact that there are 5 different versions of it, and none of them will be in scope if you just import `Text.Parsec`. These different versions all use different backing data structures for reading in the data, and can have different performance characteristics. To access them, import one of the following modules:

- `Text.Parsec.ByteString`
- `Text.Parsec.ByteString.Lazy`
- `Text.Parsec.Text`
- `Text.Parsec.Text.Lazy`
- `Text.Parsec.String`

As a general rule of thumb, use either `ByteString` or `Text` unless you definitely need laziness, in which case use the lazy version of one of the aforementioned. `String` should be avoided as it is the least performant.

## Monomorphism Restriction and Flexible Contexts

It's worth taking a moment to talk about two language extensions: the monomorphism restriction and flexible contexts. Normally it's considered good practice in Haskell to have explicit type declarations for any top-level terms. This is problematic when using Parsec for two reasons: the types are a bit complicated and possibly won't even compile. To illustrate this, if we loaded our final example into `ghci` and used `:type float` to get `float`'s generated type, we would get this:

``` haskell
float :: ParsecT String u Data.Functor.Identity.Identity Float
```

Which seems fine, but here's an exercise: below is the complete final example with nothing hidden. Replace the last line in `main` with "`putStrLn input`" so that it no longer references `float` and try to execute it:

``` active haskell
import Control.Monad
import Text.Parsec
import Control.Applicative hiding ((<|>))

(<++>) a b = (++) <$> a <*> b
(<:>) a b = (:) <$> a <*> b

number = many1 digit

plus = char '+' *> number

minus = char '-' <:> number

integer = plus <|> minus <|> number

float = fmap rd $ integer <++> decimal <++> exponent
    where rd       = read :: String -> Float
          decimal  = option "" $ char '.' <:> number
          exponent = option "" $ oneOf "eE" <:> integer

main = forever $ do putStrLn "Enter a float: "
                    input <- getLine
                    parseTest float input
```

You should get an error message starting with "`No instance for (Stream s0 m0 Char) arising from a use of 'digit'`". The problem is that the reference to `float` is affecting `float`'s type, and subsequently the types of everything else. So why does it break when that reference is removed? The answer is the monomorphism restriction. Without getting into the details, the monomorphism restriction can result in type checking failing for some classes of correct Haskell programs. In this case, the reference to `float` in `main` puts a restriction on `float`'s type that shouldn't be there, causing `float` to have a type that is too specific and that happens to avoid the monomorphism restriction. `float`'s actual type should be this:

``` haskell
float :: Stream s m Char => ParsecT s u m Float
```

But if you explicitly declare `float` to have this type you will still get an error! Luckily, we can get `float` to have the correct type if we simply keep the explicit type declarations off and disable the monomorphism restriction by putting this line at the top of our source file:

``` haskell
{-# LANGUAGE NoMonomorphismRestriction #-}
```

Putting this at the top will allow the above example to compile whether or not `main` references `float`. `float` will also have the correct type regardless. But how do we explicitly declare the type of `float`? The problem is the `Stream` constraint has a specific type in `Char` instead of being all type variables. We can get around this problem with another language pragma: flexible contexts.

``` haskell
{-# LANGUAGE FlexibleContexts #-}
```

This will allow us to explicitly define the type of `float` with no issues. In fact, if we declare the types of all our top-level values then we can forgo disabling the monomorphism restriction.

What exactly these extensions do is complex and subtle, but luckily we can avoid worrying about that if we keep to these three simple rules:

- If you have top-level parsers without type declarations, use NoMonomorphismRestriction.
- If you have top-level parsers with type declarations, use FlexibleContexts.
- If you have a mixture of both, use both.

## Conclusions

Thanks to the power of Haskell and the Parsec library, we were able to make a parser for floating point numbers in only 10 effective lines of code. The code is clean and easy to understand, and with no effort on our part it provides good error messages when parsing fails. So the next time you find yourself writing another regular expression, just consider how nice it would be to instead write a parser with Parsec.
# Monadic Parser

In this post I give two implementations of a parser presented in the paper _Monads for functional programming_ by Philip Wadler. The first implementation is essentially the one presented in the paper. The second one uses Haskell _do notation_, so it is somewhat cleaner and easier to follow.

My main motivation was to confirm there is a bug in the paper: the implementation of `term'` should use biased choice instead of choice.

## First implementation

In the first implementation the monadic type `Parser a` is a synonym for the type `String -> [(a, String)]`.

``` active haskell

import Data.Char

type Parser a = String -> [(a, String)]

unit :: a -> Parser a
unit t = \s -> [(t, s)]

bind :: Parser a -> (a -> Parser b) -> Parser b
m `bind` k = \s -> [(x, y) | (u, v) <- m s, (x, y) <- k u v]

data Term = Con Int | Div Term Term
            deriving (Show)

parseTerm :: String -> Term
parseTerm = fst . head . term

term :: Parser Term
term = factor `bind` term'

term' :: Term -> Parser Term
term' t = (lit '/' `bind` \_ -> factor `bind` \u -> term' (Div t u)) `bchoice` unit t

factor :: Parser Term
factor = (number `bind` \n -> unit (Con n)) `choice`
            (lit '(' `bind` \_ -> term `bind` \t -> lit ')' `bind` \_ -> unit t)

zero :: Parser a
zero = \s -> []

choice :: Parser a -> Parser a -> Parser a
m `choice` n = \s -> m s ++ n s

-- Biased choice
bchoice :: Parser a -> Parser a -> Parser a
m `bchoice` n = \s -> if null (m s) then n s else m s

filt :: Parser a -> (a -> Bool) -> Parser a
m `filt` p = m `bind` \t -> if p t then unit t else zero

item :: Parser Char
item [] = []
item (a : x) = [(a, x)]

digit :: Parser Char
digit = item `filt` isDigit

lit :: Char -> Parser Char
lit c = item `filt` \c' -> c == c'

reiterate :: Parser a -> Parser [a]
reiterate m = (m `bind` \t -> reiterate m `bind` \ts -> unit (t : ts)) `bchoice` unit []

number :: Parser Int
number = reiterate digit `bind` \ds -> unit (read ds :: Int)

main = print $ parseTerm "1972/2/23"

```
## Second Implementation

The second implementation defines a new type for the parser, instead of just a synonym. The new type can be promoted to an instance of the Monad typeclass, which enables the usage of Haskell _do notation_. That results in code that is somewhat cleaner and easier to follow.

``` active haskell

import Data.Char

newtype Parser a = Parser { parse :: String -> [(a, String)] }

instance Monad Parser where
    return t = Parser $ \s -> [(t, s)]
    m >>= k  = Parser $ \s -> [(x, y) | (u, v) <- parse m s, (x, y) <- parse (k u) v]

data Term = Con Int | Div Term Term
            deriving (Show)

parseTerm :: String -> Term
parseTerm = fst . head . parse term

term :: Parser Term
term = do
    t <- factor
    term' t

term' :: Term -> Parser Term
term' t = divFactor `bchoice` return t
    where divFactor = do
            lit '/'
            u <- factor
            term' $ Div t u

factor :: Parser Term
factor = numTerm `choice` parenTerm
    where numTerm = do
            n <- number
            return $ Con n
          parenTerm = do
            lit '('
            t <- term
            lit ')'
            return t

zero :: Parser a
zero = Parser $ \s -> []

choice :: Parser a -> Parser a -> Parser a
m `choice` n = Parser $ \s -> parse m s ++ parse n s

-- Biased choice
bchoice :: Parser a -> Parser a -> Parser a
m `bchoice` n = Parser $ \s -> if null (parse m s) then parse n s else parse m s

filt :: Parser a -> (a -> Bool) -> Parser a
m `filt` p = do
    t <- m
    if p t then return t else zero

item :: Parser Char
item = Parser item'
    where item' [] = []
          item' (a : x) = [(a, x)]

digit :: Parser Char
digit = item `filt` isDigit

lit :: Char -> Parser Char
lit c = item `filt` \c' -> c == c'

reiterate :: Parser a -> Parser [a]
reiterate m = multiple `bchoice` return []
    where multiple = do
            t <- m
            ts <- reiterate m
            return $ t : ts

number :: Parser Int
number = do
    ds <- reiterate digit
    return (read ds :: Int)

main = print $ parseTerm "1972/2/23"

```
# Quasiquotation 101


In the [previous episode](https://www.fpcomplete.com/user/marcin/template-haskell-101) we have looked at Template Haskell, and I promised to follow up with quasiquotation. So here we go.

Note: the examples are based on http://www.haskell.org/haskellwiki/Quasiquotation
but are a result of my work with them, rather then verbatim quotes.

Apart from Template Haskell, I also assume at least basic familiarity with Parsec parsing.

## Parsing Expressions

Let's start with a simple data type and parser for arithmetic expressions


```active haskell
{-# START_FILE Expr.hs #-}
{-# LANGUAGE DeriveDataTypeable #-}
module Expr(Exp(..),pExp,parse) where
import Text.ParserCombinators.Parsec
import Data.Char(digitToInt)
import Data.Typeable
import Data.Data
import Control.Applicative((<$>),(*>),(<*))

--show
data Exp = EInt Int 
         | EAdd Exp Exp
         | ESub Exp Exp
         | EMul Exp Exp
         | EDiv Exp Exp
           deriving(Show,Typeable,Data)

pExp :: Parser Exp
--/show
pExp = pTerm `chainl1` spaced addop
addop :: Parser (Exp->Exp->Exp)
addop   =   fmap (const EAdd) (char '+')
          <|> fmap (const ESub) (char '-')
            
pTerm = spaced pNum `chainl1` spaced mulop
mulop :: Parser (Exp->Exp->Exp)
mulop = pOps [EMul,EDiv] ['*','/']

pNum :: Parser Exp
pNum = fmap (EInt . digitToInt) digit

pOps :: [a] -> [Char] -> Parser a
pOps fs cs = foldr1 (<|>) $ map pOp $ zip fs cs

whenP :: a -> Parser b -> Parser a
whenP = fmap . const

spaced :: Parser a -> Parser a
spaced p = spaces *> p <* spaces

pOp :: (a,Char) -> Parser a
pOp (f,s) = f `whenP` char s

parseExp :: Monad m => (String, Int, Int) -> String -> m Exp
parseExp (file, line, col) s =
    case runParser p () "" s of
      Left err  -> fail $ show err
      Right e   -> return e
  where
    p = do updatePosition file line col
           spaces
           e <- pExp
           spaces
           eof
           return e

updatePosition file line col = do
   pos <- getPosition
   setPosition $
     (flip setSourceName) file $
     (flip setSourceLine) line $
     (flip setSourceColumn) col $
     pos

{-# START_FILE TestExpr.hs #-}
--show
import Expr
test1 = parse pExp "test1" "1 - 2 - 3 * 4 "
main = print test1
--/show
```

Now let's say we need some expresion trees in our program. For this kind of expressions we could (almost) get by  with `class Num` hack:

``` haskell
instance Num Exp where
    fromInteger = EInt . fromInteger
    (+) = EAdd
    (*) = EMul
    (-) = ESub
  
testExp :: Exp
testExp = (2 + 2) * 3
```

...but it is neither extensible nor, in fact, nice.

Of course as soon as we have a parser ready we could use it to build expressions

``` haskell
testExp = parse pExp "testExp" "1+2*3"
```
...but then potential errors in the expression texts remain undetected until runtime, and also this is not flexible enough: what if we wanted a simplifier for expressions, along the lines of

``` haskell
simpl :: Exp -> Exp
simpl (EAdd (EInt 0) x) = x
```

what if we could instead write

``` haskell
simpl :: Exp -> Exp
simpl (0 + x) = x
```

turns out with quasiquotation we can do just that (albeit with a slightly different syntax), so to whet your appetite:

```active haskell
{-# START_FILE Simpl2.hs #-}
{-# LANGUAGE  QuasiQuotes #-}
import ExprQuote2
import Expr2
--show
simpl :: Exp -> Exp
simpl [expr|0 + $x|] = x

main = print $ simpl [expr|0+2|]
--/show
{-# START_FILE ExprQuote2.hs #-}
-- Based on http://www.haskell.org/haskellwiki/Quasiquotation
module ExprQuote2 where

import Data.Generics
import qualified Language.Haskell.TH as TH
import Language.Haskell.TH.Quote

import Expr2
--show
expr  :: QuasiQuoter
expr  =  QuasiQuoter 
  { quoteExp = quoteExprExp
  , quotePat = quoteExprPat
  , quoteDec = undefined
  , quoteType = undefined
  }
--/show
quoteExprExp s = do
  pos <- getPosition
  exp <- parseExp pos s
  dataToExpQ (const Nothing) exp

-- dataToExpQ :: Data a => (forall b. Data b => b -> Maybe (Q Exp)) -> a -> Q Exp

quoteExprPat s = do
  pos <- getPosition
  exp <- parseExp pos s
  dataToPatQ (const Nothing `extQ` antiExprPat) exp

antiExprPat :: Exp -> Maybe (TH.Q TH.Pat)
antiExprPat (EMetaVar v) = Just $ TH.varP (TH.mkName v)
antiExprPat _ = Nothing

getPosition = fmap transPos TH.location where
  transPos loc = (TH.loc_filename loc,
                  fst (TH.loc_start loc),
                  snd (TH.loc_start loc))

{-# START_FILE Expr2.hs #-}
{-# LANGUAGE DeriveDataTypeable #-}
module Expr2 where
import Text.ParserCombinators.Parsec
import Data.Char(digitToInt)
import Data.Typeable
import Data.Data
import Control.Applicative((<$>),(*>),(<*))

-- show
data Exp = EInt Int 
         | EAdd Exp Exp
         | ESub Exp Exp
         | EMul Exp Exp
         | EDiv Exp Exp
         | EMetaVar String
           deriving(Show,Typeable,Data)
                   
pExp :: Parser Exp
-- /show
pExp = pTerm `chainl1` spaced addop

pTerm = spaced pFactor `chainl1` spaced mulop

pFactor = pNum <|> pMetaVar

addop :: Parser (Exp->Exp->Exp)
addop = fmap (const EAdd) (char '+')
          <|> fmap (const ESub) (char '-')
            
mulop :: Parser (Exp->Exp->Exp)
mulop = pOps [EMul,EDiv] ['*','/']

pNum :: Parser Exp
pNum = fmap (EInt . digitToInt) digit

pMetaVar = char '$' >> EMetaVar <$> ident

small   = lower <|> char '_'
large   = upper
idchar  = small <|> large <|> digit <|> char '\''
ident = do { c <- small; cs <- many idchar; return (c:cs) }

pOps :: [a] -> [Char] -> Parser a
pOps fs cs = foldr1 (<|>) $ map pOp $ zip fs cs

whenP :: a -> Parser b -> Parser a
whenP = fmap . const

spaced :: Parser a -> Parser a
spaced p = spaces *> p <* spaces

pOp :: (a,Char) -> Parser a
pOp (f,s) = f `whenP` char s

test1 = parse pExp "test1" "1 - 2 - 3 * 4 "
test2 = parse pExp "test2" "$x - $y*$z"

parseExp :: Monad m => (String, Int, Int) -> String -> m Exp
parseExp (file, line, col) s =
    case runParser p () "" s of
      Left err  -> fail $ show err
      Right e   -> return e
  where
    p = do updatePosition file line col
           spaces
           e <- pExp
           spaces
           eof
           return e

updatePosition file line col = do
   pos <- getPosition
   setPosition $
     (flip setSourceName) file $
     (flip setSourceLine) line $
     (flip setSourceColumn) col $
     pos

```

as we can see, a QuasiQuoter consists of quasiquoters for expressions, patterns, declarations and types (the last two remain undefined in our example). Let us start with the (perhaps simplest) quasiquoter for expressions:


``` haskell
quoteExprExp s = do
  pos <- getPosition
  exp <- parseExp pos s
  dataToExpQ (const Nothing) exp
```

## Quasiquoting Expressions

There are three steps:

* record the current position in Haskell file (for parse error reporting);
* parse the expression into our abstract syntax;
* convert our abstract syntax to its Template Haskell representation.

The first step is accomplished using [Language.Haskell.TH.location](http://hackage.haskell.org/packages/archive/template-haskell/2.8.0.0/doc/html/Language-Haskell-TH.html#v:location) and converting it to something usable by Parsec:

``` haskell
getPosition = fmap transPos TH.location where
  transPos loc = (TH.loc_filename loc,
                  fst (TH.loc_start loc),
                  snd (TH.loc_start loc))
```

Parsing is done using our expression parser introduced at the beginning - nothing exciting here, but then comes the last part: generating Template Haskell, which seems like quite a task. Luckily we can save us some work using facilities for generic programming provided by [Data.Data](http://hackage.haskell.org/packages/archive/base/4.6.0.1/doc/html/Data-Data.html) combined with an almost magical Template Haskell function [dataToExpQ](http://hackage.haskell.org/packages/archive/template-haskell/latest/doc/html/Language-Haskell-TH-Quote.html#v:dataToExpQ).

So far, we are halfway through to our goal: we can use the quasiquoter on the right hand side of function definitions:

``` haskell
testExp :: Exp
testExp = [expr|1+2*3|]
```

## Quasiquoting patterns

To be able to write thins like

``` haskell
simpl [expr|0 + $x|] = x
```

we need to write a quasiquoter for patterns. However, let us start with something less ambitious -  a quasiquoter for constant patterns, allowing us to write

``` haskell
{-# START_FILE TestExprQuote1.hs #-}
{-# LANGUAGE  QuasiQuotes #-}
import ExprQuote1
import Expr

-- show
testExp :: Exp
testExp = [expr|1+2*3|]

f1 :: Exp -> String
f1 [expr| 1 + 2*3 |] = "Bingo!"
f1 _ = "Sorry, no bonus" 

main = putStrLn $ f1 testExp
-- /show
```

This can be done similarly to the quasiquoter for expressions:
* record the current position in Haskell file (for parse error reporting);
* parse the expression into our abstract syntax;
* convert our abstract syntax to its Template Haskell representation.

Only the last part needs to gbe slightly different - this time we need to construct Template Haskell pattern representation:

``` haskell
quoteExprPat :: String -> TH.Q TH.Pat
quoteExprPat s = do
  pos <- getPosition
  exp <- parseExp pos s
  dataToPatQ (const Nothing) exp

```

The functions `quoteExprExp` and `quoteExprPat` differ in two respects:

* use `dataToPatQ` instead of `dataToExpQ`
* the result type is different (obviously)

## Antiquotation

The quasiquotation mechanism we have seen so far allows us to translate domain-specific code into Haskell and `inject` it into our program. Antiquotation, as the name suggests goes in the opposite direction: embeds Haskell entities (e.g. variables) in our DSL.

This sounds complicated, but isn't really. Think HTML templates:

``` html
<html>
    <head>
        <title>#{pageTitle} 
```

The meaning is hopefully obvious - the value of program variable `pageTitle` should be embedded in the indicated place. In our expression language we might want to write

```
twice :: Exp -> Exp
twice e = [expr| $e + $e |]

testTwice = twice [expr| 3 * 3|]
```

This is nothing revolutionary. Haskell however, uses variables not only in expressions, but also in patterns, and here the story becomes a little interesting.


Recall the pattern quasiquoter:

``` haskell
quoteExprPat :: String -> TH.Q TH.Pat
quoteExprPat s = do
  pos <- getPosition
  exp <- parseExp pos s
  dataToPatQ (const Nothing) exp

```

You might have wondered about the `const Nothing` previously, and that is exactly the place we may add own extensions to the standard `Data` to `Pat` translation.

Let us extend our expression syntax and parser with metavariables (variables from the metalanguage):

```haskell
{-# LANGUAGE DeriveDataTypeable #-}
module Expr2 where
import Text.ParserCombinators.Parsec
import Data.Char(digitToInt)
import Data.Typeable
import Data.Data
import Control.Applicative((<$>),(*>),(<*))

-- show
data Exp = EInt Int 
         | EAdd Exp Exp
         | ESub Exp Exp
         | EMul Exp Exp
         | EDiv Exp Exp
         | EMetaVar String
           deriving(Show,Typeable,Data)
                   
pExp :: Parser Exp
pExp = pTerm `chainl1` spaced addop

pTerm = spaced pFactor `chainl1` spaced mulop
pFactor = pNum <|> pMetaVar

pMetaVar = char '$' >> EMetaVar <$> ident
-- /show

small   = lower <|> char '_'
large   = upper
idchar  = small <|> large <|> digit <|> char '\''
ident = do { c <- small; cs <- many idchar; return (c:cs) }

addop :: Parser (Exp->Exp->Exp)
addop = fmap (const EAdd) (char '+')
          <|> fmap (const ESub) (char '-')
            
mulop :: Parser (Exp->Exp->Exp)
mulop = pOps [EMul,EDiv] ['*','/']

pNum :: Parser Exp
pNum = fmap (EInt . digitToInt) digit


pOps :: [a] -> [Char] -> Parser a
pOps fs cs = foldr1 (<|>) $ map pOp $ zip fs cs

whenP :: a -> Parser b -> Parser a
whenP = fmap . const

spaced :: Parser a -> Parser a
spaced p = spaces *> p <* spaces

pOp :: (a,Char) -> Parser a
pOp (f,s) = f `whenP` char s

test1 = parse pExp "test1" "1 - 2 - 3 * 4 "
test2 = parse pExp "test2" "$x - $y*$z"

parseExp :: Monad m => (String, Int, Int) -> String -> m Exp
parseExp (file, line, col) s =
    case runParser p () "" s of
      Left err  -> fail $ show err
      Right e   -> return e
  where
    p = do updatePosition file line col
           spaces
           e <- pExp
           spaces
           eof
           return e

updatePosition file line col = do
   pos <- getPosition
   setPosition $
     (flip setSourceName) file $
     (flip setSourceLine) line $
     (flip setSourceColumn) col $
     pos

```

The antiquoter is defined as an extension for the `dataToPatQ`:

``` haskell
antiExprPat :: Exp -> Maybe (TH.Q TH.Pat)
antiExprPat (EMetaVar v) = Just $ TH.varP (TH.mkName v)
antiExprPat _ = Nothing

```

* metavariables are translated to `Just` TH variables
* for all the other cases we say `Nothing` - allowing `dataToPatQ` use its default rules

And that's it! Now we can write

``` haskell
eval [expr| $a + $b|] = eval a + eval b
eval [expr| $a * $b|] = eval a * eval b
eval (EInt n) = n
```

## Exercises

* Extend the expression simplifier with more rules.

* Add antiquotation to `quoteExprExp`

* Extend the expression quasiquoter to handle metavariables for
  numeric constants, allowing to implement simplification rules like

```
simpl [expr|$int:n$ + $int:m$|] = [expr| $int:m+n$ |]
```

(you are welcome to invent your own syntax in place of `$int: ... $`)
# 3. Tokenizer

**Ex 1**.
Write a function that takes an `Operator` and returns one of the characters, `'+'`, `'-'`, `'*'`, or `'/'`.
``` active haskell
data Operator = Plus | Minus | Times | Div

opToChar :: Operator -> Char
opToChar = undefined

main = print $ opToChar Plus
```
@@@ Show solution
``` active haskell
data Operator = Plus | Minus | Times | Div

opToChar :: Operator -> Char
opToChar Plus  = '+'
opToChar Minus = '-'
opToChar Times = '*'
opToChar Div   = '/'

main = print $ opToChar Plus
```
@@@

## Lists

**Ex 1**. Implement `norm` that takes a list of `Double`s and returns the square root (`sqrt`) of the sum of squares of its elements.

``` active haskell
norm :: [Double] -> Double
norm lst = undefined

main = print (norm [1.1, 2.2, 3.3])
```

@@@ Show solution
``` active haskell
norm :: [Double] -> Double
norm lst = sqrt (squares lst)

squares :: [Double] -> Double
squares [] = 0.0
squares (x : xs) = x * x + squares xs

main = print (norm [1.1, 2.2, 3.3])
```
@@@

**Ex 2**. Implement the function `decimate` that skips every other element of a list. 
``` active haskell
decimate :: [a] -> [a]
decimate = undefined

-- should print [1, 3, 5]
main = print (decimate [1, 2, 3, 4, 5])
```

@@@ Show solution
``` active haskell
decimate :: [a] -> [a]
decimate (a:_:rest) = a : decimate rest
decimate (a:_) = [a]
decimate _ = []

main = print (decimate [1, 2, 3, 4, 5])
```
@@@

**Ex 3**. Implement a function that takes a pair of lists and returns a list of pairs. For instance `([1, 2, 3, 4], [1, 4, 9])` should produce `[(1, 1), (2, 4), (3, 9)]`. Notice that the longer of the two lists is truncated if necessary. Use nested patterns.

``` active haskell
zipLst :: ([a], [b]) -> [(a, b)]
zipLst = undefined

main = print $ zipLst ([1, 2, 3, 4], "Hello")
```

@@@ Show solution
``` active haskell
zipLst :: ([a], [b]) -> [(a, b)]
zipLst ((x : xs), (y: ys)) = (x, y) : zipLst (xs, ys)
zipLst (_, _) = []

main = print $ zipLst ([1, 2, 3, 4], "Hello")
```

Incidentally, there is a two-argument function `zip` in the Prelude that does the same thing:

``` active haskell
main = print $ zip [1, 2, 3, 4] "Hello"
```

@@@

## Single-Character Tokenizer

``` active haskell
import Data.Char

data Operator = Plus | Minus | Times | Div
    deriving (Show, Eq)

data Token = TokOp Operator
           | TokIdent String
           | TokNum Int
    deriving (Show, Eq)

operator :: Char -> Operator
operator c | c == '+' = Plus
           | c == '-' = Minus
           | c == '*' = Times
           | c == '/' = Div

tokenize :: String -> [Token]
tokenize [] = []
tokenize (c : cs) 
    | elem c "+-*/" = TokOp (operator c) : tokenize cs
    | isDigit c  = TokNum (digitToInt c) : tokenize cs
    | isAlpha c  = TokIdent [c]          : tokenize cs
    | isSpace c  = tokenize cs
    | otherwise  = error $ "Cannot tokenize " ++ [c]

main = print $ tokenize " 1 + 4 / x "
```

## Lambdas

``` haskell
\x -> expression
```

**Ex 1.** Implement function `squares` that takes a list of integers and returns the list of their squares. Use higher order functions and lambdas.

``` active haskell
squares :: [Int] -> [Int]
squares = undefined

main = print $ squares [1..10]
```

@@@ Show solution
``` active haskell
squares :: [Int] -> [Int]
squares = map (\x -> x * x)

main = print $ squares [1..10]
```
@@@

## Folding

``` haskell
foldl :: (a -> b -> a) -> a -> [b] -> a
```

**Ex 1.** Use `foldl` to calculate the sum of squares given a list of doubles. 
``` active haskell
squares :: [Int] -> Int
squares = foldl (\acc x -> ???) 0

main = print $ squares [3, 4, 5]
```

@@@ Show solution
``` active haskell
squares :: [Int] -> Int
squares = foldl (\acc x -> acc + x * x) 0

main = print $ squares [3, 4, 5]
```
@@@


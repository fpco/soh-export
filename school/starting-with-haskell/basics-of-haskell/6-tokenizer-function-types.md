# 6. Tokenizer: Function Types

We haven't gotten too far in our implementation of the symbolic calculator yet, but we've already learned a lot. We know how to work with list, and `String`s in particular, and we have defined the `Token` data type. It's time to start implementing the tokenizer function:

``` haskell
tokenize :: String -> [Token]
```

## Categorizing Characters

An imperative programmer would implement the tokenizer as a loop for processing consecutive characters in the string. 

An object-oriented programmer would implement a tokenizer as a stateful object with a getter that returns the current token and an incrementer that moves to the next token, consuming one or more characters from the string in the process. 

A functional programmer looks at the tokenizer as a function that picks the first character of the string, categorizes it, turns it into a token, and then tokenizes the rest of the string. (We'll tackle multi-character tokens in the next tutorial.) 

The application of the tokenizer to the rest of the string is the standard recursive step in the algorithm.

Here's a very simple tokenizer that recognizes digits and non-digit characters:

``` active haskell
import Data.Char

data Token = Digit | Alpha
    deriving (Show, Eq)

tokenize :: String -> [Token]
tokenize (c : rest) = 
    if isDigit c
    then Digit : tokenize rest
    else Alpha : tokenize rest
tokenize [] = []

main = print $ tokenize "passwd123"
```

I used the library function `isDigit`. This function is not defined in the Prelude, it's defined in a different library called `Data.Char`. I had to import it explicitly using the `import` statement at the top of the file -- somewhat analogous to C's or Java's `#include` statement. 

`Data.Char` defines several useful functions, like `isSpace`, `isAlpha`, `isAlphaNum`, and many more. You can look them up in Haskell's [Hoogle database](http://hackage.haskell.org/package/base-4.6.0.1/docs/Data-Char.html).

We could have easily implemented `isDigit` from scratch using the Prelude function `elem`, which tests whether its first argument is an element of the second argument (which is a list):

``` active haskell
isDigit :: Char -> Bool
isDigit c = elem c "0123456789"

main = print $ isDigit '3'
```
Here we are testing whether `c` is an element of the list of characters (string) "0123456789".

We could have also implemented the function `elem` from scratch, except that, up to now, I've been avoiding functions that require more than one argument. That's because I wanted to defer the explanation of currying until you get comfortable with single-argument functions.

## Currying

Defining a multi-argument function is easy -- it's the type signature that requires some getting used to. 

So, ignoring type signatures for a moment, here's the recursive implementation of `isElem`:

``` active haskell
isElem c (d : rest) = if c == d 
                      then True 
                      else isElem c rest
isElem _ [] = False

main = do
    print $ isElem '3' "abc"
    print $ isElem '3' "123"
```
Nothing surprising here. You just specify more than one argument, and you can do pattern matching on each of them. 

The fun part is that Haskell allows you to call a function using fewer arguments than there are formal parameters in its definition. In our example, it's okay to call `isElem` with just one argument, say, character `'3'`. What you get back is not a `Bool` but something that expects one more argument, a list, to produce a `Bool`. In other words you get a function `[Char]->Bool`. 

``` active haskell
isElem c (d : rest) = if c == d 
                      then True 
                      else isElem c rest
isElem _ [] = False
-- show
is3elem :: [Char] -> Bool
is3elem = isElem '3'

main = print $ is3elem "123"
```

Let me repeat this, because it's very important: we have curried the two-argument function `isElem` by providing the first argument, `'3'`. The result is a function that expects a list of `Char` (the second argument to `isElem`). We are storing this curried function in the variable `is3elem`. We can then call `is3elem` with a list of `Char` and get back a `Bool`. 

By the way, we could have defined `is3elem` using standard function definition syntax:

``` active haskell
isElem c (d : rest) = if c == d 
                      then True 
                      else isElem c rest
isElem _ [] = False
-- show
is3elem' :: [Char] -> Bool
is3elem' str = isElem '3' str

main = print $ is3elem' "123"
```

It almost looks like the first definition was obtained by "dividing" both sides of the second definition by `str`. After such simplification, the only trace of the `[Char]` argument is in the signature of `is3elem`. So if you see a function definition that is missing some arguments that are specified in its signature, you're looking at a curried definition. You'll see a lot of code written in this style, which has its own name: *point-free notation*. We'll talk more about it in the future.

Back to `isElem`: What's its type signature? By our reasoning, we can look at it as a function that takes a `Char` and returns a function `([Char]->Bool)`. Being able to return a function from a function is one of the perks of "functions being first-class citizens in Haskell." (The others are storing functions in variables, which we've already seen, and passing functions as arguments to other functions, which we'll see soon.)

Indeed, this is a valid signature of the funtion `isElem`:

``` haskell
isElem :: Char -> ([Char] -> Bool)
```
However, since the arrow `->` is right associative, the parentheses are not necessary and are usually omitted, as in:

``` haskell
isElem :: Char -> [Char] -> Bool
```

One more observation: `isElem`, as well as `elem`, will work not only for `Char`, but for any type that supports equality comparison. In particular, since we defined `Token` as `deriving Eq`, it supports equality comparison and can be used with `elem`. 

``` active haskell
data Token = Digit | Alpha
    deriving (Show, Eq)

main = print $ elem Digit [Digit, Alpha]
```
We'll come back to this when we study type classes.

To summarize: multi-argument functions can always be curried, and this is reflected in the way their type signatures are written. For instance, the signature:

``` haskell
f :: a -> b -> c -> d
```
tells us that `f` is a function of three arguments of types `a`, `b`, and `c`, returning `d`. You can also treat it as a function of two arguments `a` and `b` returning a function `(c ->d)`. Or a one-argument function taking `a` returning a two argument function `(b -> c -> d)`.

Currying is extremely useful and it's a pity that most other languages don't support it out of the box. Multi-argument functions in those languages are written and behave as if they were always taking tuples: In Haskell, the syntax `f (x, y, z)` would be interpreted as a function taking a tuple of three elements. In fact such function are sometimes called *uncurried*.

Parentheses and commas in the function syntax impede currying. Scala has special syntax for curriable functions: multiple pairs of parentheses. But you have to anticipate the need for currying when defining the function -- the default doesn't support it.

## Tokenizing Operators: Guards

The tokenizer has to recognize operators, identifiers, and numbers. Let's start with operators. We could categorize them using a bunch of nested if/then/else clauses, but that would be awkward. Let's use a different mechanism: guards. Just like you can split a function definition by patterns, you can further specialize it based on more general predicates (Boolean expressions). Let's define a function `operator` that converts a character to `Operator`:

``` active haskell
data Operator = Plus | Minus | Times | Div
    deriving (Show, Eq)

operator :: Char -> Operator
operator c | c == '+' = Plus
           | c == '-' = Minus
           | c == '*' = Times
           | c == '/' = Div

main = print $ operator '*'
```
There are four bodies of the function `operator` corresponding to four different guards (a body of a function starts after the equal sign). For example, the guard `c == '+'` corresponds to the body `Plus`, etc. Guards are placed after vertical bars. They are tested in order of appearance.

What happens when no guard is satisfied? You can try it by calling `operator` with the "wrong" character. You'll get a runtime error `PatternMatchFail` and the program will abort. This might be good enough if all you need is a bona fide assertion. In general, these kinds of non-exhaustive patterns are to be avoided. You can always end your list of guards with `otherwise` (which is syntactic sugar for `True`) and provide a default body. You'll see an example of this later.

## Single Character Tokenizer

Before we implement full blown tokenizers for numbers and identifiers, let's first tackle a simplified problem. Let's restrict numbers to single digits, and identifiers to single characters. This way we'll only need to process one character at a time and we can use simple recursion. Here's our recursive skeleton:

``` haskell
tokenize :: String -> [Token]
tokenize [] = []
tokenize (c : cs) = ... : tokenize cs
```

We'll have to categorize the current character, convert it to a token, and then tokenize the rest of the string (here I'm pattern matching the string as a list of characters). The result is the current token prepended to the rest of tokens. 

We'll do categorization using guards. Let's start with operators:

``` haskell
tokenize :: String -> [Token]
tokenize [] = []
tokenize (c : cs) 
    | elem c "+-*/" = TokOp (operator c) : tokenize cs
    | otherwise  = error $ "Cannot tokenize " ++ [c]
```

The guard checks if `c` is an element of the list of four characters `"+-*/"`. If it is, it constructs a `Token` using the `TokOp` constructor, passing it the result of the call to `operator c` (the function we defined earlier). This token is combined using `:` with the list returned by the recursive call, `tokenize cs`.

I also added the catch all guard that calls `error`. `error` is a function that takes a `String`, displays it, and aborts the program. (In order to satisfy the type checker, `error` is polymorphic in its return type. In this example, the type checker expects a list of `Token`s; so that's the type it will pick for the return type of `error`.)

I constructed the string by concatenating `"Cannot tokenize "` with a single character string `[c]`. I couldn't use `c` directly, because the concatenation operator `++` expects a list of `Char`, not a `Char`. So I created a one-element list `[c]`.

The tokenization of one-character numbers and identifiers is pretty simple:

``` haskell
| isDigit c  = TokNum (digitToInt c) : tokenize cs
| isAlpha c  = TokIdent [c]          : tokenize cs
```
Here, I converted a digit to an integer using `digitToInt`, and a single character to a string using `[c]`.

Finally, our tokenizer should also be able to discard white space between tokens:
``` haskell
| isSpace c  = tokenize cs
```

We are ready to test our first tokenizer:

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

Next time we'll work on tokenizing multi-character numbers and identifiers and learn about mutual recursion.

**Ex 1.** Rewrite the implementation of Fibonacci numbers using guards instead of the `if` statement (it should become much closer to the mathematical definition):

``` active haskell
-- Old definition:
-- fib n = if n > 2 then fib (n - 1) + fib (n - 2) else 1
fib n | n == 1    = ...
      | ...       = ...
      | otherwise = ...

main = print (fib 20)
```
@@@ Show solution
``` active haskell
fib :: Int -> Int
fib n | n == 1    = 1
      | n == 2    = 1
      | otherwise = fib (n-1) + fib (n-2)

main = print (fib 20)
```
@@@

**Ex 2.** Implement function `cat` that concatenates two lists. 

``` active haskell
cat :: [a] -> [a] -> [a]
cat = undefined

main = putStrLn $ cat "Hello " "World!"
```

@@@ Show hint
You want to create a list whose first element is the first element of the first list (if any) followed by *the rest of the first list concatenated with the second list*.
@@@

@@@ Show solution
``` active haskell
cat :: [a] -> [a] -> [a]
cat [] ys = ys
cat (x : xs) ys = x : cat xs ys

main = putStrLn $ cat "Hello " "World!"
```
@@@

**Ex 3.** Use `cat` from previous exercise and currying to define a function `pig` that prepends "pig" to any string.

``` active haskell
cat :: [a] -> [a] -> [a]
cat = undefined

pig :: String -> String
pig = undefined

main = putStrLn $ pig "sty"
```

@@@ Show solution
``` active haskell
cat :: [a] -> [a] -> [a]
cat [] ys = ys
cat (x : xs) ys = x : cat xs ys

pig :: String -> String
pig = cat "pig"

main = putStrLn $ pig "sty"
```
@@@

**Ex 4.** Implement function `toInts` that takes a number in the form of a string and returns a list of its digits as integers.

``` active haskell
import Data.Char

toInts :: String -> [Int]
toInts = undefined

main = print $ toInts "2013"
```

@@@ Show solution
``` active haskell
import Data.Char

toInts :: String -> [Int]
toInts [] = []
toInts (c : cs) = digitToInt c : toInts cs

main = print $ toInts "2013"
```
@@@

**Ex 5.** Implement function `sumDig` that takes a number in the form of a string and calculates the sum of its digits. Make use of the function from the previous exercise.

``` active haskell
import Data.Char

toInts :: String -> [Int]
toInts [] = []
toInts (c : cs) = digitToInt c : toInts cs

sumDig :: String -> Int
sumDig = undefined

main = print $ sumDig "30750"
```
@@@ Show hint
Define an auxiliary recursive function `acc` that takes the sum-so-far and the remaining digits, and returns the total. Call it with appropriate arguments.
``` haskell
acc :: Int -> [Int] -> Int
```
@@@

@@@ Show solution
``` active haskell
import Data.Char

toInts :: String -> [Int]
toInts [] = []
toInts (c : cs) = digitToInt c : toInts cs

sumDig :: String -> Int
sumDig str = acc 0 (toInts str)

acc :: Int -> [Int] -> Int
acc a [] = a
acc a (i:is) = acc (a + i) is

main = print $ sumDig "30750"
```
@@@
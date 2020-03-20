# 7. Tokenizer: Higher Order Functions

[Previously](https://www.fpcomplete.com/user/bartosz/basics-of-haskell/6-tokenizer-function-types) we have implemented a single-character proof-of-concept tokenizer. What we really need is to be able to recognize multi-character tokens such as identifiers and numbers. But before we get there, let's explore some new functional techniques. 

## Higher Order Functions and Lambdas

It's a good programming practice to separate independent concerns. The code we came up with so far was a mixture of two such concerns:

1. Traversing a data structure
2. Performing an operation on each element

Try to identify the two in our implementation of `tokenize`:

``` haskell
tokenize :: String -> [Token]
tokenize [] = []
tokenize (c : cs) 
    | elem c "+-*/" = TokOp (operator c) : tokenize cs
    | isDigit c  = TokNum (digitToInt c) : tokenize cs
    | isAlpha c  = TokIdent [c]          : tokenize cs
    | isSpace c  = tokenize cs
    | otherwise  = error $ "Cannot tokenize " ++ [c]
```

These kinds of traversals are so common that functional languages abstract them into higher order functions. A *higher order function* is a function that takes another function as an argument. 

This higher order functional approach has been so successful that it was eventually adopted by imperative languages. The C++ STL is based on the separation of concerns and higher order functions: traversal is abstracted into iterators, and generic algorithms accept functions (function pointers, function objects, or lambdas) as arguments. If you're familiar with the STL algorithms: `std::transform`, `std::copy_if`, and `std::remove_if`, you'll have no problem understanding what follows.

Let me start with the function `map` defined in the Prelude:

``` haskell
map :: (a -> b) -> [a] -> [b]
```

The first argument to `map` is a function `(a -> b)`. Notice that the parentheses around this argument are mandatory because the arrow associates to the right. The second argument is a list of `a`, and the result is the list of `b`. The type signature pretty much says it all: `map` applies the function `(a -> b)` to everly element of a list to produce a new list. The implementation is pretty straightforward too:

``` active haskell
import Data.Char -- for the example
import Prelude hiding (map)
-- show
map :: (a -> b) -> [a] -> [b]
map _ [] = []
map f (a : as) = f a : map f as

main = print $ map toUpper "hello world!"
```

Notice that `map` is generic in both type variables, `a` and `b`.

### Map and Filter in Action

Now suppose we have a helper function `tokenizeChar` that turns a single character into a token. We can rewrite our tokenizer using `map`:

``` haskell
tokenize :: String -> [Token]
tokenize str = map tokenizeChar str
```
This can be simplified further using currying ("divide" both sides by `str`):

``` haskell
tokenize :: String -> [Token]
tokenize = map tokenizeChar
```

Here's the function `tokenizeChar`:

``` haskell
tokenizeChar :: Char -> Token
tokenizeChar c | elem c "+-*/" = TokOp (operator c)
               | isDigit c  = TokNum (digitToInt c)
               | isAlpha c  = TokIdent [c]
               | isSpace c  = TokSpace
               | otherwise  = error $ "Cannot tokenize " ++ [c]
```

Notice that the skipping of whitespace cannot be done using `map` because `map` cannot change the *shape* of the list (the length, in this case). That's why I had to introduce a new token, `TokSpace` to replace white space characters. 

Fortunately, there is another higher-order function, `filter`, for removing stuff from a list:

``` haskell
filter :: (a -> Bool) -> [a] -> [a]
```

It takes a predicate function, `(a -> Bool)`, and applies it to each element of the list. If the predicate returns `True` the element is kept in the list, otherwise it's removed. `filter` is defined in the Prelude, but we could have as well implemented it ourselves -- it's so easy:

``` active haskell
import Data.Char
import Prelude hiding (filter)
-- show
filter :: (a -> Bool) -> [a] -> [a]
filter _ [] = []
filter p (a : as) = if p a 
                    then a : filter p as
                    else filter p as
main = putStrLn $ filter isDigit "1x+3y"
```

We are now ready to implement the function `deSpace` that filters out `TokSpace` tokens.

``` haskell 
deSpace :: [Token] -> [Token]
deSpace = filter notSpace 

notSpace :: Token -> Bool
notSpace t = t /= TokSpace
```

(This definition will only work with tokens that are `deriving Eq`. Can you tell why?  Recall that `/=` means *not equal*)

Since the auxiliary function `notSpace` is so simple, we can inline it using a lambda -- the anonymous function:

``` haskell 
deSpace :: [Token] -> [Token]
deSpace = filter (\t -> t /= TokSpace)
```

The syntax for lambdas is very simple. You start with a backslash, which looks a little like the Greek letter lambda, λ; follow it with the list of arguments; an arrow `->`; and the body of the function -- an expression. Lambda syntax doesn't support multiple patterns or guards, but otherwise lambdas are just like definitions of regular functions. Lambdas are so useful that they are now part of C++11 and Java 8.

Here's the final version of the single-character tokenizer:

``` active haskell
import Data.Char

data Operator = Plus | Minus | Times | Div
    deriving (Show, Eq)

data Token = TokOp Operator
           | TokIdent String
           | TokNum Int
           | TokSpace
    deriving (Show, Eq)

operator :: Char -> Operator
operator c | c == '+' = Plus
           | c == '-' = Minus
           | c == '*' = Times
           | c == '/' = Div

tokenize :: String -> [Token]
tokenize = map tokenizeChar

tokenizeChar :: Char -> Token
tokenizeChar c | elem c "+-*/" = TokOp (operator c)
               | isDigit c  = TokNum (digitToInt c)
               | isAlpha c  = TokIdent [c]
               | isSpace c  = TokSpace
               | otherwise  = error $ "Cannot tokenize " ++ [c]

deSpace :: [Token] -> [Token]
deSpace = filter (\t -> t /= TokSpace) 

main = print $ deSpace $ tokenize " 1 + 4 / x "
```

Did you notice something interesting? This program doesn't exhibit any recursion at all. Recursion is hidden in the implementation of higher order functions. 

A good program in any language should take advantage of higher order abstractions. Haskell makes it easier, but even in imperative languages it pays to raise the level of abstraction. For instance, the use of plain loops is now being actively discouraged in modern C++ in favor of range-based `for` or STL algorithms like `foreach`, `transform`, `accumulate`, etc.; all of which was made considerably easier with the use of lambdas.

**Ex 1.** Implement the function `toInts` from the previous tutorial using `map`. This function takes a string of digits and creates a list of `Int`s corresponding to these digits.

``` active haskell
import Data.Char -- (digitToInt)

toInts :: String -> [Int]
toInts = undefined

main = print $ toInts "30750"
```
@@@ Show solution
``` active haskell
import Data.Char

toInts :: String -> [Int]
toInts = map digitToInt

main = print $ toInts "30750"
```
@@@

**Ex 2.** Implement function `squares` that takes a list of integers and returns the list of their squares. Use higher order functions and lambdas.

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

**Ex 3.** Implement function `inCircle2` that takes a list of 2-D points and returns only those that fit inside the circle of radius 2. 

``` active haskell
type Point = (Double, Double)
inCircle2 :: [Point] -> [Point]
inCircle2 = undefined

main = print $ inCircle2 [(0, 0), (2, -2), (1, -1), (1.9, 0.1), (10, 1)]
```

@@@ Show solution
``` active haskell
type Point = (Double, Double)
inCircle2 :: [Point] -> [Point]
inCircle2 = filter (\(x, y) -> sqrt (x*x + y*y) <= 2.0)

main = print $ inCircle2 [(0, 0), (2, -2), (1, -1), (1.9, 0.1), (10, 1)]
```
@@@


## Tokenizing Identifiers

Unfortunately, recognizing multi-character tokens can't be reduced to `map` and `filter`. The former doesn't modify the shape of the list; the latter does, but only by deleting elements. So let's go back to the recursive version of the tokenizer and let's start with multi-character identifiers.

It's easy to recognize the start of an identifier: it must be an alphabetic character. We even have a predicate `isAlpha` just for that purpose. This first character can be followed by zero or more alphanumeric characters. There is another predicate, `isAlphaNum`, for that purpose. Notice that the tokenizer has to change its behavior while processing an identifier: digits inside an identifier are *not* treated as numbers, as they would normally be.  

First, let's modify the appropriate part of `tokenize` to look for more than one character:
``` haskell
tokenize (c : cs) 
    ...
    | isAlpha c = identifier c cs
    ...
```

The new function `identifier` takes the already recognized alphabetic character, plus the rest of the input for further processing. We'll run the input through a helper function, `alnums`, that consumes and aggregates alphanumeric characters. What should we do if there's still more input after the run of anphanumeric characters? We need the function `alnums` to return the leftover input *together* with the list of recognized characters -- we'll return a pair of lists:

``` haskell
alnums :: String -> (String, String)
```
So here's the implementation of `identifier` using `alnums`:

``` haskell
identifier c cs = let (str, cs') = alnums cs in
                  TokIdent (c:str) : tokenize cs'
```
We have to store (and pattern match) the pair returned by `alnums` in temporary variables, because we'll do different things with different components of the pair. Defining local variables (local binding) is done using the `let`/`in` expression. The `let` part defines local variables, and the `in` part is an expression that uses those variables. Variables bound in `let` can be pattern matched. One `let` statement may contain multiple definitions.

``` haskell
let (pattern1) = expr1
    (pattern2) = expr2
    var = expr3
in
    expression
```

It's important to understand that `let` is not a statement -- it's an expression. It has a value: the value defined by the `in` expression. 

The visibility of local variables is restricted to the `let`/`in` scope. 

### Mutual Recursion

Notice what `identifier` does with the unconsumed input, `cs'`. It recursively calls our main `tokenize` function. We could have done the same trick as with `identifier` -- returning both the result and the unprocessed string -- but that would unnecessarily complicate the code for processing other tokens. What I've done instead is to use *mutual recursion*, where multiple functions recurse into each other. 

### Accumulating

The remaining task is to implement the helper function `alnums`. This function should accumulate alphanumeric characters into a list. As usual, we will recurse into the input list, but this time we have to carry along the accumulator -- the list of alphanumerics that were recognized so far. The standard trick is to define an auxiliary recursive function, let's call it `als`, that takes the accumulator along with the rest of the input. To start the recursion, we will pass an empty accumulator to this function. 

``` haskell
alnums :: String -> (String, String)
alnums str = als "" str
```

Let's first consider the conditions that terminate recursion. One is the end of input, and the other is a non-alphanumeric character. In both cases `als` should return the current accumulator paired with the rest (if any) of the input. 

When `als` recognizes an alphanumeric character, `c`, it will hold on to it and immediately call `als` with the rest of the input. This call will return the list of alphanumeric characters,`acc'`, and the unused tail of the input, `cs'`. We prepend the retained character to the front of the accumulator, and pair it with the new tail: `(c:acc', cs')`. Here it is:

``` active haskell
import Data.Char

alnums :: String -> (String, String)
alnums str = als "" str
  where
    als acc [] = (acc, [])
    als acc (c : cs) | isAlphaNum c = 
                           let (acc', cs') = als acc cs 
                           in (c:acc', cs')
                     | otherwise = (acc, c:cs)

main = print $ alnums "R2D2+C3Po"
```

Instead of making `als` a top-level function, I introduced the new construct, `where`. Whatever is defined inside the `where` block -- it could be definitiona of functions or variables -- is visible to the main body of the function but not outside of it. Here, the function `als` is defined in the `where` clause. (Conversely, the arguments to the main function are accessible inside the `where` clause, but we're not using this property in this example). 

I could have used `let` instead of `where` to define `als`, but I think it would be less readable. See for yourself:

``` active haskell
import Data.Char

alnums :: String -> (String, String)
alnums str = 
  let
    als acc [] = (acc, [])
    als acc (c : cs) | isAlphaNum c = 
                           let (acc', cs') = als acc cs 
                           in (c:acc', cs')
                     | otherwise = (acc, c:cs)
  in
    als "" str

main = print $ alnums "R2D2+C3Po"
```

In general, though, these two constructs are not exchangeable: `let`/`in` can be used anywhere an expression is expected; whereas `where` is tied to the end of a function definition. Moreover, if you have a function defined using multiple patterns and guards, the definitions in the `where` clause are going to be visible accross all bodies of the function; whereas `let` is local to each body.

### Folding

Traversal with accumulation is also a very common pattern and is encapsulated in `foldl` and `foldr` (fold left and fold right) -- higher order functions defined in the Prelude. For instance, here's the signature of `foldl`:

``` haskell
foldl :: (a -> b -> a) -> a -> [b] -> a
```

`a` is the type of the accumulator and `[b]` is the input list type. `foldl` traverses the list from left to right, calling the function `(a -> b -> a)` with two arguments: the current accumulator and the current element. The function returns the new accumulator, which is then used in the next iteration. 

**Ex 4.** Use `foldl` to calculate the sum of squares given a list of doubles. 
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

**Ex 5.** The accumulator in `foldl` can also be a list. With this in mind, implement function `rev` that reverses a list.

``` active haskell
rev :: [a] -> [a]
rev = foldl ???

main = print $ rev "spot on"
```

@@@ Show solution
``` active haskell
rev :: [a] -> [a]
rev = foldl (\acc a -> a : acc) []

main = print $ rev "spot on"
```

This function is available in the Prelude under the name <hoogle>reverse</hoogle>.
@@@

**Ex 6.** Just as a proof of concept, implement a version of `alnums` using `foldl`, even though it's going to be awkward and inefficient.

``` active haskell
import Data.Char

alnums :: String -> (String, String)
alnums str = undefined

main = do
    print $ alnums "R2D2+C3Po"
    print $ alnums "a14"
```

@@@ Show hint
Since `foldl` must consume the whole input list, the accumulator must not only build the result string but also the remaining input string (yes, that's extremely inefficient). It must also keep track of the state: "Am I gathering alphanumeric characters or am I accumulating the tail of the list?" Use the following type as the accumulator:

``` haskell
type Accum = (Bool, String, String)
```

Also, since we have to process the input left to right, we'll be appending (rather than prepending) characters to either list. Appending (operator `++`), is also a very inefficient operation.
@@@

@@@ Show solution
``` active haskell
import Data.Char

type Accum = (Bool, String, String)

alnums :: String -> (String, String)
alnums str = let (_, als, rest) = foldl f (True, [], []) str
             in (als, rest)
  where
    f (True, als, rest) c  | isAlphaNum c = (True, als ++ [c], rest)
                           | otherwise = (False, als, [c])
    f (False, als, rest) c = (False, als, rest ++ [c])

main = do
    print $ alnums "R2D2+C3Po"
    print $ alnums "a14"
```
It's easy to figure out that this implementation runs in N² time, where N is the size of the whole input. Clearly, this is not recommended.
@@@

## Tokenizing Numbers

The tokenizer for numbers is very similar to the tokenizer for identifiers. As before, we'll use mutual recursion:

``` haskell
tokenize (c : cs) 
    ...
    | isDigit c = number c cs
    
number c cs = 
   let (digs, cs') = digits cs in
   TokNum (read (c : digs)) : tokenize cs'
```
I used `read` to convert a string to an `Int`. 

The function `digits` is the equivalent of `alnums`, except that it gathers digits rather than alphanumeric characters: 

``` haskell
digits :: String -> (String, String)
digits str = digs "" str
  where
    digs :: String -> String -> (String, String)
    digs acc [] = (acc, [])
    digs acc (c : cs) | isDigit c = 
                            let (acc', cs') = digs acc cs 
                            in (c:acc', cs')
                      | otherwise = (acc, c:cs)
```

In fact, if you compare the two functions, they only differ in one place: The predicate `isAlphaNum` is replaced by `isDigit`. This clearly calls for refactoring: We should make the predicate an argument to a more general (higher order) function. Let's call this function `span`:

``` active haskell
import Data.Char
import Prelude hiding (span)
-- show
span :: (a -> Bool) -> [a] -> ([a], [a])
span pred str = 
  let -- define a helper function 'spanAcc'
    spanAcc acc [] = (acc, [])
    spanAcc acc (c : cs) | pred c = 
                             let (acc', cs') = spanAcc acc cs 
                             in (c:acc', cs')
                         | otherwise = (acc, c:cs)
  in
    spanAcc [] str

main = print $ span isAlphaNum "R2D2 + C3Po"
```

`span` is so useful that it's included in the Prelude. We'll use it in the final version of the tokenizer, which follows.

## The Tokenizer

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
    | isDigit c = number c cs
    | isAlpha c = identifier c cs
    | isSpace c = tokenize cs
    | otherwise = error $ "Cannot tokenize " ++ [c]

identifier c cs = let (str, cs') = span isAlphaNum cs in
                  TokIdent (c:str) : tokenize cs'

number c cs = 
   let (digs, cs') = span isDigit cs in
   TokNum (read (c : digs)) : tokenize cs'

main = do
    print $ tokenize "12 + 24 / x1"
```

In the next installment we are going to implement the parser. We'll be using the results of the simple exercise below, so try to work on it, or just peek at the solution.

**Ex 7.** Extend the tokenizer above to recognize more tokens: `LParen` and `RParen` corresponding to `(` and `)`; as well as `TokAssign` for `=`. 

@@@ Show solution
This is the new `Token` definition:

``` haskell
data Token = TokOp Operator
           | TokAssign
           | TokLParen
           | TokRParen
           | TokIdent String
           | TokNum Double
           | TokEnd
    deriving (Show, Eq)
```
These are the changes to `tokenize`:

``` haskell
tokenize :: String -> [Token]
tokenize [] = []
tokenize (c : cs) 
    | elem c "+-*/" = TokOp (operator c) : tokenize cs
    | c == '='  = TokAssign : tokenize cs
    | c == '('  = TokLParen : tokenize cs
    | c == ')'  = TokRParen : tokenize cs
    | isDigit c = number c cs
    | isAlpha c = identifier c cs
    | isSpace c = tokenize cs
    | otherwise = error $ "Cannot tokenize " ++ [c]
```
@@@
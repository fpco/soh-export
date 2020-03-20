# 4. Symbolic Calculator: Recursion

Now that we are done with the [preliminaries](https://www.fpcomplete.com/user/bartosz/basics-of-haskell/3-pure-functions-laziness-io), I'd like to show you how to design and develop a small application -- a symbolic calculator. It's a console application: the user types in expressions that are evaluated, and the results are displayed. To make it more interesting, the calculator supports symbolic variables that can be assigned and re-assigned and used in expressions. Here's an example of a user session:

![Calculator session](http://www.bartosz.com/images/SoH/Calc.png)

In fact you can run the calculator right here, on the spot:

``` active haskell
import Text.Parsec
import Text.Parsec.String
import Text.Parsec.Token
import Text.Parsec.Expr
import Text.Parsec.Language
import qualified Data.Map as M
import qualified Control.Monad.State as S
import Control.Monad.Error
import Control.Monad.Identity

-- Lexer

def = emptyDef { identStart  = letter
               , identLetter = alphaNum 
               , opStart     = oneOf "+-*/="
               , opLetter    = oneOf "+-*/="
               }

lexer :: TokenParser ()
lexer = makeTokenParser def

-- Expression tree

data Expression = Constant Double
                | Identifier String
                | Addition Expression Expression
                | Subtraction Expression Expression
                | Multiplication Expression Expression
                | Division Expression Expression
                | Negation Expression
                | Assignment Expression Expression
                deriving Show

-- Parser

parseNumber :: Parser Expression
parseNumber = do
    v <- naturalOrFloat lexer
    case v of
        Left  i -> return $ Constant $ fromIntegral i
        Right n -> return $ Constant n

parseIdentifier :: Parser Expression
parseIdentifier = do
   i <- identifier lexer
   return $ Identifier i
   
parseExpression :: Parser Expression
parseExpression = (flip buildExpressionParser) parseTerm [
   [ Prefix (reservedOp lexer "-" >> return Negation)
   , Prefix (reservedOp lexer "+" >> return id) 
   ]
 , [ Infix (reservedOp lexer "*" >> return Multiplication) AssocLeft
   , Infix (reservedOp lexer "/" >> return Division) AssocLeft
   ]
 , [ Infix (reservedOp lexer "+" >> return Addition) AssocLeft
   , Infix (reservedOp lexer "-" >> return Subtraction) AssocLeft 
   ]
 , [ Infix (reservedOp lexer "=" >> return Assignment) AssocRight
   ]
 ]

parseTerm :: Parser Expression
parseTerm = parens lexer parseExpression 
        <|> parseNumber
        <|> parseIdentifier

parseInput :: Parser Expression
parseInput = do
    whiteSpace lexer
    ex <- parseExpression
    eof
    return ex

-- Evaluator

type SymTab = M.Map String Double

type Evaluator a = S.StateT SymTab (ErrorT String Identity) a

runEvaluator :: Evaluator Double -> SymTab -> Either String (Double, SymTab)
runEvaluator calc symTab = runIdentity $ runErrorT $ S.runStateT calc symTab

eval :: Expression -> Evaluator Double

eval (Constant x) = return x

eval (Identifier i) = do
    symtab <- S.get
    case M.lookup i symtab of
        Nothing -> fail $ "Undefined variable " ++ i
        Just e  -> return e

eval (Addition eLeft eRight) = do
    lft <- eval eLeft
    rgt <- eval eRight
    return $ lft + rgt

eval (Subtraction eLeft eRight) = do
    lft <- eval eLeft
    rgt <- eval eRight
    return $ lft - rgt

eval (Multiplication eLeft eRight) = do
    lft <- eval eLeft
    rgt <- eval eRight
    return $ lft * rgt

eval (Division eLeft eRight) = do
    lft <- eval eLeft
    rgt <- eval eRight
    return $ lft / rgt

eval (Negation e) = do
    val <- eval e
    return $ -val

eval (Assignment (Identifier i) e) = do
    val <- eval e
    S.modify (M.insert i val)
    return val

eval (Assignment _ _) =
    fail "Left of assignment must be an identifier"

defaultVars :: M.Map String Double
defaultVars = M.fromList
   [ ("e", exp 1)
   , ("pi", pi)
   ]
   
--runEvaluator returns Either String (Double, SymTab Double)

calculate :: SymTab -> String -> (String, SymTab)
calculate symTab s = 
    case parse parseInput "" s of
    Left  err -> ("error: " ++ (show err), symTab)
    Right exp -> case runEvaluator (eval exp) symTab of
                 Left  err              -> ("error: " ++ err, symTab)
                 Right (val, newSymTab) -> (show val, newSymTab)

loop :: SymTab -> IO ()
loop symTab = do
    line <- getLine
    if null line
    then return ()
    else do
        let (result, symTab') = calculate symTab line
        putStrLn result
        loop symTab'
   
main = loop defaultVars
-- show 
-- Enter expressions, one per line. Empty line to quit --
```

This is not the implementation I'll be describing. I wrote this one using Haskell libraries such as Parsec (one of the standard parsing libraries) and several monad transformers. In this series of tutorials I'd like to implement the same functionality from scratch, so you'll be able to clearly see each step and learn the language in the process.

## The Design

- At the very high level, the calculator is a loop that gets a line of text from the user and then calculates and displays the result. 
- The calculation is done is three steps:
    - Lexical analysis: The string is converted to tokens
    - Parsing: Building an expression tree
    - Evaluation: The expression is evaluated.

In the first phase of implementation we won't worry about error handling and symbolic variables -- we'll add them later.

Notice that there's nothing Haskell-specific in this design -- it's just a piece of good old software engineering. Some people worry that programming in Haskell means re-learning everything from scratch. This is based on seriously underestimating the amount of software engineering that is common to all programming tasks -- independent of the language. 

## Designing with Types

We haven't talked about types yet because, even though Haskell is a strongly typed language, it has a powerful type inference system. This feature makes quick prototyping easy. Sometimes you just want to write a function and not worry about types. The compiler will figure them out. (C++11 introduced a modicum of type inference with the keyword `auto`.) 

On the other hand, software *design* in Haskell often starts with types. Let's try this approach.

### 1. Lexical analyzer

Lexical analyzer is implemented as a function `tokenize` that takes a string (of type `String`) and returns a list of tokens. We'll define the `Token` data type later. A list of tokens has the type `[Token]` -- the square brackets are used to create lists (both list *types*, like `[Int]`, and list *literals*, like `[1, 2, 3]`). Finally, a *function* type is constructed with an arrow `->` between the type of the argument and the type of the result (we'll get to multi-argument functions later). Putting all this together, we can write the Haskell type signature for the function `tokenize` as follows:
``` haskell
tokenize :: String -> [Token]
```
This is read as: Tokenize is a function taking a string and returning a list of tokens. The double colon is used to introduce a type signature. 

Type names must always start with capital letters, as in `String` or `Double` (except for names constructed from special characters, like the list type, `[]`). 


### 2. Parser 

The parsing function takes a list of tokens and produces an expression. We'll define the `Expression` type later. For now, this is the type of `parse`:
``` haskell
parse :: [Token] -> Expression
```

### 3. Evaluator

We'll make `evaluate` take an `Expression` and return a value of the built in type `Double` (double precision floating point number).
``` haskell
evaluate :: Expression -> Double
```
We can define dummy data types for `Token` and `Expression`, and dummy function bodies; and fire up the compiler to typecheck our design:

``` active haskell
data Token
data Expression

tokenize :: String -> [Token]
tokenize = undefined

parse :: [Token] -> Expression
parse = undefined

evaluate :: Expression -> Double
evaluate = undefined

main :: IO ()
main = putStrLn "It works, so are we done yet?"
```
You might wonder how `undefined` plays with the type checker. It turns out that the type of `undefined` is the *bottom* of the type hierarchy, which means it can be implicitly converted to any type. For instance, in the definition of `tokenize` the type of `undefined` becomes the function type: `String->[Token]`.

The type of `main` is always `IO ()`: the type of `IO` monadic action that produces no result (only side effects). The type `()` itself is called *unit* -- loosely corresponding to `void` in C-like languages.

## Recursion

Our design calls for a loop that accepts user input and displays the results. All loops in Haskell are implemented either using recursion or using (higher-order) functions whose implementation uses recursion. 

You might be concerned about the performance or recursion and the possibility of blowing the stack -- in most cases this is not a problem since the compiler is able to turn most recursions into loops. This is called *tail recursion* optimization, where the recursive call at the very end of a function is simply turned into a goto to the beginning of the function. More serious performance concerns arise occasionally from Haskell's laziness but we'll talk about it later.

With that in mind, we are ready to implement the top-level loop of our calculator:

``` active haskell
main :: IO ()
main = do
    line <- getLine
    putStrLn line
    main
```

You can think of `main` as first calling `getLine`, storing the result in the variable `line`, then calling `putStrLn` with that `line`, and then calling itself again. This will create an infinite loop, but no stack will be hurt in the process, since this is a typical case of tail recursion. 

Of course, what really happens when the program is running is slightly different because of the `IO` monad and general laziness. So it would be more appropriate to say that `main` is an `IO` action that is a sequence of three other actions: the ones returned by `getLine`, `putStrLn`, and `main`. The last action, when the time comes to execute it, will produce three new actions, etc. But everything happens on the need to run basis, so the inner `main` will not be evaluated until the (blocking) action produced by `getLine` delivers its result. 

Thinking about recursion rather than looping might initially seem unnatural, but it quickly becomes second nature. The main reason Haskell doesn't use loops is because of immutability: Loops in imperative languages usually have some kind of mutable counter or a mutable pointer. It's relatively easy to replace those loops with recursion. But first we need to learn about conditionals: We have to be able to break out of recursion at some point.

### Conditional

A conditional in Haskell is just a simple `if`, `then`, `else` construct. The `else` is mandatory. 

Anything between `if` and `then` is the condition (you don't even have to surround it with parentheses), and it must evaluate to a Boolean. You can pretty much use the familiar equality and comparison operators, `>`, `>=`, `<`, `<=`, `==`, to create Boolean values; except for the not-equal operator which is `/=`. You can also combine them using `&&` and `||` for logical *and* and *or*, and `not` for *not*.

However, unlike in imperative languages, the Haskell if/then/else is not a statement but an expression (similar to C's (`?:`) construct). It evaluates to either the `then` or the `else` expression, both of which have to be of the same type. For instance:

``` active haskell
main = do
    putStrLn "Enter a number"
    str <- getLine
    print (if read str >= 1 then 1 else 0)
```
Here, the if/then/else expression that is the argument to `print` evaluates to either 1 or 0. 

You might be wandering about the short-circuitting properties of if/then/else or the binary Boolean operators `&&` and `||`. In most languages the property of not evaluating the branch that is not taken has to be built into the language as a special feature. Otherwise constructs like:

```
x = (p != nullptr) ? *p : 0
```
wouldn't work properly. In Haskell, short-circuiting is just the side effect of laziness. The branch not taken is never used, so it won't be evaluated.

Several explanations are in order: I used the function `read` to turn a string into a value. It's an interesting function -- it's overloaded on the return type. Here, the compiler deduced that an integral value was needed because it was compared to another integral value, 1. (Try experimenting with this code by inputing a floating point number. Then change the 1 in the if clause to 1.0 and see if the behavior changes.)

We are now ready to convert a simple imperative loop that prints numbers from 0 to 4 to Haskell. Here's the C++ loop:

```
for (int i = 0; i < 5; ++i)
    std::cout << i << std::endl
```
And here's its recursive counterpart written in Haskell:

``` active haskell
loop :: Int -> IO ()
loop n = do
    if n < 5
    then do
        putStrLn (show n)
        loop (n + 1)
    else
        return ()

main :: IO ()
main = loop 0
```

The Haskell code looks straightforward, although it's more verbose than its C++ counterpart. That's because I made it control-driven -- which is closer to the imperative version -- rather than data-driven. In Haskell one should really try to think at a higher abstraction level. Here, the goal is to print a list of integers from 0 to 4, so it would be more natural to start with such a list: `[0, 1, 2, 3, 4]` or, using a handy shorthand, `[0..4]`; and apply a function to it. We'll see examples of this approach later.

Let's talk about types: `loop` returns a "void" `IO` action, so both branches of the if must also return an `IO ()` action. The first branch is a sequence of two actions (hence the use of `do` in that branch), the last of which is indeed of the type `IO ()` (that's the result of calling `loop`). The second branch is more interesting. At first sight you might not even notice anything out of the ordinary: Well, it does return a unit value `()`, which is of the type unit `()`. But how does this value become an `IO ()` action? The trick is that `return` is not a built-in keyword, it's actually an important monadic function (every monad has it). 

The `return` function turns whatever value it's given into a monadic value: here it turns `()` into `IO ()`. It could also turn "Hello!" into `IO String`, etc. We'll see more examples of using `return` to "return" a value from a `do` block in the future. Also notice the use of the `Int` type -- it's a fixed precision integer.

In the [next installment](https://www.fpcomplete.com/user/bartosz/basics-of-haskell/5-tokenizer-data-types) we'll start implementing the lexical analyzer and learn more about data types.

## Exercises

**Ex 1**. Print squares of numbers from 1 to 10.
``` active haskell
loop :: Int -> IO ()
loop n = undefined

main :: IO ()
main = loop 1
```

@@@ Show solution
``` active haskell
loop :: Int -> IO ()
loop n = do
    if n <= 10
    then do
        putStrLn (show (n * n))
        loop (n + 1)
    else
        return ()

main :: IO ()
main = loop 1
```
@@@

**Ex 2**. No exposition of recursion is complete without factorial. Use the following property: Factorial of n is n times the factorial of (n - 1), and the factorial of 0 is 1.

``` active haskell
fact :: Int -> Int
fact n = undefined

main = print (fact 20)
```

@@@ Show solution
``` active haskell
fact :: Int -> Int
fact n = if n > 0 then n * fact (n - 1) else 1

main = print (fact 20)
```
@@@

**Ex 3**. The evaluation of factorial starts returning incorrect results right about n = 21 because of the `Int` overflow. Try implementing a version that uses the infinite precision `Integer` instead of `Int`.

``` active haskell
fact :: Int -> Int
fact n = if n > 0 then n * fact (n - 1) else 1

fullFact :: ...
fullFact n = ...

main = do
    print (fact 23)
    print (fullFact 23)
```

@@@ Show solution
``` active haskell
fact :: Int -> Int
fact n = if n > 0 then n * fact (n - 1) else 1

fullFact :: Integer -> Integer
fullFact n = if n > 0 then n * fullFact (n - 1) else 1

main = do
    print (fact 23)
    print (fullFact 23)
```
@@@

**Ex 4**. No exposition of recursion is complete without Fibonacci numbers. (I'm using these mathematical examples because we haven't learned about data structures. In general, Haskell is not just about math.) Use the following property of Fibonacci numbers: The n'th Fibonacci number is the sum of the (n-1)'st and the (n-2)'nd, and the first and second Fibonacci numbers are both 1.

``` active haskell
fib :: Int -> Int
fib n = undefined

main = print (fib 20)
```

@@@ Show solution
``` active haskell
fib :: Int -> Int
fib n = if n > 2 then fib (n - 1) + fib (n - 2) else 1

main = print (fib 20)
```
@@@

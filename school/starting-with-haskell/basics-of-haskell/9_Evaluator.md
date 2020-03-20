# 9. Evaluator

Now that we know how to [parse arithmetic expressions](https://www.fpcomplete.com/user/bartosz/basics-of-haskell/8_Parser), let's talk about evaluating our parse trees. First, let's implement the simplified evaluator which ignores symbolic variables. This turns out to be a very simple operation: you traverse the expression tree and evaluate every node. Our leaf nodes are literal numbers -- evaluation means extracting the number. For non-leaf nodes we first recurse into their children and then combine the results. Notice how recursive traversal fits very well with the recursive definition of the treee. 


## The Easy Evaluator

The evaluator is implemented as a function `evaluate` that takes an expression tree and returns a number:
``` haskell
evaluate :: Tree -> Double
```
`Tree` has several constructors, so we'll pattern match on each of them. Let's start with the number node -- its value is just the number that was passed to its constructor:

``` haskell
evaluate (NumNode x) = x
```
The recursive nodes follow the following pattern: We call `evaluate` on each child to get the numbers and then perform a node-specific operation on them. 

Here it is (You can run this code and do simple arithmetic. Try it!):

``` active haskell
import Data.Char

data Operator = Plus | Minus | Times | Div
    deriving (Show, Eq)

data Token = TokOp Operator
           | TokAssign
           | TokLParen
           | TokRParen
           | TokIdent String
           | TokNum Double
           | TokEnd
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
    | c == '='  = TokAssign : tokenize cs
    | c == '('  = TokLParen : tokenize cs
    | c == ')'  = TokRParen : tokenize cs
    | isDigit c = number c cs
    | isAlpha c = identifier c cs
    | isSpace c = tokenize cs
    | otherwise = error $ "Cannot tokenize " ++ [c]

identifier :: Char -> String -> [Token]
identifier c cs = let (name, cs') = span isAlphaNum cs in
                  TokIdent (c:name) : tokenize cs'

number :: Char -> String -> [Token]
number c cs = 
   let (digs, cs') = span isDigit cs in
   TokNum (read (c : digs)) : tokenize cs'

---- parser ----

data Tree = SumNode Operator Tree Tree
          | ProdNode Operator Tree Tree
          | AssignNode String Tree
          | UnaryNode Operator Tree
          | NumNode Double
          | VarNode String
    deriving Show

lookAhead :: [Token] -> Token
lookAhead [] = TokEnd
lookAhead (t:ts) = t

accept :: [Token] -> [Token]
accept [] = error "Nothing to accept"
accept (t:ts) = ts

expression :: [Token] -> (Tree, [Token])
expression toks = 
   let (termTree, toks') = term toks
   in
      case lookAhead toks' of
         (TokOp op) | elem op [Plus, Minus] -> 
            let (exTree, toks'') = expression (accept toks') 
            in (SumNode op termTree exTree, toks'')
         TokAssign ->
            case termTree of
               VarNode str -> 
                  let (exTree, toks'') = expression (accept toks') 
                  in (AssignNode str exTree, toks'')
               _ -> error "Only variables can be assigned to"
         _ -> (termTree, toks')

term :: [Token] -> (Tree, [Token])
term toks = 
   let (facTree, toks') = factor toks
   in
      case lookAhead toks' of
         (TokOp op) | elem op [Times, Div] ->
            let (termTree, toks'') = term (accept toks') 
            in (ProdNode op facTree termTree, toks'')
         _ -> (facTree, toks')

factor :: [Token] -> (Tree, [Token])
factor toks = 
   case lookAhead toks of
      (TokNum x)     -> (NumNode x, accept toks)
      (TokIdent str) -> (VarNode str, accept toks)
      (TokOp op) | elem op [Plus, Minus] -> 
            let (facTree, toks') = factor (accept toks) 
            in (UnaryNode op facTree, toks')
      TokLParen      -> 
         let (expTree, toks') = expression (accept toks)
         in
            if lookAhead toks' /= TokRParen 
            then error "Missing right parenthesis"
            else (expTree, accept toks')
      _ -> error $ "Parse error on token: " ++ show toks

parse :: [Token] -> Tree
parse toks = let (tree, toks') = expression toks
             in
               if null toks' 
               then tree
               else error $ "Leftover tokens: " ++ show toks'

---- evaluator ----
-- show

evaluate :: Tree -> Double
evaluate (SumNode op left right) = 
    let lft = evaluate left
        rgt = evaluate right
    in
        case op of
          Plus  -> lft + rgt
          Minus -> lft - rgt

evaluate (ProdNode op left right) = 
    let lft = evaluate left
        rgt = evaluate right
    in
        case op of
          Times -> lft * rgt
          Div   -> lft / rgt

evaluate (UnaryNode op tree) =
    let x = evaluate tree 
    in case op of
         Plus  -> x
         Minus -> -x

evaluate (NumNode x) = x

-- dummy implementation
evaluate (AssignNode str tree) = evaluate tree

-- dummy implementation
evaluate (VarNode str) = 0

main = (print . evaluate . parse . tokenize) "x1 = -15 / (2 + x2)"
```

I implemented `main` as a pipeline of functions, each passing its result to the next one. Notice that functions compose right to left reflecting the fact that they take arguments on the right and return results on the left.

## Threading the Symbol Table

How are we going to implement symbolic variables in our calculator? It seems obvious that `evaluate` acting on an assignment node must have a side effect -- it must modify the value of a variable. And this new value must be visible during the rest of the session, whenever the corresponding `VarNode` accesses it.

In functional programming we shun side effects so we have to use a different implementation strategy: we will thread the symbol table through every call to `evaluate`. 

``` haskell
evaluate :: Tree -> SymTab -> (Double, SymTab)
```

When `evaluate` needs to modify the symbol table (add new or modify an existing variable), it constructs a **new copy** of the symbol table, with the modification included. It then returns that copy. We've seen this idiom already in the implementation of the parser -- we were threading the token list in a similar manner. 

If you're worried about the performance impact of copying potentially large data structures, know that a properly implemented symbol table has its copy/modify operation comparable in cost with a modification of a mutable symbol table. I mentioned earlier that in functional languages you use [persistent data structures](http://en.wikipedia.org/wiki/Persistent_data_structure), which can be cloned by modifying just a handful of pointers, internally. Incidentally, the use of persistent data structures has spread to imperative languages as well because they can be easily made thread safe. We'll use such a persistent data structure for our symbol table. (Keep in mind, though, that if you are 100% sure that the table is the bottleneck of some algorithm, you can always switch to `Data.HashTable` library.)

We'll be using an efficient implementation of a balanced binary tree for our symbol table, taken from the `Data.Map` library. There is a slight problem when using this library: it defines a function `lookup` that has the same name as the Prelude function that operates on lists. Since we want to call `lookup` with a map, we have to resolve this name conflict. The standard way to do this is to use `Data.Map` as a namespace by importing it as `qualified`:

``` haskell
import qualified Data.Map
```

To access `lookup` from `Data.Map`, we would have to qualify it as `Data.Map.lookup`. Since this is pretty verbose, we'll provide an alias `M` for this namespace:

``` haskell
import qualified Data.Map as M
```
Any name from `Data.Map` will now have to be prefixed with `M.`. In particular, we'll be able to call `M.lookup` without the fear of conflict.

Our symbol table is a `Map` that is specialized for strings as keys and doubles as values (notice how we qualify it as `M.Map`). 

``` haskell
type SymTab = M.Map String Double
```

By the way, `Map` requires that the keys be comparable (they have to be instances of the `Cmp` type class, of which later). It so happens that our keys are `String`s, which are (lexicographically) comparable.

You may create an empty `Map` by calling the function `M.empty` or, if you want to pre-fill it, use the `M.fromList` instead:

``` haskell
M.fromList [("pi", pi), ("e", exp 1)]

```
As you can see, `fromList` accepts a list of key/value pairs. The value `pi` and the function `exp` are defined in the Prelude.

Notice that no constructors of `Map` are available to the client; only functions `empty`, `fromList`, and `singleton` (the latter to create a single-element list). That's because the Haskell module system (of which soon) supports **data hiding**, which lets the creator of the library select what to export and what to keep private (here, the constructors of `Map`). 

The symbol table must not only be threaded through all recursive invocations of `eval`. It must also be passed between invocations of the main tokenize/parse/evaluate loop. We want user-defined variables to store values throughout the session. Here's the modified top-level of our calculator that ensures the longevity of the symbol table:

``` haskell
main = do
   loop (M.fromList [("pi", pi)])

loop symTab = do
   str <- getLine
   if null str
   then
      return ()
   else
      let toks = tokenize str
          tree = parse toks
          (val, symTab') = evaluate tree symTab
      in do
          print val
          loop symTab'
```
We pass `symTab` to `evaluate` and obtain a new symbol table `symTab'`, together with the result. We then pass this modified symbol table, with user defined and modified symbolic variables, to the next iteration of the loop. 

### Symbol Table Lookup

For uniformity, we will encapsulate both the symbol table lookup and insertion in functions that have a very similar signature to `evaluate` itself:

``` haskell
evaluate :: Tree -> SymTab -> (a, SymTab)
```

For generality, I replaced `Double` with a type parameter `a`. This will come in handy when defining `addSymbol`. 

Here's the lookup (we substituted `String` for `Tree` in the signature):

``` haskell
lookUp :: String -> SymTab -> (Double, SymTab)
lookUp str symTab = 
    case M.lookup str symTab of
      Just v -> (v, symTab)
      Nothing -> error $ "Undefined variable " ++ str
```
We are calling the function `M.lookup` from the `Map` package. This function may fail if the key is not present in the map. So let's talk about failure.

Programmers come up with lots of ways of implementing failure -- often very hacky. For instance you may return a "special" value, like -1, an empty string, or a null pointer. The problem with special values is that it's easy to forget to check for them in the calling code. And cases of inadvertently dereferencing a null pointer or accessing the minus-first element of an array are endemic. 

In Haskell there are many ways of dealing with failure or, as it's sometimes called, with *partial functions* -- functions that are not defined for all possible arguments. The simplest such method is to return a `Maybe` value. `Maybe` is defined in the Prelude as:

``` haskell
data Maybe a = Nothing | Just a
```
It works for any type `a`, so it can be used to encapsulate any result. What's important is that you can hardly forget to check the result of a function returning a `Mabye`, because, in order to get at the value inside it, you are virtually forced to do pattern matching. We've done that in the implementation of `lookUp`: we pattern matched on the constructor `Just` to retrieve the value of the type `Double`; and we pattern-matched on `Nothing` to display an error. We'll implement a more sophisticated error checking later but for now, exiting with an error message will do.

You may find versions of `Mabye` in other languanges as well: `boost::optional` in C++, nullable in C#, `option` in ML, and so on.

### Symbol Table Update

Here's the function `addSymbol`, which inserts a new string/value pair into the symbol table. The function `M.insert` will overwrite the previous pair, if the key already exists in the map. That's why we can use `addSymbol` for both creating new symbolic variables and modifying them later.

``` haskell
addSymbol :: String -> Double -> SymTab -> ((), SymTab)
addSymbol str val symTab = 
    let symTab' = M.insert str val symTab
    in ((), symTab')
```
Notice that, since there is no interesting value to be returned from `addSymbol`, I replaced `a` in the generic type signature with unit `()` (a.k.a., `void` in C-like languages).

As I explained before, the symbol table is not mutable: `insert` returns a new copy of it, which we then return from `addSymbol`, so it can be threaded through the rest of the code. 

Why did I choose the symbol-table-threading interface for `lookUp` and `addSymbol`? It's because I have bigger unification plans for them. You might have noticed that the code is becoming -- seemingly unnecessarily -- complicated because of Haskell's insistence on purity and immutability. Why can't Haskell compromise like other functional languages? THe answer is  that Haskell *doesn't have to*. I'm only introducing this complexity -- this imperative rot -- to later show you the ways you can get rid of it. In Haskell you can have the cake and eat it too -- It's called *monads*. In particular, this kind of threading of state from function to function can be easily hidden from view by using the appropriate monad. But monads are not magic! By the time we are through with the complex implementation, you'll know exactly what a monad hides under the hood. So bear with me a little longer.

### Implementing evaluate

The rest of the implementation of the evaluator is pretty straightforward. You just have to remember that every call might return a modified symbol table, and that you have to pass it to subsequent calls. Here's `evaluate` for the `SumNode`:

``` haskell
evaluate :: Tree -> SymTab -> (Double, SymTab)

evaluate (SumNode op left right) symTab = 
    let (lft, symTab')  = evaluate left symTab
        (rgt, symTab'') = evaluate right symTab'
    in
        case op of
          Plus  -> (lft + rgt, symTab'')
          Minus -> (lft - rgt, symTab'')
```

The evaluators for `VarNode` and the `AssignNote` are easily implemented in terms of the primitives `lookUp` and `addSymbol`:

``` haskell
evaluate (VarNode str symTab) = lookUp str symTab

evaluate (AssignNode str tree) symTab = 
    let (v, symTab')  = evaluate tree symTab
        (_, symTab'') = addSymbol str v symTab'
    in (v, symTab'')
```
The complete implementation of the calculator is at the end of this tutorial. 

## Module System

The calculator project is small enough to fit in a single file. Nevertheless, it's a good idea to split it into multiple modules. I have posted the multi-module version of the project on [my github](https://github.com/BartoszMilewski/SymCalc1), where you can view or clone it. In that version I put lexer, parser, and evaluator in separate modules/files. There is also a main module with `main` and the main `loop`. 

So far we've been only seeing library modules through `import` statements. Now we can see how these modules are created. A module corresponds to a file -- the name of the file (minus the extension .hs) is also the name of the module -- the compiler enforces it, except for single-file projects. The contents of each file starts with a module header, which contains the name of the module and an optional list of exports. If the exports are not listed, all top-level names are automatically exported. 

Here's the header of the `Lexer` module from the file `Lexer.hs`.  

``` haskell
module Lexer (Operator(..), Token(..), tokenize, lookAhead, accept) where
```
In parentheses, you see the list of exported definitions of data types and functions. When exporting a data type, you have the option to export none, some, or all of its constructors. For instance, I could have exported only `Plus` and `Minus` constructors of `Operator`, and no constructors of `Token`:

``` haskell
module Lexer (Operator(Plus, Minus), Token, tokenize, lookAhead, accept) where
```
If constructors are not exported, you can still use the type in other modules that import it, but you can't create or pattern match values of that type (that's exactly what to module `Data.Map` did). 

It's common to export all constructors of a given type, and that's what the double-dot shortcut is for. Here we have `Operator(..)` and `Token(..)`. 

When another module imports the module `Lexer`, all exported definitions become visible (unless you use selective imports, as below). Importing is not transitive, though. If `Parser` imports `Lexer` and `Main` imports `Parser`, the definitions from `Lexer` are not visible in `Main`. That's why the module `Main` had to import all of `Parser`, `Lexer`, and `Evaluator`:

``` haskell
module Main where

import qualified Data.Map as M
import Lexer (tokenize)
import Parser (parse)
import Evaluator (evaluate)
```
In `Main`, we only need one function from each module so we use *selective* imports by specifying a (comma-separated, if more than one) lists of imported declarations.

The module system of Haskell is much cleaner than, for instance, that of C++ or Java. Also, you don't need separate header files -- Haskell parser can quickly scan files for exports and imports.

## Expression Problem

In this particular project we've been dealing, so far, with multiple expression nodes and just one function, `evaluate`. In principle, though, we could come up with many other functions acting on nodes (see Ex 2). You can imagine a two-dimensional matrix of possibilites: the nodes on one axis, the functions on another. An interesting question is, how easy it is to extend this matrix. How easy it is to add more node types, and how easy it is to define new functions. 

If we were implementing the evaluator in an object-oriented language, chances are we would define an interface `Node` with a virtual function `evaluate`. All specific nodes would be described by classes that inherit from `Node`. Our `evaluate` would then be overridden separately for each type of node. 

Suppose you want to write an *extensible* library based on this scheme. In Haskell it would be easy for the clients of our library to implement new functions acting on nodes. Adding new node constructors, on the other hand, would require modifying the definition of `Tree` and adding a new pattern to all existing functions that operate on `Tree`, including the ones defined in the library module. Our library would be open to new functions but closed to new variations of data.

In an object oriented language, the opposite is true: adding a function requires the modification of the top `Node` and all its descendants -- a new virtual function has to be added to `Node`, and all its overrides in existing nodes have to be implemented. You can't do that without touching the library. However, adding a new node is easy: you just define a new class that inherits from `Node` and implement its virtual functions. You don't need to touch the library in order to add new nodes. Object-oriented libraries are open to new data types, but closed to new functions. 

This tradeoff is a symptom of the well known [expression problem](http://en.wikipedia.org/wiki/Expression_problem) faced by library writes. Fortunately, Haskell offers an elegant solution to the expression problem, which I will describe in the [next tutorial](https://www.fpcomplete.com/user/bartosz/basics-of-haskell/10_Error_Handling) (it requires type classes).

## Exercises

**Ex 1.** Implement function `translate`, which takes a dictionary and a list of strings and returns a list of translated strigs. If a string is not in a dictionary, it should be replaced with "whatchamacallit". For bonus points, try using the higher order `map` function from the Prelude, and the `where` clause. Remember that a function defined inside `where` has access to the arguments of the outer function.
``` active haskell
import qualified Data.Map as M

type Dict = M.Map String String

translate :: Dict -> [String] -> [String]
translate = undefined

testTranslation :: Dict -> IO ()
testTranslation dict = do
    print $ translate dict ["where", "is", "the", "colosseum"]

testInsertion :: Dict -> IO Dict
testInsertion dict = do
    return $ M.insert "colosseum" "colosseo" dict
    
main = 
    let dict = M.fromList [("where", "dove"), ("is", "e"), ("the", "il")]
    in do
          testTranslation dict
          dict'  <- testInsertion dict
          testTranslation dict'
          putStrLn "The original dictionary is unchanged:"
          testTranslation dict
```

@@@ Show solution

``` active haskell
import qualified Data.Map as M

type Dict = M.Map String String

translate :: Dict -> [String] -> [String]
translate dict words = map trans words
  where
    trans :: String -> String
    trans w =
      case M.lookup w dict of
        (Just w') -> w'
        Nothing   -> "whatchamacallit"

testTranslation :: Dict -> IO ()
testTranslation dict = do
    print $ translate dict ["where", "is", "the", "colosseum"]

testInsertion :: Dict -> IO Dict
testInsertion dict = do
    return $ M.insert "colosseum" "colosseo" dict
    
main = 
    let dict = M.fromList [("where", "dove"), ("is", "e"), ("the", "il")]
    in do
          testTranslation dict
          dict'  <- testInsertion dict
          testTranslation dict'
          putStrLn "The original dictionary is unchanged:"
          testTranslation dict
```
@@@

**Ex 2.** Implement function `paren` that takes an expression tree and turns it into a string with fully parenthesized expression. For instance, when acting on `testExpr` it should produce the string `(x = ((2.0 * (y = 5.0)) + 3.0))`

``` active haskell
data Operator = Plus | Minus | Times | Div
    deriving (Show, Eq)

data Tree = SumNode Operator Tree Tree
          | ProdNode Operator Tree Tree
          | AssignNode String Tree
          | UnaryNode Operator Tree
          | NumNode Double
          | VarNode String
    deriving Show

paren :: Tree -> String
paren = undefined

-- x = 2 * (y = 5) + 3
testExpr = AssignNode "x" (SumNode Plus 
                             (ProdNode Times 
                               (NumNode 2.0) 
                               (AssignNode "y" (NumNode 5)))
                             (NumNode 3))

main = print $ paren testExpr
```

@@@ Show solution
``` active haskell
data Operator = Plus | Minus | Times | Div
    deriving (Show, Eq)

data Tree = SumNode Operator Tree Tree
          | ProdNode Operator Tree Tree
          | AssignNode String Tree
          | UnaryNode Operator Tree
          | NumNode Double
          | VarNode String
    deriving Show

paren :: Tree -> String
paren (SumNode op left right) =
    case op of
      Plus  -> bin " + " left right
      Minus -> bin " - " left right

paren (ProdNode op left right) =
    case op of
      Times -> bin " * " left right
      Div   -> bin " / " left right

paren (AssignNode var tree) =
    let treeS = paren tree 
    in "(" ++ var ++ " = " ++ treeS ++ ")"

paren (UnaryNode op tree) =
    let treeS = paren tree
        opS = case op of 
                Plus -> " +"
                Minus -> " -"
    in "(" ++ opS ++ treeS ++ ")"

paren (NumNode x) = show x

paren (VarNode var) = var

bin :: String -> Tree -> Tree -> String
bin op left right =
    let leftS = paren left
        rightS = paren right
    in
        "(" ++ leftS ++ op ++ rightS ++ ")"

-- x = 2 * (y = 5) + 3
testExpr = AssignNode "x" (SumNode Plus 
                             (ProdNode Times 
                               (NumNode 2.0) 
                               (AssignNode "y" (NumNode 5)))
                             (NumNode 3))

main = print $ paren testExpr
```
@@@

** Ex 3.** The code below creates a frequency map of words in a given text. Fill in the implementation of `indexWords`, which counts the frequency of each word in a list of words; and `splitWords`, which splits a string into (lowercased) words, removing punctuation and newlines in the process. You might want to use the function <hoogle search="Data.Map.findWithDefault">findWithDefault</hoogle> from `Data.Map` and the function <hoogle search="Prelude.words">words</hoogle> from the Prelude. For bonus points try using <hoogle search="Prelude.map">map</hoogle> and <hoogle search="Prelude.foldl">foldl</hoogle>.

``` active haskell
{-# START_FILE main.hs #-}
import qualified Data.Map as M
import Data.Char (toLower)
import Data.List (sortBy)

type Index = M.Map String Int

indexWords ::  Index -> [String] -> Index
indexWords = undefined

splitWords :: String -> [String]
splitWords = undefined

mostFrequent :: [String] -> [(String, Int)]
mostFrequent wrds =
    let index = indexWords M.empty wrds
    in take 9 (sortBy cmpFreq (M.toList index))
  where
    cmpFreq :: (String, Int) -> (String, Int) -> Ordering
    cmpFreq (w1, n1) (w2, n2) = compare n2 n1

main = do
    text <- readFile "moby.txt"
    print $ mostFrequent (splitWords text)

{-# START_FILE moby.txt #-}
Call me Ishmael. Some years ago - never mind how long precisely - having little 
or no money in my purse, and nothing particular to interest me on shore, I thought 
I would sail about a little and see the watery part of the world. It is a way 
I have of driving off the spleen, and regulating the circulation. Whenever I find 
myself growing grim about the mouth; whenever it is a damp, drizzly November 
in my soul; whenever I find myself involuntarily pausing before coffin warehouses, 
and bringing up the rear of every funeral I meet; and especially whenever 
my hypos get such an upper hand of me, that it requires a strong moral principle 
to prevent me from deliberately stepping into the street, and methodically 
knocking people's hats off - then, I account it high time to get to sea as soon as I can.
```

@@@ Show solution

``` active haskell
{-# START_FILE main.hs #-}
import qualified Data.Map as M
import Data.Char (toLower)
import Data.List (sortBy)

type Index = M.Map String Int

indexWords ::  Index -> [String] -> Index
indexWords index = 
    foldl acc index 
  where
    acc :: Index -> String -> Index
    acc ind word = 
      let n = M.findWithDefault 0 word ind in
      M.insert word (n + 1) ind

splitWords :: String -> [String]
splitWords = words . map (\c -> if elem c ".,;-\n" then ' ' else toLower c)

mostFrequent :: [String] -> [(String, Int)]
mostFrequent wrds =
    let index = indexWords M.empty wrds
    in take 9 (sortBy cmpFreq (M.toList index))
  where
    cmpFreq :: (String, Int) -> (String, Int) -> Ordering
    cmpFreq (w1, n1) (w2, n2) = compare n2 n1

main = do
    text <- readFile "moby.txt"
    print $ mostFrequent (splitWords text)

{-# START_FILE moby.txt #-}
Call me Ishmael. Some years ago - never mind how long precisely - having little 
or no money in my purse, and nothing particular to interest me on shore, I thought 
I would sail about a little and see the watery part of the world. It is a way 
I have of driving off the spleen, and regulating the circulation. Whenever I find 
myself growing grim about the mouth; whenever it is a damp, drizzly November 
in my soul; whenever I find myself involuntarily pausing before coffin warehouses, 
and bringing up the rear of every funeral I meet; and especially whenever 
my hypos get such an upper hand of me, that it requires a strong moral principle 
to prevent me from deliberately stepping into the street, and methodically 
knocking people's hats off - then, I account it high time to get to sea as soon as I can.
```
@@@

## The Symbolic Calculator So Far

Below is the source code for the whole project: 

``` active haskell
{-# START_FILE Main.hs #-}
module Main where

import qualified Data.Map as M
import Lexer (tokenize)
import Parser (parse)
import Evaluator (evaluate)

main = do
   loop (M.fromList [("pi", pi), ("e", exp 1.0)])

loop symTab = do
   str <- getLine
   if null str
   then
      return ()
   else
      let toks = tokenize str
          tree = parse toks
          (val, symTab') = evaluate tree symTab
      in do
          print val
          loop symTab'
{-# START_FILE Lexer.hs #-}
module Lexer (Operator(..), Token(..), tokenize, lookAhead, accept) where

import Data.Char

data Operator = Plus | Minus | Times | Div
    deriving (Show, Eq)

data Token = TokOp Operator
           | TokAssign
           | TokLParen
           | TokRParen
           | TokIdent String
           | TokNum Double
           | TokEnd
    deriving (Show, Eq)

lookAhead :: [Token] -> Token
lookAhead [] = TokEnd
lookAhead (t:ts) = t

accept :: [Token] -> [Token]
accept [] = error "Nothing to accept"
accept (t:ts) = ts

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

identifier :: Char -> String -> [Token]
identifier c cs = let (name, cs') = span isAlphaNum cs in
                  TokIdent (c:name) : tokenize cs'

number :: Char -> String -> [Token]
number c cs = 
   let (digs, cs') = span isDigit cs in
   TokNum (read (c : digs)) : tokenize cs'

operator :: Char -> Operator
operator c | c == '+' = Plus
           | c == '-' = Minus
           | c == '*' = Times
           | c == '/' = Div
{-# START_FILE Parser.hs #-}
module Parser (Tree(..), parse)where

import Lexer

data Tree = SumNode Operator Tree Tree
          | ProdNode Operator Tree Tree
          | AssignNode String Tree
          | UnaryNode Operator Tree
          | NumNode Double
          | VarNode String
    deriving Show

parse :: [Token] -> Tree
parse toks = let (tree, toks') = expression toks
             in
               if null toks' 
               then tree
               else error $ "Leftover tokens: " ++ show toks'

expression :: [Token] -> (Tree, [Token])
expression toks = 
   let (termTree, toks') = term toks
   in
      case lookAhead toks' of
         (TokOp op) | elem op [Plus, Minus] -> 
            let (exTree, toks'') = expression (accept toks') 
            in (SumNode op termTree exTree, toks'')
         TokAssign ->
            case termTree of
               VarNode str -> 
                  let (exTree, toks'') = expression (accept toks') 
                  in (AssignNode str exTree, toks'')
               _ -> error "Only variables can be assigned to"
         _ -> (termTree, toks')

term :: [Token] -> (Tree, [Token])
term toks = 
   let (facTree, toks') = factor toks
   in
      case lookAhead toks' of
         (TokOp op) | elem op [Times, Div] ->
            let (termTree, toks'') = term (accept toks') 
            in (ProdNode op facTree termTree, toks'')
         _ -> (facTree, toks')

factor :: [Token] -> (Tree, [Token])
factor toks = 
   case lookAhead toks of
      (TokNum x)     -> (NumNode x, accept toks)
      (TokIdent str) -> (VarNode str, accept toks)
      (TokOp op) | elem op [Plus, Minus] -> 
            let (facTree, toks') = factor (accept toks) 
            in (UnaryNode op facTree, toks')
      TokLParen      -> 
         let (expTree, toks') = expression (accept toks)
         in
            if lookAhead toks' /= TokRParen 
            then error "Missing right parenthesis"
            else (expTree, accept toks')
      _ -> error $ "Parse error on token: " ++ show toks

{-# START_FILE Evaluator.hs #-}
module Evaluator (evaluate) where

import Lexer
import Parser
import qualified Data.Map as M

type SymTab = M.Map String Double

evaluate :: Tree -> SymTab -> (Double, SymTab)

evaluate (SumNode op left right) symTab = 
    let (lft, symTab') = evaluate left symTab
        (rgt, symTab'') = evaluate right symTab'
    in
        case op of
          Plus  -> (lft + rgt, symTab'')
          Minus -> (lft - rgt, symTab'')

evaluate (ProdNode op left right) symTab = 
    let (lft, symTab') = evaluate left symTab
        (rgt, symTab'') = evaluate right symTab'
    in
        case op of
          Times -> (lft * rgt, symTab)
          Div   -> (lft / rgt, symTab)

evaluate (UnaryNode op tree) symTab =
    let (x, symTab') = evaluate tree symTab
    in case op of
         Plus  -> (x, symTab')
         Minus -> (-x, symTab')

evaluate (NumNode x) symTab = (x, symTab)

evaluate (VarNode str) symTab = lookUp str symTab

evaluate (AssignNode str tree) symTab = 
    let (v, symTab')  = evaluate tree symTab
        (_, symTab'') = addSymbol str v symTab'
    in (v, symTab'')

lookUp :: String -> SymTab -> (Double, SymTab)
lookUp str symTab = 
    case M.lookup str symTab of
      Just v -> (v, symTab)
      Nothing -> error $ "Undefined variable " ++ str

addSymbol :: String -> Double -> SymTab -> ((), SymTab)
addSymbol str val symTab = 
    let symTab' = M.insert str val symTab
    in ((), symTab')

```



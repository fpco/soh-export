# 8. Parser

I the [previous installment](https://www.fpcomplete.com/user/bartosz/basics-of-haskell/7-tokenizer-higher-order-functions), we finished implementing the tokenizer, a.k.a., the lexer. We implemented it as a (pure) function that takes a string of characters and produces a list of tokens. This list of tokens will now serve as the input to the next stage of our calculator, the parser, which applies the rules of grammar to tokens in order to create an expression tree. We already have the the signature for our parser:

``` haskell
parse :: [Token] -> Expression
```
What remains is to define the approapriate data structures and implement the parsing functions.

## The Grammar

Let's first define a simple grammar using some sort of BNF notation. We expect the result of parsing a line of user input to be an Expression. We'll consider three forms of expressions:

1. An additive expression that starts with a Term followed by either plus or minus, followed by another expression. This is a typical recursive definition for parsing expressions of the type `Term + Term - Term ... etc.`. Example: `x - 5 + y`.
2. An assignment of the form: Identifier equals Expression. For simplicity, I chose to treat an assignment as an expression, as it is in C++, rather than a separate statement. Example: `x = 2 + 2`
3. Finally, a lonely Term is also considered an expression. Example: `44`.

Terms are more tightly bound than expressions, corresponding to higher precedence of multiplicative vs. additive operators. We'll consider two forms or terms:

1. Factor followed by a multiplication or division sign, followed by another term. This production corresponds to terms of the form `Factor * Factor / Factor ...`. Example: `2 * x / 2`.
2. A Term could also be a lonely Factor. Example: `44`.

Finally, the most tightly bound production is that of the Factor, which can be one of:

1. A Number, like `147`
2. An Identifier (variable name), like `x11`
3. A unary plus or minus in front of a Factor, like `-x` or `+12`. You can convince yourself that there is no ambiguity between binary and unary uses of these operators.
4. A parenthesized Expression, like `(a + b/2)`

Here's the more formal description of this grammar:

```
Expression <- Term [+-] Expression 
            | Identifier '=' Expression
            | Term
Term       <- Factor [*/] Term
            | Factor
Factor     <- Number 
            | Identifier 
            | [+-] Factor 
            | '(' Expression ')' 
```

To be honest, there's a slight problem with this grammar: the associativity is wrong. Operators of the same precedence associate to the right rather than to the left so, for instance, `5 - 3 + 2` is interpreted as `5 - (3 + 2)`. There is a way to transform this grammar to the left-associative one, but it comes at a cost of making the parser slightly more complicated, so I'll leave it as an exercise to the reader.

## The Parse Tree

The productions of this grammar map nicely into the structure of a tree. For instance, the leaf nodes are either variables or numbers. Expressions generate binary addition/subtraction nodes. Terms produce binary multiplication/division nodes. There are also nodes for unary plus or minus, and the assignment node. We end up with this tree-like recursive data structure:

``` haskell
data Tree = SumNode Operator Tree Tree
          | ProdNode Operator Tree Tree
          | AssignNode String Tree
          | UnaryNode Operator Tree
          | NumNode Double
          | VarNode String
  deriving Show
```

Our grammar will always produce parse trees of this type. The opposite is not true: some trees are invalid according to our grammar. (This kind of mismatch tells us that we have to be prepared to deal with runtime parsing errors.)

Let's have a look at a simple example. This input:

``` haskell
"x1 = -15 / (2 + x2)"
```
should produce the following parse tree:

![Parse Tree](http://bartosz.com/images/Basics/ParseTree.PNG)

## Top-Down Recursive Parsing

Our grammar is ideal for top-down recursive parsing. It doesn't use left recursion and every production requires only one lookahead token. If you're not familiar with the theory of parsing, all you need to know that there is an almost trivial mapping from grammar to implementation. Each production is translated into a function, and each function needs only to look at one token to decide which branch to take. We will need three parsing functions: `expression`, `term`, and `factor`, corresponding to the three productions of our grammar. These functions will consume tokens and produce nodes of the parse tree. In imperative languages, token consumption is implemented using some kind of mutable pointer or a call to a stateful function. Can we implement parsing using pure functions? 

### State without Mutation

At this point I could tell you to use the State monad and be done with it. Instead I will show you how to solve this problem with pure functions and discover the State monad later in the process. 

The trick is to make all the parsing functions not only take the token list as input, but also return the unconsumed part of the token list attached to the output. We've done this on a smaller scale before, with functions like `alnums`, `digits`, and their generalization, `span`. Except that now we'll apply this pattern across a whole family of functions.

Here are the type signatures for our (pure) parsing functions:

``` haskell
expression :: [Token] -> (Tree, [Token])
term       :: [Token] -> (Tree, [Token])
factor     :: [Token] -> (Tree, [Token])
```

It's usually a good idea to provide some helper functions to access tokens. Traditionally, you'd want to look one token ahead and, when you use it in a production, call `accept` to remove it from the list (actually, return the tail of the list).

``` haskell
lookAhead :: [Token] -> Token
lookAhead [] = TokEnd
lookAhead (c:cs) = c

accept :: [Token] -> [Token]
accept [] = error "Nothing to accept"
accept (t:ts) = ts
```

I introduced a new sythesized token, `TokEnd`. The alternative would be to check for the end of input every time you access the token list, but that would complicate the code unnecessarily. Since we recognize tokens using pattern matching, `TokEnd` will usually be dealt with by the fallthrough wildcard pattern.

A word about performance: Behind the scenes, a list is implemented using pointers. Since the list is guaranteed never to be modified, accessing the tail of the list, as in `accept`, doesn't incur the overhead of copying. Instead it returns a pointer to the second element of the list (or null). If the original list is no longer accessible (goes out of scope), the garbage collector will eventually free the skipped element. In general, because of immutability, garbage collectors in Haskell can be more efficient and easily parallelized. 


## Expression

As I mentioned earlier, translating a grammar to a top-down recursive parser is almost mechanical. Here's the production for Expression:
```
Expression <- Term [+-] Expression 
            | Identifier '=' Expression
            | Term
```
And here's its translation into a parser: We'll first try to parse a Term by calling the function `term`. Then we'll look ahead at the next token. If it's a `TokOp` containing `Plus` or `Minus` we will create a `SumNode`. If it's a `TokAssign`, we'll create an `AssignNode`. Otherwise we'll just return the lonely Term. 

With each node, we have to also return the remaining token list.

``` haskell
expression :: [Token] -> (Tree, [Token])
expression toks = 
   let (termTree, toks') = term toks
   in
      case lookAhead toks' of
         -- Term [+-] Expression
         (TokOp op) | elem op [Plus, Minus] -> 
            let (exTree, toks'') = expression (accept toks') 
            in (SumNode op termTree exTree, toks'')
         -- Identifier '=' Expression
         TokAssign ->
            case termTree of
               VarNode str -> 
                  let (exTree, toks'') = expression (accept toks') 
                  in (AssignNode str exTree, toks'')
               _ -> error "Only variables can be assigned to"
         -- Term
         _ -> (termTree, toks')
```
A reminder: `TokAssign` was introduced in an exercise in the last tutorial. 

I used the Haskell construct, `case`/`of` for pattern matching. It is analogous to the pattern matching used in defining multiple bodies for a function, but can be used as an expression anywhere in your code. 

The code between `case` and `of` is an expression that is to be pattern matched. The block following `of` lists the patterns, each of them followed by an arrow `->` and an expression. All expressions have to be of the same type. The value of the whole `case`/`of` exrpession is the result of the first expression whose pattern was matched. 

Notice that the assignment branch, `Identifier '=' Expression`, requires that the left hand side (the `termNode`) be a `VarNode`. This condition is checked at runtime and, if not met, results in an error.

## Term

This is the grammar rule:

```
Term       <- Factor [*/] Term
            | Factor
```

and that's its straightforward translation into code:

``` haskell
term :: [Token] -> (Tree, [Token])
term toks = 
   let (facTree, toks') = factor toks
   in
      case lookAhead toks' of
         (TokOp op) | elem op [Times, Div] ->
            let (termTree, toks'') = term (accept toks') 
            in (ProdNode op facTree termTree, toks'')
         _ -> (facTree, toks')
```

## Factor

We proceed with the Factor production the same way:

```
Factor     <- Number 
            | Identifier 
            | [+-] Factor 
            | '(' Expression ')' 
```

Again, tokens `LParen` and `RParen` were introduced in an exercise in the previous tutorial.

``` haskell
factor :: [Token] -> (Tree, [Token])
factor toks = 
   case lookAhead toks of
      (TokNum x)     -> (NumNode x, accept toks)
      (TokIdent str) -> (VarNode str, accept toks)
      (TokOp op) | elem op [Plus, Minus] -> 
            let (facTree, toks') = factor (accept toks) 
            in (UnaryNode facTree, toks')
      TokLParen      -> 
         let (expTree, toks') = expression (accept toks)
         in
            if lookAhead toks' /= TokRParen 
            then error "Missing right parenthesis"
            else (expTree, accept toks')
      _ -> error $ "Parse error on token: " ++ show toks

```

## Parser

The parsing starts at the top. We are expecting an Expression at the top level, so we call `expression` and pattern match the result. The expression should exhaust all tokens -- if it doesn't, it's an error. 

``` haskell
parse :: [Token] -> Tree
parse toks = let (tree, toks') = expression toks
             in
               if null toks' 
               then tree
               else error $ "Leftover tokens: " ++ show toks'
```

The function `null` returns `True` if the argument is an empty list. 

Finally, here's the complete parser together with the (extended) tokenizer. The implementation is pretty straightforward but it contains a lot of boilerplate code. Simplifying it by learning common Haskell idioms will be the goal of the next few tutorials.

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

main = (print . parse . tokenize) "x1 = -15 / (2 + x2)"
```

The output of the test program, after some manual formatting, should look like this (compare with the earlier picture):

```
AssignNode "x1" 
  (ProdNode Div 
     (UnaryNode Minus 
        (NumNode 15.0)) 
     (SumNode Plus 
        (NumNode 2.0)
        (VarNode "x2")))
```

In the next installment, we'll implement the evaluator for our expression trees. 

## Exercises

**Ex 1.** The shape of a binary tree may be encoded using matching pairs of parentheses. The string of parentheses obtained this way matches the following grammar:

```
Root  <- Par
Expr  <- Par Par
Par   <- '(' Expr ')'
       | '(' ')'
```
Write a parser based on this grammar:

``` active haskell
data Token = TokLParen | TokRParen | TokEnd
    deriving (Show, Eq)

lookAhead :: [Char] -> Token
lookAhead [] = TokEnd
lookAhead (c:cs)| c == '(' = TokLParen
                | c == ')' = TokRParen
                | otherwise = error $ "Bad input: " ++ (c:cs)

accept :: [Char] -> [Char]
accept [] = error "Nothing to accept"
accept (c:cs) = cs

data Tree = Node Tree Tree | Leaf
    deriving Show

root, expr, par :: [Char] -> (Tree, [Char])

root = undefined
expr = undefined
par  = undefined

parse str = let (tree, str') = root str
            in
                if null str' 
                then tree 
                else error $ "Unconsumed string " ++ str'

main = print $ parse "(()(()()))"
        
```

@@@ Show solution
``` active haskell
data Token = TokLParen | TokRParen | TokEnd
    deriving (Show, Eq)

lookAhead :: [Char] -> Token
lookAhead [] = TokEnd
lookAhead (c:cs)| c == '(' = TokLParen
                | c == ')' = TokRParen
                | otherwise = error $ "Bad input: " ++ (c:cs)

accept :: [Char] -> [Char]
accept [] = error "Nothing to accept"
accept (c:cs) = cs

data Tree = Node Tree Tree | Leaf
    deriving Show

root, expr, par :: [Char] -> (Tree, [Char])

root = par

expr toks = 
    let (p, toks')   = par toks 
        (p', toks'') = par toks'
    in (Node p p', toks'')
    
par toks = 
    case lookAhead toks of
      TokLParen ->
        case lookAhead (accept toks) of
          TokRParen -> (Leaf, accept (accept toks))
          _ -> let (e, toks') = expr (accept toks) 
               in  if lookAhead toks' == TokRParen
                   then (e, accept toks')
                   else error $ "Missing closing paren in: " ++ show toks'
      _ -> error $ "Bad expression: " ++ show toks

parse str = let (tree, str') = root str
            in
                if null str' 
                then tree 
                else error $ "Unconsumed string " ++ str'

main = print $ parse "(()(()()))"
        
```
@@@

**Ex 2.** Write a parser that splits a string into a list of words using space characters as separators (use function `isSpace`). 

``` active haskell
import Data.Char

type Word = String

sentence :: String -> [Word]
sentence = undefined

-- returns a word and the rest of input
word :: String -> (Word, String)
word = undefined
                          
main = print $ sentence "Ceci n'est pas une phrase"
```

@@@ Show solution
``` active haskell
import Data.Char

type Word = String

sentence :: String -> [Word]
sentence "" = []
sentence str = let (w, str') = word str
               in w : sentence str'

word :: String -> (Word, String)
word "" = ("", "")
word (c:cs) | isSpace c = ("", cs)
            | otherwise = let (w, cs') = word cs
                          in (c:w, cs')

main = print $ sentence "Ceci n'est pas une phrase"
```
You can also implement `word` using `span`:

``` haskell
word str = let (w, str')  = span (not . isSpace) str
               (_, str'') = span isSpace str'
           in (w, str'')
```
@@@

**Ex 3.** Generalize the `sentence` parser from (Ex 2) to take a pluggable parser. The new function is called `several` and takes as an argument a generic function `String->(a, String)`, which is supposed to parse a string and return the result of type `a` together with the leftover string. Use it to split a string into a list of numbers.

``` active haskell
import Data.Char

type Parser a = String -> (a, String)

several :: Parser a -> String -> [a]
several = undefined

num :: Parser Int
num str = undefined

main = print $ several num "12 4 128"
```

@@@ Show solution
``` active haskell
import Data.Char

type Parser a = String -> (a, String)

several :: Parser a -> String -> [a]
several p "" = []
several p str = let (a, str') = p str
                    as = several p str'
                in a:as

num :: Parser Int
num str = 
    let (digs, str') = span isDigit str
        (_, str'')   = span isSpace str'
    in (read digs, str'')

main = print $ several num "12 4 128"
```
@@@
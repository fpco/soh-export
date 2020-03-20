# 5. Tokenizer: Data Types

In the [previous tutorial](https://www.fpcomplete.com/user/bartosz/basics-of-haskell/4-symbolic-calculator-recursion) I sketched the desing of a calculator and implemented the top-level input/output loop. This is a typical pattern in Haskell: the top level is implemented in the `IO` monad (after all, the signature of `main` is `IO ()`) but, as you descend to the lower levels, you enter the realm of side-effect-free pure functions. The first such function is `tokenize` with the following signature:

``` haskell
tokenize :: String -> [Token]
```
Before we can start implementing it, we have to define the `Token` data type and learn more about `String`s. 

## Haskell Data Types

There is one major difference between data in imperative languages and data in Haskell. Haskell data is immutable. Once you construct a data item, it will forever stay the same. 

Well, it's not entirely true because of another property of Haskell: laziness. Calling a constructor of a data type is not the same as *evaluating* it. It's only when you actually peek inside a data item that the constructor is evaluated, and only the part that you're looking at. 

But for all intents and purposes, the state of a data item remains frozen after its construction. Moreover, every data item *remembers* the way it's been constructed. It remembers which constructor was used and what values were passed to it. 

But how can you write programs without mutable data? Actually, those of us who had to deal with concurrent programming in imperative languages had to learn (often the hard way) to eschew mutability whenever possible. The fewer opportunities for those hard to reproduce and debug low-level data races, the more reliable your code. This is one more reason to learn programming in Haskell even if your job requries the use of imperative languages: You'll learn how to solve problems without mutable variables.

In Haskell you'll often see mutation replaced by construction. Instead of modifying one element of a data structure, you construct a copy of it with the appropriate change in place. This trick could be prohibitively expensive if you use the wrong data structures. We'll be steering away from such data structures in favor of the so called *persistent* data structures, which don't require a lot of copying when they are modified. For instance, the workhorse of Haskell data structures is the list, not the array of the vector. We'll talk more about this later.

## Enumerated Data Types

The simplest data types just enumerate all possible values. For instance, `Bool` is an enumeration of `True` and `False` (as defined in the Prelude, the Haskell's standard library):

``` haskell
data Bool = True | False
```

A data structure definition is introduced by the keyword `data`. `Bool` is the name of the type we are defining. The right hand side of the equal sign lists the *constructors* separated by vertical bars. When you create a new `Bool` value, you use one of these two constructors. Constructor names must start with a capital letter and must be unique per file (two data structures can't share the same constructor name). 

When you want to inspect a `Bool` value, you *match* it with one of the constructors (remember, a value *remembers* how it was constructed). There are several ways of matching values to constructors in Haskell. Let's start with the simplest one: Defining a function using multiple equations. Instead of defining a function with one equation, like this:

``` active haskell
boolToInt :: Bool -> Int
boolToInt b = if b then 1 else 0

main = print $ boolToInt False
```
you may split it into two equations corresponding to two constructor patterns, `True` and `False`:

``` active haskell
boolToInt :: Bool -> Int
boolToInt True  = 1
boolToInt False = 0

main = print $ boolToInt False
```

Patterns are matched in order, so when `boolToInt` is called with `False`, the runtime first tries to match it to `True` and fails, so it moves to the second pattern `False` and succeeds. (All equations for the same function must be consecutive.)

(Note: In order to save on parentheses I will start using the function application operator `$` that I introduced in the first tutorial. It's been a long time, so here's a quick recap: `$` separates a function call from its argument. It's very useful when the argument is another function call, because function calls bind to the left. In our example, without the `$` or parenteheses, the function calls would bind: `(print boolToInt) False`, and would fail to compile. Operator `$` has very low precedence so the thing to its right will be evaluated before the function to the left is called, and it binds to the right.)

Here's a useful enumeration that we will use in our project:

``` haskell
data Operator = Plus | Minus | Times | Div
```

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

## Token

Our tokenizer should recognize operators, identifiers, and numbers. We can enumerate the four operators, but we can't enumerate all possible indentifiers or numbers. For those tokens we need to store additional information: a `String` and an `Int` respectively. Here's the definition of `Token`:

``` haskell
data Token = TokOp Operator
           | TokIdent String
           | TokNum Int
    deriving (Show, Eq)
```
All three constructors now take arguments. The `TokOp` constructor takes a value of the type `Operator`, `TokIdent` takes a `String`, and `TokNum` takes an `Int`. For instance, you can create a `Token` using (`TokIdent "x"`), etc.

I'll explain the `deriving` clause in more detail when we talk about type classes. For now it will suffice to know that `deriving` `Show` means that there is a way to convert any `Token` to string (either by calling `show` or by `print`'ing it), and `deriving` `Eq` means that we can compare `Token`s for (in-)equality. The compiler is clever enough to implement this functionality all by itself (if it can't, it will issue an error).

Pattern matching on these constructors is more interesting: We not only match the constructor name but also the value with which it was originally called. Here's a definition of a function `showContent` that uses this kind of pattern matching:

``` active haskell
-- show
data Token = TokOp Operator
           | TokIdent String
           | TokNum Int
    deriving (Show, Eq)

showContent :: Token -> String
showContent (TokOp op) = opToStr op
showContent (TokIdent str) = str
showContent (TokNum i) = show i

token :: Token
token = TokIdent "x"

main = do
    putStrLn $ showContent token
    print token
-- /show
data Operator = Plus | Minus | Times | Div
    deriving (Show, Eq)

opToStr :: Operator -> String
opToStr Plus  = "+"
opToStr Minus = "-"
opToStr Times = "*"
opToStr Div   = "/"
```

Notice that non-trivial constructor patterns require parentheses. In these patterns the argument to the constructor is replaced by a (lower-case) variable that is to be bound to the value stored inside the `Token`. For instance, in the `(TokIdent str)` pattern, `str` will be bound to the string that was used in the construction of the matched token. If the token was constructed using `TokIdent "x"`, `str` will be bound to `"x"`. (For immutable variables we prefer to use the word "bind" rather than "assign.")

In general, constructors may take many arguments of various types, and they can all be matched by patterns.

**Ex 2**.
Define a data type `Point` with one constructor `Pt` that takes two `Double`s, corresponding to the x and y coordinates of a point. Write a function `inc` that takes a `Point` and returns a new `Point` whose coordinates are one more than the original coordinates. Use pattern matching.

``` active haskell
data Point = Pt ... 
    deriving Show

inc :: Point -> Point
inc ... = ...

p :: Point
p = Pt (-1) 3

main = print $ inc p
```
@@@ Show solution
``` active haskell
data Point = Pt Int Int 
    deriving Show

inc :: Point -> Point
inc (Pt x y) = Pt (x + 1) (y + 1)

p :: Point
p = Pt (-1) 3

main = print $ inc p

```
@@@

By the way, we've seen pattern matching previously applied to pairs. The constructor of a pair is `(,)`. 

**Ex 3**.
Solve the previous exercise using pairs rather than `Point`s.
``` active haskell
inc :: (Int, Int) -> (Int, Int)
inc ... = ...

p :: (Int, Int)
p = ...

main = print $ inc p
```
@@@ Show solution
``` active haskell
inc :: (Int, Int) -> (Int, Int)
inc (x, y) = (x + 1, y + 1)

p :: (Int, Int)
p = (-1, 3)

main = print $ inc p
```
@@@

## Lists and Recursion

In Haskell a `String` is a list of characters. Admittedly, list storage and processing is less space/time efficient than the processing of arrays of characters in imperative languages. However, unless your application is string-intensive, the convenience of list manipulation overcomes these shortcomings. And it's easy enough to replace `String` with the more efficient array-based `ByteString` in string-intensive applications. 

Since we'll be manipulating strings -- and strings are list of characters -- we need to learn about lists first.

First we have to ask ourselvest: What is a list? If you're thinking, "Singly-linked or doubly-linked?", you are talking about implementation, not the essence of a list. So what's the essence of a list? Like any abstract data type, list is defined by operations you can perform on it. The most essential operation is the *creation* of a list. 

One should be able to create a new list by prepending an element to an existing list. This operation is often called "cons," a word taken from Lisp jargon. Notice that this definition is self-referential -- you create a list from a list. To start somewhere, you should also be able to create a list from nothing -- an empty list. Here's a definition of a list of integers that is based just on this description:

``` haskell
data List = Cons Int List | Empty
```
The fact that this definition is recursive shouldn't bother us in the least. The important thing is that it lets us create arbitrary lists:

``` haskell
lst0, lst1, lst2 :: List
lst0 = Empty        -- empty list
lst1 = Cons 1 lst0  -- one-element list
lst2 = Cons 2 lst1  -- two-element list
```
This definition can also be used in pattern matching. For instance, here's a function that checks if a list is a singleton:
``` active haskell
data List = Cons Int List | Empty

singleton :: List -> Bool
singleton (Cons _ Empty) = True
singleton _ = False

main = do
   print $ singleton Empty
   print $ singleton $ Cons 2 Empty
   print $ singleton $ Cons 3 $ Cons 4 Empty
```
In this example, I made use of a wildcard pattern `_`. Let me remind you that his pattern matches anything (without evaluating it). For instance, in the first clause of `singleton` I'm discarding the integer stored in the list. In the second clause I'm ignoring the whole list, because I know that the first clause, which catches one-element lists, is tried first. 

Most importantly, because list is defined recursively, it's easy to implement recursive algorithms for it. For instance, to calculate the sum of all list elements it's enough to say that the sum is equal to the first element plus the sum of the rest. And, of course, the sum of an empty list is zero. So here we go:

``` active haskell
data List = Cons Int List | Empty

sumLst :: List -> Int
sumLst (Cons i rest) = i + sumLst rest
sumLst Empty = 0

lst = Cons 2 (Cons 4 (Cons 6 Empty))

main = do
   print (sumLst lst)
   print (sumLst Empty)
```

But you don't want to be defining a new list type for each possible element type. Fortunately, static polymorphism in Haskell is embarassingly easy. No need for the verbose `template<typename T>` ugliness. You just parameterize types by specifying a type argument. You may define a generic list by replacing `Int` by a type parameter `a` (type parameters must start with lower case and are typically taken from the beginning of the alphabet):

``` haskell
data List a = Cons a (List a) | Empty
```

`List a` in this definition is a *generic type*; `List` itself is called a *type constructor*, because you can use it to construct a new type by providing a type argument, as in `List Int`, or `List (List Char)` (a list of lists of characters). To avoid confusion, the constructors on the right hand side of a `data` definition are often called *data constructors*, as opposed to the *type constructor* on the left.

In reality, you don't need to define a list type -- its definition is built into the language, and it's syntax is very convenient. The type name for a list consists of a pair of square brackets with the type varaible between them; `Cons` is replaced by an infix colon, `:`; and the `Empty` list is an empty pair of square brackets, `[]`. You may think of the built-in list type as defined by this equation:
``` haskell
data [a] = a : [a] | []
```
Let me rewrite the previous example with this new notation:

``` active haskell
sumLst :: [Int] -> Int
sumLst (i : rest) = i + sumLst rest
sumLst [] = 0

lst = [2, 4, 6]

main = do
   print (sumLst lst)
   print (sumLst [])
```

There is another convenient feature: special syntax for list literals. Instead of writing a series of constructors, `2:8:64:[]`, you can write `[2, 8, 64]`.

Pattern matching may be *nested*. For instance, you may match the first three elements of a list with the pattern `(a : (b : (c : rest)))` or, taking advantage of the right associativity of `:`, simply `(a : b : c : rest)`.

Finally, this is the definition of `String`:

``` haskell
type String = [Char]
```
`String` comes with some syntactic sugar of its own: When defining string literals, you can write `"Hello"` instead of the more verbose `['H', 'e', 'l', 'l', 'o']` . 

Here, the `type` keyword introduces a *type synonym* (like the `typedef` in C). You can always go back and treat a `String` as a list of `Char` -- in particular, you may pattern match it like a list. We'll be doing a lot of this in the implementation of `tokenize`. Type synonyms increase the readability of code and lead to better error messages, but they don't create new types.

In the [next tutorial](https://www.fpcomplete.com/user/bartosz/basics-of-haskell/6-tokenizer-function-types) we'll continue to work on the tokenizer and learn about guards and touch upon currying.

## Exercises

**Ex 4**. Implement `norm` that takes a list of `Double`s and returns the square root (`sqrt`) of the sum of squares of its elements.

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

**Ex 5**. Implement the function `decimate` that skips every other element of a list. 
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

**Ex 6**. Implement a function that takes a pair of lists and returns a list of pairs. For instance `([1, 2, 3, 4], [1, 4, 9])` should produce `[(1, 1), (2, 4), (3, 9)]`. Notice that the longer of the two lists is truncated if necessary. Use nested patterns.

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
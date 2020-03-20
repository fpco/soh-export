# Basic Haskell Syntax

# Syntax elements

(Just so I don't forget: This web system sometimes hangs on perfectly correct Haskell and I'm still working out why...)

## Basic data types

Haskell has the basic components you'd expect:

``` active haskell
-- one-line comments begin with --

{- area comments are enclosed in {- -},
   which can be nested -}

-- integers
n = 10

-- floating point numbers
x = 2.718281828459045

-- Unicode strings
s = "Where is task force thirty four?"

-- tuples: fixed number of items, any types
t = (n, s, x)

-- lists: any number of items, same type
zs = [1, 2, 3, 4]

-- lists can also be constructed using the : operator and the empty list []
ws = 1 : 2 : 3 : 4 : []

-- lists as ranges
zRange = [0, 2 .. 10]

-- and getting stuff done:

main = do
    putStrLn s -- Output s with a new line at the end
    print s -- Show s as a string literal with a new line
    print x
    print zs
    print ws
    print zRange
```

## Function definitions and application

Function application is by juxtaposition or infix operators.  (Note: The compiler on this site is oddly picky.
You need spaces around `*` etc. for some reason, probably to do with markdown.)

``` active haskell
-- define a function f that takes one number and returns one number

f x = x ^ 2 + 3 * x + 1

x0 = 2.5
y0 = f 2.5

-- define a function g that takes two numbers and returns one number

g a b = a ^ 2 + b ^ 2

c = g 3 4

main = do
    print (x0, y0)
    print (3, 4, g 3 4)
```

You can also use the `\ ... -> ...` notation to create an anonymous function, also known as a lambda expression:

``` active haskell
-- define a function f that takes one number and returns one number

f = \x -> x ^ 2 + 3 * x + 1

x0 = 2.5
y0 = f 2.5

-- define a function g that takes two numbers and returns one number

g = \a b -> a ^ 2 + b ^ 2

c = g 3 4

main = do
    print (x0, y0)
    print (3, 4, g 3 4)
```


## Conditioning via if-then-else

The easiest conditioning is done with `if`-`then`-`else`:

``` active haskell
absoluteValue x =
    if x < 0 then -x else x
    
main = do
    print (absoluteValue 10)
    print (absoluteValue (-6.5)) -- unary minus is tricky :-(
    print (absoluteValue 0)
```

## Instead of loops

There are no looping constructions (`while`, `for`).  Instead, you use recursion and write looping functions:

``` active haskell
collatz n =
    if n == 1
    then 1 -- base case
    else
        if n `mod` 2 == 0   -- same as: mod n 2 == 0
        then collatz (n `div` 2)
        else collatz (3 * n + 1)
    
main = do
    print (collatz 1)
    print (collatz 2)
    print (collatz 3)
    print (collatz 4)
    print (collatz 5)
```

And you can do list comprehensions:

``` active haskell
f x = x ^ 2 + 3 * x + 1

points = [(x, f x) | x <- [-5 .. 5] ]

main = do
    print points
```

which can include a condition to filter the results:

``` active haskell
f x = x ^ 2 + 3 * x + 1

points =
    [(x, f x)
    | x <- [-10 .. 10],
      x `mod` 3 == 0  ]

main = do
    print points
```

You can also do Cartesian products of lists in comprehensions:

``` active haskell
square n =
    [(i, j)
    | i <- [0 .. n],
      j <- [0 .. n] ]

upperTriangle n =
    [(i, j)
    | i <- [0 .. n],
      j <- [i .. n] ]

evenSumSquare n =
    [(i, j)
    | i <- [0 .. n],
      j <- [0 .. n],
      0 == mod (i + j) 2]

main = do
    print (square 3)
    print (upperTriangle 3)
    print (evenSumSquare 3)
```

## Local definitions

Local definitions are built using `let`-`in` and `where`.  All definition groups are mutually recursive and can shadow symbols defined in a surrounding scope.  With `where`, the definitions come after the expression in which they are used:

``` active haskell
import Data.Complex

-- Solutions to a * x^2 + b * x + c == 0
quadraticSolve a b c = (rPlus, rMinus)
    where
        discriminant = sqrt (b ^ 2 - 4 * a * c)
        rPlus = ((-b) + discriminant) / (2 * a)
        rMinus = ((-b) - discriminant) / (2 * a)
        
main = do
    print (quadraticSolve 1 1 (-6))
    print (quadraticSolve 1 0 1)
    print (quadraticSolve (1 :+ 0) 0 (1 :+ 0))
    print (quadraticSolve (1 :+ 0) (1 :+ 0) (6 :+ 0))
```

The syntax for `let` is `let` followed by definitions, then `in`, and the expression in which those definitions are used:

``` active haskell
import Data.Complex

-- Solutions to a * x^2 + b * x + c == 0
quadraticSolve a b c =
    if a == 0
    then error "Leading coefficient can't be zero"
    else
        let
            discriminant = sqrt (b ^ 2 - 4 * a * c)
            rPlus = ((-b) + discriminant) / (2 * a)
            rMinus = ((-b) - discriminant) / (2 * a)
        in
            (rPlus, rMinus)

main = do
    print (quadraticSolve 1 1 (-6))
    print (quadraticSolve 1 0 1)
    print (quadraticSolve (1 :+ 0) 0 (1 :+ 0))
    print (quadraticSolve (1 :+ 0) (1 :+ 0) (6 :+ 0))
    print (quadraticSolve 0 0 0)
```

You can put `let` in list comprehensions, and those symbols are available in the generator expression:

``` active haskell
f x = x ^ 2 + 3 * x - 20

points =
    [(x, y)
    | x <- [-10 .. 10],
      let y = f x,
      y >= 0 ]

main = do
    print points
```


## Exercises

### List of squares

Complete this program so it prints out a list of the first n square numbers

```active haskell
listOfSquares n = ???

main = do
    print (listOfSquares 10)
    print (listOfSquares 20)
```

Solution:
@@@
```active haskell
listOfSquares n =
    [k^2 | k <- [1 .. n] ]

main = do
    print (listOfSquares 10)
    print (listOfSquares 20)
```
@@@

### Pythagorean triples

Complete this program to print out a list of Pythagorean triples with numbers between 3 and 50.

A Pythagorean triple is three integers `(a,b,c)` such that `a^2 + b^2 == c^2`.

``` active haskell
pythagoreanTriples = ???

main =
    print pythagoreanTriples
```

Easiest solution:
@@@
``` active haskell

pythagoreanTriples =
    [(a, b, c)
    | a <- [3 .. 50],
      b <- [3 .. 50],
      c <- [3 .. 50],
      c^2 == a^2 + b^2]

main =
    print pythagoreanTriples
```
@@@

A bit better:

@@@
``` active haskell
pythagoreanTriples =
    [(a, b, c)
    | a <- [3 .. 50],
      b <- [a .. 50],
      c <- [b .. 50],
      c^2 == a^2 + b^2]

main =
    print pythagoreanTriples
```
@@@


# The list data type and pattern matching

## Definition and deconstruction of a list

A list of items of type t is either an empty list or a value of type t followed by a list of items of type t.  You could define your own list type as follows:

``` haskell
data List t = Null | Cons t (List t)
```

But Haskell has lists built in, and special notation to make them easier to use.  The definition is roughly as follows:

``` haskell
data [t] = [] | t : [t]
```

Here's how to determine the length of a list (a standard library function):

``` haskell
length [] = 0
length (x:xs) = 1 + length xs
```

The `map` function (or combinator) takes a function and a list of items and returns a list built by applying the function to each item.  Here are two implementations of `map`:

``` haskell
map f xs = [f x | x <- xs]
```

``` haskell
map f [] = []
map f (x:xs) = f x : map f xs
```

You can also use the `case` syntax instead of multiple equations:

``` haskell
map f xs =
    case xs of
        [] -> []
        x:xs -> f x : map f xs
```

Lists can be concatenated.  The `++` operator does this.  Here is an implementation:

``` haskell
-- this is how to declare a right-associative binary operator with precedence level 5
infixr 5 ++ 

[] ++ ys = ys
(x:xs) ++ ys = x : (xs ++ ys)

```

## Exercises

### Sum of items

Haskell has a built-in function `sum` that returns the sum of all items in a list of numbers.  Write an implementation:

``` active haskell
mySum = ???

main = do
    print (sum [1,4,7,20]) -- built in library function
    print (mySum [1,4,7,20])
```
Solution:
@@@
``` active haskell
mySum [] = 0
mySum (x:xs) = x + mySum xs

main = do
    print (sum [1,4,7,20]) -- built in library function
    print (mySum [1,4,7,20])
```
@@@

### Reverse a list

Haskell has a built-in function `reverse` that takes a list and returns a list of the same items in reverse order.  Write an implementation:

``` active haskell
myReverse = ???

main = do
    print (reverse [1,4,7,20]) -- built in library function
    print (myReverse [1,4,7,20])
```
Solutions:
@@@
Using `++`:

``` active haskell
myReverse [] = []
myReverse (x:xs) = myReverse xs ++ [x]

main = do
    print (reverse [1,4,7,20]) -- built in library function
    print (myReverse [1,4,7,20])
```
but that's not so good, because it ends up traversing the list multiple times
@@@
@@@
Using an accumulator:

``` active haskell
myReverse xs = loop xs []
    where
        loop [] ys = ys
        loop (x:xs) ys = loop xs (x:ys)

main = do
    print (reverse [1,4,7,20]) -- built in library function
    print (myReverse [1,4,7,20])
```
@@@


## Partial applications and combinators

Since Haskell allows you to build partially applied functions and pass them around, you can do things like this:

``` active haskell
add x y = x + y

numbers = [6, 5, 0, 2]

-- Partial evaluation means that (add 1) is the same as (\y -> 1 + y)
-- so we can do this:

biggerNumbers = map (add 1) numbers

main = do
    print numbers
    print biggerNumbers
```

Infix operators also have a special partial application syntax, and that lets us abbreviate the previous example into :

``` active haskell
numbers = [6, 5, 0, 2]

-- Partial evaluation means that (1 +) is the same as (\y -> 1 + y)
-- so we can do this:

biggerNumbers1 = map (1 +) numbers

-- and you can go the other way:
biggerNumbers2 = map (+ 1) numbers

main = do
    print numbers
    print biggerNumbers1
    print biggerNumbers2
```
By the way, this is why the `-` operator is sort of funky.  The notation `(-2)` could either mean `\x -> x - 2` or negative two.  The standard defines a special syntactic rules that 

- `(x-)` means `\y -> x - y` as expected
- `(-x)` always means `negate x`
- even for literals unary `-` is translated into negate, so `-3.14` means `negate 3.14` 
- `f -x` means subtraction (`f` minus `x`) rather than `f` applied to `negate x`
- A `subtract` function is available, `subtract 2 = \x -> x - 2`, to make up for the missing meaning of `(-2)`.

``` active haskell
numbers = [6, 5, 0, 2]

smallerNumbers = map (subtract 1) numbers

reflectedNumbers = map (10 -) numbers

main = do
    print numbers
    print smallerNumbers
    print reflectedNumbers
```

Also, Haskell has special backtick notation for using a function named by letters as an infix operator:
``` haskell
r = 10 `div` 3
-- same as:
r = div 10 3
```
and special parentheses notation for using an infix operator named by symbols as a function in prefix form:
``` haskell
x = (+) 1 2
-- same as
x = 1 + 2
```

## Storing a set of numbers in a tree

Here's how to define a binary tree data structure inductively much like a list.  A tree is either an empty leaf node, or a branch with two subtrees and a value.  Values in the left subtree must be less than the branch's value, and values in the right subtree must be greater.

``` haskell
data Tree elt =
    Leaf
    | Branch elt (Tree elt) (Tree elt)
    deriving (Read, Show, Eq, Ord)
```

- `data` is a key word defining an algebraic data type

- `Tree` is the type we are defining.  It must begin with a capital letter

- `elt` is a type variable.  It must begin with a lower case letter

- What follows the = is a list of alternatives separated by `|`

- `Leaf` means that a tree can be an empty object, marked by the `Leaf` data constructor.  Data constructors must begin with a capital letter.

- `Branch ...` means that a tree can also be a triple of an element and two trees, marked by the `Branch` data constructor

- `deriving ...` well, just trust me, and we'll get back to what `deriving` means later

Let's create some test trees and search them:

``` active haskell
data Tree elt =
    Leaf
    | Branch elt (Tree elt) (Tree elt)
    deriving (Read, Show, Eq, Ord)

singleton elt = Branch elt Leaf Leaf

-- pretty printing
prettyPrint Leaf = ""
prettyPrint (Branch v left right) =
    "(" ++ prettyPrint left ++ ") < " ++ show v ++ " < (" ++ prettyPrint right ++ ")"

-- Nothing is included in an empty leaf.
treeIncludes x Leaf = False

-- If a tree is a branch, it might contain the element.
-- If the value we're looking for is greater than that,
-- it certainly isn't in the tree.
-- Otherwise, it might be in one of the subtrees.

treeIncludes x (Branch y left right) =
    if x == y then True
    else if x > y then treeIncludes x right
    else treeIncludes x left
    
exTree = Branch 2 (singleton 1) (singleton 3)

main = do
    print exTree
    putStrLn (prettyPrint exTree)
    print (treeIncludes 2 exTree)
    print (treeIncludes 10 exTree)
```

To add an item to a tree, we have to create a new tree with the item in it.

``` active haskell
data Tree elt =
    Leaf
    | Branch elt (Tree elt) (Tree elt)
    deriving (Read, Show, Eq, Ord)

singleton elt = Branch elt Leaf Leaf

-- pretty printing
prettyPrint Leaf = ""
prettyPrint (Branch v left right) =
    "(" ++ prettyPrint left ++ ") < " ++ show v ++ " < (" ++ prettyPrint right ++ ")"

-- Nothing is included in an empty leaf.
treeIncludes x Leaf = False

-- If a tree is a branch, it might contain the element.
-- If the value we're looking for is greater than that,
-- it certainly isn't in the tree.
-- Otherwise, it might be in one of the subtrees.

treeIncludes x (Branch y left right) =
    if x == y then True
    else if x > y then treeIncludes x right
    else treeIncludes x left

-- show
-- If the tree is empty, insertion is easy: just create a singleton.
treeInsert x Leaf = singleton x

-- If the tree is a branch:
treeInsert x t@(Branch y left right) =
    -- If x is already in the tree, then no need to insert it again
    if x == y then t
    -- If x is greater than y, insert it to the right
    else if x > y then Branch y left (treeInsert x right)
    -- Else x < y, so insert it to the left
    else Branch y (treeInsert x left) right
    
exTree = Branch 20 (singleton 10) (singleton 30)

main = do
    putStrLn (prettyPrint exTree)
    putStrLn (prettyPrint (treeInsert 5 exTree))
    putStrLn (prettyPrint (treeInsert 15 exTree))
    putStrLn (prettyPrint (treeInsert 25 exTree))
    putStrLn (prettyPrint (treeInsert 35 exTree))
-- /show
```

Deleting from a tree is the same kind of thing.

``` active haskell
data Tree elt =
    Leaf
    | Branch elt (Tree elt) (Tree elt)
    deriving (Read, Show, Eq, Ord)

singleton elt = Branch elt Leaf Leaf

-- pretty printing
prettyPrint Leaf = ""
prettyPrint (Branch v left right) =
    "(" ++ prettyPrint left ++ ") < " ++ show v ++ " < (" ++ prettyPrint right ++ ")"

-- Nothing is included in an empty leaf.
treeIncludes x Leaf = False

-- If a tree is a branch, it might contain the element.
-- If the value we're looking for is greater than that,
-- it certainly isn't in the tree.
-- Otherwise, it might be in one of the subtrees.
treeIncludes x (Branch y left right) =
    if x == y then True
    else if x > y then treeIncludes x right
    else treeIncludes x left

-- If the tree is empty, insertion is easy: just create a singleton.
treeInsert x Leaf = singleton x

-- If the tree is a branch:
treeInsert x t@(Branch y left right) =
    -- If x is already in the tree, then no need to insert it again
    if x == y then t
    -- If x is greater than y, insert it to the right
    else if x > y then Branch y left (treeInsert x right)
    -- Else x < y, so insert it to the left
    else Branch y (treeInsert x left) right

--show
-- If the tree is empty, deletion is easy:
treeDelete x Leaf = Leaf

-- If there's a branch
treeDelete x (Branch y left right) =
    -- maybe have to delete it from a subtree
    if x < y then Branch y (treeDelete x left) right
    else if x > y then Branch y left (treeDelete x right)
    else -- x == y, have to delete it from this node in the middle, which is trickier
        case left of
            -- If the left subtree is empty, we're deleting the root
            Leaf ->
                right
            branch ->
                let
                    (m, newLeft) = treePop left
                in
                    Branch m newLeft right

-- Delete the right-most (maximum element) from the tree, returning it and the new tree
treePop (Branch y left Leaf) =
    (y, left)
    
treePop (Branch x left right) =
    let
        (z, newRight) = treePop right
    in
        (z, Branch x left newRight)
                

exTree = Branch 20 (singleton 10) (singleton 30)

main = do
    putStrLn (prettyPrint exTree)
    putStrLn (prettyPrint (treeDelete 10 exTree))
    putStrLn (prettyPrint (treeDelete 20 exTree))
    putStrLn (prettyPrint (treeDelete 30 exTree))
    -- /show
```

## Exercises

Use this workspace:

``` active haskell
data Tree elt =
    Leaf
    | Branch elt (Tree elt) (Tree elt)
    deriving (Read, Show, Eq, Ord)

singleton elt = Branch elt Leaf Leaf

-- pretty printing
prettyPrint Leaf = ""
prettyPrint (Branch v left right) =
    "(" ++ prettyPrint left ++ ") < " ++ show v ++ " < (" ++ prettyPrint right ++ ")"

-- Nothing is included in an empty leaf.
treeIncludes x Leaf = False

-- If a tree is a branch, it might contain the element.
-- If the value we're looking for is greater than that,
-- it certainly isn't in the tree.
-- Otherwise, it might be in one of the subtrees.
treeIncludes x (Branch y left right) =
    if x == y then True
    else if x > y then treeIncludes x right
    else treeIncludes x left

-- If the tree is empty, insertion is easy: just create a singleton.
treeInsert x Leaf = singleton x

-- If the tree is a branch:
treeInsert x t@(Branch y left right) =
    -- If x is already in the tree, then no need to insert it again
    if x == y then t
    -- If x is greater than y, insert it to the right
    else if x > y then Branch y left (treeInsert x right)
    -- Else x < y, so insert it to the left
    else Branch y (treeInsert x left) right

-- If the tree is empty, deletion is easy:
treeDelete x Leaf = Leaf

-- If there's a branch
treeDelete x (Branch y left right) =
    -- maybe have to delete it from a subtree
    if x < y then Branch y (treeDelete x left) right
    else if x > y then Branch y left (treeDelete x right)
    else -- x == y, have to delete it from this node in the middle, which is trickier
        case left of
            -- If the left subtree is empty, we're deleting the root
            Leaf ->
                right
            branch ->
                let
                    (m, newLeft) = treePop left
                in
                    Branch m newLeft right

-- Delete the right-most (maximum element) from the tree, returning it and the new tree
treePop (Branch y left Leaf) =
    (y, left)
    
treePop (Branch x left right) =
    let
        (z, newRight) = treePop right
    in
        (z, Branch x left newRight)
                

exTree = Branch 20 (singleton 10) (singleton 30)

main = do
    putStrLn "Put some test cases here"
```

### From list

Write a function that takes a list of items (in no particular order) and builds a tree, with everything in the proper order:

``` haskell
treeFromList xs = ???
```
### To list

Write a function that builds a list of the elements of a tree in ascending order.  Try to do this in a way that doesn't repeatedly traverse partial lists along the way...

``` haskell
treeToList t = ???
```

### Tree map

Write an equivalent of `map` for trees:

``` haskell
treeMap f t = ???
```

### Solutions
@@@
``` active haskell
data Tree elt =
    Leaf
    | Branch elt (Tree elt) (Tree elt)
    deriving (Read, Show, Eq, Ord)

singleton elt = Branch elt Leaf Leaf

-- pretty printing
prettyPrint Leaf = ""
prettyPrint (Branch v left right) =
    "(" ++ prettyPrint left ++ ") < " ++ show v ++ " < (" ++ prettyPrint right ++ ")"

-- Nothing is included in an empty leaf.
treeIncludes x Leaf = False

-- If a tree is a branch, it might contain the element.
-- If the value we're looking for is greater than that,
-- it certainly isn't in the tree.
-- Otherwise, it might be in one of the subtrees.
treeIncludes x (Branch y left right) =
    if x == y then True
    else if x > y then treeIncludes x right
    else treeIncludes x left

-- If the tree is empty, insertion is easy: just create a singleton.
treeInsert x Leaf = singleton x

-- If the tree is a branch:
treeInsert x t@(Branch y left right) =
    -- If x is already in the tree, then no need to insert it again
    if x == y then t
    -- If x is greater than y, insert it to the right
    else if x > y then Branch y left (treeInsert x right)
    -- Else x < y, so insert it to the left
    else Branch y (treeInsert x left) right

-- If the tree is empty, deletion is easy:
treeDelete x Leaf = Leaf

-- If there's a branch
treeDelete x (Branch y left right) =
    -- maybe have to delete it from a subtree
    if x < y then Branch y (treeDelete x left) right
    else if x > y then Branch y left (treeDelete x right)
    else -- x == y, have to delete it from this node in the middle, which is trickier
        case left of
            -- If the left subtree is empty, we're deleting the root
            Leaf ->
                right
            branch ->
                let
                    (m, newLeft) = treePop left
                in
                    Branch m newLeft right


-- Delete the right-most (maximum element) from the tree, returning it and the new tree
treePop (Branch y left Leaf) =
    (y, left)
    
treePop (Branch x left right) =
    let
        (z, newRight) = treePop right
    in
        (z, Branch x left newRight)


treeToList t = loop t []
    where
        loop Leaf tailElts = tailElts
        loop (Branch x left right) tailElts =
            loop left (x : loop right tailElts)


treeFromList [] = Leaf

treeFromList (x:xs) = treeInsert x (treeFromList xs)


treeMap f t =
    treeFromList (map f (treeToList t))

exTree1 = treeFromList [10, 30, 20]

exTree2 = treeFromList [-5 .. 5]

f x = x ^ 2 + 3 * x - 20

exTree3 = treeMap f exTree2

main = do
    putStrLn (prettyPrint exTree1)
    print (treeToList exTree1)
    putStrLn (prettyPrint exTree3)
    print (map f [-5 .. 5])
```
@@@

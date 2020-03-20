# Type classes

# Why type classes

We'd really like to be able to use `==` and other notation for more than one thing.  But if Haskell were just strictly typed with no additional features, there'd be no way to write

``` haskell
x :: Int
...
if x == 0 then ...

s :: String
...
if s == "yes" then ...
```

We'd need the impossible:
``` haskell
(==) :: Int -> Int -> Bool
(==) :: String -> String -> Bool
```

(By the way, ML avoids the problem by defining things like `+` and `-` that only work for integers, and `+.` and `-.` that only work for floating point numbers, etc.)

Haskell has a tool called *type classes* that are a huge help with this notation problem:
``` haskell

class Eq t where
    (==) :: t -> t -> Bool
  
instance Eq Int where
    (==) = {- something built in -}
```

Here's an example of searching a list:
``` active haskell

listIncludes x [] = False
listIncludes x (y:ys) =
    if x == y then True else listIncludes x ys
    
main = do
    print (listIncludes 3 [1,2,3,4,5])
    print (listIncludes 10 [1,2,3,4,5])
    print (listIncludes "green" ["red", "blue", "green"])
```

So what's going on?  The type of `listIncludes` is
``` haskell
listIncludes :: Eq t => t -> [t] -> Bool
```
which means, `listIncludes` is a function that takes a `t` and a list of `t`s and returns a `Bool`, but the *context* out in front of the bold arrow `=>` contains the constraint `Eq t`, which means that the function also requires `t` to be an instances of type class `Eq`.

What the compiler does is roughly as follows:

While compiling `listIncludes`, it sneaks in an extra parameter called a dictionary, and the call to `==` is done by fetching the apropriate function out of that dictionary.  I'm going to make up some notation here for internal forms that the compiler might work with to give you some idea of what it's doing:
``` haskell
dEqLookup== :: DictionaryEq t -> (t -> t -> Bool)
dEqLookup== d = ...
...
listIncludesImpl :: DictionaryEq t -> t -> [t] -> Bool 
listIncludesImpl dEq x xs = ...
    if (dEqLookup== dEq) x y then ...
```
Everywhere in your program that you call `listIncludes`, the compiler implicitly passes a dictionary.  So it might transform `listIncludes 3 [1,2,3,4,5]` into
``` haskell
listIncludesImpl dEqInt 3 [1,2,3,4,5]
```
and continue compiling from there.

Sometimes the type constraint sort of percolates outward.  Example: Here's a function that takes an item, and a list of lists of items, and returns the first of those lists that contains that item if such a list exists, using Haskell's `Maybe` type.
``` haskell
-- The built-in Maybe type is the standard way of specifying something
-- that might not exist.
-- You might think of it like pointer that is allowed to be null in C or Java.

data Maybe t = Nothing | Just t

firstThatIncludes :: Eq t => t -> [[t]] -> Maybe [t]

firstThatIncludes x (ys : moreYs) =
    if listIncludes x ys
    then Just ys
    else firstThatIncludes x moreYs

firstThatIncludes x [] =
    Nothing
```
Even though `firstThatIncludes` doesn't call `==`, it calls a function `listIncludes` that *does* call `==`.  Which means that the compiler has to get a dictionary for `class Eq` from somewhere to pass implicitly to `listIncludes`.  So the dictionary must be passed in to `firstThatIncludes` as well, which means `Eq t` has to become a constraint in the context of the type of `firstThatIncludes`.

The compiler does have other options, by the way.  Sometimes it can determine exactly which version of an overloaded function it needs, and it just inlines it.  In the case of `listIncludes 3 [1,2,3,4,5]`, the compiler can tell that it's dealing with integers, so it looks up the implementation of `==` from the `instance Eq Int` dictionary and probably uses that directly.

Here's how to implement `Eq` for lists by the way:
``` haskell
instance Eq t => Eq [t] where
    [] == []  =  True
    (x:xs) == (y:ys)  =  (x == y) && (xs == ys)
    _ == _ = False
```
Explanation:
- This gives an implementation of `class Eq` for the list type parameterized by `t`
- It requires an implementation of `class Eq` for `t`, and this is indicated by the constraint `Eq t` in the context of the `instance` declaration
- The definition of `==` is just plain baffling to look at.  Here's another way to write it out:
``` haskell
instance Eq t => Eq [t] where
    -- The empty list equals the empty list
    (==) [] []  =  True
    
    -- Two nonempty lists are equal if their first elements are equal
    -- and their tails are equal 
    (==) (x:xs) (y:ys)  =  
        (x == y) && (xs == ys)
    
    -- An empty list is never equal to a non empty list.

    (==) [] (_:_) = False
    (==) (_:_) [] = False 
    
        -- Since pattern equations defining a function are checked in order,
        -- we could abbreviate those last two cases down to
        -- (==) _ _ = False
        -- Every case that could return True is handled by something above
```
Here's how to do `Eq` for pairs:
``` haskell
instance (Eq t1, Eq t2) => Eq (t1, t2) where
    (x1, x2) == (y1, y2) = (x1 == y1) && (x2 == y2)
```
Note that this time, the context includes two constraints because a pair might contain two items of different types.

# Built-in classes and arithmetic

The arithmetic operators are overloaded through a couple of built-in classes:
``` haskell
class Num t where
    (+) :: t -> t -> t
    (*) :: t -> t -> t
    ...
```
and `Int` and `Double` are instances of `Num`.  But division is sort of funky, so it goes in separate classes:
``` haskell
class Num t => Fractional t where
    (/) :: t -> t -> t
    ...
    
class (Real t, Enum t) => Integral t where
    div :: t -> t -> t
    mod :: t -> t -> t
    ...
```

# Built-in classes and converting data to and from strings

Two important built-in classes are `Show` and `Read`.  These are required for these all-important functions that convert data objects to and from readable strings:
``` haskell
show :: Show a => a -> String
read :: Read a => String -> a
print :: Show a => a -> IO ()
```

`Show` isn't too bad.  It includes the `show` function, plus a couple of others that are sometimes useful for improved efficiency.

`Read` can be complicated to implement, because it has to deal with operator precedence.

# Getting the compiler to derive instances

For a lot of data types, the compiler can automatically create reasonable instances for the classes `Read`, `Show`, `Eq`, and `Ord` plus a few others.  Equality is checked by matching data structures exactly, and ordering is done field by field, with constructors ordered by the order in which they're listed in the definition. Here's the syntax:
``` haskell
data ChessPiece = Pawn | Bishop | Knight | Rook | Queen | King
    deriving (Read, Show, Eq, Ord)

data ChessBoard = ChessBoard [(ChessPieces, Int, Int)] [(ChessPiece, Int, Int)]
    deriving (Read, Show, Eq, Ord)
    
-- This is built in:
data Maybe t = Nothing | Just t
    deriving (Read, Show, Eq, Ord)
```
Frequently, you need your data types to be instances of these four classes.  Reading and showing are really useful for debugging, even if you don't plan to ever store your data items to a file in that format.  `Eq` and `Ord` are required if you're going to put data into a container like `Data.Set` or `Data.Map`.

# Exercises

## Type checking

Go back to the tree example in [Basic Haskell Syntax](https://www.fpcomplete.com/user/garrett.mitchener/Basic%20Haskell%20Syntax) and put in type signatures for all the functions.  You'll have to use the `Ord` type class, which is for things that can be ordered using `<`, `>`, etc.

## Some instances

Go back to the set example in [Basic Haskell Syntax](https://www.fpcomplete.com/user/garrett.mitchener/Basic%20Haskell%20Syntax) and implement an instance for trees for the class `Eq`.  For our purposes, two trees are equal if they contain the same items, perhaps in a different branching structure, so these trees have set-like semantics.

## Trying out Ord

Make sure you understand these:
``` active haskell
main = do
    print (Nothing < Just 1)
    print (Nothing > Just 1)
    print (Just 1 < Just 2)
    print (Just 1 > Just 2)
```
Then, figure out ahead of time what this will print out, then try it:
``` active haskell
data Point = Point Int Int
    deriving (Read, Show, Eq, Ord)
    
data GraphicsPrimitive =
    LineSegment Point Point
    | BezierCurve Point Point Point Point
    deriving (Read, Show, Eq, Ord)

a = Point 0 0
b = Point 25 100
c = Point 75 100
d = Point 100 0

l = LineSegment a b
v = BezierCurve a b c d

main = do
    print l
    print (a == b)
    print (a == Point 0 0)
    print (l < v)
    print (v < l)
```
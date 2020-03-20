# Haskell Fast & Hard (Part 4)

### Types

![Dali, the madonna of port Lligat](http://yannesposito.com/Scratch/img/blog/Haskell-the-Hard-Way/salvador-dali-the-madonna-of-port-lligat.jpg)

---

Too long; didn't read:

- `type Name = AnotherType` is just an alias and the compiler doesn't do any difference between `Name` and `AnotherType`.
- `data Name = NameConstructor AnotherType` make a difference.
- `data` can construct structures which can be recursives.
- `deriving` is magic and create functions for you.

---

In Haskell, types are strong and static.

Why is this important? It will help you _greatly_ to avoid mistakes.
In Haskell, most bugs are caught during the compilation of your program.
And the main reason is because of the type inference during compilation.
It will be easy to detect where you used the wrong parameter at the wrong place for example.

#### Type inference

Static typing is generally essential to reach fast execution time.
But most statically typed languages are bad at generalizing concepts.
Haskell's saving grace is that it can _infer_ types.

Here is a simple example.
The `square` function in Haskell:

``` haskell
square x = x * x
```

This function can `square` any Numeral type.
You can provide `square` with an `Int`, an `Integer`, a `Float` a `Fractional` and even `Complex`. Proof by example:

``` active haskell
import Data.Complex

square x = x*x

main = do
  print $ square 2
  print $ square 2.1
  print $ square (2 :+ 1)
```


`x :+ y` is the notation for the complex (<i>x + ib</i>).

Now compare with the amount of code necessary in C:

``` c
int     int_square(int x) { return x*x; }

float   float_square(float x) {return x*x; }

complex complex_square (complex z) {
    complex tmp;
    tmp.real = z.real * z.real - z.img * z.img;
    tmp.img = 2 * z.img * z.real;
}

complex x,y;
y = complex_square(x);
```

For each type, you need to write a new function.
The only way to work around this problem is to use some meta-programming trick.
For example using the pre-processor.
In C++ there is a better way, the C++ templates:

``` c++
#include <iostream>
#include <complex>
using namespace std;

template<typename T>
T square(T x)
{
    return x*x;
}

int main() {
    // int
    int sqr_of_five = square(5);
    cout << sqr_of_five << endl;
    // double
    cout << (double)square(5.3) << endl;
    // complex
    cout << square( complex<double>(5,3) )
         << endl;
    return 0;
}
```

C++ does a far better job than C.
For more complex function the syntax can be hard to follow:
look at
[this article](http://bartoszmilewski.com/2009/10/21/what-does-haskell-have-to-do-with-c/)
for example.

In C++ you must declare that a function can work with different types.
In Haskell this is the opposite.
The function will be as general as possible by default.

Type inference gives Haskell the feeling of freedom that dynamically
typed languages provide.
But unlike dynamically typed languages, most errors are caught before the execution.
Generally, in Haskell:

> "if it compiles it certainly does what you intended"


#### Type construction

You can construct your own types.
First you can use aliases or type synonyms.

``` active haskell
type Name   = String
type Color  = String

showInfos :: Name ->  Color -> String
showInfos name color =  "Name: " ++ name
                        ++ ", Color: " ++ color
name :: Name
name = "Robin"
color :: Color
color = "Blue"
main = putStrLn $ showInfos name color
```



But it doesn't protect you much.
Try to swap the two parameter of `showInfos` and run the program:

``` active haskell
type Name   = String
type Color  = String

showInfos :: Name ->  Color -> String
showInfos name color =  "Name: " ++ name
                        ++ ", Color: " ++ color
name :: Name
name = "Robin"
color :: Color
color = "Blue"
-- show
main = putStrLn $ showInfos color name
-- /show
```

It will compile and execute.
In fact you can replace Name, Color and String everywhere.
The compiler will treat them as completely identical.

Another method is to create your own types using the keyword `data`.

``` active haskell
data Name   = NameConstr String
data Color  = ColorConstr String

showInfos :: Name ->  Color -> String
showInfos (NameConstr name) (ColorConstr color) =
      "Name: " ++ name ++ ", Color: " ++ color

name  = NameConstr "Robin"
color = ColorConstr "Blue"
main = putStrLn $ showInfos name color
```

Now if you switch parameters of `showInfos`, the compiler complains!
A possible mistake you could never do again.
The only price is to be more verbose.

Also remark constructor are functions:

``` haskell
NameConstr  :: String -> Name
ColorConstr :: String -> Color
```

The syntax of `data` is mainly:

``` haskell
data TypeName =   ConstructorName  [types]
                | ConstructorName2 [types]
                | ...
```

Generally the usage is to use the same name for the
DataTypeName and DataTypeConstructor.

Example:

``` haskell
data Complex a = Num a => Complex a a
```

Also you can use the record syntax:

``` haskell
data DataTypeName = DataConstructor {
                      field1 :: [type of field1]
                    , field2 :: [type of field2]
                    ...
                    , fieldn :: [type of fieldn] }
```

And many accessors are made for you.
Furthermore you can use another order when setting values.

Example:

``` haskell
data Complex a = Num a => Complex { real :: a, img :: a}
c = Complex 1.0 2.0
z = Complex { real = 3, img = 4 }
real c ⇒ 1.0
img z ⇒ 4
```

##### Exercises:

1) Declare the data type `Knight`  in the following program:

``` active haskell
data Knight = undefined

galaad = Knight { name = "Galaad, the pure"
                , quest = "To seek the Holy Grail"
                , favoriteColor = "The blue... No the red! AAAAAAHHHHHHH!!!!" }

showCharacter :: Knight -> String
showCharacter knight = "What is your name?\n"
    ++ "My name is " ++ name knight
    ++ "\nWhat is your quest?\n"
    ++ quest knight
    ++ "\nWhat is your favorite color?\n"
    ++ favoriteColor knight

main = do
  putStrLn $ showCharacter galaad

```

@@@ Solution
``` active haskell
data Knight = {-hi-}Knight { name :: String
                     , quest :: String
                     , favoriteColor :: String }{-/hi-}

galaad = Knight { name = "Galaad, the pure"
                , quest = "To seek the Holy Grail"
                , favoriteColor = "The blue... No the red! AAAAAAHHHHHHH!!!!" }

showCharacter :: Knight -> String
showCharacter knight = "What is your name?\n"
    ++ "My name is " ++ name knight
    ++ "\nWhat is your quest?\n"
    ++ quest knight
    ++ "\nWhat is your favorite color?\n"
    ++ favoriteColor knight

main = do
  putStrLn $ showCharacter galaad

```
@@@

2) Somebody changed the showCharacter to make it more readable.
Unfortunately he mades some mistake.
Change the type declaration such that the compiler complains, and then correct the showCharacter function.

``` active haskell
data Knight = Knight { name :: String
                     , quest :: String
                     , favoriteColor :: String }

showNameQuestion :: String -> String
showNameQuestion someName = "What is your name? My name is " ++ someName

showQuestQuestion :: String -> String
showQuestQuestion someQuest = "What is your quest? " ++ someQuest

showColorQuestion :: String -> String
showColorQuestion someColor = "What is your favorite color? " ++ someColor
    
showCharacter :: Knight -> String
showCharacter knight = showNameQuestion (favoriteColor knight) ++ "\n"
                       ++ showQuestQuestion (name knight ) ++ "\n"
                       ++ showColorQuestion (quest knight)

galaad = Knight { name = "Galaad, the pure"
                , quest = "To seek the Holy Grail"
                , favoriteColor = "The blue... No the red! AAAAAAHHHHHHH!!!!" }

main = do
  putStrLn $ showCharacter galaad

```

@@@ Solution
``` active haskell
{-hi-}data Name = Name String
data Quest = Quest String
data Color = Color String{-/hi-}

data Knight = Knight { name :: Name
                     , quest :: Quest
                     , favoriteColor :: Color }

showNameQuestion :: {-hi-}Name{-/hi-} -> String
showNameQuestion {-hi-}(Name someName){-/hi-} = "What is your name? My name is " ++ someName

showQuestQuestion :: {-hi-}Quest{-/hi-} -> String
showQuestQuestion {-hi-}(Quest someQuest){-/hi-} = "What is your quest? " ++ someQuest

showColorQuestion :: {-hi-}Color{-/hi-} -> String
showColorQuestion {-hi-}(Color someColor){-/hi-} = "What is your favorite color? " ++ someColor
    
showCharacter :: Knight -> String
{-
-- This version doesn't compile, try to uncomment to verify
showCharacter knight = showNameQuestion (favoriteColor knight) ++ "\n"
                       ++ showQuestQuestion (name knight ) ++ "\n"
                       ++ showColorQuestion (quest knight)
-}

showCharacter knight = showNameQuestion (name knight) ++ "\n"
                       ++ showQuestQuestion (quest knight ) ++ "\n"
                       ++ showColorQuestion (favoriteColor knight)

galaad = Knight { name = {-hi-}Name{-/hi-} "Galaad, the pure"
                , quest = {-hi-}Quest{-/hi-} "To seek the Holy Grail"
                , favoriteColor = {-hi-}Color{-/hi-} "The blue... No the red! AAAAAAHHHHHHH!!!!" }

main = do
  putStrLn $ showCharacter galaad

```
@@@


#### Recursive type

You already encountered a recursive type: lists.
You can re-create lists, but with a more verbose syntax:

``` haskell
data List a = Empty | Cons a (List a)
```

If you really want to use an easier syntax you can use an infix name for constructors.

``` haskell
infixr 5 :::
data List a = Nil | a ::: (List a)
```

The number after `infixr` is the priority.

If you want to be able to print (`Show`), read (`Read`), test equality (`Eq`) and compare (`Ord`) your new data structure you can tell Haskell to derive the appropriate functions for you.

``` haskell
infixr 5 :::
data List a = Nil | a ::: (List a)
              deriving (Show,Read,Eq,Ord)
```

When you add `deriving (Show)` to your data declaration, Haskell create a `show` function for you.
We'll see soon how you can use your own `show` function.

``` haskell
convertList [] = Nil
convertList (x:xs) = x ::: convertList xs
```

``` active haskell
infixr 5 :::
data List a = Nil | a ::: (List a)
              deriving (Show,Read,Eq,Ord)
convertList [] = Nil
convertList (x:xs) = x ::: convertList xs
-- show
main = do
      print (0 ::: 1 ::: Nil)
      print (convertList [0,1])
-- /show
```

#### Trees

![Magritte, l'Arbre](http://yannesposito.com/Scratch/img/blog/Haskell-the-Hard-Way/magritte-l-arbre.jpg)

We'll just give another standard example: binary trees.

``` haskell
import Data.List

data BinTree a = Empty
                 | Node a (BinTree a) (BinTree a)
                              deriving (Show)
```

We will also create a function which turns a list into an ordered binary tree.

``` haskell
treeFromList :: (Ord a) => [a] -> BinTree a
treeFromList [] = Empty
treeFromList (x:xs) = Node x (treeFromList (filter (<x) xs))
                             (treeFromList (filter (>x) xs))
```

Look at how elegant this function is.
In plain English:

- an empty list will be converted to an empty tree.
- a list `(x:xs)` will be converted to a tree where:

  - The root is `x`
  - Its left subtree is the tree created from members of the list `xs` which are strictly inferior to `x` and
  - the right subtree is the tree created from members of the list `xs` which are strictly superior to `x`.

``` active haskell
import Data.List

data BinTree a = Empty
                 | Node a (BinTree a) (BinTree a)
                              deriving (Show)
treeFromList :: (Ord a) => [a] -> BinTree a
treeFromList [] = Empty
treeFromList (x:xs) = Node x (treeFromList (filter (<x) xs))
                             (treeFromList (filter (>x) xs))
-- show
main = print $ treeFromList [7,2,4,8]
-- /show
```

This is an informative but quite unpleasant representation of our tree.

Just for fun, let's code a better display for our trees.
I simply had fun making a nice function to display trees in a general way.
You can safely skip this part if you find it too difficult to follow.

We have a few changes to make.
We remove the `deriving (Show)` from the declaration of our `BinTree` type.
And it might also be useful to make our BinTree an instance of (`Eq` and `Ord`).
We will be able to test equality and compare trees.

``` haskell
data BinTree a = Empty
                 | Node a (BinTree a) (BinTree a)
                  deriving (Eq,Ord)
```

Without the `deriving (Show)`, Haskell doesn't create a `show` method for us.
We will create our own version of `show`.
To achieve this, we must declare that our newly created type `BinTree a`
is an instance of the type class `Show`.
The general syntax is:

``` haskell
instance Show (BinTree a) where
   show t = ... -- You declare your function here
```

Here is my version of how to show a binary tree.
Don't worry about the apparent complexity.
I made a lot of improvements in order to display even stranger objects.

``` haskell
-- declare BinTree a to be an instance of Show
instance (Show a) => Show (BinTree a) where
  -- will start by a '<' before the root
  -- and put a : a begining of line
  show t = "< " ++ replace '\n' "\n: " (treeshow "" t)
    where
    -- treeshow pref Tree
    --   shows a tree and starts each line with pref
    -- We don't display the Empty tree
    treeshow pref Empty = ""
    -- Leaf
    treeshow pref (Node x Empty Empty) =
                  (pshow pref x)

    -- Right branch is empty
    treeshow pref (Node x left Empty) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " left)

    -- Left branch is empty
    treeshow pref (Node x Empty right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    -- Tree with left and right children non empty
    treeshow pref (Node x left right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "|--" "|  " left) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    -- shows a tree using some prefixes to make it nice
    showSon pref before next t =
                  pref ++ before ++ treeshow (pref ++ next) t

    -- pshow replaces "\n" by "\n"++pref
    pshow pref x = replace '\n' ("\n"++pref) (show x)

    -- replaces one char by another string
    replace c new string =
      concatMap (change c new) string
      where
          change c new x
              | x == c = new
              | otherwise = x:[] -- "x"
```

The `treeFromList` method remains identical.

``` haskell
treeFromList :: (Ord a) => [a] -> BinTree a
treeFromList [] = Empty
treeFromList (x:xs) = Node x (treeFromList (filter (<x) xs))
                             (treeFromList (filter (>x) xs))
```

And now, we can play:

``` active haskell
data BinTree a = Empty
                 | Node a (BinTree a) (BinTree a)
                  deriving (Eq,Ord)

-- declare BinTree a to be an instance of Show
instance (Show a) => Show (BinTree a) where
  -- will start by a '<' before the root
  -- and put a : a begining of line
  show t = "< " ++ replace '\n' "\n: " (treeshow "" t)
    where
    -- treeshow pref Tree
    --   shows a tree and starts each line with pref
    -- We don't display the Empty tree
    treeshow pref Empty = ""
    -- Leaf
    treeshow pref (Node x Empty Empty) =
                  (pshow pref x)

    -- Right branch is empty
    treeshow pref (Node x left Empty) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " left)

    -- Left branch is empty
    treeshow pref (Node x Empty right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    -- Tree with left and right children non empty
    treeshow pref (Node x left right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "|--" "|  " left) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    -- shows a tree using some prefixes to make it nice
    showSon pref before next t =
                  pref ++ before ++ treeshow (pref ++ next) t

    -- pshow replaces "\n" by "\n"++pref
    pshow pref x = replace '\n' ("\n"++pref) (show x)

    -- replaces one char by another string
    replace c new string =
      concatMap (change c new) string
      where
          change c new x
              | x == c = new
              | otherwise = x:[] -- "x"

treeFromList :: (Ord a) => [a] -> BinTree a
treeFromList [] = Empty
treeFromList (x:xs) = Node x (treeFromList (filter (<x) xs))
                             (treeFromList (filter (>x) xs))
-- show
main = do
  putStrLn "Int binary tree:"
  print $ treeFromList [7,2,4,8,1,3,6,21,12,23]
-- /show
```

Now it is far better!
The root is shown by starting the line with the `<` character.
And each following line starts with a `:`.
But we could also use another type.

``` active haskell
data BinTree a = Empty
                 | Node a (BinTree a) (BinTree a)
                  deriving (Eq,Ord)

-- declare BinTree a to be an instance of Show
instance (Show a) => Show (BinTree a) where
  -- will start by a '<' before the root
  -- and put a : a begining of line
  show t = "< " ++ replace '\n' "\n: " (treeshow "" t)
    where
    -- treeshow pref Tree
    --   shows a tree and starts each line with pref
    -- We don't display the Empty tree
    treeshow pref Empty = ""
    -- Leaf
    treeshow pref (Node x Empty Empty) =
                  (pshow pref x)

    -- Right branch is empty
    treeshow pref (Node x left Empty) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " left)

    -- Left branch is empty
    treeshow pref (Node x Empty right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    -- Tree with left and right children non empty
    treeshow pref (Node x left right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "|--" "|  " left) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    -- shows a tree using some prefixes to make it nice
    showSon pref before next t =
                  pref ++ before ++ treeshow (pref ++ next) t

    -- pshow replaces "\n" by "\n"++pref
    pshow pref x = replace '\n' ("\n"++pref) (show x)

    -- replaces one char by another string
    replace c new string =
      concatMap (change c new) string
      where
          change c new x
              | x == c = new
              | otherwise = x:[] -- "x"

treeFromList :: (Ord a) => [a] -> BinTree a
treeFromList [] = Empty
treeFromList (x:xs) = Node x (treeFromList (filter (<x) xs))
                             (treeFromList (filter (>x) xs))
-- show
main = do
  putStrLn "\nString binary tree:"
  print $ treeFromList ["foo","bar","baz","gor","yog"]
-- /show
```


As we can test equality and order trees, we can
make tree of trees!

```active haskell
data BinTree a = Empty
                 | Node a (BinTree a) (BinTree a)
                  deriving (Eq,Ord)

-- declare BinTree a to be an instance of Show
instance (Show a) => Show (BinTree a) where
  -- will start by a '<' before the root
  -- and put a : a begining of line
  show t = "< " ++ replace '\n' "\n: " (treeshow "" t)
    where
    -- treeshow pref Tree
    --   shows a tree and starts each line with pref
    -- We don't display the Empty tree
    treeshow pref Empty = ""
    -- Leaf
    treeshow pref (Node x Empty Empty) =
                  (pshow pref x)

    -- Right branch is empty
    treeshow pref (Node x left Empty) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " left)

    -- Left branch is empty
    treeshow pref (Node x Empty right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    -- Tree with left and right children non empty
    treeshow pref (Node x left right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "|--" "|  " left) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    -- shows a tree using some prefixes to make it nice
    showSon pref before next t =
                  pref ++ before ++ treeshow (pref ++ next) t

    -- pshow replaces "\n" by "\n"++pref
    pshow pref x = replace '\n' ("\n"++pref) (show x)

    -- replaces one char by another string
    replace c new string =
      concatMap (change c new) string
      where
          change c new x
              | x == c = new
              | otherwise = x:[] -- "x"

treeFromList :: (Ord a) => [a] -> BinTree a
treeFromList [] = Empty
treeFromList (x:xs) = Node x (treeFromList (filter (<x) xs))
                             (treeFromList (filter (>x) xs))
-- show
main = do
  putStrLn "\nBinary tree of Char binary trees:"
  print ( treeFromList
           (map treeFromList ["baz","zara","bar"]))
-- /show
```


This is why I chose to prefix each line of tree display by `:` (except for the root).

![Yo Dawg Tree](http://yannesposito.com/Scratch/img/blog/Haskell-the-Hard-Way/yo_dawg_tree.jpg)

``` active haskell
data BinTree a = Empty
                 | Node a (BinTree a) (BinTree a)
                  deriving (Eq,Ord)

-- declare BinTree a to be an instance of Show
instance (Show a) => Show (BinTree a) where
  -- will start by a '<' before the root
  -- and put a : a begining of line
  show t = "< " ++ replace '\n' "\n: " (treeshow "" t)
    where
    -- treeshow pref Tree
    --   shows a tree and starts each line with pref
    -- We don't display the Empty tree
    treeshow pref Empty = ""
    -- Leaf
    treeshow pref (Node x Empty Empty) =
                  (pshow pref x)

    -- Right branch is empty
    treeshow pref (Node x left Empty) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " left)

    -- Left branch is empty
    treeshow pref (Node x Empty right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    -- Tree with left and right children non empty
    treeshow pref (Node x left right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "|--" "|  " left) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    -- this shows a tree using some prefixes to make it nice
    showSon pref before next t =
                  pref ++ before ++ treeshow (pref ++ next) t

    -- pshow replaces "\n" by "\n"++pref
    pshow pref x = replace '\n' ("\n"++pref) (show x)

    -- replaces one char by another string
    replace c new string =
      concatMap (change c new) string
      where
          change c new x
              | x == c = new
              | otherwise = x:[] -- "x"

treeFromList :: (Ord a) => [a] -> BinTree a
treeFromList [] = Empty
treeFromList (x:xs) = Node x (treeFromList (filter (<x) xs))
                             (treeFromList (filter (>x) xs))
-- show
main = do
  putStrLn "\nTree of Binary trees of Char binary trees:"
  print $ (treeFromList . map (treeFromList . map treeFromList))
             [ ["YO","DAWG"]
             , ["I","HEARD"]
             , ["I","HEARD"]
             , ["YOU","LIKE","TREES"] ]
-- /show
```

Which is equivalent to

``` haskell
print ( treeFromList (
          map treeFromList
             [ map treeFromList ["YO","DAWG"]
             , map treeFromList ["I","HEARD"]
             , map treeFromList ["I","HEARD"]
             , map treeFromList ["YOU","LIKE","TREES"] ]))
```

Notice how duplicate trees aren't inserted;
there is only one tree corresponding to `"I","HEARD"`.
We have this for (almost) free, because we have declared Tree to be an instance of `Eq`.

See how awesome this structure is.
We can make trees containing not only integers, strings and chars, but also other trees.
And we can even make a tree containing a tree of trees!


### Infinite Structures

![Escher](http://yannesposito.com/Scratch/img/blog/Haskell-the-Hard-Way/escher_infinite_lizards.jpg)

It is often stated that Haskell is _lazy_.

In fact, if you are a bit pedantic, you should state that [Haskell is _non-strict_](http://www.haskell.org/haskellwiki/Lazy_vs._non-strict).
Laziness is just a common implementation for non-strict languages.

Then what does not-strict means? From the Haskell wiki:

> Reduction (the mathematical term for evaluation) proceeds from the outside in.
>
> so if you have `(a+(b*c))` then you first reduce `+` first, then you reduce the inner `(b*c)`

For example in Haskell you can do:

``` active haskell
-- numbers = [0,1,2,..]
numbers :: [Integer]
numbers = 0:map (1+) numbers

take' n [] = []
take' 0 l = []
take' n (x:xs) = x:take' (n-1) xs

main = print $ take' 10 numbers
```

And it stops.

How?

Instead of trying to evaluate `numbers` entirely,
it evaluates elements only when needed.

Also, note in Haskell there is a notation for infinite lists

```
[1..]   ⇔ [1,2,3,4...]
[1,3..] ⇔ [1,3,5,7,9,11...]
```

And most functions will work with them.
Also, there is a built-in function `take` which is equivalent to our `take'`.




Suppose we don't mind having an ordered binary tree.
Here is an infinite binary tree:

``` haskell
nullTree = Node 0 nullTree nullTree
```

A complete binary tree where each node is equal to 0.
Now I will prove you can manipulate this object using the following function:

``` haskell
-- take all element of a BinTree
-- up to some depth
treeTakeDepth _ Empty = Empty
treeTakeDepth 0 _     = Empty
treeTakeDepth n (Node x left right) = let
          nl = treeTakeDepth (n-1) left
          nr = treeTakeDepth (n-1) right
          in
              Node x nl nr
```

See what occurs for this program:

``` active haskell
import Data.List
data BinTree a = Empty
                 | Node a (BinTree a) (BinTree a)
                  deriving (Eq,Ord)
-- declare BinTree a to be an instance of Show
instance (Show a) => Show (BinTree a) where
  -- will start by a '<' before the root
  -- and put a : a begining of line
  show t = "< " ++ replace '\n' "\n: " (treeshow "" t)
    where
    treeshow pref Empty = ""
    treeshow pref (Node x Empty Empty) =
                  (pshow pref x)

    treeshow pref (Node x left Empty) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " left)

    treeshow pref (Node x Empty right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    treeshow pref (Node x left right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "|--" "|  " left) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    -- This shows a tree using some prefixes to make it nice
    showSon pref before next t =
                  pref ++ before ++ treeshow (pref ++ next) t

    -- pshow replace "\n" by "\n"++pref
    pshow pref x = replace '\n' ("\n"++pref) (" " ++ show x)

    -- replace on char by another string
    replace c new string =
      concatMap (change c new) string
      where
          change c new x
              | x == c = new
              | otherwise = x:[] -- "x"
nullTree = Node 0 nullTree nullTree
-- take all element of a BinTree
-- up to some depth
treeTakeDepth _ Empty = Empty
treeTakeDepth 0 _     = Empty
treeTakeDepth n (Node x left right) = let
          nl = treeTakeDepth (n-1) left
          nr = treeTakeDepth (n-1) right
          in
              Node x nl nr
-- show
main = print $ treeTakeDepth 4 nullTree
-- /show
```

This code compiles, runs and stops.

Just to heat up your neurones a bit more,
let's make a slightly more interesting tree:

``` haskell
iTree = Node 0 (dec iTree) (inc iTree)
        where
           dec (Node x l r) = Node (x-1) (dec l) (dec r)
           inc (Node x l r) = Node (x+1) (inc l) (inc r)
```

Another way to create this tree is to use a higher order function.
This function should be similar to `map`, but should work on `BinTree` instead of list.
Here is such a function:

``` haskell
-- apply a function to each node of Tree
treeMap :: (a -> b) -> BinTree a -> BinTree b
treeMap f Empty = Empty
treeMap f (Node x left right) = Node (f x)
                                     (treeMap f left)
                                     (treeMap f right)
```

_Hint_: I won't talk more about this here.
If you are interested by the generalization of `map` to other data structures,
search for functor and `fmap`.

Our definition is now:

``` haskell
infTreeTwo :: BinTree Int
infTreeTwo = Node 0 (treeMap (\x -> x-1) infTreeTwo)
                    (treeMap (\x -> x+1) infTreeTwo)
```

Look at the result for

``` active haskell
import Data.List
data BinTree a = Empty
                 | Node a (BinTree a) (BinTree a)
                  deriving (Eq,Ord)
-- declare BinTree a to be an instance of Show
instance (Show a) => Show (BinTree a) where
  -- will start by a '<' before the root
  -- and put a : a begining of line
  show t = "< " ++ replace '\n' "\n: " (treeshow "" t)
    where
    treeshow pref Empty = ""
    treeshow pref (Node x Empty Empty) =
                  (pshow pref x)

    treeshow pref (Node x left Empty) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " left)

    treeshow pref (Node x Empty right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    treeshow pref (Node x left right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "|--" "|  " left) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    showSon pref before next t =
                  pref ++ before ++ treeshow (pref ++ next) t

    -- pshow replace "\n" by "\n"++pref
    pshow pref x = replace '\n' ("\n"++pref) (" " ++ show x)

    -- replace on char by another string
    replace c new string =
      concatMap (change c new) string
      where
          change c new x
              | x == c = new
              | otherwise = x:[] -- "x"
iTree = Node 0 (dec iTree) (inc iTree)
        where
           dec (Node x l r) = Node (x-1) (dec l) (dec r)
           inc (Node x l r) = Node (x+1) (inc l) (inc r)
-- apply a function to each node of Tree
treeMap :: (a -> b) -> BinTree a -> BinTree b
treeMap f Empty = Empty
treeMap f (Node x left right) = Node (f x)
                                     (treeMap f left)
                                     (treeMap f right)
infTreeTwo :: BinTree Int
infTreeTwo = Node 0 (treeMap (\x -> x-1) infTreeTwo)
                    (treeMap (\x -> x+1) infTreeTwo)

treeTakeDepth _ Empty = Empty
treeTakeDepth 0 _     = Empty
treeTakeDepth n (Node x left right) = let
          nl = treeTakeDepth (n-1) left
          nr = treeTakeDepth (n-1) right
          in
              Node x nl nr
-- show
main = print $ treeTakeDepth 4 infTreeTwo
-- /show
```

[continue to next part](https://www.fpcomplete.com/school/haskell-fast-hard/haskell-fast-hard-part-5)
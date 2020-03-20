# Functional programming elevator pitch

# The Functional Programming Elevator Pitch

What is functional programming about?  This tutorial will try to tell you why it's worth your time to learn the functional style.  We're focusing on Haskell, the most popular *pure* functional language, to see where these ideas take us in the extreme.

You will see that code you write in Haskell is highly modular and reusable - because the language makes it easy to write higher-order functions that have very particular jobs and are able to move through a program to the right spots the same way that data does.  Haskell allows you to create data-types that capture some of the *meaning* of your algorithms, so that the compiler can check your reasoning, not just your use of punctuation.  This eliminates a huge class of frustrating bugs.  And Haskell's pure functional discipline allows the Haskell compiler to understand much more about your program than is the case in other languages - the result is aggressive compiler optimization, a much better concurrency model, great code-testing tools, and the eradication of SegFaults, NullPtrExceptions, and the like.


Our goal in this post isn't to teach you Haskell (although you will pick up some basics).  We just want to demo Haskell to you, in the hope that you will be convinced to give Haskell a good try.  Please don't be frightened by the unusual syntax - it will all become clear.

## First-class functions: Build components with interlocking precision

FP is about treating functions like 'first-class citizens' in the language.  In most languages, functions take values as arguments and return a value as a result.  Functional languages allow you to treat functions themselves as values.  A function can take other functions as arguments, and return a function as a result.  The syntax for calling a function is also different.  `e = max(a,b)` would be written as `e = max a b` in Haskell.  

```  active haskell
import Data.Char
import Data.List

-- Helper functions split a sentence into words and punctuation
isWordPart l  = notElem l " .,?!-\"' "
toParts s = groupBy (\x y -> isWordPart x && isWordPart y) s

modSentence :: (String -> Bool) -> (String -> String) -> String -> String
modSentence pred mod s = concat . 
                         map (\w -> if pred w then mod w else w) .
                         toParts $ s

-- show
-- Two different tests for whether a word 'w' is in the list
isName  w =  elem w ["Erika","Mark","Katharine"]
isPlace w =  elem w ["lab", "zoo"]

-- Two functions that return either the capitalized
-- or abbreviated form of the input word
uppercaseWord   w = map toUpper w
abbreviatedWord w = [ toUpper (head w), '.' ]

mySentence = "Are Erika and Mark in the lab? Katharine is at the zoo."

main = print $ modSentence isName uppercaseWord mySentence 
-- /show
```

Two of the arguments to the function `modSentence` are themselves functions.  You can swap them out for other functions that fit in that context (`isPlace` and `abbreviatedWord`).

This example focuses on functions that work on strings.  But the method applies to functions over all data types, and you will see later that data types in in Haskell play a bigger role than they do in most other languages.

Detail: how is this different from function pointers in c, for example?  Click "Show more" to find out.
@@@
The first uses you will find for higher-order functions (functions that take other functions as arguments or return other functions) are the same ones you would think of for function pointers in languages that support them: for example a sorting algorithm that can be called with different ordering functions.

Function pointers are an very powerful feature.  Functional languages take this feature and make it pervasive in the language.  You can see this already in the syntax for Haskell function calls.

For example, `foldl` is a very general higher-order function that takes three arguments: (A) a function that combines two elements to make a third (B) An initial element and (C) list of elements [e1, ..., eN]

`foldl f i ls` returns the function `f` 'folded' over the initial element and every element in the list like so: 
`f( ... f( f( a e1) e2 ) ... en)`

``` active haskell
-- show
a = foldl (+) 0 [5 .. 10]
b = foldl (++) "These " ["are ", "some ", "words!"]
-- /show
main = do
    putStrLn $ "a: " ++ show a
    putStrLn $ "b: " ++ show b
```

What would happen if we left off the last argument?  It's reasonable to think that this is an error, just as it would be an to try to compute `y = min(a,);`  But in fact, it is very common to leave out arguments in Haskell.  The return value of `foldl (+) 0` (the above function call, minus its last argument) is a function that stores (+) and 0, and runs when finally given its third argument, a list of elements.

``` active haskell
-- 'Partial application' of foldl to two of its three arguments
myFa = foldl (+) 0

-- myFa is now a function on one argument
main = print (myFa [5 .. 10])
```

It turns out that `myFa` is a pretty useful function!  Try changing the function's name to listSum.  

Then change the name again to `listProduct` and see if you can change its definition so that it returns the product of the elements in a list, instead of their sum.

@@@

Haskell syntax makes it as easy as possible to build up new functions out of old ones and to pass them around in your code.  This allows you to [move up and down the ladder of abstraction](http://worrydream.com/LadderOfAbstraction/) very efficiently. 

## Strong static typing: "If it compiles, it works."

You will be very happy with how *often* your programs work as expected the first time they pass compilation.  The reason for this is that *Algebraic Data Types* make it easy to design types that can only take values that make sense in your program, and *Pattern Matching* helps you pass these well-formed values between functions.

### Algebraic Data Types

An ADT enumerates the values that a data type may take, with some of these values coming with extra data tacked on.  Double colon lines are optional 'type annotations' that convey a value's type to the reader of the program and to the compiler.

``` active haskell 
-- show
-- The definition of a Palatability Type
data Palatability = Delicious      -- Palatability values may be Delicious
                  | NotDelicious   -- OR NotDelicious

-- The definition of type Taste
data Taste = Sweet                 -- Values of type Taste may be Sweet 
           | Salty                 -- OR Salty
           | Umami                 -- OR ... ...
           | Bitter Palatability   -- Bitter and Sour values must specify
           | Sour Palatability     -- whether they are good or bad


-- Example value of type Taste
cookieTaste :: Taste               -- Here is a type annotation
cookieTaste = Sweet

brusselsSproutsTaste = Bitter NotDelicious

harpoonIpaTaste =   Bitter Delicious

someTestTaste = Sweet Delicious -- An accident that the compiler will catch
                                -- Sweet values don't come with Palatability
                                -- data
anotherTestTaste  = Bitter

-- /show
main = print "OK"
```
Try to make the above code compile by finding and fixing the malformed values.  Programming Haskell in the real world is like this - a back-and-forth between you and the compiler until your use of types lines up with the definition you created for them.

### Pattern Matching

Where Algebraic Data Types give structure to values, Pattern Matching allows functions to take advantage of that structure.  A function with an ADT parameter may 'match' each possible value in turn, and bind any accompanying data for use within the function. 

In this example, a sensor reading can only be extracted from a SensorReading value tagged as 'ValidReading'.  If a function tried to extract a value from an invalid sample, that would be a type error, and the program would simply fail to compile.  If a programmer forgot to check for the possibility of error by handling the HardwareError case, the compiler would issue a warning.  We are using types to enforce invariants in the code.  The ability to move a program's *semantics* up into the type system, where they can be checked by the compiler, provides a powerful bug-deterrent.

``` active haskell
--show
data SensorReading = ValidReading Float  
            | HardwareError

updateHeading :: SensorReading -> Float -> Float
updateHeading (ValidReading r) oldHeading = oldHeading + r
updateHeading HardwareError oldHeading = oldHeading
-- /show
main = print "OK"
```

Try as you might - it is impossible to accidentally read a value from a SensorReading carrying the `HardwareError` tag.  The compiler won't let you.

Using C structs, or objects Python, the programmer would have *remember* to check validity flags by hand in every function using SensorReadings; or values would need to be accessed through special helper functions.  Forgetting to exercise such discipline in other languages can result in errors that are silent all the way up until the project is deployed / the analysis is done and the paper is published / the flight-control system is up in the air.  Using ADT's and pattern matching, more errors are caught during compilation and fixed before they can wreak havoc.

Pattern matching also provides a very expressive way to manipulate data structures.  Here we define a binary tree type, which may either be `Empty`, or a `Branch` with two sub-trees and a node value.  The function `insert` breaks down `Branch` values into named parts that get used in the function.  See the full code for more pattern matching examples.

``` active haskell

-- show
data CharTree = Empty 
              | Branch CharTree Char CharTree
  deriving (Show)

-- depth takes a tree and returns its depth
depth :: CharTree -> Integer
depth Empty           = 0                            -- Handle the empty case
depth (Branch l el r) = 1 + max (depth l) (depth r)  -- Handle the branch case

-- insert takes a tree and a character c, returning a new tree with c inserted
insert :: CharTree -> Char -> CharTree
insert Empty el = Branch Empty el Empty   -- Handle the empty case
insert (Branch l c r) el                  -- Recursively handle Branch case...
 | el <= c   = Branch (insert l el) c r   --   when element <= center node
 | el >  c   = Branch l c (insert r el)   --   when element >  center node
-- /show 
 
isElement :: CharTree -> Char -> Bool
isElement Empty el = False
isElement (Branch l c r) el 
 | el == c   = True
 | el < c    = isElement l el
 | el > c    = isElement r el


-- See the note about function pointers above to learn about foldl
-- show
-- Insert all the characters from this string into the tree in turn
myTree = foldl insert Empty "Hello World!" 
myTestChar = 'o'
-- /show

reportTest :: Char -> String
reportTest testChar
 | isElement myTree testChar = "We found your element - " ++ [testChar]
 | otherwise                 = "Could not find " ++ [testChar] 
 
main = do 
    putStrLn $ "Depth: " ++ show (depth myTree)
    putStrLn $ reportTest myTestChar
    putStrLn $ "The tree: " ++ show myTree

```

Note: Pattern matching is a big help during code refactoring, as the compiler tells you the location of every function that needs to be changed to reflect a change that you make in a data type.  This feature isn't turned on yet on the School of Haskell website, but it works just fine in your project.

@@@

### Pattern Matching guides you through code refactorings

Data structures often change as a program grows.  Haskell makes the process of synchronizing the rest of the program far less painful than it is in languages with weaker type systems.  Add `Tornado` as a flight condition and recompile.  The compiler makes sure that every function handles all possible input values, and will tell you which functions need to be modified to accommodate your refactoring.  (In the case of a tornado, set the wingDirection to Up and thrust to Afterburners)

``` active haskell

data Direction = WLeft | WRight | WUp | WDown
  deriving (Show)

data ThrustStrength = EnginesOff | Thrust Integer | Afterburners
  deriving (Show)

data FlightCondition = Calm
                     | LightWind Direction

-- Given weather conditions and our current wing direction,
-- what should our new wing direction be?
wingResponse :: FlightCondition -> Direction -> Direction
wingResponse Calm d               = d      -- In calm conditions stay the course
wingResponse (LightWind WLeft) _  = WRight -- Bank against light wind
wingResponse (LightWind WRight) _ = WLeft  -- Bank against light wind
wingResponse (LightWind _) d      = d      -- In up/down wind, stay the course 

thrustResponse Calm t = t                           
thrustResponse (LightWind _) (Thrust t) = Thrust (t+10)  -- Increase power against wind
-- /show

main = print $ wingResponse (LightWind WUp) WLeft

```

@@@

### Generic programming from the ground up

In functional programming you call it 'polymorphism', and its use predates the templates and generics that you see in c++, Java, etc.  Haskell's support for polymorphic data types and polymorphic functions is excellent in terms of syntactic simplicity, expressiveness, and speed.  Here is an example of a polymorphic tree.  You can see that the syntax is so natural that there was never a need to make trees non-generic in the first place.

``` active haskell
import Data.Map hiding (insert, foldl)
-- show
-- A tree with nodes that can take any type
data MyTree a = MyEmptyTree | Branch (MyTree a) a (MyTree a)
  deriving (Show)

-- insert works for MyTrees of any type that can be ordered
insert :: Ord a => MyTree a -> a -> MyTree a
insert MyEmptyTree el = Branch MyEmptyTree el MyEmptyTree
insert (Branch l c r) el
 | el <= c    = Branch (insert l el) c r 
 | el >  c    = Branch l c (insert r el) 
  
-- A MyTree of Strings   
myTree :: MyTree String
myTree = foldl insert MyEmptyTree ["Ada","Brent","Conel","Doaitse","Ertugrul"]

-- A MyTree of (Map from String to Integer),
-- just to demonstrate that we can make trees of ANY orderable type
myTree2 :: MyTree (Map String Integer)
myTree2 = foldl insert MyEmptyTree 
    [fromList [("c++", 6),("ocaml", 9),("haskell",10),("python",7)]
    ,fromList [("LeBron", 250),("Rajon", 186),("Kyrie",191)]
    ,fromList [("Big Mac",550),("Whopper",670),("Baconator",970)] ]
-- /show
main = do {print myTree; print myTree2}

```

## Functional Purity and Immutable Data: A different way of thinking about programming

Functions in FP are modeled after mathematical functions: instead of performing a sequence of operations, they express strict mathematical relationships between arguments and a return value.  Imperative programs are *sequences* of commands that each have an effect on memory or on the computer's output.  *Functional programs are collections of statements that are true at all times.*  

Purity and immutability are what makes functional programming seem so alien.  They are also what gives functional programming its power.  When using imperative languages (c, c++, Java, Python, C#, etc etc), the programmer has to keep a mental timeline of their program's execution.  Immutability in FP frees functional programmers from having to think about sequences of changes - they focus instead on the validity of the mathematical relationships between types they design.  

Here are a few surprising examples of this.

###Surprise #1: Variables never change their value

``` active haskell
-- show
a = 5      -- ERROR! To quote Miran Lipovača, "If you say that a is 5, 
a = a + 1  -- you can't say it's something else later because you just 
           -- said it was 5. What are you, some kind of liar?"
-- /show
main = print "OK"
```

What we do instead depends on our reason for wanting to increment the value `a` in the first place.

``` active haskell
a = 5
-- Simply refer to the incremented value
b = (a + 1)

-- Define a function that refers to a incremented
incr a = a + 1
b = incr a
```

###Surprise #2: No for loops

``` haskell
-- There is no such thing in Haskell, 
-- because the values of i, a, and b are changing
a = [1, 2, 3, 4, 5]
b = 1.5
for (i = 0; i < 5; i++)
  a(i) = 2 * a(i)
end

for (i = 0; i < 100, i++)
  b = b^2 - 1
end
```

Wherever a for loop is used in imperative code, functional languages offer combinators that more precisely capture the nature of that particular loop.  Two short examples:

``` active haskell
-- show
a = [1,2,3,4,5]

-- map maps a function (here, the number-doubling function)
-- over all elements in a list
r1 = map (* 2) a

-- iterate recursively applies a function to its argument
-- to any depth.  Take and last pull out the value you're
-- interested in from that list
r2 = last . take 100 $ iterate (\x -> x^2 - 1) 1.5
-- /show

main = do
  putStrLn $ "r1: " ++ show r1
  putStrLn $ "r2: " ++ show r2
  
```

It's natural to wonder, Without being able to change variables, how can you possibly get any work done?  The general answer is, You quickly get used to working with recursion, and you get used to the combinators that abstract over recursion.  Believe it or not, coding in this style is a lot of fun, and if you learn Haskell, you will probably find yourself re-writing these combinators for your own use in the other languages you work with.

###Surprise #3: Functions other than main can't do any I/O

Functions that do I/O aren't functions in the mathematical sense, because they *do things* instead of expressing relationships.  A function that does input could return different values each time it was run, and a function that does output can't be treated as an algebraic entity that can be combined with others and safely moved around, as terms often are in actual algebra.

``` haskell
printAndDouble :: Integer -> Integer  
printAndDouble n = print n;   -- Not allowed!  We give up this sort of
                   (n * 2)    -- thing to pay for Haskell's other features
```
If functions can't do I/O, how does a functional program interact with the world, read data, communicate over the internet?  They use "monads".

## A Monad safely mediates communication between the real world and pure functions

Data in the world (on a disk, on the internet, from the keyboard) may only enter and leave a purely functional program through a dedicated channel called the `IO` monad.  In general, monads offer specific patterns that simulate sequential code execution, in much the same way that `map` and `iterate` simulate particular patterns found in `for` loops, without actually mutating data.

Haskell's type system enforces the rule that only values that are 'in the IO monad' may interact with the real world.  A functional program is structured so that the majority of its functions are pure, communicating with the real world using only a few functions in the IO monad.

``` active haskell
import System.IO
import Control.Monad

shiftLetter :: Integer -> Char -> Char
shiftLetter n c
 | n == 0      = c
 | n > 0       = shiftLetter (n - 1) (succ c)
 | n < 0       = shiftLetter (n + 1) (pred c)

-- simulating sequential code in the IO monad
main :: IO ()
main = do
    hSetBuffering stdout NoBuffering
    hSetBuffering stdin  NoBuffering
    putStr "How many lines do you want to encode: "
    nLines <- readLn
    putStr "How deep do you want each line to be encoded: "
    depth  <- readLn
    sequence_ . replicate nLines $ do
      putStr "Enter a string: "
      entry <- getLine
      putStrLn $ "Encoded: " ++ (map (shiftLetter depth) entry) ++ "\n"
```

That is not to say that occurrences of `putStrLn` and `getLine` can only exist directly within the `main` function.  On the contrary, `putStrLn` is a function that can be passed to and from other higher-order functions, integrated into lists and data structures, and glued to other IO-performing functions using monad combinators.  As you learn Haskell, you will discover how monads can help you raise the expressiveness and abstraction level of sequential code. A common metaphor for monads is 'a programmable semicolon', because they allow you to flexibly define what it means to do actions in a sequence.

You may ask, "What exactly *is* a Monad?"  A very rough answer is: any data type for which you can explain how to program the semicolon.  Too abstract?  It will all become clear as you see them more and use them in your own code.

## Haskell can be Pragmatic

### Haskell code runs really fast

Although Haskell is a very high-level language, it's now a top contender in code speed shootouts.  Haskell compiles to LLVM and to native code on Linux, OS X, and Windows.  Unoptimized code compiled with GHC (the most commonly used of Haskell's several compilers) generally runs within 1/5 the speed of unoptimized c programs, and quite a bit faster than Python.  Heavily optimized Haskell code is generally within half the speed of heavily optimized c; in some cases faster.


### Haskell has pretty strong library offerings

[Hackage](http://hackage.haskell.org) is an online database with many high-quality, community-written libraries covering lots of areas (linear algebra, crypto, web frameworks, etc. etc. etc.).  The School of Haskell website is powered by Yesod, a powerful Haskell web framework.  There are Haskell bindings to OpenGL, GUI libraries, databases, CUDA, and on.  Libraries for concurrency and parallelism are particularly strong.

### Haskell has incredible Unit Testing

Haskell's QuickCheck library is an absolutely wonderful unit-testing framework.  You use QuickCheck to test algorithms by defining a collection of properties of that algorithm that should always be true.  QuickCheck uses the function types to intelligently generate randomized data sets (as many as you like - the default is 100) that get into far more corner-cases than any unit-test writer would ever care to write by hand.

``` active haskell
import Data.Map hiding (insert, foldl)
import Test.QuickCheck

-- A tree with nodes that can take any type
-- show
data MyTree a = MyEmptyTree | Branch (MyTree a) a (MyTree a)
  deriving (Show)
-- /show

-- depth recursively descends the tree to find its maximum depth
depth :: MyTree a -> Integer
depth MyEmptyTree = 0
depth (Branch l _ r) = 1 + max (depth l) (depth r)

-- nElem recursively counts elements
nElem :: MyTree a -> Integer
nElem MyEmptyTree = 0
nElem (Branch l _ r ) = 1 + (nElem l) + (nElem r)

-- show
-- insert works for MyTrees of any type that can be ordered
insert :: Ord a => MyTree a -> a -> MyTree a
insert MyEmptyTree el = Branch MyEmptyTree el MyEmptyTree
insert (Branch l c r) el
 | el <= c    = Branch (insert l el) c r 
 | el >  c    = Branch l c (insert r el) 
-- /show
 
isElem :: Ord a => a -> MyTree a -> Bool
isElem _ MyEmptyTree = False
isElem el (Branch l c r)
 | el == c  = True
 | el <  c  = isElem el l
 | el >  c  = isElem el r
 
-- show
-- QuickCheck will generate random inputs to test these properties
prop_insIsElem :: (MyTree Double) -> Double -> Bool
prop_insIsElem tree el =  isElem el (insert tree el)

prop_prevIsElem :: (MyTree Double) -> Double -> Double -> Bool
prop_prevIsElem tree el el' = isElem el (insert (insert tree el) el')

prop_insChangeDepth :: (MyTree Double) -> Double -> Bool
prop_insChangeDepth tree el = (depth (insert tree el)) - depth(tree) <= 1

prop_depthLowerBound :: (MyTree Double) -> Bool
prop_depthLowerBound tree = (depth tree) >= 
                            floor (logBase 2 $ fromIntegral(nElem tree))
-- /show
instance (Arbitrary a, Ord a) => Arbitrary (MyTree a) where
  arbitrary = do
    elems <- listOf arbitrary
    return (foldl insert MyEmptyTree elems)
    
main = do
  let args = stdArgs { maxSuccess = 10, maxSize = 10, chatty = True }
  putStrLn "Checking insertion properties."
  quickCheckWith args prop_insIsElem
  quickCheckWith args prop_prevIsElem
  putStrLn "Checking depth properties."
  quickCheckWith args prop_insChangeDepth
  quickCheckWith args prop_depthLowerBound

```

## Haskell has personality

Haskell is backed by a lot of amazing people - both great engineers and great teachers.  It has a cool [history](http://www.youtube.com/watch?v=7NPBrWDzO2A) and a [formal background](http://www.youtube.com/watch?v=2PJ_DbKGFUA) that might inspire you to learn some abstract math.  The #haskell IRC channel and haskell-cafe mailing list are famously friendly and educational places to hang out.  

Seasoned Haskellers flock to new language features and take a peculiar interest in teaching about them.  [Haskell Weekly News](http://contemplatecode.blogspot.com/) is consistently filled with links to blog posts explaining interesting corners of the language and reddit stories about new experiments and abstractions.  

You can challenge yourself for a very long time before you can reach to all corners of the Haskell world; and as a research language, it is always growing.  

## You can do it!  Where to go next.

It's rumored that Haskell is hard.  It's probably more accurate to say that Haskell makes easy things a little tricky; while bring things that would otherwise be extremely difficult within reach.

It's important to remember that no one was born knowing Haskell.  We all came to it out of curiosity and a sense of adventure.  You *will* have to study in order to get Haskell, but that's half of the fun.

If you are ready to start learning the language, take a look at [Learn You a Haskell for Great Good](http://learnyouahaskell.com) and [Real World Haskell](http://book.realworldhaskell.org).  Check out the many tutorials here at [School of Haskell](https://www.fpcomplete.com/).  Find other resources at [haskell.org](http://haskell.org).  And download the [Haskell Platform](http://haskell.org/platform).  Have fun!

**


Feel free to help out or suggest improvements for this post on [github](https://github.com/ImAlsoGreg/FPEP/).
# Introductory Haskell Materials

# __[http://bit.ly/hintro](http://bit.ly/hintro)__

Description from the [DevBeat 2013 site](http://venturebeat.com/events/devbeat2013/)

*Wednesday, November 13, 2013, Master Class: Haskell*

Nearly everyone who learns Haskell seems to turn into a raving Haskell fan. They say things like “I can write 80% less code and still get native multicore performance” and “I can easily understand code I wrote two years ago” and “my programs often work the first time they compile" and “now when I have to use Java, I feel like I’m being punished.” What makes Haskell so productive, fast, maintainable, and bug-free? And can you learn it without your head exploding? In this talk Aaron, a raving fan himself, will walk you through the basics of Haskell. You’ll see how to use interactive Haskell tools to become productive and learn quickly. By the end of the talk you will know what Haskell’s about AND you’ll be ready to use free online Haskell resources to easily learn more.

## Materials 

In addition to this article, I used some code in the IDE during this lecture. The IDE is browser-based; [get your account here](https://www.fpcomplete.com/business/haskell-center/overview/) so you can follow along.

- [Code snippets](https://www.fpcomplete.com/user/FPguy/devbeat13-samples) 
- [Simple Sudoku solver](https://www.fpcomplete.com/user/FPguy/sudoku-devbeat) 

# Agenda + code illustrations 

*Credits: In preparing this material I made use of contributions to the School of Haskell by Yann Esposito, Bartosz Milewski, and Brent Yorgey.*


## Haskell features

- Purely functional, no side effects
- Powerful type system
- Laziness
- Use of monads

## Haskell benefits

- Fast time to initial working version
- Maintainability, abstraction, reuse
- Low bugs, mostly caught at compile time
- Native C++ like performance

## Typical uses of Haskell

- Fundamental data analysis for Web and Mobile
    - Bump
    - Facebook
- Unique/competitive [big] data analysis
    - New York Times
- Financial analysis
    - Netrium
    - Standard Chartered Bank
    - OTAS Technologies
- Scientific modeling
    - Amgen
- Adding headroom to Ruby/Python projects
    - Janrain


## Concise syntax

- low punctuation, significant indentation
- function call syntax

```active haskell
add a b = a + b 
main = do
  print (add 6 5)
  print (6 `add` 5)
 
```

- Operators like `+` are just functions with names made of symbol characters
- Defaults to infix notation for traditional nice syntax
- !#$%&\*+./<=>?@\^|-~ (and non-ascii Unicode symbol or punctuation)


```active haskell
a !@ b = a + b
a ∆ b = a - b

main = do
  print (6 !@ 5)
  print ((!@) 6 5)
  print (6 ∆ 5)
 
```
- `a $ b c d e` sugar for `a (b c d e)`
```active haskell
main = do
  print (5 + 2 * 3)
  print $ 5 + 2 * 3
```

- Function composition: `(a . b) c` means `a (b c)`

```active haskell
double x = x + x
sixteenTimes = double . double . double . double
main = do
  print $ double 2
  print $ (double . double . double) 2
  print $ sixteenTimes 2

```
- Type definitions can be omitted & deduced

## Some interesting types

- Numbers: Integer Int Float Double Complex and more, all in type class Num

```active haskell
import Data.Ratio   -- defines type Rational in type class Num
oneThird = 1 % 3
fiveEighths = 5 % 8
main = print (2 * oneThird * fiveEighths)

```

- Lists [a,b,c] or [n1,n2..nn] 
- Infinite lists (note: lazy evaluation) [n1,n2..]
- List comprehensions
```active haskell
ns = [1..20]
evens = [2,4..20]
allPos = [1,2..]
favoritePos = [x | x <- allPos, x `mod` 10 == 0]
main = do
  print $ take 5 ns
  print $ take 5 allPos
  print $ take 5 favoritePos
  
```
- Huge set of utility functions `import Data.List`
- Tuples

```active haskell
a = (5,2)
b = (10,1)
items = a : (replicate 3 b)
main = do
  print items
```

- `type` creates synonyms of existing types
- `data` creates new types
```active haskell
data Breakfast = Cereal | Yogurt | Egg Integer 
  deriving (Show, Eq)
  
a = Yogurt
b = Egg 3

moreEgg (Egg n) = Egg (n+1)
moreEgg x = x -- Adding an egg to anything else has no effect

addEggs k (Egg n) = if k < 1 then Egg n else addEggs (k-1) (Egg (n+1))
addEggs k x = x 

main = do
  print a
  print b
  print $ moreEgg a
  print $ moreEgg b
  print $ (moreEgg . moreEgg . moreEgg) b
  print $ addEggs 14 (Egg 2)
  print $ (map moreEgg) [Yogurt, Yogurt, Egg 4, Cereal, Egg 1]
```

- constructors are functions
- `Maybe` can be `Just a` (where a is some type) or `Nothing`

```active haskell
import Data.Maybe
x = Just 5
y = Nothing
safeAdd :: Maybe Int -> Maybe Int -> Maybe Int
safeAdd (Just a) (Just b) = Just (a+b)
safeAdd _ _ = Nothing
main = do
  print $ safeAdd x y
  print $ safeAdd (Just 2) (Just 5)
```

- `Either` can be `Left a` or `Right b` (e.g. use Left String for errors, Right Int for answers)
- Record syntax, accessors
- Recursive types
- `deriving` is like inheritance

## Making a function

- Function types
- Styles of handling inputs
```haskell
tryDivi :: Double -> Double -> Maybe Double
tryDivi a b = if b==0 then Nothing else Just (a / b)   -- conditional

tryDivide :: Double -> Double -> Maybe Double
tryDivide _ 0 = Nothing      -- pattern matching
trydivide a b = Just (a / b)

tryDiv :: Double -> Double -> Maybe Double
tryDiv a b 
  | b==0 = Nothing           -- guards
  | otherwise = Just (a / b)
```
- Eliminating final argument ("eta reduction")


```active haskell
inc x = x + 1
inc' = (+ 1)     -- final arg on left = final arg on right
add a b = a + b
addFive = add 5  -- partial evaluation returns a function: "Currying"
inc'' = add 1
main = do
  print $ inc' 10
  print $ addFive 10
  
```




## Higher-order functions

- Function as parameter

```active haskell
increment = (+ 1)
doTwice f x = f (f x)
main = do
  print $ increment 5
  print $ doTwice increment 5
  print $ map increment [1..9]
```

- Lambda (written as backslash) for anonymous functions

```active haskell
main = do
  print $ map (\x -> x * x + 1)  [1..9]
```

## Eliminating iterators

- Recursion, tail recursion
```active haskell
-- Apply a function to every list element using recursion
-- Works for a list of any type a and returns a list of any type b
--   assuming that f is a function that takes an a and returns a b
-- Note: this trivial function is equivalent to map
apply :: (a -> b) -> [a] -> [b]
apply f [] = []
apply f [x] = [f x]
apply f (x : xs) = (f x) : (apply f xs)
main = do
  print $ apply (* 2) [1..5]
  print $ apply (* 2) []
  print $ apply (\x -> x * x) [1..5]
  print $ take 20 $ apply show [10,20..]
```

- `fold` reduces a whole list to a single result
- Important to choose a variant that doesn't use huge temporary memory


## Monads

- A burrito: a value-added container for oblivious but valuable contents
- A typeclass (keyword `class`) is like an `interface` in OOP
- `Monad` is a typeclass using `>>=` ("bind") and `return` and `>>`
- Wrap a type into a value-added type
- Used for sequencing and passing on value-added state

## I/O 

- Uses the IO () monad
- Special `do` syntax for sequences of monads looks like imperative code
- Separates side-effectful code from purely functional code
- `putStr` and `putStrLn`
- `print` for `(putStrLn . show)`

## Sample analytics app, and deployment

- Simple Sudoku solver [(Open in IDE)](https://www.fpcomplete.com/user/FPguy/sudoku-devbeat)
- Running in the IDE / Console
- Application Server: easily deploy and manage

## Sample web app

- "Gravity Fun" physics simulation
- JavaScript front end
- JSON to communicate with server-based physics model
- Run in the IDE / Browser tab
- Application Server: easily deploy and manage

## Parallel Processing

- Native high-performance threads
- Lack of side effects means safe concurrency
- Software Transactional Memory
- Parallel Haskell
- Cloud Haskell
- Get started trivially:

```active haskell
import Control.Parallel.Strategies

slowFib x = if x<3 then 1 else (slowFib (x-1)) + (slowFib (x-2))
inputs = [17, 7, 15, 5, 18, 2, 15, 3, 24]

main = do
  print $ map slowFib inputs            -- sequential
  print $ (parMap rpar) slowFib inputs  -- parallel
```

## some interesting packages

- Yesod or Happstack to make a Web server
- Fay or GHCJS or Haste to make JavaScript
- Database access

# Resources to learn and use Haskell 

To learn Haskell, from beginner through advanced topics, I strongly recommend the free but powerful online [School of Haskell](https://www.fpcomplete.com/school) including its [Starting with Haskell](https://www.fpcomplete.com/school/starting-with-haskell) section. 

The commercial IDE for Haskell, which you can try for free in your browser, is [FP Haskell Center.](https://www.fpcomplete.com/business/haskell-center/overview/). [Here](http://www.youtube.com/watch?v=ZHSBwlm5C8U) is a screencast by me showing how it works.

If you prefer your learning in book form, the text [Learn You a Haskell for Great Good](http://learnyouahaskell.com) is a good choice and so is [Real World Haskell](http://book.realworldhaskell.org). These sites include the full text of the the books online, as well as links to order the e-book or dead-tree version.

Online Haskell discussions are found notably [on reddit](http://haskell.reddit.com) and [on Stack Overflow](http://stackoverflow.com/questions/tagged/haskell). There are also [Haskell mailing lists.](http://www.haskell.org/haskellwiki/Mailing_lists)

On-site Haskell classes for companies are available from FP Complete and from Well-Typed.

Online headquarters for the Haskell open-source community is [haskell.org](http://www.haskell.org). The site includes a wiki of diverse Haskell information, and links for downloading open-source Haskell tools.

The main online database of Haskell packages is Hackage. There is a compatibility-checked subset called Stackage, also used in FP Haskell Center.

The main online help repository for Haskell is Hoogle. It is available as a website and within FP Haskell Center.

The biggest Haskell conference is the annual [ACM SIGPLAN Haskell Symposium.](http://www.haskell.org/haskell-symposium/2013/)

[Here](http://www.youtube.com/watch?v=Fqi0Xu2Enaw) is a video introducing Haskell at an overview/manager level. Show it to your manager!


# Sample commercial uses of Haskell

Haskell is used commercially in various industries including finance, biotechnology, defense, and a range of Internet companies.

[Detailed commercial case studies](https://www.fpcomplete.com/page/case-studies) are published by FP Complete.

[Another interesting case study](http://source.opennews.org/en-US/articles/model-analysis/) about Haskell usage at the New York Times.

The [Haskell in Industry](http://www.haskell.org/haskellwiki/Haskell_in_industry) Wiki page lists further examples.

[Haskell Talks](https://www.fpcomplete.com/business/blogs/video-blog/) is a recently started series of video interviews with Haskell users.



# About the speaker

![Picture of the Author](https://www.fpcomplete.com/media/591dcad7-fdf1-4d8f-b3d6-ad4d61a94511.jpeg)

Aaron Contorer is CEO and founder of FP Complete, specializing in Haskell tools and services.  His longer bio is [here](https://www.fpcomplete.com/business/about/leadership/). 

Dr. Dobb's Journal recently published his article [In Praise of Haskell.](http://www.drdobbs.com/architecture-and-design/in-praise-of-haskell/240163246)

Aaron [tweets as @contorer](https://twitter.com/Contorer) and [is also on LinkedIn](https://www.linkedin.com/in/acontorer). You can contact him personally at aaron *//at//* fpcomplete *//dot//* com.


*(c) 2013 FP Complete Corporation.*





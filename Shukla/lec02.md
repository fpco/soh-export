# CS 1501: Introduction to Haskell, Lecture 2

## What is this?
You can run Haskell code directly from his website. 

Every code block in [these lecture slides](http://shuklan.com/haskell/lec02.html) points to one of the sections below.

## Jumping into GHCi

``` active haskell
main = putStrLn (show number)
-- show
number = 4317 * 103
-- /show
```

``` active haskell
main = putStrLn (show output)
-- show
x = 3.0
y = 4.0
output = sqrt( x^2 + y^2 )
-- /show
```

``` active haskell
main = putStrLn (show number)
-- show
number = 2^1000
-- /show
```


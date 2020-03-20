# Infinite subsets of natural numbers

## Natural numbers in Haskell

We will show how to use Haskell in order to solve mathematical questions about natural numbers. The problems below are motivated by the [Project Euler](http://projecteuler.net/). The sketches or partial solutions are meant to inspire you to perform further investigation of the problems on your own. You may need to do a substantial optimization of the code, in particular of the underlying data structures, in order to get fast solutions working on inputs proposed in the [Project Euler](http://projecteuler.net/).
This is an example how Haskell handles infinite objects:

``` active haskell
naturals = [1..]
firstTen = take 10 naturals

main = print ( firstTen )
```

We decided to represent natural numbers as an infinite list.
Haskell allows certain operations on infinite objects, in particular selecting elements from a given list.

## Prime numbers

Let us analyze an example. A given natural number n is _prime_ if it is bigger or equal 2 and is not divisible by any natural number except 1 and n. 
As a preparatory step in our definition of prime numbers we define a function _ldf_. For two given numbers k and n  the function _ldf_ returns the smallest divisor of n which is bigger or equal to k.

``` active haskell
ldf :: Integer -> Integer -> Integer
ldf k n | rem n k == 0   = k
        | otherwise      = ldf (k+1) n

main = do
    print ( ldf 2 5 )
    print ( ldf 7 36 )
```

Now notice, that we can select prime numbers from the naturals using the following procedure:

``` active haskell
naturals = [1..]
ldf :: Integer -> Integer -> Integer
ldf k n | rem n k == 0   = k
        | k^2 > n        = n
        | otherwise      = ldf (k+1) n
```


@@@
```
prime :: Integer -> Bool
prime n | n < 1          = error "this is not a positive number"
        | n == 1         = False
        | otherwise      = ldf 2 n == n
       
primes = filter prime naturals

main = do 
    print ( take 10 primes )
```
@@@

Motivated by [Problem 7](http://projecteuler.net/problem=7) from the Project Euler list we may easily figure out the 501st prime number

``` active haskell
naturals = [1..]
ldf :: Integer -> Integer -> Integer
ldf k n | rem n k == 0   = k
        | k^2 > n        = n
        | otherwise      = ldf (k+1) n

prime :: Integer -> Bool
prime n | n < 1          = error "this is not a positive number"
        | n == 1         = False
        | otherwise      = ldf 2 n == n

primes = filter prime naturals
```

@@@
```
main = do 
    print ( primes !! 500 )
```
@@@


Exercise. Modify the code and find 500 number which is not divisible by 2,3,5,7,11.

## Collatz sequence

Collatz sequence is an [interesting sequence](http://en.wikipedia.org/wiki/Collatz_conjecture) defined in recurential terms. The limit behavior of the sequence is not completely understood. However, we may implement the Collatz sequence in Haskell and ask about its properties such as the longest Collatz sequence _starting from a given number n_. The is a version of [Problem 14](http://projecteuler.net/problem=14) from the Project Euler. 

``` active haskell
import Data.List

collatz :: Integer -> Maybe (Integer, Integer) 
collatz n | n == 1         = Nothing
          | rem n 2 == 0   = Just ( n, div n 2 )    
          | otherwise      = Just ( n, 3*n+1 )
                 
collatzSeq :: Integer -> [Integer]
collatzSeq n = unfoldr collatz n
 
collatzNumbers = [ length ( collatzSeq n ) | n <- [1..10^3] ]
longestCollatz = maximum collatzNumbers

main = do 
    print ( collatzSeq 15 )
    print ( longestCollatz )
    print ( elemIndex longestCollatz collatzNumbers )
```

Exercise 2. Implement the [Syracuse function](http://en.wikipedia.org/w/index.php?title=Collatz_conjecture&action=edit&section=25) and find the longest Syracuse sequence for odd integers below 1000. 

## Declarative programming 

The declarative style of programming implies that some problems find immediate solutions in Haskell. As an example, consider [Problem 29](http://projecteuler.net/problem=29) from the Project Euler. 


``` active haskell
import Data.List

powers = [ a^b | a <- [2..100], b <- [2..100] ]
purePowers = nub powers

main = do 
    print ( length powers )
    print ( length purePowers )
```

Another example, where the declarative programming allows an immediate solution is
[Problem 31](http://projecteuler.net/problem=31)

``` active haskell
import Data.List

change = [ [a,b,c,d,e,f,g,h] | a  <- [0..200],b <- [0..100],c <- [0..50],d <- [0..20],e <- [0..10],f <- [0..4],g <- [0..2],h <- [0..1], a+2*b+5*c+10*d+20*e+50*f+100*g+200*h == 200 ]

main = print ( length change )
```

## A bit more involved problem with prime numbers

Different prime factors, [Problem 47](http://projecteuler.net/problem=47)

``` active haskell
import Data.List

naturals = [1..]

ldf :: Integer -> Integer -> Integer
ldf k n | rem n k == 0   = k
        | k^2 > n        = n
        | otherwise      = ldf (k+1) n

prime :: Integer -> Bool
prime n | n < 1          = error "this is not a positive number"
        | n == 1         = False
        | otherwise      = ldf 2 n == n
       
primes = filter prime naturals
```

@@@
```
divides :: Integer -> Integer -> Bool
divides n m = rem m n == 0

primeFactors :: Integer -> [Integer]
primeFactors x = unfoldr findFactor x
   where
       findFactor 1 = Nothing
       findFactor x = Just (nextFactor, x `div` nextFactor)
           where nextFactor = head $ filter (\f -> f `divides` x) primes

factors :: Integer -> [Integer]
factors n = nub ( primeFactors n ) 

triples = [ n | n <- [1..10^3], length ( factors n ) == 3, length ( factors (n+1) ) == 3, length ( factors (n+2) ) == 3]

pairs = [ n | n <- [1..10^3], length ( factors n ) == 2, length ( factors (n+1) ) == 2]

main = do
    print ( primeFactors 21 )
    print ( primeFactors 32 )
    print ( factors 32 )
    print ( head triples )
    print ( head pairs )
```
@@@
# Polynomials

#What is a Polynomial?
Here, we will implement standard mathematical operations on polynomials. 

If you do not know what a polynomial is, here is an example:

``` haskell
f x = x**2 + 3*x + 2
```

In the example, `f` is a _polynomial function_, and the right-hand-side is an expression of a polynomial.

##How will we represent a Polynomial?
More specifically, we will represent polynomials as a list of coefficients:

``` haskell
(x**2 + 3*x + 2) {- is represented by: -} [2,3,1]

(20) {- is represented by: -} [20]

(x) {- is represented by: -} [0,1]

(c0 + c1*x + c2*x**2 + ...) {- is represented by: -} [c0,c1,c2,..]

```

##Proper and Improper Representations

Mathematically, something like (`0*x**2 + x + 0`) is equal to (`x`), but one of them contains more symbols than the other.

For simplicity, we will say:

``` haskell
(0*x**2 + x + 0) {- is represented by: -} [0,1,0]
(x)  {- is represented by: -} [0,1]
```

so that by looking at the expression of a polynomial, we can immediately get a list of coefficients.

This decision implies that a polynomial like `0` has an infinity of representations:

``` haskell
listOfAllRepresentationsOfZero = [[],[0],[0,0],[0,0,0],..]
```

To avoid unnecessary memory usage, we would like to use `[]` to represent `0`; it is the minimal representation. So, we say that the minimal representation is the proper representation, and any other representation is improper.

It is possible to define a function which takes any finite representation, and turns it into a proper representation:

``` active haskell
import Data.List
-- show This gets rid of all the zeroes at the end of the representation.
toProper [] = []
toProper p = if (last p /= 0) -- this means there is nothing to do.
             then p 
             else toProper $ init p -- this means the last zero can be discarded.

main = do print $ toProper []
          print $ toProper [0,0,0]
          print $ toProper [0,1]
          print $ toProper [0,1,0]
-- /show
```

#Operations

First, we implement operations like `(+)` for representations which are not necessarily proper. Afterwards, we will use `toProper` to "close" these operations for proper polynomials.

##You can add two Polynomials
``` active haskell
addPoly p1 p2 = if (length p1 >= length p2)
                then zipWith (+) p1 (p2 ++ repeat)
                else addPoly p2 p1
                
{-
if you have 

(2*x**2 + 3*x + 2) + (x)

you rewrite

(2*x**2 + 3*x + 2) + (0*x**2 + x + 0) = (2+0)*x**2 + (3+1)*x + (2+0)
-}

main = print $ [0,1] `addPoly` [2,3,2]
```

##You can multiply a polynomial by a constant

``` active haskell
multiplyBy a p1 = map (a*) p1

{-
if you have 
4 * (2*x**2 + 3*x + 2) = (4*2)*x**2 + (4*3)*x + (4*2) 
-}

main = do print $ 4 `multiplyBy` [2,3,2]
```
We see that this operation is already closed over proper representations.

##You can multiply by `x`

``` active haskell
multiplyByX p = 0:p

{-
if you have 
(x) * (x**2 + 3*x + 4) = (x**3 + 3*x**2 + 4*x + 0) 
-}

main = do print $  multiplyByX [4,3,1]
```

We see that this is not closed over proper representations. However, it is almost closed. 

Exercise: Find the only case where this transforms a proper representation into an improper one.

##We can multiply two polynomials

``` active haskell
-- show
multPoly [] p2 = []
multPoly (p:p1) p2 = let pTimesP2 = multiplyBy p p2
                         xTimesP1Timesp2 = multiplyByX $ multPoly p1 p2
                     in addPoly pTimesP2 xTimesP1Timesp2    

{- 
if we have
(2*x + 1) * (x**2 + 3*x + 2) = 1 * (x**2 + 3*x + 2) + (x) * (2) * (x**2 + 3*x + 2)
-}

main = print $ multPoly [1,2] [2,3,1]

-- /show

addPoly p1 p2 = if (length p1 >= length p2)
                then zipWith (+) p1 (p2 ++ repeat 0)
                else addPoly p2 p1
multiplyBy a p1 = map (a*) p1
multiplyByX p = 0:p
```

##We can negate a polynomial (coefficient-wise)

``` active haskell
negatePoly = map negate

main = print $ negatePoly [0,1]
```

##We can derive a polynomial
``` active haskell
derive [] = []
derive (_:ps) = zipWith (*) ps [1..]

main = do print $ derive [0,1]
          print $ derive [0,1,4,2]

```

#Instances

Here are three important instances: `Num`, `Show`, `Eq`.

``` active haskell
import Data.List
multPoly [] p2 = []
multPoly (p:p1) p2 = let pTimesP2 = multiplyBy p p2
                         xTimesP1Timesp2 = multiplyByX $ multPoly p1 p2
                     in addPoly pTimesP2 xTimesP1Timesp2    
addPoly p1 p2 = if (length p1 >= length p2)
                then zipWith (+) p1 (p2 ++ repeat 0)
                else addPoly p2 p1
multiplyBy a p1 = map (a*) p1
multiplyByX p = 0:p

negatePoly :: Num a => [a] -> [a]
negatePoly = map negate



toProper [] = []
toProper p = if (last p /= 0) -- this means there is nothing to do.
             then p 
             else toProper $ init p -- this means the last zero can be discarded.

-- show This allows us to instantiate typeclasses to which regular lists belong.
newtype Poly a = P [a] 
-- /show



-- show A polynomial is sort of a number. We enforce closure by properP, which only creates proper Ps.
--By using modules, we could only export properP, not P. This would simply get rid 
--of all those improper representations. However, some things are more efficient 
--with improper representations, so we do not want to lose that power. Depending 
--on the application, this constraint might be removed.
properP :: (Num a, Eq a) => [a] -> Poly a
properP = P . toProper

instance (Num a, Eq a) => Num (Poly a) where
    (P a) + (P b) = properP $ addPoly a b
    (P a) * (P b) = properP $ multPoly a b
    negate (P a) = properP $ negatePoly a
    abs = undefined
    signum = undefined
    fromInteger i = properP [fromIntegral i]
-- /show

-- show Shows polynomials how we write them mathematically.
-- It would be harder to parse.
showPoly [] = show 0
showPoly p =  let cOs = zip p [0..]
                  nonZeroCOs = filter (\(c,_) -> c /= 0) cOs
                  cShow c = if c == 1 
                            then "" 
                            else show c
                  nShow n = case n of 
                              0 -> ""
                              1 -> "x" 
                              m -> "x^" ++ show m
                  cnShow c n = if c == 1 && n == 0 
                               then show 1 
                               else intercalate " " $ filter (/="") [cShow c, nShow n]            
                  terms = map (\(c,n) -> cnShow c n) nonZeroCOs
              in intercalate " + " (reverse terms)    

instance (Show a, Eq a, Num a) => Show (Poly a) where
    show (P a) = showPoly $ toProper a
-- /show

-- show Eqs two polynomials
instance (Num a, Eq a) => Eq (Poly a) where
    (P a) == (P b) = toProper a == toProper b
-- /show

-- show Try it!
main = do print (P [1,3,4] - P [0,1,0,0])
          print $ P []
          print $ P [0,0]
          print (P [3,2,5] * P [4,1,1])
          print (P [3,2,5] * P [4,1,1] == properP [4,1,1] * P [3,2,5,0,0])
                    
-- /show

```
Exercise: 1) modify the instance of Show so that multiplication is denoted by `*` (as in `2*x + 1`). 2) (Bonus) modify the instance of Show so that things like `2 x^2 + -3` become `2 x^2 - 3`. 

#Polynomials as functions

``` haskell
-- show
listOfPowers x = map (\n -> x**n) [0..]

makeFunction p = \x -> sum $ zipWith (*) p (listOfPowers x)
-- /show
```

#Important Families of Polynomials

``` active haskell
import Data.List
multPoly [] p2 = []
multPoly (p:p1) p2 = let pTimesP2 = multiplyBy p p2
                         xTimesP1Timesp2 = multiplyByX $ multPoly p1 p2
                     in addPoly pTimesP2 xTimesP1Timesp2    
addPoly p1 p2 = if (length p1 >= length p2)
                then zipWith (+) p1 (p2 ++ repeat 0)
                else addPoly p2 p1
multiplyBy a p1 = map (a*) p1
multiplyByX p = 0:p

negatePoly :: Num a => [a] -> [a]
negatePoly = map negate



toProper [] = []
toProper p = if (last p /= 0) -- this means there is nothing to do.
             then p 
             else toProper $ init p -- this means the last zero can be discarded.

newtype Poly a = P [a] 
properP :: (Num a, Eq a) => [a] -> Poly a
properP = P . toProper

instance (Num a, Eq a) => Num (Poly a) where
    (P a) + (P b) = properP $ addPoly a b
    (P a) * (P b) = properP $ multPoly a b
    negate (P a) = properP $ negatePoly a
    abs = undefined
    signum = undefined
    fromInteger i = properP [fromIntegral i]
showPoly [] = show 0
showPoly p =  let cOs = zip p [0..]
                  nonZeroCOs = filter (\(c,_) -> c /= 0) cOs
                  cShow c = if c == 1 
                            then "" 
                            else show c
                  nShow n = case n of 
                              0 -> ""
                              1 -> "x" 
                              m -> "x^" ++ show m
                  cnShow c n = if c == 1 && n == 0 
                               then show 1 
                               else intercalate " " $ filter (/="") [cShow c, nShow n]            
                  terms = map (\(c,n) -> cnShow c n) nonZeroCOs
              in intercalate " + " (reverse terms)    

instance (Show a, Eq a, Num a) => Show (Poly a) where
    show (P a) = showPoly $ toProper a
instance (Num a, Eq a) => Eq (Poly a) where
    (P a) == (P b) = toProper a == toProper b
                    
-- show Legendre Polynomials
x = P [0,1]
coef a = properP [a]
coeff `mono` order = properP (replicate order 0 ++ [coeff]) 

--this is the zeroth legendre polynomial
leg0 = coef 1
--this is the 1st legendre poly.
leg1 = x 
--this is the relationship between the current legendre 
--polynomial and the two previous ones.
legN n legP legPP = coef (1/n) * (coef (2*n-1) * x * legP + coef (1-n) * legPP) 

--the infinite list of legendre polynomials
legs = leg0 : leg1 : zipWith3 legN [2..] (tail legs) legs
getLegendre n = legs !! n

derive (P []) = (P [])
derive (P (_:ps)) = P $ zipWith (*) ps [1..]
getLegendre' = derive . getLegendre
-- /show

--show Chebyshev Polynomials
ch0 = coef 1
ch1 = x 
chN chPrev chPrevPrev =   2 * x * chPrev - chPrevPrev 

chebys = ch0:ch1: zipWith chN (tail chebys) chebys
getCheby n = chebys !! n
-- /show

-- show Try it!
main = do putStrLn "Legendre polynomials:"
          putStrLn $ intercalate "\n" $ map show $ take 10 legs
          putStrLn "Associated Legendre polynomials:"
          putStrLn $ intercalate "\n" $ map show $  map getLegendre' [0..9]
          putStrLn "Chebyshev Polynomials:"
          putStrLn $ intercalate "\n" $ map show $ take 10 chebys
-- /show
```
Exercise: Define the set of all legendre polynomials using 1) a recursion, 2) an iteration. 3) (Bonus) Why is it a bad idea to do this recursively for n = 1000? 4) Is it a good idea to use an iteration for n = 1000 ?

#Conclusion
Now that we have the power of polynomials, there are many important applications to mention. See my tutorial on Numerical Methods for the actual applications.

We can do interpolation with polynomials. The idea is to construct a polynomial which interpolates a set of points.

We can find the roots of simple polynomials exactly:

``` haskell

realquadroots (P [c,b,a]) = let disc = b**2 - 4 * a * c
                            in if disc < 0 
                               then []
                               else if disc == 0
                                    then [-b/(2*a)]
                                    else [(-b + sqrt disc)/(2*a),(-b - sqrt disc)/(2*a)]
                                    
```

We can also use `makeFunction` to find the roots of simple polynomials approximately (with Newton's method, for instance).

You can also use the legendre polynomials in more complex techniques such as the Gaussian Quadrature.

#Bonus Round: Getting rid of Improper Representations.

Here is how you ensure the utter annilation of Improper Polynomials. This technique can be used for your own applications with a similar structure.

The idea is that we will make `P` dissapear, leaving only `properP`. You see, if all of our operations are closed, and there is no way to introduce Improper representations, then _it becomes impossible to get an Improper polynomial_.

First, we have to introduce explicitely a function we used without knowing it.

``` haskell

toList (P p) = p

```

Because we want to still be able to work directly on the list (this is necessary if we want to define additional functions).

So now, we put what we have done so far in a module, and only export the functions in the parantesis. 
The user of our module will work in another file (Main.hs).

``` active haskell
{-# START_FILE Poly.hs #-}
-- show
module Poly (toList, makeFunction, properP, Poly(), x, coef, mono, legs, getLegendre, getLegendre', derive, chebys, getCheby) where

--Note: the instances for Show, Num, and Eq are exported.

--There is actually more stuff here, but it is all a repeat of what we saw previously.
-- /show
import Data.List

toList (P p) = p

listOfPowers x = map (\n -> x**n) [0..]

makeFunction (P p) = \x -> sum $ zipWith (*) p (listOfPowers x)

multPoly [] p2 = []
multPoly (p:p1) p2 = let pTimesP2 = multiplyBy p p2
                         xTimesP1Timesp2 = multiplyByX $ multPoly p1 p2
                     in addPoly pTimesP2 xTimesP1Timesp2    
addPoly p1 p2 = if (length p1 >= length p2)
                then zipWith (+) p1 (p2 ++ repeat 0)
                else addPoly p2 p1
multiplyBy a p1 = map (a*) p1
multiplyByX p = 0:p

negatePoly :: Num a => [a] -> [a]
negatePoly = map negate



toProper [] = []
toProper p = if (last p /= 0) -- this means there is nothing to do.
             then p 
             else toProper $ init p -- this means the last zero can be discarded.

newtype Poly a = P [a] 
properP :: (Num a, Eq a) => [a] -> Poly a
properP = P . toProper

instance (Num a, Eq a) => Num (Poly a) where
    (P a) + (P b) = properP $ addPoly a b
    (P a) * (P b) = properP $ multPoly a b
    negate (P a) = properP $ negatePoly a
    abs = undefined
    signum = undefined
    fromInteger i = properP [fromIntegral i]
showPoly [] = show 0
showPoly p =  let cOs = zip p [0..]
                  nonZeroCOs = filter (\(c,_) -> c /= 0) cOs
                  cShow c = if c == 1 
                            then "" 
                            else show c
                  nShow n = case n of 
                              0 -> ""
                              1 -> "x" 
                              m -> "x^" ++ show m
                  cnShow c n = if c == 1 && n == 0 
                               then show 1 
                               else intercalate " " $ filter (/="") [cShow c, nShow n]            
                  terms = map (\(c,n) -> cnShow c n) nonZeroCOs
              in intercalate " + " (reverse terms)    

instance (Show a, Eq a, Num a) => Show (Poly a) where
    show (P a) = showPoly $ toProper a
instance (Num a, Eq a) => Eq (Poly a) where
    (P a) == (P b) = toProper a == toProper b

x = P [0,1]
coef a = properP [a]
coeff `mono` order = properP (replicate order 0 ++ [coeff]) 

--this is the zeroth legendre polynomial
leg0 = coef 1
--this is the 1st legendre poly.
leg1 = x 
--this is the relationship between the current legendre 
--polynomial and the two previous ones.
legN n legP legPP = coef (1/n) * (coef (2*n-1) * x * legP + coef (1-n) * legPP) 

--the infinite list of legendre polynomials
legs = leg0 : leg1 : zipWith3 legN [2..] (tail legs) legs
getLegendre n = legs !! n

derive (P []) = (P [])
derive (P (_:ps)) = P $ zipWith (*) ps [1..]
getLegendre' = derive . getLegendre

ch0 = coef 1
ch1 = x 
chN chPrev chPrevPrev =   2 * x * chPrev - chPrevPrev 

chebys = ch0:ch1: zipWith chN (tail chebys) chebys
getCheby n = chebys !! n

{-# START_FILE Main.hs #-}
import Poly
import Data.List

main = do putStrLn "Legendre polynomials:"
          putStrLn $ intercalate "\n" $ map show $ take 10 legs
          putStrLn "Testing the arithmetic operations"
          print (properP [1,3,4] - properP [0,1,0,0])
          print $ properP []
          print $ properP [0,0]
          print (properP [3,2,5] * properP [4,1,1])
          putStrLn "Testing the Equality test"
          print (properP [3,2,5] * properP [4,1,1] == properP [4,1,1] * properP [3,2,5,0,0])
          putStrLn "Testing derivation"
          print (derive (properP [1,0,2]))
          
```
Exercise: 1) write a test for `makeFunction`, 2) try to replace `properP` by `P` in Main.hs; what happens?

3) (Bonus \*) Try to implement Polynomials as a Vector (from <hoogle>Data.Vector.Unboxed</hoogle>) of coefficient instead of a list (do all this in Poly.hs), but you must keep Main.hs the way it is, and have all the tests give you the same thing as they give you now.

Have fun.


# Bonus Round 2: Integration

In the same way that polynomials can be derived exactly according to calculus, so too can they be integrated.

The integral of a polynomial is a polynomial, which is rather nice. (as I will discuss later, this kind of property is often very good if you want your code to be powerful and relatively small)

Of course, integration (indefinite integration) does not fully specify what the result should be. We know that the following property must hold:

``` haskell

derive (integrate p) == p

```

This only specifies integration up to a constant. In other words, 

``` haskell

integrate (derive p) == p + c

```

where `c` is a constant polynomial.


For simplicity, we use the convention that integration always gives a polynomial going through the origin. For instance,

``` haskell

integrate (P [1]) == P [0,1]
```

## Why not just do definite integration?

The ultimate goal is definite integration, but suppose we want to visualise the integral of some polynomial. In that case, it is more efficient (and more interesting) to first calculate the indefinite integral, and then evaluate it between various points plot the curve.

Exercise: Try to implement polynomial integration.

Solution: 

@@@

``` haskell

integrate (P p) = P (0 : zipWith (/) p (map fromIntegral [1..length p]))

```

@@@

# Solution to using vectors and optimising a bit.

As you can see below, programs using vectors are a little less descriptive of the meaning, and more descriptive of the procedures. However, after understanding the list version, you can use the vector version without difficulty. Notice also how I didn't convert the `show` function to a vector style. The `show` function doesn't need to be super-fast, and the tools to work with `String`s are using lists (because `String` is just `[Char]`). This is the kind of compromise in favor of convenience you should always be making.

A good rule of thumb is this: functions that take some `a` and makes a new `a` are potential bottlenecks, because some bigger function could call this smaller function a billion times in some program. However, a function that takes some `a` and returns something that only *you* will understand ... well, this will never be the bottleneck; you and your understanding will (you can't look at a billion strings representing a polynomial faster than your computer can print them).

@@@

``` haskell
module PolyV (toVec, makeFunction, properP, Poly(), x, coef, mono , legs, getLegendre, getLegendre', derive, integrate, chebys, getCheby)  where

import Data.List (intercalate)
import Numeric.Polynomial
import qualified Data.Vector.Generic as G
import           Data.Vector.Generic  (Vector)
import qualified Data.Vector.Unboxed as U


toVec :: Poly t -> U.Vector t
toVec (P p) = p
{-# INLINE toVec #-}

makeFunction :: (Num a, U.Unbox a) => Poly a -> (a -> a)
makeFunction (P p) = \x ->  evaluatePolynomial x p
{-# INLINE makeFunction #-}

multPoly :: (Eq a, Num a, U.Unbox a) => U.Vector a -> U.Vector a -> U.Vector a
multPoly   xs  myYs     = if U.null myYs
                          then U.empty
                          else let 
                            y = U.head myYs
                            ys = U.tail myYs
                            mul 0 bs = 0 `U.cons` bs
                            mul x bs = (x*y) `U.cons` addPoly (U.map (x*) ys) bs
                            in U.foldr mul U.empty xs
{-# INLINE multPoly #-}                           







addPoly :: (Num a, U.Unbox a) => U.Vector a -> U.Vector a -> U.Vector a
addPoly p1 p2 = let
    n1 = U.length p1 
    n2 = U.length p2
    in if n1 >= n2
       then let 
          (fittingPart, rest) = U.splitAt n2 p1
          in  (U.zipWith (+) fittingPart p2) U.++ rest
       else addPoly p2 p1
{-# INLINE addPoly #-}

multiplyBy :: (Num b, U.Unbox b) => b -> U.Vector b -> U.Vector b
multiplyBy a p1 = U.map (a*) p1
{-# INLINE multiplyBy #-}

multiplyByX :: (Num a, U.Unbox a) => U.Vector a -> U.Vector a
multiplyByX p = 0 `U.cons` p
{-# INLINE multiplyByX #-}


negatePoly ::(U.Unbox b0, Num b0) => U.Vector b0 -> U.Vector b0
negatePoly = U.map negate
{-# INLINE negatePoly #-}


toProper ::  (Eq a, Num a, U.Unbox a) => U.Vector a -> U.Vector a 
toProper p = if (U.null p || U.last p /= 0) 
             then     p -- this means there is nothing to do.        
             else     toProper $ U.init p -- this means the last zero can be discarded.
{-# INLINE toProper #-}

newtype Poly a = P (U.Vector a) 

properP :: (Eq a, Num a, U.Unbox a) => U.Vector a -> Poly a
properP = P . toProper
{-# INLINE properP #-}


showPoly mp = if U.null mp 
             then show 0
             else let
                p = U.toList mp
                cOs = zip p [0..]
                nonZeroCOs = filter (\(c,_) -> c /= 0) cOs
                cShow c = if c == 1 
                          then "" 
                          else show c
                nShow n = case n of 
                            0 -> ""
                            1 -> "x" 
                            m -> "x^" ++ show m
                cnShow c n = if c == 1 && n == 0 
                             then show 1 
                             else intercalate " " $ filter (/="") [cShow c, nShow n]            
                terms = map (\(c,n) -> cnShow c n) nonZeroCOs
                in intercalate " + " (reverse terms)    
              
instance (Num a, Eq a, U.Unbox a) => Num (Poly a) where
    (P a) + (P b) = properP $ addPoly a b
    (P a) * (P b) = properP $ multPoly a b
    negate (P a) = properP $ negatePoly a
    abs = undefined
    signum = undefined
    fromInteger i = properP $ U.singleton $ fromIntegral i


{-
instance (Show a, Eq a, Num a) => Show (Poly a) where
    show (P a) = showPoly $ toProper a
-}

instance (Num a, Eq a, U.Unbox a) => Eq (Poly a) where
    (P a) == (P b) = toProper a == toProper b


x :: (Num a0, U.Unbox a0) => Poly a0
x = P $  0 `U.cons` U.singleton 1

coef :: (Eq a, Num a, U.Unbox a) => a -> Poly a
coef a = properP $ U.singleton a

mono :: (Eq a, Num a, U.Unbox a) => a -> Int -> Poly a
coeff `mono` order = properP (U.replicate order 0 `U.snoc` coeff) 

--this is the zeroth legendre polynomial
leg0 :: (Eq a, Num a, U.Unbox a) => Poly a
leg0 = coef 1
--this is the 1st legendre poly.
leg1 :: (Num a, U.Unbox a) => Poly a
leg1 = x 
--this is the relationship between the current legendre 
--polynomial and the two previous ones.

legN n legP legPP = coef (1/n) * (coef (2*n-1) * x * legP + coef (1-n) * legPP) 

--the infinite list of legendre polynomials
legs :: (Eq a, Num a, U.Unbox a, Fractional a) => [Poly a]
legs = leg0 : leg1 : zipWith3 legN (map fromIntegral [2..]) (tail legs) legs

getLegendre :: (Eq a, Num a, U.Unbox a, Fractional a) => Int -> Poly a
getLegendre n = legs !! n

derivePoly p = if U.null p
               then p
               else U.zipWith (*) (U.tail p) (U.enumFromN 1 (U.length p - 1))

derive :: (Fractional a, U.Unbox a) => Poly a -> Poly a
derive = P . derivePoly . toVec

integratePoly p = 0 `U.cons` U.zipWith (/) p (U.enumFromN 1 (U.length p))

integrate :: (Fractional a, U.Unbox a) => Poly a -> Poly a
integrate = P . integratePoly . toVec

getLegendre' :: (Eq a, Num a, U.Unbox a, Fractional a) => Int -> Poly a
getLegendre' = derive . getLegendre

ch0 :: (Eq a, Num a, U.Unbox a) => Poly a
ch0 = coef 1

ch1 :: (Num a, U.Unbox a) => Poly a
ch1 = x 

chN chPrev chPrevPrev =   2 * x * chPrev - chPrevPrev 

chebys :: (Eq a, Num a, U.Unbox a) => [Poly a]
chebys = ch0:ch1: zipWith chN (tail chebys) chebys
getCheby n = chebys !! n


```
@@@
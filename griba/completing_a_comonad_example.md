# Completing a Comonad example

Hi! This is about completing an example from a Gabriel Gonzalez' splendid article [Comonads are objects](http://www.haskellforall.com/2013/02/you-could-have-invented-comonads.html), section "The command pattern" that I couldn't make it work.

With the Comonad *[extend](http://hackage.haskell.org/package/comonad/docs/Control-Comonad.html#v:extend)* method, you can turn a getter method `(w a -> b)` into an object transformer `(w a -> w b)`, and thus apply multiple transformations.

The object type in the article was `(Kelvin, Kelvin -> a)`, but using it like this I suspect that the Comonad instance taken by the compiler was the Tuple-2 one where the inner type is its second element one, and the example didn't compile.

Defining a *newtype* around it, with instances for Functor and Comonad ([Comonads are Functors](http://hackage.haskell.org/package/comonad/docs/Control-Comonad.html)), makes the example work.

Using the "stack" template "simple", paste the following on `Main.hs`

```haskell
{-# LANGUAGE PackageImports, GeneralizedNewtypeDeriving, FlexibleInstances #-}

module Main where

import Data.Function ((&))   -- (&): backwards application
import "HUnit" Test.HUnit

import "comonad" Control.Comonad (Comonad(..))

-- 

newtype Kelvin = Kelvin { getKelvin :: Double } deriving (Num, Fractional, Show)

newtype Celsius = Celsius { getCelsius :: Double }
    deriving (Num, Fractional, Show)


-- wrapping the (data, function) pair

newtype W a = W (Kelvin, Kelvin -> a)

instance Functor W where

    -- fmap :: (a -> b) -> w a -> w b
    
    fmap g (W (k, f)) = W (k, g . f)


instance Comonad W where

    -- the dual of monad's return
    extract (W (k, f)) = f k
    
    -- the dual of join
    duplicate (W (k, f)) = W (k ,\k' -> W (k', f))
    
    -- laws:
    -- extract . duplicate = id
    -- fmap extract . duplicate = id
    -- duplicate . duplicate = fmap duplicate . duplicate

---

kelvinToCelsius :: Kelvin -> Celsius
kelvinToCelsius (Kelvin t) = Celsius (t - 273.15)

initialThermostat :: W Celsius
initialThermostat = W (298.15, kelvinToCelsius)

up :: W a -> a
up (W (t, f)) = f (t + 1)

down :: W a -> a
down (W (t, f)) = f (t - 1)

toString :: W Celsius -> String
toString (W (t, f)) = show (getCelsius (f t)) ++ " Celsius"

mymethod :: W Celsius -> String
mymethod obj = obj 
                  & extend up 
                  & extend up 
                  & toString

test1 = TestCase $ assertEqual "for the initial obj.:" (initialThermostat & toString) "25.0 Celsius"
test2 = TestCase $ assertEqual "for the extended obj.:" (initialThermostat & mymethod) "27.0 Celsius"

tests = TestList [TestLabel "test1" test1, TestLabel "test2" test2]

main :: IO Counts
main = runTestTT tests

```
Then add the packages *comonad* and *HUnit* to the project .cabal file.

```bash
$ stack exec myproject
Cases: 2  Tried: 2  Errors: 0  Failures: 0

```
Using *ghci* we can show the types of applying the Comonad `extend` method to the getters:
```
$ stack ghci
GHCi, version 8.4.4: http://www.haskell.org/ghc/  :? for help
[1 of 1] Compiling Main ...  
Ok, one module loaded.
Loaded GHCi configuration from ...

*Main> :t up
up :: W a -> a

*Main> :t extend up
extend up :: W b -> W b

*Main> :t toString
toString :: W Celsius -> String

*Main> :t extend toString
extend toString :: W Celsius -> W String

*Main> 

```
Let's see how `extend up` transforms `W (k, f)` through its simplification:

```haskell
-- since `extend f = fmap f . duplicate`

fmap up $ duplicate $ W (k, f)

-- from the Comonad instance

fmap up $ W (k ,\k' -> W (k', f))

-- from the Functor instance

 W (k , up . (\k' -> W (k', f)))
 
 W (k , \k' -> up (W (k', f)))
 
 -- from the definition of `up`

 W (k , \k' -> f (k' +1))

```






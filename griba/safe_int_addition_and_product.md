# Int Overflow And Maybe result expressions on weird Int results

Bounded Int{N} types arithmetic, for certain values give results that don't match the decimal arithmetic we use for counting.

No **arithmetic Overflow** is thrown in haskell Int{N} sum, product and casting between Int{N} types.

Here is a solution, with a CPP macro for exception enabled derived newtypes that use it.


### Defining a specific exception type

```haskell
data IntOverflow = IntSumOverflow String | IntNegOverflow 
                   | IntProdOverflow String
                   | IntCastOverflow String | IntOutOfRange String
                   deriving (Show, Typeable)

instance Exception IntOverflow
```

### Int addition, checking the result sign

The [two's complement](https://en.wikipedia.org/wiki/Two's_complement) addition that overflows the maxBound value alters the result leftmost bit, the sign. Some examples: 

```haskell
$ ghci
GHCi, version 7.10.3: http://www.haskell.org/ghc/  :? for help
Prelude> :set -XNegativeLiterals
Prelude> import Data.Int

Prelude Data.Int> 127 + 1 :: Int8
-128
Prelude Data.Int> 127 + 127 :: Int8
-2
Prelude Data.Int> (-128) + (-1) :: Int8
127
Prelude Data.Int> (-128) + (-128) :: Int8
0
```

Different sign operands don't pose overflow problems, as it equates to a substraction.

Let's use the function [*signum*](http://hackage.haskell.org/package/base/docs/Prelude.html#v:signum) that returns {-1,0,1} upon sign and zero tests.

```haskell
-- show parenthesis on negative values
showPar :: (Num a, Eq a, Show a) => a -> String
showPar x = if signum x == (-1) then "(" ++ show x ++ ")" else show x

--------------------

intAddEx :: (Num a, Eq a, Show a) => a -> a -> a
intAddEx x y
  | signum x == signum y =
             if signum result == signum x
               then result
               else let msg = showPar x ++ " + " ++ showPar y
                    in throw (IntSumOverflow msg)
  | otherwise = result -- no risk of overflow when sign differs
  where result = x + y

intAddMay :: (Num a, Eq a) => a -> a -> Maybe a
intAddMay x y
  | signum x == signum y =
             if signum result == signum x
               then Just result
               else Nothing
  | otherwise = Just result -- no risk of overflow when sign differs
  where result = x + y
```

### Int negate, the case of the minimum integer value

```haskell
Prelude Data.Int> - (-128) :: Int8  -- minimum Int{N} negation
-128                                -- (-0x80::Int8) == 0x80 !!

```
Checking code:

```haskell
intNegateEx :: (Num a, Eq a) => a -> a
intNegateEx x = if x == -x then throw IntNegOverflow
                           else -x

intNegateMay:: (Num a, Eq a) => a -> Maybe a
intNegateMay x = if x == -x     -- this is the case for 0x80 :: Int8
                  then Nothing
                  else Just (-x)
```

### Int product, with 2^numberOfBits bounds

Since `2^x * 2^y = 2^(x+y)` all products where the sum of bits of the absolute values is less or equal than the maximum are safe.

```haskell
import Data.Bits (FiniteBits(..)) -- the FiniteBits class for finite bitSize integer types

-- we need to take into account the minimum integer value (sign bit followed by zeros) whose absolute value is bigger than the highest positive value

bits :: (Integral a, FiniteBits a) => a -> Int
bits v
  | v == -v = finiteBitSize v   -- the data type size
  | otherwise = finiteBitSize v - countLeadingZeros (abs v) -- positive bitMask length

intProdEx :: (Integral a, FiniteBits a, Show a) => a -> a -> a
intProdEx x y
  | abs x <= 1 || abs y <= 1 = x * y
  | bits x + bits y < finiteBitSize x = x * y   -- no prod overflow
  | otherwise = throw $ IntProdOverflow $ showPar x ++ " * " ++ showPar y

```
but this check rejects good products (e.g. `31 * 4 :: Int8`) that can be accepted through the following proposal:

### Int product, through unlimited precision promotion and result cast checking


```haskell
intProdEx :: (Integral a, Show a) => a -> a -> a
intProdEx x y
  | abs x <= 1 || abs y <= 1 = x * y
  | otherwise = let integerProd = (fromIntegral x :: Integer) * fromIntegral y
                    result = fromInteger integerProd -- cast to return type
                in if integerProd == fromIntegral result
                      then result
                   else let msg = showPar x ++ " * " ++ showPar y
                        in throw (IntProdOverflow msg)


intProdMay :: (Integral a) => a -> a -> Maybe a
intProdMay x y
  | abs x <= 1 || abs y <= 1 = Just (x * y)
  | otherwise = let integerProd = (fromIntegral x :: Integer) * fromIntegral y  
                    result = fromInteger integerProd -- cast to return type parameter
                in if integerProd == fromIntegral result
                      then Just result
                      else Nothing
                      
```

### Cast between Integral types

```haskell
intCastEx :: (Integral a, Show a, Integral b,
              Typeable a, Typeable b) => a -> b
intCastEx x = let result = fromIntegral x     -- cast to output type
              in if x == fromIntegral result  -- cast result to input type and check
                   then result                -- output
                   else let msg = showPar x 
                               ++ " :: from " ++ show (typeOf x)
                               ++ " to " ++ show (typeOf result)
                        in throw (IntCastOverflow msg)

intCastMay :: (Integral a, Integral b) => a -> Maybe b
intCastMay x = let result = fromIntegral x     -- cast to output type parameter
               in if x == fromIntegral result  -- cast result to input type and check
                    then Just result           -- output
                    else Nothing
```

### Out of range numeric literals

Actually, out of range literals give a compiler warning and a different value.

But using the *default* clause the warning doesn't show:

```haskell
Prelude> import Data.Int
Prelude Data.Int> 129::Int8
                           
<interactive>:3:1: Warning:                                                                          
    Literal 129 is out of the Int8 range -128..127
    
-127
Prelude Data.Int> default (Int8)
Prelude Data.Int> 129
-127  -- ??? cannot match out-of-range input ; no warning !!!
```

Defining a specific *fromInteger* for the derived types being defined, checking the converted value:

```haskell

intFromIntegerEx :: Integral a => Integer -> a
intFromIntegerEx integer = 
     let result = fromInteger integer
     in if integer == fromIntegral result
           then result
           else throw (IntOutOfRange $ showPar integer)

```

### Defining types that use the above

A *CPP macro* for newtype and instances definition.

CPP macros don't generate newline characters, so single line notation (curly brackets and semicolons) must be used:

```haskell
-----------------
-- Macro for newtype and instances definition
#define BLOC( IntEN, IntN) \
newtype IntEN = IntEN IntN deriving (Show, Eq, Ord, Enum, Bounded, Real, Integral) ; \
instance Num IntEN where {\
    IntEN x + IntEN y = IntEN (intAddEx x y) ; \
    IntEN x * IntEN y = IntEN (intProdEx x y) ; \
    abs (IntEN x) = IntEN (abs x) ; \
    signum (IntEN x) = IntEN (signum x) ; \
    fromInteger = IntEN . intFromIntegerEx ; \
    negate (IntEN x) = IntEN (intNegateEx x) ; \
         } 

-----------------

BLOC( IntE, Int)
BLOC( IntE8, Int8)
BLOC( IntE16, Int16)
BLOC( IntE32, Int32)
BLOC( IntE64, Int64)

-----------------
```

### Everything together now

``` active haskell
{-# LANGUAGE CPP, DeriveDataTypeable, GeneralizedNewtypeDeriving #-}
module IntEx (
  IntE(..), IntE8(..), IntE16(..), IntE32(..), IntE64(..),
  intAddEx, intProdEx, intCastEx,
  intAddMay, intProdMay, intCastMay,
  ) where

import Control.Exception (Exception, throw)
import Data.Typeable
import Data.Int (Int8, Int16, Int32, Int64)


data IntOverflow = IntSumOverflow String | IntNegOverflow 
                   | IntProdOverflow String
                   | IntCastOverflow String | IntOutOfRange String
                   deriving (Show, Typeable)

instance Exception IntOverflow

showPar :: (Num a, Eq a, Show a) => a -> String
showPar x = if signum x == (-1) then "(" ++ show x ++ ")" else show x

--------------------

intAddEx :: (Num a, Eq a, Show a) => a -> a -> a
intAddEx x y
  | signum x == signum y =
             if signum result == signum x
               then result
               else let msg = showPar x ++ " + " ++ showPar y
                    in throw (IntSumOverflow msg)
  | otherwise = result -- no risk of overflow when sign differs
  where result = x + y

intAddMay :: (Num a, Eq a) => a -> a -> Maybe a
intAddMay x y
  | signum x == signum y =
             if signum result == signum x
               then Just result
               else Nothing
  | otherwise = Just result -- no risk of overflow when sign differs
  where result = x + y

-----------------

intNegateEx :: (Num a, Eq a) => a -> a
intNegateEx x = if x == -x then throw IntNegOverflow
                           else -x

intNegateMay:: (Num a, Eq a) => a -> Maybe a
intNegateMay x = if x == -x     -- this is the case for 0x80 :: Int8
                  then Nothing
                  else Just (-x)

-----------------

intProdEx :: (Integral a, Bounded a, Show a) => a -> a -> a
intProdEx x y
  | abs x <= 1 || abs y <= 1 = x * y
  | otherwise = let integerProd = (fromIntegral x :: Integer) * fromIntegral y
                    result = fromInteger integerProd -- cast to return type
                in if integerProd == fromIntegral result
                      then result
                   else let msg = showPar x ++ " * " ++ showPar y
                        in throw (IntProdOverflow msg)

intProdMay :: (Integral a, Bounded a) => a -> a -> Maybe a
intProdMay x y
  | abs x <= 1 || abs y <= 1 = Just (x * y)
  | otherwise = let integerProd = (fromIntegral x :: Integer) * fromIntegral y  
                    result = fromInteger integerProd -- cast to return type parameter
                in if integerProd == fromIntegral result
                      then Just result
                      else Nothing

-----------------

intFromIntegerEx :: (Integral a) => Integer -> a
intFromIntegerEx integer =
     let result = fromInteger integer
     in if integer == fromIntegral result
           then result
           else throw (IntOutOfRange $ showPar integer)

-----------------
-- Macro for newtype and instances definition
#define BLOC( IntEN, IntN) \
newtype IntEN = IntEN IntN deriving (Show, Eq, Ord, Enum, Bounded, Real, Integral) ; \
instance Num IntEN where {\
    IntEN x + IntEN y = IntEN (intAddEx x y) ; \
    IntEN x * IntEN y = IntEN (intProdEx x y) ; \
    abs (IntEN x) = IntEN (abs x) ; \
    signum (IntEN x) = IntEN (signum x) ; \
    fromInteger = IntEN . intFromIntegerEx ; \
    negate (IntEN x) = IntEN (intNegateEx x) ; \
         }

-----------------

BLOC( IntE, Int)
BLOC( IntE8, Int8)
BLOC( IntE16, Int16)
BLOC( IntE32, Int32)
BLOC( IntE64, Int64)

-----------------

intCastEx :: (Integral a, Show a, Integral b,
              Typeable a, Typeable b) => a -> b
intCastEx x = let result = fromIntegral x     -- cast to output type
              in if x == fromIntegral result  -- cast result to input type and check
                   then result                -- output
                   else let msg = showPar x 
                               ++ " :: from " ++ show (typeOf x)
                               ++ " to " ++ show (typeOf result)
                        in throw (IntCastOverflow msg)

intCastMay :: (Integral a, Integral b) => a -> Maybe b
intCastMay x = let result = fromIntegral x     -- cast to output type parameter
               in if x == fromIntegral result  -- cast result to input type and check
                    then Just result           -- output
                    else Nothing

```

### Using the *default* clause with the newly defined types

Since the IntE{N} type has *fromInteger* from the *Num* typeclass instance, you can preset the literals type with the *default* clause

```bash
$ ghci
GHCi, version 7.10.3: http://www.haskell.org/ghc/  :? for help
Prelude> :load IntEx
[1 of 1] Compiling IntEx          ( IntEx.hs, interpreted )
Ok, modules loaded: IntEx.

*IntEx> import Data.Int
*IntEx Data.Int> default (Int8)

*IntEx Data.Int> 129
-127  -- ??? cannot match out-of-range input ; no warning !!!

*IntEx Data.Int> default (IntE8)

*IntEx Data.Int> 129
IntE8 *** Exception: IntOutOfRange "129"

*IntEx Data.Int> 4
IntE8 4

```


### Checking

```bash
$ ghci
GHCi, version 7.10.3: http://www.haskell.org/ghc/  :? for help
Prelude> :load IntEx
[1 of 1] Compiling IntEx          ( IntEx.hs, interpreted )
Ok, modules loaded: IntEx.

*IntEx> (minBound::IntE8, maxBound::IntE8)
(IntE8 (-128),IntE8 127)

*IntEx> (126::IntE8) +1
IntE8 127

*IntEx> (127::Int8) +1
-128  -- ??? not in decimal arithmetics !!!

*IntEx> (127::IntE8) +1
IntE8 *** Exception: IntSumOverflow "127 + 1"

*IntEx> :set -XNegativeLiterals

*IntEx> (-128) + (-1) :: IntE8
IntE8 *** Exception: IntSumOverflow "(-128) + (-1)"

*IntEx> (-128) - (-1) :: IntE8
IntE8 (-127)

*IntEx> (-128) + (-128) :: IntE8
IntE8 *** Exception: IntSumOverflow "(-128) + (-128)"
-----------

*IntEx> (127::Int8) *2
-2    -- ??? not in decimal arithmetics !!!

*IntEx> (127::IntE8) *2
IntE8 *** Exception: IntProdOverflow "127 * 2"
-----------

*IntEx> import Data.Int

*IntEx Data.Int> fromIntegral (128::Int16) :: Int8
-128  -- ??? out-of-range input cannot match result !!!

*IntEx Data.Int> intCastEx (127::Int16) :: Int8
127

*IntEx Data.Int> intCastEx (128::Int16) :: Int8
*** Exception: IntCastOverflow "128 :: from Int16 to Int8"

*IntEx> intCastEx (128::IntE16) :: IntE8
IntE8 *** Exception: IntOutOfRange "128"

```

### Reflexion

All this is nice, but *exceptions* that can throw everywhere are hard to debug.

It would be better to use the Maybe result operation versions, compose them as monadic expressions, and check the final result, throwing routine specific exceptions in case of Nothing (meaning Int{N} Overflow).


### Using the Maybe result ops. and extra newtypes

Adding the following code for IntM{N} newtypes.

```haskell
-----------------

class Num a => NumMaybe a where
  (+?),(-?),(*?):: a -> a -> Maybe a

#define BLOC2( IntMN, IntN) \
newtype IntMN = IntMN IntN deriving (Show, Eq, Ord, Enum, Bounded, Num, Real, Integral) ; \
instance NumMaybe IntMN where {\
    IntMN x +? IntMN y = fmap IntMN (intAddMay x y) ; \
    IntMN x -? IntMN y = fmap IntMN (intNegateMay y >>= \minusY -> intAddMay x minusY) ; \
    IntMN x *? IntMN y = fmap IntMN (intProdMay x y) ; \
         }

-----------------

BLOC2( IntM, Int)
BLOC2( IntM8, Int8)
BLOC2( IntM16, Int16)
BLOC2( IntM32, Int32)
BLOC2( IntM64, Int64)

```

Use:

```haskell
$ghci
GHCi, version 7.10.3: http://www.haskell.org/ghc/  :? for help
Prelude> :load prova4
[1 of 1] Compiling IntEx            ( prova4.hs, interpreted )
Ok, modules loaded: IntEx.

*IntEx> :{   -- multiline entry mode

*IntEx| let aRiskyCalc :: NumMaybe a => a -> a -> a -> Maybe a
*IntEx|     aRiskyCalc x y z = do
*IntEx|          r <- x +? y
*IntEx|          s <- r *? z
*IntEx|          (-100) -? s
*IntEx| :}
*IntEx> aRiskyCalc 2 4 2 :: Maybe IntM8
Just (IntM8 (-112))
*IntEx> aRiskyCalc 2 4 8 :: Maybe IntM8
Nothing         -- the calculation overflowed !!!
```
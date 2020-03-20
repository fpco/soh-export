# 1HaskellADay

This page contains solutions to [#1HaskellADay](https://twitter.com/1HaskellADay) puzzles. It doesn't contain all solutions and it certainly doesn't contain best solutions (whatever it might mean). Solutions to puzzles are ordered by date, descending, and are hidden by default.

## May 14th, 2014

[Puzzle](http://lpaste.net/revision/2870923760335060992)

```haskell
{- Find the previous element of a look-and-say sequence element if it exists
   Look and say is defined here: https://en.wikipedia.org/wiki/Look-and-say_sequence

   Example:

   >>> lookAndSayPrec [4,1,3,7]
   Just [1,1,1,1,7,7,7]

   >>> lookAndSayPrec []
   Nothing

-}
lookAndSayPrec :: [Int] -> Maybe [Int]
lookAndSayPrec = undefined
```

### Solution

@@@
```haskell active
module Main where

{- Find the previous element of a look-and-say sequence element if it exists
   Look and say is defined here: https://en.wikipedia.org/wiki/Look-and-say_sequence

   Example:

   >>> lookAndSayPrec [4,1,3,7]
   Just [1,1,1,1,7,7,7]

   >>> lookAndSayPrec []
   Nothing

-}

import Data.List

lookAndSayPrec :: [Int] -> Maybe [Int]
lookAndSayPrec xs
    | length xs < 2 || (odd $ length xs) = Nothing
    | otherwise = Just $ lasPrec xs

lasPrec (x1:x2:xs) = replicate x1 x2 ++ lasPrec xs
lasPrec _ = []

main = do
    print $ lookAndSayPrec [4, 1, 3, 7]
    print $ lookAndSayPrec [3, 2, 5, 1, 1, 8]
    print $ lookAndSayPrec []
    print $ lookAndSayPrec [1, 3, 6]
    print $ lookAndSayPrec [4]
```
@@@

## April 18th, 2014

[Puzzle](http://lpaste.net/revision/8517000115964084224)

```haskell
{- | combinations 
   Builds all the combinations of length n of the elements of the initial list.

   Examples:

   >>> combinations 2 [0,1]
   [[0,0],[0,1],[1,0],[1,1]]

   >>> combinations 3 ['a','b']
   ["aaa","aab","aba","abb","baa","bab","bba","bbb"]
   
   >>> combinations (-2) ['a'..'z']
   [""]
-}
combinations :: Int -> [a] -> [[a]]
combinations = undefined
```

### Solution

For point-free solution, I found tremendous help [here](http://stackoverflow.com/a/13426526/97431).

@@@
```haskell active
module Main where

{- | combinations
   Builds all the combinations of length n of the elements of the initial list.

   Examples:

   >>> combinations 2 [0,1]
   [[0,0],[0,1],[1,0],[1,1]]

   >>> combinations 3 ['a','b']
   ["aaa","aab","aba","abb","baa","bab","bba","bbb"]

   >>> combinations (-2) ['a'..'z']
   [""]
-}

combinations :: Int -> [a] -> [[a]]
combinations len elems = sequence $ replicate len elems

combinationsPF :: Int -> [a] -> [[a]]
combinationsPF = (sequence .) . replicate

main = do
    print $ combinations 2 [0, 1] -- [[0,0],[0,1],[1,0],[1,1]]
    print $ combinations 3 ['a', 'b'] -- ["aaa","aab","aba","abb","baa","bab","bba","bbb"]
    print $ combinations (-2) ['a' .. 'z'] -- [""]
    --
    print $ combinationsPF 2 [0, 1] -- [[0,0],[0,1],[1,0],[1,1]]
    print $ combinationsPF 3 ['a', 'b'] -- ["aaa","aab","aba","abb","baa","bab","bba","bbb"]
    print $ combinationsPF (-2) ['a' .. 'z'] -- [""]
```
@@@

## April 15th, 2014

[Puzzle](http://lpaste.net/revision/5805301582249590784)

```haskell
{- | lowestFreeInt
   Find the lowest non-negative integer not in the list

   (Thanks to @clementd for this one)

   Example:

   >>> lowestFreeInt [0..10]
   11

   >>> lowestFreeInt [1..10]
   0

   >>> lowestFreeInt $ [0..9] ++ [2000,1999..11]
   10
-}
lowestFreeInt :: [Int] -> Int
lowestFreeInt = undefined
```

### Solution

@@@
```haskell active
module Main where

{- | lowestFreeInt
   Find the lowest non-negative integer not in the list

   (Thanks to @clementd for this one)

   Example:

   >>> lowestFreeInt [0..10]
   11

   >>> lowestFreeInt [1..10]
   0

   >>> lowestFreeInt $ [0..9] ++ [2000,1999..11]
   10
-}
import Data.List ((\\))

lowestFreeInt :: [Int] -> Int
lowestFreeInt = head .  (\\) [0..]

main = do
    print $ lowestFreeInt [0..10] -- 11
    print $ lowestFreeInt [1..10] -- 0
    print $ lowestFreeInt $ [0..9] ++ [2000,1999..11] -- 10
```
@@@

## April 8th, 2014

[Puzzle](http://lpaste.net/revision/3166527577426755584)

```haskell
module HAD.Y2014.M04.D08.Exercise where

{- | replicateF
   replicate and chain an endofunction

   Examples:

   prop> x + 10 == replicateF 10 (+1) x

   prop> 10 * x == replicateF 10 (+x) 0

   prop> replicate 10 x == replicateF 10 (x:) []
-}
replicateF :: Int -> (a -> a) -> a -> a
replicateF = undefined
```

### Solution

@@@
```haskell active
module Main where

{- | replicateF
   replicate and chain an endofunction

   Examples:

   prop> x + 10 == replicateF 10 (+1) x

   prop> 10 * x == replicateF 10 (+x) 0

   prop> replicate 10 x == replicateF 10 (x:) []
-}

-- common sense solution
replicateF :: Int -> (a -> a) -> a -> a
replicateF 0 f zero = zero
replicateF times f zero = replicateF (times - 1) f (f zero)

-- with fold
replicateF' :: Int -> (a -> a) -> a -> a
replicateF' times f zero = foldr (\fn elem -> fn elem) zero (replicate times f)

-- ... which is pretty much
replicateA :: Int -> (a -> a) -> a -> a
replicateA times f zero = foldr ($) zero (replicate times f)

main = do
    print $ replicateF 10 (+1) 0 -- 10
    print $ replicateF 10 (+ 3) 0 -- 30
    print $ replicateF 10 (1:) [] -- [1,1,1,1,1,1,1,1,1,1]
    print $ replicateF' 10 (+1) 0 -- 10
    print $ replicateF' 10 (+ 3) 0 -- 30
    print $ replicateF' 10 (1:) [] -- [1,1,1,1,1,1,1,1,1,1]
    print $ replicateA 10 (+1) 0 -- 10
    print $ replicateA 10 (+ 3) 0 -- 30
    print $ replicateA 10 (1:) [] -- [1,1,1,1,1,1,1,1,1,1]

```
@@@

## April 4th, 2014

[Puzzle](http://lpaste.net/revision/5976894132839251968)

```haskell
module HAD.Y2014.M04.D04.Exercise where

-- | countFigures count the different figures that composes a number
--
-- Examples:
--
-- >>> countFigures 1
-- 1
-- >>> countFigures 1000000
-- 2
-- >>> countFigures 123
-- 3
-- >>> countFigures (-12)
-- 2
-- >>> countFigures 1234567890
-- 10
-- >>> countFigures 00001
-- 1
-- >>> countFigures 0
-- 1
--
countFigures :: Integral a => a -> Int
countFigures = undefined
```

### Solution

@@@
```haskell active
module Main where

-- | countFigures count the different figures that composes a number
--
-- Examples:
--
-- >>> countFigures 1
-- 1
-- >>> countFigures 1000000
-- 2
-- >>> countFigures 123
-- 3
-- >>> countFigures (-12)
-- 2
-- >>> countFigures 1234567890
-- 10
-- >>> countFigures 00001
-- 1
-- >>> countFigures 0
-- 1
--

import Data.List

countFigures :: Integral a => a -> Int
countFigures = length . group . sort . show . abs . fromIntegral

main = do
    print $ countFigures 1
    print $ countFigures 1000000
    print $ countFigures 123
    print $ countFigures (-12)
    print $ countFigures 1234567890
    print $ countFigures 0001
    print $ countFigures 0
```
@@@

## April 3rd, 2014

[Puzzle](http://lpaste.net/revision/1109818273681113088)

```haskell
module HAD.Y2014.M04.D03.Exercise where

-- | foo
-- Types. Powerful enough to get it right.
--
foo :: (a ->  b) -> [a] -> [(a,b)]
foo = undefined
```

### Solution

@@@
```haskell active
module Main where

-- | foo
-- Types. Powerful enough to get it right.
--

foo :: (a ->  b) -> [a] -> [(a, b)]
foo f list = zip list $ map f list
```
@@@

## April 2nd, 2014

[Puzzle](http://lpaste.net/revision/2754796237017317376)

```haskell
module HAD.Y2014.M04.D02.Exercise where

-- | update update the nth element of a list
-- if the index is not a valid index, it leaves the list unchanged
--
-- Examples
--
-- >>> update (-2) 10 [0..4]
-- [0,1,2,3,4]
--
-- >>> update 6 10 [0..4]
-- [0,1,2,3,4]
--
-- >>> update 2 10 [0..4]
-- [0,1,10,3,4]
--
update :: Int -> a -> [a] -> [a]
update = undefined
```

### Solution

@@@
```haskell active
module Main where

-- | update update the nth element of a list
-- if the index is not a valid index, it leaves the list unchanged
--
-- Examples
--
-- >>> update (-2) 10 [0..4]
-- [0,1,2,3,4]
--
-- >>> update 6 10 [0..4]
-- [0,1,2,3,4]
--
-- >>> update 2 10 [0..4]
-- [0,1,10,3,4]
--
update :: Int -> a -> [a] -> [a]
update pos elem list
    | pos >= len || pos < 0 = list
    | otherwise             = take pos list ++ [elem] ++ drop (pos + 1) list
    where len = length list

main = do
    print $ update (-2) 10 [0..4]
    print $ update 6 10 [0..4]
    print $ update 2 10 [0..4]
    print $ update 4 10 [0..4]
    print $ update 0 10 [0..4]
```
@@@

## March 31st, 2014

[Puzzle](http://lpaste.net/revision/8924126783280775168)

```haskell
module HAD.Y2014.M02.D31.Exercise where

-- $setup
-- >>> import Control.Applicative

-- | emptyIndices List the indices of a list of maybes that contains Nothing
--
-- prop> (all (isNothing) .) . map . (!!) <*> emptyIndices $ xs
--
emptyIndices :: [Maybe a] -> [Int]
emptyIndices = undefined
```

### Solution

@@@
```haskell active
module Main where

-- $setup
-- >>> import Control.Applicative

-- | emptyIndices List the indices of a list of maybes that contains Nothing
--
-- prop> (all (isNothing) .) . map . (!!) <*> emptyIndices $ xs
--

import Data.Maybe

emptyIndices :: [Maybe a] -> [Int]
emptyIndices = map fst . filter (isNothing . snd) . zip [0..]

main = do
    print $ emptyIndices []
    print $ emptyIndices [Just 1, Just 2, Just 3]
    print $ emptyIndices [Just 1, Nothing, Just 2, Nothing, Nothing]
```
@@@

## March 24th, 2014

[Puzzle](http://lpaste.net/7140908906611474432)

```haskell
module HAD.Y2014.M03.D24.Exercise where

-- | squareList builds a list of x lists of size x from a given list of elements
-- If there aren't enough elements, fill the square with the second parameter
-- Examples:
--
-- >>> squareList 2  0 [0..]
-- [[0,1],[2,3]]
--
-- >>> squareList 2 0 [1]
-- [[1,0],[0,0]]
--
-- >>> squareList 3 () $ repeat ()
-- [[(),(),()],[(),(),()],[(),(),()]]
--
squareList :: Int -> a -> [a] -> [[a]] 
squareList = undefined
```

### Solution

@@@
```haskell active
module Main where

-- | squareList builds a list of x lists of size x from a given list of elements
-- If there aren't enough elements, fill the square with the second parameter
-- Examples:
--
-- >>> squareList 2  0 [0..]
-- [[0,1],[2,3]]
--
-- >>> squareList 2 0 [1]
-- [[1,0],[0,0]]
--
-- >>> squareList 3 () $ repeat ()
-- [[(),(),()],[(),(),()],[(),(),()]]
--
squareList :: Int -> a -> [a] -> [[a]] 
squareList size spare list = take size $ squareList' size (list ++ repeat spare)

squareList' :: Int -> [a] -> [[a]]
squareList' 0 _ = []
squareList' size list = head:tail where
    head = head'
    tail = squareList' size tail'
    (head', tail') = splitAt size list

main = do
    print $ squareList 2 0 [0..]
    print $ squareList 2 0 [1]
    print $ squareList 3 () $ repeat ()
    print $ squareList 4 0 []
```
@@@

## March 19th, 2014

[Puzzle](http://codepad.org/QollsjCw)

```haskell
-- $setup
-- >>> import Control.Applicative ((<*>))
-- >>> import Data.List (isInfixOf)
-- >>> import Test.QuickCheck

-- Level: Easy
-- Pointfree: yes


-- | mostRepeatedElem
-- Returns the element with the longest (consecutive) repetition and the
-- repetition number
-- If there are tie, the last most repeated element is returned
-- It returns error on empty string
-- 
-- Examples:
--
-- >>> mostRepeatedElem "hello world!"
-- ('l',2)
--
-- >>> mostRepeatedElem [1,1,2,2]
-- (2,2)
--
-- prop> (flip isInfixOf <*> uncurry (flip replicate) . mostRepeatedElem) . getNonEmpty

mostRepeatedElem :: Eq a => [a] -> (a,Int)
mostRepeatedElem = undefined
```

### Solution

@@@
```haskell active
module Main where

-- $setup
-- >>> import Control.Applicative ((<*>))
-- >>> import Data.List (isInfixOf)
-- >>> import Test.QuickCheck

-- Level: Easy
-- Pointfree: yes


-- | mostRepeatedElem
-- Returns the element with the longest (consecutive) repetition and the
-- repetition number
-- If there are tie, the last most repeated element is returned
-- It returns error on empty string
--
-- Examples:
--
-- >>> mostRepeatedElem "hello world!"
-- ('l',2)
--
-- >>> mostRepeatedElem [1,1,2,2]
-- (2,2)
--
-- prop> (flip isInfixOf <*> uncurry (flip replicate) . mostRepeatedElem) . getNonEmpty

import Data.List
import Control.Applicative

mostRepeatedCmp :: Eq a => (a, Int) -> (a, Int) -> Ordering
mostRepeatedCmp (_, fc) (_, sc) = fc `compare` sc

mostRepeatedElem :: Eq a => [a] -> (a, Int)
mostRepeatedElem ls = maximumBy mostRepeatedCmp $ zip heads lengths
    where grouped      = group ls
          heads        = map head grouped
          lengths      = map length grouped

main = do
    print $ mostRepeatedElem "hello world"
    print $ mostRepeatedElem [1, 1, 2, 2]
    print $ mostRepeatedElem [1,1,1,3,3,3,6,6,6,9,9,9,9,9,9,1,1]
    print $ mostRepeatedElem [1,1,1,3,3,3,6,6,6,9,9,9,9,9,9,1,1,1,1,1,1,1,1]
```
@@@

## March 17th, 2014

[Puzzle](http://codepad.org/4ZT5ZCwH)

```haskell
-- | maximumList replace each element in a list by its maximum so far
-- 
-- Examples:
--
-- >>> maximumList [1..4]
-- [1,2,3,4]
--
-- >>> maximumList [10,9..7]
-- [10,10,10,10]
--
-- prop> and . (zipWith (<=) <*> tail) . maximumList

-- maximumList :: Find a general signature
maximumList = undefined
```

### Solution

A common sense solution:

@@@
```haskell active
module Main where

-- | maximumList replace each element in a list by its maximum so far
--
-- Examples:
--
-- >>> maximumList [1..4]
-- [1,2,3,4]
--
-- >>> maximumList [10,9..7]
-- [10,10,10,10]
--
-- prop> and . (zipWith (<=) <*> tail) . maximumList

maximumList :: Ord a => [a] -> [a]
maximumList [] = []
maximumList xs = maximumList' (head xs) xs

maximumList' :: Ord a => a -> [a] -> [a]
maximumList' _ [] = []
maximumList' max (x:xs)
  | max > x   = max:maximumList' max xs
  | otherwise = x:maximumList' x xs

main = do
    print $ take 10 $ maximumList [100, 99..]
    print $ maximumList [10, 9.. 7]
    print $ maximumList [1..4]
```
@@@

## March 13th, 2014

[Puzzle](http://codepad.org/gXEcCgBI)

```haskell
-- | pairToList Transform a pair of same type elements in a list of two
-- elements.
--
-- Of course, the major challenge is to find a point free function
-- (without lambda). And, if you want more fun, do it without (++).
--
-- prop> replicate 2 (x :: Int) == pairToList (x,x)
--
-- prop> (\(f,s) -> [f,s]) x == pairToList x
--
pairToList :: (a,a) -> [a]
pairToList = undefined
```

### Solution

Couldn't really come up with a point-free solution, although it seemed like it won't be hard.

@@@
```haskell active
module Main where

-- | pairToList Transform a pair of same type elements in a list of two
-- elements.
--
-- Of course, the major challenge is to find a point free function
-- (without lambda). And, if you want more fun, do it without (++).
--
-- additional challenge: don't use flip or reverse
--
-- prop> replicate 2 (x :: Int) == pairToList (x,x)
--
-- prop> (\(f,s) -> [f,s]) x == pairToList x
--

-- simple solution
pairToList :: (a,a) -> [a]
pairToList (f, s) = [f,s]

-- second
pairToList2 :: (a,a) -> [a]
pairToList2 p = fst p : [snd p]

-- third
pairToList3 :: (a,a) -> [a]
pairToList3 arg = map ($ arg) [fst, snd]

-- fourth
pairToList4 :: (a,a) -> [a]
pairToList4 arg = flip map [fst, snd] ($ arg)

main = do
    print $ pairToList (1, 2)
    print $ pairToList2 (1, 2)
    print $ pairToList3 (1, 2)
    print $ pairToList4 (1, 2)
```
@@@

## March 12th, 2014

[Puzzle](http://codepad.org/PL9JcBz5)

```haskell
-- | localMax Given an entry list, it outputs the its list of local maxima.
-- A Local maximum is a an element greater than its predecessor and than its
-- successor.
--
-- No point-free today, at least for my solution.
--
-- Examples:
--
-- >>> localMax [0 .. 1000]
-- []
--
-- >>> localMax [1000 .. 0]
-- []
--
-- >>> localMax [2,2,1,5,4]
-- [5]
--
-- >>> take 4 . localMax $ [0..] >>= (\y -> [y,y+2])
-- [2,3,4,5]
--
localMax :: Ord a => [a] -> [a]
localMax = undefined

main = print $ localMax [2,2,1,5,4] -- [5]
```

### Solution

@@@
```haskell active
module Main where

-- | localMax Given an entry list, it outputs the its list of local maxima.
-- A Local maximum is a an element greater than its predecessor and than its
-- successor.
--
-- No point-free today, at least for my solution.
--
-- Examples:
--
-- >>> localMax [0 .. 1000]
-- []
--
-- >>> localMax [1000 .. 0]
-- []
--
-- >>> localMax [2,2,1,5,4]
-- [5]
--
-- >>> take 4 . localMax $ [0..] >>= (\y -> [y,y+2])
-- [2,3,4,5]
--
localMax :: Ord a => [a] -> [a]
-- | take lists of length 3, filter those by min criteria and take the middle element
localMax = map (!! 1) . filter hasMin . threes

threes :: [a] -> [[a]]
threes (x:y:z:rest) = [x,y,z] : threes (y:z:rest)
threes _ = []

hasMin :: Ord a => [a] -> Bool
hasMin [x,y,z] = y > x && y > z
hasMin _ = False

main = do
  print $ localMax [2,2,1,5,4] -- [5]
  print $ take 4 . localMax $ [0..] >>= (\y -> [y,y+2])
```
@@@

## March 11th, 2014

[Puzzle](http://codepad.org/l3Eyjdzs)

```haskell
-- | lcAlphabetFrom
-- Display the alaphabet in lower cas, starting from the letter given in
-- parameter.
-- If the parameter is not a lowercase letter, displays the alphabet from 'a'
--
-- Point-free is quite easy
--
-- Examples:
--
-- >>> lcAlphabetFrom 'a'
-- "abcdefghijklmnopqrstuvwxyz"
--
-- >>> lcAlphabetFrom 'e'
-- "efghijklmnopqrstuvwxyzabcd"
--
-- >>> lcAlphabetFrom '`'
-- "abcdefghijklmnopqrstuvwxyz"
--
-- >>> lcAlphabetFrom '{'
-- "abcdefghijklmnopqrstuvwxyz"

lcAlphabetFrom :: Char -> String
lcAlphabetFrom = undefined

main = do
  print $ lcAlphabetFrom 'a' -- "abcdefghijklmnopqrstuvwxyz"
  print $ lcAlphabetFrom 'e' -- "efghijklmnopqrstuvwxyzabcd"
  print $ lcAlphabetFrom '`' -- "abcdefghijklmnopqrstuvwxyz"
  print $ lcAlphabetFrom '{' -- "abcdefghijklmnopqrstuvwxyz"
```

### Solution

@@@
```haskell active
module Main where

import Data.Char

-- | lcAlphabetFrom
-- Display the alaphabet in lower cas, starting from the letter given in
-- parameter.
-- If the parameter is not a lowercase letter, displays the alphabet from 'a'
--
-- Point-free is quite easy
--
-- Examples:
--
-- >>> lcAlphabetFrom 'a'
-- "abcdefghijklmnopqrstuvwxyz"
--
-- >>> lcAlphabetFrom 'e'
-- "efghijklmnopqrstuvwxyzabcd"
--
-- >>> lcAlphabetFrom '`'
-- "abcdefghijklmnopqrstuvwxyz"
--
-- >>> lcAlphabetFrom '{'
-- "abcdefghijklmnopqrstuvwxyz"

lcAlphabetFrom :: Char -> String
lcAlphabetFrom c
  | isAsciiLower c = prettyAlphabet c
  | otherwise = ['a'..'z']

prettyAlphabet :: Char -> String
prettyAlphabet from = head ++ tail
  where head = [from .. 'z']
        tail = ['a' .. pred $ from]

main = do
  print $ lcAlphabetFrom 'a' -- "abcdefghijklmnopqrstuvwxyz"
  print $ lcAlphabetFrom 'e' -- "efghijklmnopqrstuvwxyzabcd"
  print $ lcAlphabetFrom '`' -- "abcdefghijklmnopqrstuvwxyz"
  print $ lcAlphabetFrom '{' -- "abcdefghijklmnopqrstuvwxyz"
```
@@@

## March 10th, 2014

[Puzzle](http://codepad.org/XRnantkj)

```haskell
-- $setup
-- >>> import Test.QuickCheck
-- >>> import Control.Applicative

-- | maybeReadPositiveInt Try to parse a positive Int
-- Can be done point-free (and it's probably funnier this way).
--
-- Examples:
--
-- prop> (==) <$> Just <*> maybeReadPositiveInt . show $ getNonNegative x
--
-- prop> Nothing == (maybeReadPositiveInt . show . negate . getPositive $ x)
--
-- >>> maybeReadPositiveInt "foo"
-- Nothing
--
-- >>> maybeReadPositiveInt "12 "
-- Nothing


maybeReadPositiveInt :: String -> Maybe Int
maybeReadPositiveInt = undefined

main = do
  print $ maybeReadPositiveInt "12" -- Just 12
  print $ maybeReadPositiveInt "-12" -- Nothing
  print $ maybeReadPositiveInt "12 a" -- Nothing
  print $ maybeReadPositiveInt "12 " -- Nothing
```

### Solution

@@@
```haskell active
module Main where

import Data.Char

-- $setup
-- >>> import Test.QuickCheck
-- >>> import Control.Applicative

-- | maybeReadPositiveInt Try to parse a positive Int
-- Can be done point-free (and it's probably funnier this way).
--
-- Examples:
--
-- prop> (==) <$> Just <*> maybeReadPositiveInt . show $ getNonNegative x
--
-- prop> Nothing == (maybeReadPositiveInt . show . negate . getPositive $ x)
--
-- >>> maybeReadPositiveInt "foo"
-- Nothing
--
-- >>> maybeReadPositiveInt "12 "
-- Nothing


maybeReadPositiveInt :: String -> Maybe Int
maybeReadPositiveInt str
  | allDigits str = return $ tryInt str
  | otherwise = Nothing

allDigits :: String -> Bool
allDigits "" = False
allDigits str = foldr (&&) True $ map isDigit str

tryInt str = read str :: Int

main = do
  print $ maybeReadPositiveInt "12" -- Just 12
  print $ maybeReadPositiveInt "-12" -- Nothing
  print $ maybeReadPositiveInt "12 a" -- Nothing
  print $ maybeReadPositiveInt "12 " -- Nothing
```
@@@

## March 7th, 2014

[Puzzle](http://codepad.org/3A1QmmdN)

```haskell
-- | trueIndexes produce an infinite list where only the index given in the list

-- in parameter are true.
-- The parameter list is supposed to be sorted set of positive numbers
--
-- Point-free: Probably hard to find!
-- Level: HARD
--
-- Examples:
-- >>> take 2 $ trueIndexes [1]
-- [False,True]
--
-- >>> take 6 $ trueIndexes [0,2,4]
-- [True,False,True,False,True,False]
--
-- >>> take 3 $ trueIndexes []
-- [False, False, False]
--
trueIndexes :: [Int] -> [Bool]
trueIndexes = undefined

main = print $ take 6 $ trueIndexes [0,2,4]
```

### Solution

@@@
```haskell active
module Main where
-- | trueIndexes produce an infinite list where only the index given in the list

-- in parameter are true.
-- The parameter list is supposed to be sorted set of positive numbers
--
-- Point-free: Probably hard to find!
-- Level: HARD
--
-- Examples:
-- >>> take 2 $ trueIndexes [1]
-- [False,True]
--
-- >>> take 6 $ trueIndexes [0,2,4]
-- [True,False,True,False,True,False]
--
-- >>> take 3 $ trueIndexes []
-- [False, False, False]
--
trueIndexes :: [Int] -> [Bool]
trueIndexes [] = repeat False
trueIndexes (x:xs)
  | xs == []  = head ++ [True] ++ repeat False
  | otherwise = head ++ [True] ++ trueIndexes newInd
  where head   = replicate x False
        newInd = map (subtract offset) xs
        offset = x + 1
-- trueIndexes [x]    = replicate x False ++ [True] ++ repeat False
-- trueIndexes (x:xs) = replicate x False ++ [True] ++ trueIndexes (map (subtract (x + 1)) xs)

main = do
  print $ take 6 $ trueIndexes []
  print $ take 6 $ trueIndexes [2]
  print $ take 2 $ trueIndexes [1]
  print $ take 6 $ trueIndexes [0, 2, 4]
  print $ take 20 $ trueIndexes [0, 3 ..]
```
@@@
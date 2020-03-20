# Euler Project #43

Auguably simpler solution.

``` active haskell
module Main where
import Data.List
import Data.Char

permute :: [Char] -> [[Char]]
permute "" = [""]
permute s  = [(x:xs)| x <- s, xs <- permute (delete x s)]

f :: Int -> Integer -> String -> Bool
f i j n = mod (read $ fst $ splitAt 3 $ snd $ splitAt i n) j == 0

f1 = f 1 2
f2 = f 2 3
f3 = f 3 5
f4 = f 4 7
f5 = f 5 11
f6 = f 6 13
f7 = f 7 17

fp :: String -> Bool
fp n = f1 n && f2 n && f3 n && f4 n && f5 n && f6 n && f7 n

sol :: [Integer]
sol = [read n | n <- permute ['0'..'9'], fp n]
-- [1406357289,1430952867,..]

main = print $ sum sol
```
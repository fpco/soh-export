# Programming Contests Skeleton

Most programming contests have input as below.
The first line: number of lines that follow and other parameters.
The second line upto the last line: data of type string or integer.

You can check the IO to work:
``` active haskell
module Main where
import Control.Applicative
import Data.List

main :: IO ()
main = do
  ks <- map read . words <$> getLine
  xs <- words <$> getLine
  print (ks :: [Int], xs :: [String])
```

Typically you better separate the solver out of the IO.
"interact" is also handy:

``` active haskell
main :: IO ()
main = interact (solve . map words . lines)

solve :: [[String]] -> String
solve s = show (l0 :: [Int]) ++ show l1
    where
    l0 = map read $ (!!0) s
    l1 = (!!1) s
```
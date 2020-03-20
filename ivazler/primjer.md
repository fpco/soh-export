# Primjer

``` active haskell
{-# START_FILE Main.hs #-}
module Main where

-- show Quick sort algoritam
qsort:: Ord a=>[a]->[a]
qsort []=[]
qsort (x:xs)=qsort manji ++ [x] ++ qsort veci
             where
               manji=[a|a<-xs,a<=x]
               veci=[b|b<-xs,b>x]
-- /show

main :: IO ()

-- show poziv
main=print (qsort [4,7,2,3,1,3])
-- /show
```
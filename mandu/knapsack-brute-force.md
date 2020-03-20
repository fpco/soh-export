# Knapsack - Brute Force

## Problem definition

![knapsack](http://upload.wikimedia.org/wikipedia/commons/f/fd/Knapsack.svg "knapsack")

_Given a set of items, each with a weight and a value, determine the number of each item to include in a collection so that the total weight is less than or equal to a given limit and the total value is as large as possible._

Reference [Knapsack Problem (wiki)](http://en.wikipedia.org/wiki/Knapsack_problem).

## Data Structures
Input is a list of tuples, `[(value, weight)]`, that represents items. From items list, all possible combinations are created. One possible combination is called an instance of the problem. Each raw instance is parsed into 3-tuple, where the first element is the sum of values, the second is the sum of weights and the third element is the raw list of items. `[(value, weight)] -> (total_value, total_weight, [(value, weight)])`

## Combinations
Function <hoogle>subsequences</hoogle> enumerates all n-combinations of n objects. This is exactly what is needed for brute force algorithm in knapsack problem.


``` active haskell 
-- show Example of subsequences
import Data.List (subsequences)
combinations = subsequences [1,2,3]
-- /show
main = print combinations

```

## Solution
     
**Note**: You can fiddle `Main.hs`, for example change `n` (problem size),     `capacity` and `seed` variables and run code.

   
``` haskell
{-# START_FILE Knapsack.hs #-}
module Knapsack (knapsack) where

import Data.List (subsequences, filter, maximumBy)
import Data.Ord (comparing)

-- Brute Force Algorithm for Knapsack Problem
-- Complexity: O(2^n) (subsequences function)

-- show Brute Force Algorithm
-- Create a list of 3-tuples from item subsequences.
combos = map parse . subsequences

-- Filter feasible solutions based on instance's total weight. 
feasible w = filter ((>=) w . snd3)

-- Select maximum value from feasible set of combinations.
knapsack w = maximumBy (comparing fst3) . feasible w . combos

-- Create 3-tuple from each instance that contains total value and weight.
parse xs = (value, weight, xs) 
  where uz     = unzip xs
        value  = sum (fst uz) 
        weight = sum (snd uz)
-- /show

-- Getter functions for 3-tuple
fst3 (x,_,_) = x 
snd3 (_,y,_) = y

```


``` active haskell 
{-# START_FILE Main.hs #-}
import System.Random (Random, mkStdGen, randomRs)
import Knapsack

main = do
  -- show Solve example problem
  -- Input values
  let n = 10 -- problem size
  let capacity = 200 
  let seed = 42
  
  -- Get random values and weights
  let values = take n (random seed)
  let weights = take n (random (seed +1))
  
  let items = zip values weights
  let result = knapsack capacity items
  
  output items result
  -- /show
  
   
output items result = do 
  putStrLn $ "Items [(value, weight)] \n" ++ show items 
  putStrLn $ "\nResult (value, weight, [items])\n" ++ show result
  
-- For given seed, create infinite list of random integers from interval.
random seed = randomRs(1,100) (mkStdGen seed)
  
{-# START_FILE Knapsack.hs #-}
-- show
-- /show
module Knapsack (knapsack) where

import Data.List (subsequences, filter, maximumBy)
import Data.Ord (comparing)

-- Brute Force Algorithm for Knapsack Problem
-- Complexity: O(2^n) (subsequences function)

-- Create a list of 3-tuples from item subsequences.
combos = map parse . subsequences

-- Filter feasible solutions based on instance's total weight. 
feasible w = filter ((>=) w . snd3)

-- Select maximum value from feasible set of combinations.
knapsack w = maximumBy (comparing fst3) . feasible w . combos

-- Create 3-tuple from each instance that contains total value and weight.
parse xs = (value, weight, xs) 
  where uz     = unzip xs
        value  = sum (fst uz) 
        weight = sum (snd uz)

-- Getter functions for 3-tuple
fst3 (x,_,_) = x 
snd3 (_,y,_) = y

```

## Complexity Analysis

`O(2^n)` (<hoogle>subsequences</hoogle> function)

- `2` outcomes: in or out
- `^n` = Each outcome creates a new branch for the remaining elements in input items.

With `n = 20` the above code will run just fine. When n is increased, execution time starts to increase rapidly. Compared to feasible size `n = 20`, how much more time and memory would be required?  

- `n = 21`     -> `2^1   = 2`
- `n = 25`     -> `2^5   = 32`
- `n = 30`     -> `2^10  = 1024`
- `n = 100`    -> `2^80  = 1.2e+24`
- `n = 1000`   -> `2^980 = 1.0e+295`

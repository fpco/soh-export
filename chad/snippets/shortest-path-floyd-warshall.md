# Shortest Path - Floyd-Warshall

## Shortest Path Problem

![graph.png](https://www.fpcomplete.com/media/5217b2d7-bb7c-4364-a280-6ea2c5775b8b.png)

_In graph theory, the shortest path problem is the problem of finding a path between two vertices (or nodes) in a graph such that the sum of the weights of its constituent edges is minimized._

Reference [Shortest Path Problem (wiki)](http://en.wikipedia.org/wiki/Shortest_path_problem)


## Floyd-Warshall Algorithm
The Floyd-Warshall algorithm compares all possible paths through the graph between each pair of vertices. Algorithm has a nice recursive definition for calculating the shortest path between two vertices: [Floyd-Warshall](http://en.wikipedia.org/wiki/Floyd%E2%80%93Warshall_algorithm#Algorithm).

To find a path from `i` to `j` the algorithm will test between the direct path `{[i,..,j]}` using vertices `{1..k}` as intermediate points along the way and using a step `k+1`: `{[i,..,k+1],[k+1,..,j]}`. Recursive base case `k=0` is the edge `[i,j]`.

Normally in applications, the shortest path reconstruction is also required. In wiki page there is a slight modification in the algorithm: When a new shortest path is found, the matrix containing the paths is updated. This method won't fit directly in the functional programming paradigm. I had some problems to design path reconstruction function and I ended up with a solutions which calls `shortest` function recursively and I found this solution quite hard to read. Anyway, it solves the desired problem and could be improved using memoization.

## Solution
   

``` active haskell
{-# START_FILE Floyd.hs #-}
module Floyd (floyd_warshall) where

floyd_warshall start end graph = (dist, [start] ++ route ++ [end])
  where dist  = shortest (start, end, length graph) graph
        route = path (start, end, length graph) graph
        
-- Calculates the value of shortest route
shortest (i,j,0) g = g !! (i-1) !! (j-1) -- Transition value from graph
shortest (i,j,k) g = min (shortest (i,j,k-1) g) $
                         (shortest (i,k,k-1) g) + (shortest (k,j,k-1) g)

-- Reconstructs the shortest path
path (i,j,0) _ = []
path (i,j,k) g
  | direct < step =  path(i,j,k-1) g
  | otherwise     = (path(i,k,k-1) g) ++ [k] ++ (path(k,j,k-1) g)
  where direct =  shortest (i,j,k-1) g
        step   = (shortest (i,k,k-1) g) + (shortest (k,j,k-1) g)


{-# START_FILE Main.hs #-}
import Data.List (transpose)
import Floyd

main = do
  -- show Example problem
  contents <- readFile "graph.txt"
  let graph = readGraph contents

  output $ floyd_warshall 1 8 graph
  -- /show

output x = do
  putStrLn $ "(Length, [nodes])"
  putStrLn $ show x

readGraph = transpose . str2int . map words . lines
str2int = map.map $ zero2inf . fromIntegral . (\xs -> read xs :: Int)
zero2inf x = if x == 0 then 1/0 else x
 
{-# START_FILE graph.txt #-}
0  0  0  0  0  0  0  0
8  0  0  0  0  0  0  0
15 13 0  0  0  0  0  0
9  0  0  0  0  0  0  0
0  0  6 13  0  0  0  0
0  0  0  0  7  0  9  0
0  0  5  0  0  0  0  0
0  0  0  0  0  4  18 0

```

## Memoization
Solution above will actually use much more computation and memory than is needed. It's the first working version and I hope I'll have time to improve it. The first improvement for this algorithm would be implementing memoization. This technique allows to use already calculated values.

## Complexity
The algorithm requires three iterations through vertices (paths between each i and j and steps 1..k), thus `O(V^3)`.
# Simple Wolfram Automata

```active haskell
import Prelude hiding (length, replicate)
import Data.Vector hiding ((!), mapM_, take)
import qualified Data.Vector as V ((!?))
import Data.Monoid
 
type Cell = Char
type Universe = Vector Cell
 
size :: Int
size = 30
 
-- a starting universe with (2 * size + 1) cells
start :: Universe
start = replicate size off <> fromList [on] <> replicate size off
 
on, off :: Cell
on = 'X'
off = ' '
 
-- Guarded index, where out-of-bounds cells are considered off
(!) :: Universe -> Int -> Cell
v ! i = maybe off id (v V.!? i)
 
-- Get the neighbors for a cell by index
neighbors :: Universe -> Int -> Cell -> [Cell]
neighbors v i _ = fmap (v!) [i-1..i+1]
 
-- Calculate the next generation
next :: Universe -> Universe
next v = fmap rule90 $ imap (neighbors v) v
 
-- Wolfram's Rule 90 automata
rule90 :: [Cell] -> Cell
rule90 "XXX" = off
rule90 "XX " = on
rule90 "X X" = off
rule90 "X  " = on
rule90 " XX" = on
rule90 " X " = off
rule90 "  X" = on
rule90 "   " = off
 
main :: IO ()
main = mapM_ putStrLn . take size . fmap toList $ iterate next start
```
# Disassociatives: Not Even Once

NB: I wrote this long before I published it, but after logging to to SoH after many months, I found it relatively cogent, if not comprehensive.  I have nothing to add right now, so I'm releasing it, but it may attract more editing in the future.

To be a mathematical monoid, the monoid operation must be associative.  Free monoids (lists) also make this assumption and encode a preference for full right association (cons lists) or full left association (snoc lists).

Mathematical magams do not make the requirement, and free magams (trees) do not make this assumption.  This allows the creator of the structure to suggest a way to "insert parentheses" but also allows the consumer, who generally provides the operation, to reassociate the structure based on the particulars of the operation.

## Floating-Point Addition

Addition.  Associative or not?  If so, rearranging parentheses shoudn't matter for the result.  Let's try it.

``` active haskell
-- /show Imports
import Numeric (showFFloat)
-- show
lassocSum :: Float
lassocSum = (((((((((100000000 + 1) + 1) + 1) + 1) + 1) + 1) + 1) + 1) + 1) + 1

rassocSum :: Float
rassocSum = 100000000 + (1 + (1 + (1 + (1 + (1 + (1 + (1 + (1 + (1 + 1)))))))))

main :: IO ()
main = fprint lassocSum >> fprint rassocSum

-- /show
fprint :: Float -> IO ()
fprint f = print $ Numeric.showFFloat Nothing f ""
```

Of course, the "right" way to add up floating point numbers depends on the numbers.  But, this does hint that forcing *any* uniform association up front might be a mistake.

## List Appends

Clearly list appends must be associative.  There's no way to introduce subtle errors in such a simple structure as a list, right?  You would be correct, as far as correctness goes, but...

``` active haskell
-- /show Imports
import Data.Time (getCurrentTime)
-- show

lol :: [[Int]]
lol = replicate 100 (replicate 100000 1)

main :: IO ()
main = do
    printTime
    print . length $ foldr1 (++) lol
    printTime
    print . length $ foldl1 (++) lol
    printTime
    
-- /show
printTime :: IO ()
printTime = getCurrentTime >>= print
```

...performance is very different.  For this operation though, full right association is the best way to go in all cases.  For snoc lists, full left association is key.  Association is especially relevant when using lists as monads, since the monad operations do not assciate the way `(++)` does.  Again, forcing *any* uniform association up front might be a mistake.

In a combination of this and the previous point, if you are merging a list of sorted lists into a single sorted list (e.g. implementing the merge phase of [Timsort](http://en.wikipedia.org/wiki/Timsort)) you'd want to merge pairs of smaller lists before merging pairs of larger lists, again for the performance benefits.

## [Matrix Chain Multiplication](http://en.wikipedia.org/wiki/Matrix_chain_multiplication)

Again, performance.  The result is the same no matter the grouping, but the time spent calculating the result can vary wildly based on the order.  It's best to do this analysis just before consuming the chain, rather than as the chain is built.
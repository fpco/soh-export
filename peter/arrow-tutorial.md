# Arrow Tutorial

<hoogle>Arrow</hoogle> is just another typeclass that is used to composite computation. A computation takes  inputs of some type and produces outputs of another type. In haskell, a funciton is a computation:

``` haskell
f :: b -> c
```

The minimal complete Arrow typeclass definition includes three functions and a set of laws.

``` haskell
class Arrow a where
    -- A function is a computation
    arr :: (b -> c) -> a b c

    -- Composite computations by feeding output of one computation to input of another
    (>>>) :: a b c -> a c d -> a b d

    -- Apply computation on first part of inputs, with the rest parts untouched.
    first :: a b c -> a (b, d) (c, d)
```

Here are some examples to help you get some intuition.

``` active haskell

import Control.Arrow
import Data.Char

-- show
f = arr even $ 42
g = arr ord >>> arr even $ '*'
h = (first $ arr even) (42, 42)
-- /show

main = do
    putStrLn $ "f = " ++ (show f)
    putStrLn $ "g = " ++ (show g) 
    putStrLn $ "h = " ++ (show h)
```

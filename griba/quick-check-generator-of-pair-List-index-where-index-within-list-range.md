# QuickCheck generator of (List, Index) where index within list range

###(List,Index) pair generator keeping the index within range

```haskell
class Arbitrary a where

  arbitrary :: Gen a

  shrink :: a -> [a]

```
QuickCheck Gen is a monad.

Here is the Arbitrary instance for lists, taken from the QuickCheck source

```haskell
instance Arbitrary a => Arbitrary [a] where
  arbitrary = sized $ \n ->
    do k <- choose (0,n)       -- list length
       sequence [ arbitrary | _ <- [1..k] ] -- :: m [a]

  shrink xs = shrinkList shrink xs

```

Let's add an index generator within [0, length list) except for empty lists giving 0 as index.

```haskell

data ListWithIdx a = ListWithIdx [a] Int deriving (Eq, Show)

instance Arbitrary a => Arbitrary (ListWithIdx a) where

  arbitrary = sized $ \n -> do
       k <- choose (0, n) -- list length
       -- added index
       i <- if k == 0 then return 0 else choose (0, k-1)
       
       list <- sequence [ arbitrary | _ <- [1..k] ]
       -- wrap them in the new type
       return $ ListWithIdx list i
```

Adding a shrink function to the instance: shrink the list and lessen the index according to lengths difference

```haskell

  shrink (ListWithIdx xs i) = 
  
     map wrap $ shrinkList shrink xs
      where
        lenXs = length xs
        
        wrap ys = assert (lenYs <= lenXs) $ 
        
                    ListWithIdx ys $ max 0 $ i - (lenXs - lenYs)
          where
            lenYs = length ys
```
Test it:

```haskell
import Data.List as L
import Test.QuickCheck.Gen
import Test.QuickCheck as Q
import Control.Exception (assert)

data ListWithIdx a = ListWithIdx [a] Int deriving (Eq, Show)

-- include Arbitrary instance for ListWithIdx here       
-- with arbitrary and shrink methods

propIndexIsInRange :: [ListWithIdx Int] -> Bool
propIndexIsInRange xs = all test xs
  where
    test (ListWithIdx list idx) = if null list
                                      then idx == 0
                                      else idx >= 0 && idx < length list
main :: IO ()
main = quickCheckWith Q.stdArgs {Q.maxSuccess = 30, Q.maxSize = 22} 
                      propIndexIsInRange

```
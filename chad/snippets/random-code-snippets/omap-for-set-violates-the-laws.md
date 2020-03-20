# omap for Set violates the laws

```haskell active
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
import qualified Data.Set as Set

newtype M = M { unM :: Int }
    deriving (Show, Num, Integral, Enum, Real)

instance Eq M where
    M a == M b = a `mod` 10 == b `mod` 10

instance Ord M where
    M a `compare` M b = (a `mod` 10) `compare` (b `mod` 10)

f :: M -> Int
f = unM

g :: Int -> M
g 1 = M 10
g x = M x

main :: IO ()
main = do
    let w = Set.fromList $ map M [1..10]
    print w
    let x = Set.map (* 10) w
    print x
    let y = Set.map (`div` 10) x
    print y
    let z = Set.map ((`div` 10) . (* 10)) w
    print z
```
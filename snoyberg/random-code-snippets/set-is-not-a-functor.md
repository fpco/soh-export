# Set is not a functor

```haskell active
import qualified Data.Set as Set

newtype AlwaysEq a = AlwaysEq { unAlwaysEq :: a }

instance Eq (AlwaysEq a) where
    _ == _ = True
instance Ord (AlwaysEq a) where
    _ `compare` _ = EQ

main :: IO ()
main = do
    let s = Set.fromList [1..3]
    print $ (Set.map unAlwaysEq . Set.map AlwaysEq) s
    print $ Set.map (unAlwaysEq . AlwaysEq) s
```
# Optimized implementation

Let's begin by using Vectors instead of Lists.

# Full implementation (also, an instance of Num to ease testing)

``` active haskell
{-# LANGUAGE MonadComprehensions #-}
import Prelude hiding (map, (++), filter, null, head, tail, sum)
import Data.Monoid
import Data.Vector hiding (accum)
import Data.Function
import qualified Data.Vector.Fusion.Stream as Stream
import qualified Data.Vector.Generic as Gen 

mySum :: (Gen.Vector v a, Monoid a) => v a -> a
{-# INLINE mySum #-}
mySum = Stream.foldl' mappend mempty . Gen.stream

newtype GA k a = GA (Vector (k,a)) deriving (Show, Eq)

class Monoid s => SimpleAlgebra s where
    (<*>) :: s -> s -> s
    
isZero x = x == mempty
clean ::  (Eq k, Monoid k) => GA k a -> GA k a
clean (GA list) = GA $ filter (not . isZero . fst) list

clean' (GA list) = GA $ filter ((/= 0) . fst) list

regroupWith f v  | null v = empty
                 | otherwise =  let 
                   x = head v
                   xs = tail v
                   (likeX, unlikeX) = unstablePartition  (((==) `on` snd) x) xs
                       in (f $ fst x `cons` map fst likeX, snd x) `cons` regroupWith f unlikeX
                    
accum list = regroupWith mySum list                    
accum' list = regroupWith sum list
regroup list = regroupWith id list


instance (Monoid a, Eq a, SimpleAlgebra k, Eq k) => Monoid (GA k a) where
    mempty = GA empty
    (GA list1) `mappend` (GA list2) =  clean $ GA $ accum (list1 ++ list2) 

instance  (Monoid a, Eq a, SimpleAlgebra k, Eq k) => SimpleAlgebra (GA k a) where
    (GA list1) <*> (GA list2) =  clean $ GA $ accum [(k1 <*> k2, a1 <> a2) | (k1,a1) <- list1, (k2,a2) <- list2]

instance (Monoid a, Eq a, Num k, Eq k) => Num (GA k a) where
  fromInteger n = let nK = fromInteger n
                  in if (nK == 0)
                     then GA empty
                     else GA $ singleton (nK, mempty) 
  (GA list1) + (GA list2) =  clean' $ GA $ accum' (list1 ++ list2) 
  (GA list1) * (GA list2) =  clean' $ GA $ accum' [(k1 * k2, a1 <> a2) | (k1,a1) <- list1, (k2,a2) <- list2]
  negate (GA list) = GA $ map (\(k,a) -> (negate k, a)) list
  
main = do print $  GA (singleton (-2,Sum 3)) + GA (singleton (1, Sum 2))
          print $ GA (singleton (-2,Sum 3)) * GA (singleton (1, Sum 2))
          print $ GA (singleton (-2,Sum 3)) + GA (singleton (1, Sum 3))
          print $ GA (singleton (-2,Sum 3)) + GA (singleton (0, Sum 2))
```

We are basically done. 
Leave me some feedback if you want me to implement something else.
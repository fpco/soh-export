# Basic implementation

# Cleaning up
First, we will define a newtype for a generated algebra.

``` active haskell

newtype GA k a = GA [(k,a)] deriving (Show, Eq)

main = print $ GA $ [(2,'a'),(-1,'c')]
```

Then, we will build our pyramid of requirements:


We search hoogle for monoid: <hoogle>monoid</monoid>.

Let's go:

``` haskell
import Data.Monoid
import Data.List

--monoid already has mempty and mappend.

-- this will be more visual.
(<+>) = mappend

class Monoid s => SimpleAlgebra s where
    (<*>) :: s -> s -> s
    
isZero x = x == mempty

instance (Monoid a, Eq a, SimpleAlgebra k) => Monoid (GA k a) where
    mempty = GA []
    (GA list1) `mappend` (GA list2) =  ... 
```

Because `list1` and `list2` are both lists of terms, you might guess that their sum will be simply the concatenation: 

``` haskell
(GA list1) `mappend` (GA list2) = GA $ (list1 ++ list2)
```
The problem is that this will never ever simplify; it will keep on growing forever. 

We would like our `GA`s to be always as simple as they can get.

So here we hit our first road block; You see, we want the following simplification:

``` haskell

2*a1 + 3*a1 = (2+3) *a1

```

Here is how I break down this task into manageable pieces:

``` haskell

2*a1 + 3*a1 = (map (+) [2,3]) *a1 = 5 * a1 

```

And here is how you do this:

### Regrouping:

``` active haskell
import Data.Monoid
import Data.List
import Data.Function

regroup [] = []
regroup (x:xs) = let (likeX, unlikeX) = partition (((==) `on` snd) x) xs
                 in (fst x:map fst likeX, snd x) : regroup unlikeX

main = do let a = [(k,a) | k <- [0..5], a <- [2..7]]
          print $ regroup a
```
Then we generalize to include a function:

``` active haskell
import Data.Monoid
import Data.List
import Data.Function

regroupWith f [] = []
regroupWith f (x:xs) = let (likeX, unlikeX) = partition (((==) `on` snd) x) xs
                       in (f $ fst x:map fst likeX, snd x) : regroupWith f unlikeX

regroup list = regroupWith id list

main = do let a = [(k,a) | k <- [0..5], a <- [2..7]]
          print $ regroup a
```
## simplifying by accumulation
Then our simplifier is just:

``` haskell
accum list = regroupWith mconcat list
```

``` active haskell
import Data.Monoid
import Data.List
import Data.Function

regroupWith f [] = []
regroupWith f (x:xs) = let (likeX, unlikeX) = partition (((==) `on` snd) x) xs
                       in (f $ fst x:map fst likeX, snd x) : regroupWith f unlikeX

accum list = regroupWith mconcat list

main = do let a = [(k,a) | k <- map Sum [0..5], a <-  [2..7]]
          print $ accum a
```

## Monoid instance (how to add two elements of our GA)

``` haskell

import Data.Monoid
import Data.List
import Data.Function

newtype GA k a = GA [(k,a)] deriving (Show, Eq)

--monoid already has mempty and mappend.
-- this will be more visual.
(<+>) :: Monoid a0 => a0 -> a0 -> a0
(<+>) = mappend

class Monoid s => SimpleAlgebra s where
    (<*>) :: s -> s -> s
    
isZero x = x == mempty
clean ::  (Eq k, Monoid k) => GA k a -> GA k a
clean (GA list) = GA $ filter (not . isZero . fst) list

regroupWith f [] = []
regroupWith f (x:xs) = let (likeX, unlikeX) = partition (((==) `on` snd) x) xs
                       in (f $ fst x:map fst likeX, snd x) : regroupWith f unlikeX
                    
accum list = regroupWith mconcat list                    
regroup list = regroupWith id list


instance (Monoid a, Eq a, SimpleAlgebra k, Eq k) => Monoid (GA k a) where
    mempty = GA []
    (GA list1) `mappend` (GA list2) =  clean $ GA $ accum (list1 ++ list2) 
```

## SimpleAlgebra instance (how to multiply in GA)

We basically multiply (by distributivity) each term of the first list with each term of the second list, and then we add all of it together.

To multiply two terms together, we multiply the `k`s in K, and the `a`s in A.
``` haskell
instance  (Monoid a, Eq a, SimpleAlgebra k, Eq k) => SimpleAlgebra (GA k a) where
    (GA list1) <*> (GA list2) =  clean $ GA $ accum [(k1 <*> k2, a1 <> a2) | 
                                                     (k1,a1) <- list1, 
                                                     (k2,a2) <- list2]
```

# Full implementation

``` haskell
import Data.Monoid
import Data.List
import Data.Function

newtype GA k a = GA [(k,a)] deriving (Show, Eq)

--monoid already has mempty and mappend.
-- this will be more visual.
(<+>) :: Monoid a0 => a0 -> a0 -> a0
(<+>) = mappend

class Monoid s => SimpleAlgebra s where
    (<*>) :: s -> s -> s
    
isZero x = x == mempty
clean ::  (Eq k, Monoid k) => GA k a -> GA k a
clean (GA list) = GA $ filter (not . isZero . fst) list

regroupWith f [] = []
regroupWith f (x:xs) = let (likeX, unlikeX) = partition (((==) `on` snd) x) xs
                       in (f $ fst x:map fst likeX, snd x) : regroupWith f unlikeX
                    
accum list = regroupWith mconcat list                    
regroup list = regroupWith id list


instance (Monoid a, Eq a, SimpleAlgebra k, Eq k) => Monoid (GA k a) where
    mempty = GA []
    (GA list1) `mappend` (GA list2) =  clean $ GA $ accum (list1 ++ list2) 

instance  (Monoid a, Eq a, SimpleAlgebra k, Eq k) => SimpleAlgebra (GA k a) where
    (GA list1) <*> (GA list2) =  clean $ GA $ accum [(k1 <*> k2, a1 <> a2) | (k1,a1) <- list1, (k2,a2) <- list2]

```

Next, we will implement this a little more efficiently.
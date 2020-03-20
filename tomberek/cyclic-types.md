# Cyclic Types

This is an attempt to capture a concept in Haskell that I have been trying to do for a while. Some of the new extensions and type-level hackery seem to help, though I suspect I am not taking advantage of everything I could. Let me know if this can be simplified or if there is a better way.

Functioning code is at the very bottom.

# Type-Cycles
Some datatypes we commonly use are recursive or mutually recursive, leading to what I call a type-cyclic structure where it is possible to traverse back to the original datatype.

This has a type-cycle of 2, even though the data actually is in a cycle of 4:
``` haskell
data A = A Int B
data B = B String A
exampleAB = A 1 $ B "one" $ A 2 $ B "two" exampleAB 
```

This has a type-cycle of 1 when using `drop 1`:
``` haskell
data List a = Cons a (List a) | Nil
```

A tree like this has a type-cycle of 1 as you traverse it manually or with a zipper:
``` haskell
data Tree a = Node a (Tree a) (Tree a) | Leaf a | TNil
```

Last example inspired by `Data.Graph` using `Data.Array`. This has a type-cycle of 1.
``` haskell
type Graph = Table [Int]
type Table a = Array Int a
nextItems :: Graph -> Int -> [Int]
nextItems = (!)

nextUnsafe :: Graph -> Int -> Int
nextUnsafe g = head.(nextItems g)
```
Repeated applications of `nextUnsafe g` will traverse the graph one-by-one. Though one can imagine a slower traversal. Taking only a single result (unsafe) when multiple or none are possible leads to a type-cycle of 2 when traversing using `outEdge` and `outNode`. This is similar to the concept of using a bipartite double cover to represent a directed graph.
``` 
nextEdge :: Graph -> Int -> (Int,Int)
nextEdge g i = (i, head $ g!i)

nextNode :: (Int,Int) -> Int
nextNode = snd
```

# Abstract away
This notion is pretty general and can be captured with the idea that they are categories with n-objects and the traversals are arrows between them. The category diagram can be represented with a multipartite graph. Zippers have abstracted the notion of traversal; we can use the same polymorphic functions to traverse [zippers](https://hackage.haskell.org/package/zippers). I wanted to see if I can make the Haskell type system enforce the notion that a type was cyclic.

## Attempt 1 - find cycles
``` haskell
class Cycle1 a where
    next :: a -> a
```
Not very promising.
``` haskell
class Cycle2 a where
    type Next a
    next :: Cycle2 (Next a) => a -> Next a
```
This is closer, but it doesn't ensure that the cycle returns to the original datatype. Starting at `a` you may go to `b` which then loops along itself. Not satisfying. It seems we need a way to identify a cycle at the type-level; enter type families:
``` haskell
data Cycle (n::Nat)
type Counter a = Count (Next a) a 1
type family Count a b n where
    Count a b 20 = Void     -- Used to terminate the type-checker
    Count b b n = Cycle n
    Count a b n = Count (Next a) b (n+1)
```
This seems to have the right capabilities. We can't use a constraint (or can we? let me know) like `class (Counter a ~ Cycle n) => Cycle a where` because we need to have `n` in the class head. Ideally it would not be manually entered by `instance` writers. With a little type in-equality help:
``` haskell
type a /=/ b = (a == b) ~ False

class (Counter a /=/ Void) => Cycle3 a where
    type Next a
    next :: Cycle3 (Next a) => a -> Next a
```
Success! This ensures that a type-cycle exists and is of a set length `n`.

## Attempt 2 - traverse faster
At the moment `next` merely traverses the structure one-by-one. That is fine for 1-cycles, but how can we do `a -> a` hops for 2-cycles, 3-cycles, etc? I could not think of a way to express this properly:
``` haskell
data Cycle (n::Nat) = Proxy
class (Cycle3 a, Counter a ~ Cycle n) => CycleN a n
    neighbor :: Count n -> a -> a
    neighbor n a = foldr ($) a $ replicate (natVal n) next
```
So instead we I tried a next typeclass:
``` haskell
data Cycle (n::Nat) {-hi-}a {-/hi-}
type Counter a = Count (Next a) a 1
type family Count a b n where
    Count b b n = Cycle n {-hi-}b {-/hi-}
    Count a b 3 = Void
    Count a b n = Count (Next a) b (n+1)
type CyclicN a r n = (Cyclic a
                    ,Cyclic (Next a)
                    ,Cyclic (Next (Next a))
                    , {-hi-}Counter a ~ Cycle n a {-/hi-},a~r)
class (CyclicN a r n) => Cyc a r n where
    neighbor :: {-hi-}a -> a{-/hi-}

instance (a ~ Next (Next (Next a))
        ,CyclicN a (Next (Next (Next a))) 3) => Cyc a a 3 where 
    neighbor = next . next . next 
instance (a ~ Next (Next a)
        ,CyclicN a (Next (Next a)) 2)        => Cyc a a 2 where 
    neighbor = next . next 
instance (a~Next a
        , CyclicN a (Next a) 1)              => Cyc a a 1 where 
    neighbor = next
```
This is an improvement. Now we have the capablity to call `next` the proper number of times to produce the right result to hop through our structure.

## Attempt 3 - Functors and Monads, oh my!
Most of these structures are not single-linked, sometimes the natural result is a list, set, `Maybe`, or anything else. Our function should not be `a -> a`, but more like `Functor => a -> f a` and if it wasn't ugly already; we have to add structure to our `Counter` and even more Constraints:
``` haskell
data Cycle (n::Nat) b {-hi-}r {-/hi-}
type Counter a = Count (Next a) a 1 {-hi-}((F a) a){-/hi-}
type family Count a b n r where
    Count b b n r = Cycle n b r
    Count a b 3 r = Void
    Count a b n r = Count (Next a) b (n+1) ((F a) r)

class (Counter a /=/ Void, {-hi-}Functor (F a){-/hi-}) => Cyclic a where
    type Next a
    {-hi-}type F a :: * -> * {-/hi-}
    {-hi-}nextSet :: (Cyclic (Next a)) => a -> F a (Next a){-/hi-}
    step :: (Cyclic (Next a)) => a -> Next a

type CyclicN a b n r = (Monad (F a),Cyclic a
                    ,Cyclic (Next a)
                    ,Cyclic (Next (Next a))
                    ,Counter a ~ Cycle n b r, b~a)
                    
class (CyclicN a b n r) => Cyc a b n r where
    neighbors :: a -> r

instance (CyclicN a e 3 b,b~F a (F c (F d a)),c~Next a,d~Next c,e~Next d) => Cyc a a 3 b where 
    neighbors = fmap (fmap nextSet . nextSet) . nextSet
instance (CyclicN a d 2 b,{-hi-}b~F a (F c a){-/hi-},c~Next a,d~Next c) => Cyc a a 2 b where 
    {-hi-}neighbors = fmap nextSet . nextSet{-/hi-}
instance (CyclicN a c 1 b,b~F a a,c~Next a) => Cyc a a 1 b where 
    neighbors = nextSet
```

Now I'm getting the hang of type-level computation, but it is *very* ugly. Any way to clean this up? Wait, we're not done yet!

I don't like having results like `Just (Just (Just 2))`. Let's smash them together whenever possible:
``` haskell
data Cycle (n::Nat) b (x :: * -> *) xr (f::Bool)
type Counter a = Count (Next a) a 1 ((F a) a) (F a == F (Next a))
type family Count a b n r f where
    Count b b n (x r) f     = Cycle n b x (x r) f
    Count a b 3 r f         = Void
    Count a b n (x r) True  = Count (Next a) b (n+1) (x r) (F a == F (Next a))
    Count a b n (x r) False = Count (Next a) b (n+1) (x ((F a) r)) False
```
and 
``` haskell
class (CyclicN a b n x xr f ) => Cyc a b n x xr f where
    neighbors :: a -> xr
    
instance (CyclicN a e 3 x xr True, Monad x
            ,xr~F a a,F a ~ F c,F c ~ F d,c~Next a,d~Next c,e~Next d) => Cyc a a 3 x xr True where 
    neighbors = nextSet >=> nextSet >=> nextSet

instance (CyclicN a d 2 x xr True ,Monad x
            ,xr~F a a,F a ~ F c ,c~Next a,d~Next c) => Cyc a a 2 x xr True where 
    neighbors =  nextSet >=> nextSet
```
Now calling `neighbors` smashes together any duplicate `Monads` in a `join`-like fasion.

# Conclusion
The type system can do quite a bit and I'm almost certain I'm doing it wrong. How can this be simpler?

# Functioning Code
``` active haskell
{-# LANGUAGE TypeFamilies
    ,DataKinds
    ,TypeOperators
    ,ConstraintKinds
    ,GADTs
    ,UndecidableInstances
    ,FlexibleContexts
    ,MultiParamTypeClasses
    ,FlexibleInstances
    ,GeneralizedNewtypeDeriving
    ,StandaloneDeriving
    #-}
import GHC.TypeLits
import Data.Type.Equality
import Data.Word
import Data.Void
import Data.Functor.Identity
import Control.Monad ((>=>))
import qualified Data.Graph as G
import Data.Array
deriving instance Show a => Show (Identity a)

main :: IO ()
main = do 
    print $ "example:                     : " ++ show example
    print $ "step        (Node 2 example) : " ++ show (step (Node 2 example))
    print $ "step $ step (Node 2 example) : " ++ show (step $ step (Node 2 example))
    print $ "neighbors   (Node 2 example) : " ++ show (neighbors (Node 2 example))
    print $ "step     (DirNode 2 example) : " ++ show (step (DirNode 2 example))
    print $ "neighbors(DirNode 1 example) : " ++ show (neighbors (DirNode 1 example))
    print $ "step              (2 :: Int) : " ++ show (step (2 :: Int))
    print $ "step $ step       (2 :: Int) : " ++ show (step $ step (2 :: Int))
    print $ "neighbors         (2 :: Int) : " ++ show (neighbors (2 :: Int))
    print $ "neighbors   (2,exampleGraph) : " ++ show (neighbors (2::Int,exampleGraph))

exampleGraph = G.buildG (0,2) [(0,1),(1,2),(2,0),(2,1)]

type DList = [(Int,[Int])]
example :: DList
example = [(0,[1]),(1,[2]),(2,[0,1])]

data Node = Node Int DList
instance Show Node where
    show (Node x _) = "Node " ++ show x ++ " example"
data DirNode = DirNode Int DList
instance Show DirNode where
    show (DirNode x _) = "DirNode " ++ show x ++ " example"
data Edge = Edge (Int,Int) DList
instance Show Edge where
    show (Edge x _) = "Edge " ++ show x ++ " example"

newtype L a = L [a] deriving (Show,Functor,Monad) -- No type-level lamdas! gar!
instance Cyclic Node where
    type Next Node = Node
    type F Node = L
    step (Node x xs) = Node (fst (xs !! head (snd (xs !! x)))) xs
    nextSet (Node x xs) = L $ map (\y -> Node y xs) (snd $ xs !! x)

instance Cyclic DirNode where
    type Next DirNode = Edge
    type F DirNode = L
    step (DirNode x xs) = Edge (x,fst (xs !! head (snd (xs !! x)))) xs
    nextSet (DirNode x xs) = L $ map (\y -> Edge (x,y) xs) (snd (xs !! x))
instance Cyclic Edge where
    type Next Edge = DirNode
    type F Edge = Identity
    step (Edge x xs) = DirNode (snd x) xs
    nextSet (Edge x xs) = Identity $ DirNode (snd x) xs
-- This instance has a cycle, but not one that returns to the original instance
{-
data BadInstance = BadInstance (Int,Int) GList deriving Show
instance Cyclic BadInstance where
    type Next BadInstance = DirNode
    type F BadInstance = Identity
    step (BadInstance x xs) = DirNode (fst x) xs 
    nextSet = Identity . step
---}
instance Cyclic Int where
    type Next Int = Integer
    type F Int = Identity
    step x = 2 * fromIntegral x
    nextSet = Identity . step
instance Cyclic Integer where
    type Next Integer = Word32
    type F Integer = Identity
    nextSet = Identity . step
    step x = 2 * fromIntegral x
instance Cyclic Word32 where
    type Next Word32 = Int
    type F Word32 = Identity
    step x = 2 * fromIntegral x
    nextSet = Identity . step

instance Cyclic [a] where
    type Next [a] = [a]
    type F [a] = Maybe
    step = tail
    nextSet (_:x:xs) = Just (x:xs)
    nextSet _ = Nothing

data Tree a = TNode a (Tree a) (Tree a) | Leaf a | Nil
instance Cyclic (Tree a) where
    type Next (Tree a) = Tree a
    type F (Tree a) = L
    step (TNode _ a _) = a  -- Can't use Forest without fully applying it! gar!
    step _ = Nil
    nextSet (TNode _ xs ys) = L [xs,ys]
    nextSet _ = L []

instance Cyclic (G.Vertex,G.Graph) where
    type Next (G.Vertex,G.Graph) = (G.Edge,G.Graph)
    type F (G.Vertex,G.Graph) = L
    step (v,g) = ( (v,head $ g!v) ,g)
    nextSet (v,g) = L $ map (\x -> ( (v,x),g)) $ g!v

instance Cyclic (G.Edge,G.Graph) where
    type Next (G.Edge,G.Graph) = (G.Vertex,G.Graph)
    type F (G.Edge,G.Graph) = L
    step ((_,v),g) = (v,g)
    nextSet ((_,v),g) = L [(v,g)]
    
-- show
data Cycle (n::Nat) b (x :: * -> *) xr (f::Bool)
type Counter a = Count (Next a) a 1 ((F a) a) (F a == F (Next a))
type family Count a b n r f where
    Count b b n (x r) f     = Cycle n b x (x r) f
    Count a b 3 r f         = Void
    Count a b n (x r) True  = Count (Next a) b (n+1) (x r) (F a == F (Next a))
    Count a b n (x r) False = Count (Next a) b (n+1) (x ((F a) r)) False

class (Counter a /=/ Void, Functor (F a),Monad (F a)) => Cyclic a where
    type Next a
    type F a :: * -> *
    nextSet :: (Cyclic (Next a)) => a -> F a (Next a)
    step :: (Cyclic (Next a)) => a -> Next a

class (Counter w /=/ Void, Functor (F2 w),Monad (F w)) => Cyclic2 w where
    type Part w
    type F2 w :: * -> *

type CyclicN a b n x xr f= (Monad (F a),Cyclic a
                    ,Cyclic (Next a)
                    ,Cyclic (Next (Next a))
                    ,Counter a ~ Cycle n b x xr f, b~a)
class (CyclicN a b n x xr f ) => Cyc a b n x xr f where
    neighbors :: a -> xr
instance (CyclicN a e 3 x xr True, Monad x
            ,xr~F a a,F a ~ F c,F c ~ F d,c~Next a,d~Next c,e~Next d) => Cyc a a 3 x xr True where 
    neighbors = nextSet >=> nextSet >=> nextSet
instance (CyclicN a e 3 x xr False
            ,xr~F a (F c (F d a)),c~Next a,d~Next c,e~Next d) => Cyc a a 3 x xr False where 
    neighbors = fmap (fmap nextSet . nextSet) . nextSet

instance (CyclicN a d 2 x xr True ,Monad x
            ,xr~F a a,F a ~ F c ,c~Next a,d~Next c) => Cyc a a 2 x xr True where 
    neighbors =  nextSet >=> nextSet
instance (CyclicN a d 2 x xr False
            ,xr~F a (F c a)      ,c~Next a,d~Next c) => Cyc a a 2 x xr False where 
    neighbors = fmap nextSet . nextSet

instance (CyclicN a c 1 x xr f
            ,xr~F a a,c~Next a) => Cyc a a 1 x xr f where 
    neighbors = nextSet

type a /=/ b = (a == b) ~ False
-- /show
```
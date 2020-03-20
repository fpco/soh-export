# KolmogorovFlatMaps

## Kolmogorov R-sets and related flat maps

We will show how to define in Haskell a `flatten` and `prioritiesMap`. Both of them quite
standard and known to Haskell programmers. The first mapping is essentially a flatMap on NestedLists:

``` active haskell
data NestedList a = Elem a | List [NestedList a] 
                    deriving (Show)
                    
flatten :: NestedList a -> [a]
flatten (Elem x) = [x]
flatten (List x) = concatMap flatten x

main = do
    print (flatten list)
      where
        list = List [List[List [Elem (2,15)]], List [ List[Elem (7,5)], List[Elem (6,4)]]]    
```

A discussion of `NestedList` one can find at [99 Haskell problems](http://www.haskell.org/haskellwiki/H-99:_Ninety-Nine_Haskell_Problems). 
More specifically, this is [problem 7](http://www.haskell.org/haskellwiki/99_questions/Solutions/7) and the above solution is one
of the listed solutions.

Now let us improve on the pretty printing side


``` active haskell
data NestedList a = Elem a | List [NestedList a] 
                    deriving (Show)
                   
-- simple pretty print, a slightly better pp' below                      
                    
pp :: Show a => NestedList a -> IO ()
pp = (mapM_ putStr) . bracketing
  where
    bracketing (Elem a)             = [show a]
    bracketing (List x)             = ["("]++(concatMap bracketing x)++[")"]

main = do
    pp list
      where
        list = List [List[List [Elem (2,15)]], List [ List[Elem (7,5)], List[Elem (6,4)]]]       
```

Now let us improve even further adding commas in appropriate places


``` active haskell
data NestedList a = Elem a | List [NestedList a] 
                    deriving (Show)

-- The ordered pairs (n,m) are kept unchanged and
-- lists are indicated by braces { and }. The number of braces to the right
-- is the implicit parity index (see prioritiesMap below)

pp' :: Show a => NestedList a -> IO ()
pp' = (mapM_ putStr) . bracketing
  where
    bracketing (Elem a)             = [show a]
    bracketing (List (x:[]))        = ["{"]++(bracketing x)++["}"]
    bracketing (List (x:xs))        = ["{"]++(bracketing x)++[","]++(concatMap bracketing xs)++["}"]

main = do
    pp' list
      where
        list = List [List[List [Elem (2,15)]], List [ List[Elem (7,5)], List[Elem (6,4)]]]   
```


We are ready to define `prioritiesMap`

``` active haskell
data NestedList a = Elem a | List [NestedList a] 
                    deriving (Show)
                    

prioritiesMap (x) = prioritiesMap'(x,0)

prioritiesMap' :: (NestedList a, Int) -> [Int]
prioritiesMap' (Elem a,n)   = [n]
prioritiesMap' (List (x:[]), n) =  prioritiesMap' (x,n+1) 
prioritiesMap' (List (x:y:xs),n) = prioritiesMap' (x,0) ++ prioritiesMap' (List( y:xs),n)
prioritiesMap' (List [],n)     = []

main = do
    print (prioritiesMap list)
      where
        list = List [List[List [Elem (2,15)]], List [ List[Elem (7,5)], List[Elem (6,4)]]]    
```


Some links related to the paper:

V. Kanovei, [Kolmogorov's ideas in the theory of operations on sets](http://www.mathnet.ru/links/455cc6de9147c8573d61ba964222ae53/rm2048.pdf)
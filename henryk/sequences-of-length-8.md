# Sequences of length 8

There are some symmetries which must be satisfied:

``` active haskell
pattern0 = [1,1,1,0,-1,-1,-1,0]
pattern1 = [0,1,1,1,0,-1,-1,-1]
pattern2 = [-1,0,1,1,1,0,-1,-1]
pattern3 = [-1,-1,0,1,1,1,0,-1]
pattern4 = [3,4,3,4,3,4,3,4]
pattern5 = [1,1,1,0,0,0,0,0,0]
pattern6 = [0,1,1,1,0,0,0,0,0]
                    
symmetry :: [Int] -> [Int] -> Int
symmetry lengths pattern = sum [ (lengths !! i) * (pattern !! i) | i <- [0..7] ]

-- the desired bounds on coordinates are [0..30] not [8..12]
good0 = [ [l0,l1,l2,l3,l4,l5,l6,l7] | l0 <- [8..12], l1 <- [8..12], 
                                      l2 <- [8..12], l3 <- [8..12],
                                      l4 <- [8..12], l5 <- [8..12], 
                                      l6 <- [8..12], l7 <- [8..12] ] 

good1 = filter symmetryZeroHolds good0
          where symmetryZeroHolds lengths = (symmetry pattern0 lengths == 0)
                
good2 = filter symmetryOneHolds good1
          where symmetryOneHolds lengths = (symmetry pattern1 lengths == 0)

good3 = filter symmetryOneHolds good2
          where symmetryOneHolds lengths = (symmetry pattern2 lengths == 0)
                
good4 = filter symmetryOneHolds good3
          where symmetryOneHolds lengths = (symmetry pattern3 lengths == 0)

good5 = filter potential288 good4
          where potential288 lengths = (8 + symmetry pattern4 lengths == 288)

good6 = filter maxCoordinate good5
	  where maxCoordinate lengths = (lengths !! 0 == maximum lengths)

good7 = filter symmetryOneHolds good6
          where symmetryOneHolds lengths = (symmetry pattern5 lengths >= 10)

good8 = filter symmetryOneHolds good7
          where symmetryOneHolds lengths = (symmetry pattern6 lengths >= 10)

main = do
    print (take 10000 good8)
    print (length good8) 
```
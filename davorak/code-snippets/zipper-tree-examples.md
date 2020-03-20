# Zipper Tree Examples

As I was writing a response to a [StackOverflow question](http://stackoverflow.com/questions/15489324/traversal-tree-with-lens-and-zippers/15489761#15489761) I realized that I, as a first time learner, would have a hard time learning the material with out experimenting with the expression in ghci. So the answer seemed like a good fit for School of Haskell.

The code below use the 
[lens](http://hackage.haskell.org/package/lens) package and a zipper data structure which you can read more about at the [wiki page](http://www.haskell.org/haskellwiki/Zipper)


``` active haskell
import Control.Lens
import Data.Tree
import Data.Tree.Lens

testTree :: Tree Integer
testTree = Node 1 [ Node 2 [ Node 4 [ Node 6 [], Node 8 [] ],
                             Node 5 [ Node 7 [], Node 9 [] ] ],
                    Node 3 [ Node 10 [], 
                             Node 11 [] ] 
                  ]

main = putStr $ drawTree $ fmap show $ testTree
```
ref: [Definition of Tree](http://hackage.haskell.org/packages/archive/containers/0.5.0.0/doc/html/src/Data-Tree.html#Tree)

We can then make a zipper out of the testTree with:

``` haskell
zipperTree = zipper testTree
```

# Viewing

To view a particular element of the tree we use the lenses in the [Data.Tree.Lens](http://hackage.haskell.org/packages/archive/lens/3.8.7.3/doc/html/Data-Tree-Lens.html) package.

There is one for both fields of the Tree data type, `root` and `branches`.

To look at the first root we can move downwards using `downward` paired with `root` and view the Integer field.

``` active haskell
import Control.Lens
import Data.Tree
import Data.Tree.Lens

testTree = Node 1 [ Node 2 [ Node 4 [ Node 6 [], Node 8 [] ],
                             Node 5 [ Node 7 [], Node 9 [] ] ],
                    Node 3 [ Node 10 [], 
                             Node 11 [] ] 
                  ]

zipperTree = zipper testTree

main = putStr $ show $ 
-- show
    zipperTree & downward root & view focus
-- /show
```

To look at the branches of the top node we can use `downward branches`. The below draws each subtree of the top node.

``` active haskell
import Control.Lens
import Data.Tree
import Data.Tree.Lens

testTree = Node 1 [ Node 2 [ Node 4 [ Node 6 [], Node 8 [] ],
                             Node 5 [ Node 7 [], Node 9 [] ] ],
                    Node 3 [ Node 10 [], 
                             Node 11 [] ] 
                  ]

zipperTree = zipper testTree

labelElems lst = map format $ zip [0..] lst
  where
    format (num, tree) = "List elem " ++ show num ++ ":\n"
                      ++ tree
                      ++ "\n"

main = sequence_ $ map putStr $ labelElems $ map drawTree $ map (fmap show) $
-- show
    zipperTree & downward branches & view focus
-- /show
```
Click on the light yellow paper looking icon in the upper right of the above code block to see the full source code and how the pretty printing works in this case.

If we wanted to see the root value of the first subtree of the top node:

``` active haskell
import Control.Lens
import Data.Tree
import Data.Tree.Lens

testTree = Node 1 [ Node 2 [ Node 4 [ Node 6 [], Node 8 [] ],
                             Node 5 [ Node 7 [], Node 9 [] ] ],
                    Node 3 [ Node 10 [], 
                             Node 11 [] ] 
                  ]

zipperTree = zipper testTree

main = putStr $ show $ 
-- show
    zipperTree & downward branches 
               & fromWithin traverse 
               & downward root 
               & view focus
-- /show
```

Here I move downward to the list of branches.  I then use `fromWithin` and use `traverse` to traverse the list, if this was a 2-tuple I could use `both` instead.

# Saving and replaying traversal paths

`saveTape` and `restoreTape` allow for you to save your position in the zipper so that it can be restored latter.

Save a position:

``` haskell
    tape = zipperTree & downward branches 
                      & fromWithin traverse 
                      & downward root 
                      & saveTape
```

Then to recreate the traversal through the tree I can:

``` haskell
    t <- (restoreTape tape testTree)
```
    

Then you can use t as the new zipper and modify it as normal:
``` active haskell
import Control.Lens
import Data.Tree
import Data.Tree.Lens

testTree = Node 1 [ Node 2 [ Node 4 [ Node 6 [], Node 8 [] ],
                             Node 5 [ Node 7 [], Node 9 [] ] ],
                    Node 3 [ Node 10 [], 
                             Node 11 [] ] 
                  ]

zipperTree = zipper testTree

main = do
    let tape = zipperTree & downward branches 
                          & fromWithin traverse 
                          & downward root 
                          & saveTape
    t <- (restoreTape tape testTree)
    putStr $ drawTree $ fmap show $
-- show
        t & focus .~ 15 & rezip
-- /show

```

The tape replays the steps that you took so can work on other trees so the follow would work with the tape as defined above:

``` active haskell
import Control.Lens
import Data.Tree
import Data.Tree.Lens

testTree = Node 1 [ Node 2 [ Node 4 [ Node 6 [], Node 8 [] ],
                             Node 5 [ Node 7 [], Node 9 [] ] ],
                    Node 3 [ Node 10 [], 
                             Node 11 [] ] 
                  ]
-- show
testTree2 = Node 1 [ Node 2 [] ]
-- /show

zipperTree = zipper testTree

main = do
    let tape = zipperTree & downward branches 
                          & fromWithin traverse 
                          & downward root 
                          & saveTape
                          
-- show
    t2 <- (restoreTape tape testTree2)
-- /show
    putStr $ drawTree $ fmap show $
-- show
        t2 & focus .~ 15 & rezip
-- /show

```


# Modifying multiple locations

If you want to modify multiple roots just hold off on reziping the zipper. The following modifies the two roots of testTree2:
``` active haskell
import Control.Lens
import Data.Tree
import Data.Tree.Lens

testTree = Node 1 [ Node 2 [ Node 4 [ Node 6 [], Node 8 [] ],
                             Node 5 [ Node 7 [], Node 9 [] ] ],
                    Node 3 [ Node 10 [], 
                             Node 11 [] ] 
                  ]
-- show
testTree2 = Node 1 [ Node 2 [] ]
-- /show

zipperTree = zipper testTree

main = do                          
    putStr $ drawTree $ fmap show $
-- show        
        zipper testTree2 & downward root 
                         & focus .~ 11            -- Modify the root of the top node
                         & upward                 -- Move back up from root
                         & downward branches
                         & fromWithin traverse    -- Traverse list of branches
                         & downward root 
                         & focus .~ 111           -- Modify the root of the only subNode
                         & rezip                  -- Convert back to type Tree
-- /show

```

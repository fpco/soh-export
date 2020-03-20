# Solution counting

# The problem

A freind of mine has an email list where he sends out a brain teaser once a week. I like subscribing to them, just to make sure my brain gets exercise on a regular basis.

The last one had this problem in it:

    How many times are our numbers found?
    Fill in the blanks to make these independent statements true:
   
      If 1 is found 2 times
      then 2 is found ___ times
      and 3 is found ___ times
      and 4 is found ___ time.

He then goes on to ask for solutions for five through nine, then complicates things by asking for hex digits.

# Testing a solution

My first thuoght was _Huh?_ After thinking about it, I realized what he was asking, and decided this was a perfect problem for tackling with lists. Except, of course, that there's you can figure out the solutions with logic. But this is about programming, not logic.

So, given that a solution can be represented as a list, where the element at position `i - 1` represents the number of times the digit `i` appears, then we can check if a solution is valid by checking that for each position in the list that the number of the position appears is one greater than the number of times the position number appears in the list. Since haskell numbers positions from 0, we have to subract one when getting the digit. That looks like so:


```active haskell
import Data.List (elemIndices)

checkList :: [Int] -> Bool
checkList is = and [checkItem i is | i <- [1 .. length is]]

checkItem :: Int -> [Int] -> Bool
checkItem i is = is !! (i - 1) == (length . elemIndices i $ is) + 1

-- testing
main = do
   print $ checkList [1, 2, 3]
   print $ checkList [2, 3, 2, 1]
```

# All the solutions?

Now that we can test a single list, it'd be nice if we could just search for all solutions of a specific lenght. Fortunately, we have a computer at hand, which excells at just that kind of thing. All we need to do is get a list of all possible solutions. In this case, that means to test solutions for 6, say, we need a list of all possible lists of the digits 1 through 6, allowing repeatitions in the list. That's easy to do in haskell, and we do it with `combinations`. Similarly, generating the list of combinations and then checking them is easy, so we do that in `solutions`:

``` active haskell
import Control.Monad (replicateM)
import Data.List (elemIndices)

checkList :: [Int] -> Bool
checkList is = and [checkItem i is | i <- [1 .. length is]]

checkItem :: Int -> [Int] -> Bool
checkItem i is = is !! (i - 1) == (length . elemIndices i $ is) + 1

combinations :: Int -> [[Int]]
combinations n = replicateM n [1 .. n]

solutions :: Int -> [[Int]]
solutions n = [p | p <- combinations n, checkList p]

-- testing
main = print $ solutions 4
```

# Pretty please?

But Haskell lists are sort of ugly. How about if we write a little function to print the solution list in a form similar to the original question? We'll have the function `display` display the entire list. It starts by displaying the first line, whichi is different, then using the helper function `display'` to recursively display the rest. The terminating case also displays a slightly different text:

``` active haskell

display :: [Int] -> String
display [] = "The only solution for the empty list is the empty solution."
display (a:as) = "1 appears " ++ show a ++ " times then\n" ++ display' 2 as

display' :: Int -> [Int] -> String
display' i (a:[]) = show i ++ " appears " ++ show a ++ " times.\n"
display' i (a:as) = show i ++ " appears " ++ show a ++ " times, and\n" ++ display' (i + 1) as

-- testing
main = putStr $ display [2, 4, 1, 3]
```

# A tool for searching

Now all we have to do is tie it all together.  We'll use `printSolutions` to print all the solutions,  and have `main` get a number from the console and then invoke `printSolutions`. Repeatedly.

``` active haskell
import Data.List (elemIndices)
import Control.Monad (replicateM)

checkList :: [Int] -> Bool
checkList is = and [checkItem i is | i <- [1 .. length is]]

checkItem :: Int -> [Int] -> Bool
checkItem i is = is !! (i - 1) == (length . elemIndices i $ is) + 1

combinations :: Int -> [[Int]]
combinations n = replicateM n [1 .. n]

display :: [Int] -> String
display [] = "The only solution for the empty list is the empty solution."
display (a:as) = "1 appears " ++ show a ++ " times then\n" ++ display' 2 as

display' :: Int -> [Int] -> String
display' i (a:[]) = show i ++ " appears " ++ show a ++ " times.\n"
display' i (a:as) = show i ++ " appears " ++ show a ++ " times, and\n" ++ display' (i + 1) as

solutions :: Int -> [[Int]]
solutions n = [p | p <- combinations n, checkList p]

printSolutions :: Int -> String
printSolutions n = "Solutions for lists of length " ++ show n ++ ":\n" ++ (concatMap display $ solutions n)

main = do
    putStrLn "What length solution to you want to look for? "
    line <- getLine
    putStr $ printSolutions $ read line
    main
```

# Comments

You can discuss this post and provide feedback to the author on the [Google + post](https://plus.google.com/100162554869434148021/posts/SJ39DcMBkGr) about it.



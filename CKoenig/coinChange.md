# coin change

# the task

given a list of coins `[c1,..,cn]` and a amount `a` we shall find a list `[a1,..,an]` of integers such that `c1*a1 + .. + cn*an = a` - but not just any - we want to minimize the number of coins given: `a1+..+an`.

[see here](http://craftsmanship.sv.cmu.edu/exercises/coin-change-kata)

I stumpled on this reading twitter, where [Mike Bild](https://twitter.com/mikebild) *solved* this using various LINQ-like expressions (well the first version of what I give you here) ... as I could not stand the *not-optimal* solution I had to brute-force a better solution (... and maybe I come back and try to improve on that some time)

## a trivial algorithm (that does give you correct change but not with fewest coins)
First define the coins used (as a `Integer`-list):

``` haskell
coins :: [Integer]
coins = [100, 25, 10, 5, 1]
```

Then the idea is to fold over the coins using the remaining amount we have to change and the count of used coins so far (so a 3 at position 2 will mean 2 - 25ct coins used)

``` haskell
change :: Integer -> [Integer]
change n = reverse . snd $ foldl next (n, []) coins
    where next (remaining, cs) coin
            | coin <= remaining = (r', cnt:cs)
            | otherwise         = (remaining, 0:cs)
            where r' = remaining `mod` coin
                  cnt = remaining `div` coin
```

Thats it - here you can try/play with it:

``` active haskell
-- /show
coins :: [Integer]
coins = [100, 25, 10, 5, 1]

change :: Integer -> [Integer]
change n = reverse . snd $ foldl next (n, []) coins
    where next (remaining, cs) coin
            | coin <= remaining = (r', cnt:cs)
            | otherwise         = (remaining, 0:cs)
            where r' = remaining `mod` coin
                  cnt = remaining `div` coin
                  
-- show
main = do
    putStrLn $ ("change for 15ct: "++) . show . change $ 15
    putStrLn $ ("change for 40ct: "++) . show . change $ 40
    putStrLn $ ("change for 99ct: "++) . show . change $ 99

```

## Why this might fail in the general case
Well sometimes ... let's change the coins and look for 30cts ...

``` active haskell
coins :: [Integer]
coins = [100, 25, 15, 10, 1]

-- /show
change :: Integer -> [Integer]
change n = reverse . snd $ foldl next (n, []) coins
    where next (remaining, cs) coin
            | coin <= remaining = (r', cnt:cs)
            | otherwise         = (remaining, 0:cs)
            where r' = remaining `mod` coin
                  cnt = remaining `div` coin
                  
-- show
main = do
    putStrLn $ ("change for 30ct: "++) . show . change $ 30

```
so 6 coins (1x25ct and 5*1ct) - WTF ... well the algorithm is too stupid in this case (the answer is of course 2x15ct) ... stay tuned - I gonna fix it ... soon

## brute-forcing our way
An easy way to fix this is to try every possible combination for small cases like the samples it's still ok

``` active haskell
import Data.List(minimumBy)
import Data.Maybe(mapMaybe)
import Control.Applicative ( (<$>) )

type Coin = Integer
type Cents = Integer

coins :: [Coin]
coins = [100, 25, 15, 10, 1]

findChange :: [Coin] -> Cents -> Maybe [Integer]
findChange coins amount = snd <$> calc amount coins
    where calc r []
            | r == 0    = Just (0, [])
            | otherwise = Nothing
          calc r (c:cs)
            | r < 0     = Nothing
            | otherwise = findMinimum . mapMaybe try $ [0 .. r `div` c]
                where try n = do
                        (n', ns) <- calc (r-n*c) cs
                        return (n + n', n:ns)
                      findMinimum [] = Nothing
                      findMinimum vs = Just . minimumBy (\ (a,_) (b, _) -> compare a b) $ vs

main :: IO ()
main = do
    putStrLn $ ("change for 30ct: "++) . show . findChange coins $ 30
```

As you can see, the code has changed a bit - but produces the right answer (2x15ct)!

Instead of trying to find the right fold-function I went for *manual* recursion for now:
You will find the algorithm inside of calc. The first few guards handle the edge-cases (nothing more to change, no-coins left to try, to much change given).
But in the case that there is still something to *give-back* and where there are coins left to change with the algorithm tries every combination and finaly turns back the one with fewest coins given recursivley.

## brute-force brakes down
Remember: I told you "it's still ok"?
Well **I lied** - try the brute-force algorithm for your normal coins `[100, 50, 25, 10, 5, 2, 1]` and (say) 2$34ct ... this will take the algorithm on my machine almost 10seconds!
Try the same for 100$ and get yourself some coffee...

## What can we do...
If you look carefully you'll see that we do the recursive-call not one time but two times - and just as with fibonacci numbers we will sureley hit the same **spot** more than once - but every time we calculate the thing again.
Just write the algorithm down as a tree and you will see that we recalculate the same branches time-and-time again...

So how can we change this?

We have to memoize the caluclated values! This is more or less the basic idea in *dynamic programming* (**DP**) - and while I will not give a complete introduction here I will try to show how you can change the algorithm to use this.

## First step towards DP
The first thing we are going to do is modify the algorithm a bit so that we can see better where the recursion is (the `case` expression below):

``` active haskell
type Coin = Integer
type Cents = Integer

defaultCoins :: [Coin]
defaultCoins = [100, 50, 25, 10, 5, 2, 1]

-- | tries change (using coins form the first parameter) to the amount of money in the second parameter with the fewest number of pieces in the change
-- | the first parameter should be a decreasing list of distinct values
takeCoin :: [Coin] -> Cents -> Maybe (Integer, [Coin])
takeCoin [] cents
    | cents == 0   = Just (0, [])
    | otherwise    = Nothing
takeCoin coins@(coin:coins') cents
    | cents < 0    = Nothing
    | coin > cents = takeCoin coins' cents
    | otherwise    = case (takeCoin coins (cents-coin), takeCoin coins' cents) of
                        (Just (n, t), Just (n',t')) -> Just $ if n <= n' then (n+1, coin:t) else (n', t')
                        (Nothing,     Just (n',t')) -> Just (n', t')
                        (Just (n,t),  Nothing)      -> Just (n+1, coin:t)

main :: IO ()
main = do
    putStrLn "let's change 2$34ct"
    let coins = defaultCoins
    let amount = 234

    putStrLn "here we go ... just wait - it'll take some time..."
    let res = takeCoin coins amount
    print res
```

This will still take long (indeed it will do slightly worse) but you can see better where the *magic* (aka recursion) is and where the edge-cases gets handled.

## enter lazy-arrays
Now how to memoize ... well the easiest thing I can think of (and indeed almost the same as with *real* DP) is to use an array.
Well now you cannot mutate arrays in Haskell but as Haskell is lazy this will be no problem!
How so? Well we just put lots of chunks into the array and let the algorithm lazily evalute those at needed.
BTW: this is the point where this algorithm can break down: the chunks need quite some memory and if your DP-table is large this will exhaust your memory (look for the knapsackproblem with large data-sets ...) but again (this time for sure): for this it will be sufficient (for me)!

So here it is - the final (quite quick) DP-solution to the problem

``` active haskell
import Data.Array

type Coin = Integer
type Cents = Integer

defaultCoins :: [Coin]
defaultCoins = [100, 50, 25, 10, 5, 2, 1]

takeCoinDP :: [Coin] -> Cents -> Maybe (Integer, [Coin])
takeCoinDP coins cents = get ltCoin cents
    where arr = array ((0,0), (ltCoin, cents)) [((i,c), takeC i c) | i <- [0..ltCoin], c <- [0..cents]]
          get i c
            | c < 0 || i < 0  = Nothing
            | c == 0          = Just (0, [])
            | otherwise       = arr!(i,c)
          ltCoin = length coins - 1
          takeC cNr cts
            | coin > cts          = get (cNr-1) cts
            | otherwise           = case (get cNr (cts-coin), get (cNr-1) cts) of
                                       (Just (n, t), Just (n',t')) -> Just $ if n+1 <= n' then (n+1, coin:t) else (n', t')
                                       (Nothing,     Just (n',t')) -> Just (n', t')
                                       (Just (n,t),  Nothing)      -> Just (n+1, coin:t)
                                       (Nothing,     Nothing)      -> Nothing
            where coin = coins !! cNr
            
main :: IO ()
main = do
    putStrLn "let's change 2$34ct"
    let coins = defaultCoins
    let amount = 234

    putStrLn "here we go ... this should be alot quicker..."
    let res = takeCoinDP coins amount
    print res
```

I hope you see the similarity to the version above.
The only *trick* here is to move the edge cases to the selection from the array (to avoid index-out-of-range exceptions and stuff).

Enjoy!
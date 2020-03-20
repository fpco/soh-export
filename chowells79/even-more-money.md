# Sending even more money

A short while ago I came across [Justin Le's take on the SEND+MORE=MONEY problem](http://blog.jle.im/entry/unique-sample-drawing-searches-with-list-and-statet).
It describes a clever way to model nondeterministic selection-without-replacement searches with `StateT [a] [] a`.
I liked the idea a lot, started playing with it, and immediately slammed into just *how* slow the naive search is.

It got me thinking about my final year at the University of Oregon.
I was on one of two (or was it three?) teams from the university to attend the Pacific Northwest regionals for the ACM Intercollegiate Programming Competition.
We were taking the train up the coast to the competition site and our coach, Eugene Luks, kept mentioning that brute-force searches were often good enough for combinatorial searches if the branches were pruned as early as possible.
This turned out to be important.
One of the problems in the competition had to do with finding permutations of numbers on a clock face such that no consecutive set of three had a sum that exceeded a given total.
I worked on that problem for my team, and the solution I put together was fast enough to complete the search in the time allotted.
Professor Luks referred to the search pattern as "branch-and-bound", and I've always remembered that name as apt.

It turns out the nondeterministic search Justin Le provided is a fully capable implementation of branch-and-bound searching for selection-without-replacement problems.
The end of his article showed an example of using `mfilter` to prune the tree early a bit.
But even that variant wasn't all that fast, so I decided to look into the problem further.

The first thing I wanted to figure out was how to count the number of selection operations being performed in the course of a search.
I immediately went for throwing a `Writer (Sum Int)` into the works to track the count.
I always mess up nesting transformers a bit, and this was no exception.
I eventually sorted it out when I remembered I wanted to count the selections in the entire search, not just in a branch.
I ended up with a stack like `StateT [a] (ListT (Writer (Sum Int))) b`.
I was slightly worried about `ListT`, but I checked the [validity precondition](https://hackage.haskell.org/package/transformers-0.4.3.0/docs/Control-Monad-Trans-List.html#t:ListT).
`Writer` is a commutative monad whenever the underlying monoid is commutative, and `Sum Int` is commutative.
And then I decided to test it.

``` active haskell
import Control.Monad.State.Strict
import Control.Monad.Trans.List
import Control.Monad.Morph
import Control.Monad.Writer.Strict
import Control.Arrow(second)

-- CS = Counting Selections
type CS a b = StateT [a] (ListT (Writer (Sum Int))) b

select' :: [a] -> [(a, [a])]
select' [] = []
select' (x:xs) = (x, xs) : [(y, x:ys) | (y, ys) <- select' xs]

select :: CS a a
select = do
    xs <- get
    lift . lift . tell . Sum  $! length xs
    hoist (ListT . return) (StateT select')

runCS :: CS a b -> [a] -> ([b], Int)
runCS a xs = second getSum . runWriter . runListT $ evalStateT a xs

main :: IO ()
main = print $ runCS select [1..3]
```

So far, so good.
It selected each element of the input list in turn, for a total of 3 selections
So I tested another couple variants.

``` active haskell
-- /show
import Control.Monad.State.Strict
import Control.Monad.Trans.List
import Control.Monad.Morph
import Control.Monad.Writer.Strict
import Control.Arrow(second)

-- CS = Counting Selections
type CS a b = StateT [a] (ListT (Writer (Sum Int))) b

select' :: [a] -> [(a, [a])]
select' [] = []
select' (x:xs) = (x, xs) : [(y, x:ys) | (y, ys) <- select' xs]

select :: CS a a
select = do
    xs <- get
    lift . lift . tell . Sum  $! length xs
    hoist (ListT . return) (StateT select')

runCS :: CS a b -> [a] -> ([b], Int)
runCS a xs = second getSum . runWriter . runListT $ evalStateT a xs
-- show
main :: IO ()
main = do
    print $ runCS (liftM2 (,) select select) [1..3]
    print $ runCS (liftM3 (,,) select select select) [1..3]
```

Huh.
I was promptly confused.
The output lists were exactly what I expected.
The selection counts were not.
I spent an hour chasing what I had done wrong.
Eventually, I found the problem — and it was in the worst place.
My expected results were wrong.
The code was correct!

I expected `6` as the result for both of those, because there were 6 elements in the list.
But that's not what I was actually counting.
I was counting the number of selection operations taking place.
It's the difference between counting the number of leaves in a tree and counting the number of nodes in the tree.
There is some relation between the two numbers, sure — but they're not the same.

Properly embarassed that incorrect expectations had caused me to spend an hour looking for bugs in correct code, I proceeded to blunder directly into an even more embarrassing problem.
I wasn't fully happy with the performance of the code.
I mean, I expected it to be slow.
I didn't expect just *how* ridiculously slow it would be.
I wasn't sure if it was buggy-slow, or just appropriately-slow for the amount of work being added over Justin Le's code.
So I grabbed a heap profile and took a look.
Sure enough, the heap use graph was a giant triangle consisting mostly of thunks.
I'd recently dismissed space leaks as not a real issue with experience.
Sure, they happen from time to time, but when you discover you have one, they're easy to fix.
And then I stared at this one cluelessly.
It was something of a foot-in-mouth moment, even if the only person who was aware of my silliness was me.

I eventually tracked the issue to `Writer` not keeping its accumulator evaluated, even when imported as `Control.Monad.Writer.Strict`.
I asked about it on IRC, and was directed to a [mailing list post](https://mail.haskell.org/pipermail/libraries/2013-March/019528.html) on the topic from Gabriel Gonzalez, pointing out the problem, the underlying cause, and suggesting a solution.

I replaced my use of `Writer (Sum Int)` with `State Int`.
I suffered a bit of anguish over the fact that I was losing my static guarantee that `ListT` was giving me a valid `Monad` instance, but I found solace in the fact that I was never branching on the current state, so I wasn't breaking any monad laws.

And that *mostly* flattened the heap profile graph.
But there was still a definite increasing pattern of there being a ton of `[]` constructors in the heap profile.
So I looked at the `Monad` instance for `ListT`, and found that its strictness depends on the strictness of the underlying `(>>=)` operation, of course.
And then I smacked my forehead and stopped importing `State` strictly.
Yeah - space leaks are caused by inappropriate strictness, too.

Finally, the heap profile graph was flat.
Ok, the changes required were small, but tracking them down was a problem.
I had to sleep on it to start tracking down the problematic location.
Space leaks are still not too hard to fix, once you pin down their *exact* cause!
(Yeah, I'm challenging fate again..  But how will I ever get better at solving tough space leaks if I don't keep running into them?)

So I was finally ready to experiment with bounding the search space better.

``` active haskell
import Control.Monad.State
import Control.Monad.Trans.List
import Control.Monad.Morph
import Data.List (foldl')

type CS a b = StateT [a] (ListT (State Int)) b

select' :: [a] -> [(a, [a])]
select' [] = []
select' (x:xs) = (x, xs) : [(y, x:ys) | (y, ys) <- select' xs]

select :: CS a a
select = do
    i <- lift . lift $ get
    xs <- get
    lift . lift . put $! i + length xs
    hoist (ListT . return) (StateT select')

runCS :: CS a b -> [a] -> ([b], Int)
runCS a xs = flip runState 0 . runListT $ evalStateT a xs

fromDigits :: [Int] -> Int
fromDigits = foldl' (\x y -> 10 * x + y) 0

sendMoreMoney :: ([(Int, Int, Int)], Int)
sendMoreMoney = flip runCS [0..9] $ do
    [s,e,n,d,m,o,r,y] <- replicateM 8 select
    let send  = fromDigits [s,e,n,d]
        more  = fromDigits [m,o,r,e]
        money = fromDigits [m,o,n,e,y]
    guard $ s /= 0 && m /= 0 && send + more == money
    return (send, more, money)

main :: IO ()
main = do
    putStrLn "This is going to take a while... But it will finish."
    print sendMoreMoney
```

Well.
That was...
Slow.

The problem is that it's doing a lot of selection.
I mean, look at that number.
Over 2.6 million selections!
Yes, that's quite a lot more than `10!/2!`.
The number of permuations only counts the *leaves* of the selection tree.
Once again, the full tree is a fair bit bigger than just the number of leaves it has.

The closer to the root things get pruned from the selection tree, the smaller the tree gets.
One of the constraints in the problem is that the initial digits, represented by `s` and `m`, can't be `0`.
Let's prune those out.


``` active haskell
-- /show
import Control.Monad.State
import Control.Monad.Trans.List
import Control.Monad.Morph
import Data.List (foldl')

type CS a b = StateT [a] (ListT (State Int)) b

select' :: [a] -> [(a, [a])]
select' [] = []
select' (x:xs) = (x, xs) : [(y, x:ys) | (y, ys) <- select' xs]

select :: CS a a
select = do
    i <- lift . lift $ get
    xs <- get
    lift . lift . put $! i + length xs
    hoist (ListT . return) (StateT select')

runCS :: CS a b -> [a] -> ([b], Int)
runCS a xs = flip runState 0 . runListT $ evalStateT a xs

fromDigits :: [Int] -> Int
fromDigits = foldl' (\x y -> 10 * x + y) 0

-- show
sendMoreMoney :: ([(Int, Int, Int)], Int)
sendMoreMoney = flip runCS [0..9] $ do
    s <- mfilter (/= 0) select
    m <- mfilter (/= 0) select
    [e,n,d,o,r,y] <- replicateM 6 select
    let send  = fromDigits [s,e,n,d]
        more  = fromDigits [m,o,r,e]
        money = fromDigits [m,o,n,e,y]
    guard $ s /= 0 && m /= 0 && send + more == money
    return (send, more, money)
-- /show
main :: IO ()
main = do
    putStrLn "This is a bit faster. A tiny bit."
    print sendMoreMoney
```

That didn't actually help that much.
Not nearly enough to make it fast.
Apparently removing only one top-level branch and one child from each of the remaing top-level branches isn't actually good enough.

But there's more structure to exploit in the problem.
For instance, the partial sums need to work out.
`d + e` needs to give you `y`...
More or less.
Let's try that out.


``` active haskell
-- /show
import Control.Monad.State
import Control.Monad.Trans.List
import Control.Monad.Morph
import Data.List (foldl')

type CS a b = StateT [a] (ListT (State Int)) b

select' :: [a] -> [(a, [a])]
select' [] = []
select' (x:xs) = (x, xs) : [(y, x:ys) | (y, ys) <- select' xs]

select :: CS a a
select = do
    i <- lift . lift $ get
    xs <- get
    lift . lift . put $! i + length xs
    hoist (ListT . return) (StateT select')

runCS :: CS a b -> [a] -> ([b], Int)
runCS a xs = flip runState 0 . runListT $ evalStateT a xs

fromDigits :: [Int] -> Int
fromDigits = foldl' (\x y -> 10 * x + y) 0

-- show
sendMoreMoney :: ([(Int, Int, Int)], Int)
sendMoreMoney = flip runCS [0..9] $ do
    [e,d,y] <- replicateM 3 select
    guard $ (d + e) `mod` 10 == y
    [s,n,m,o,r] <- replicateM 5 select
    let send  = fromDigits [s,e,n,d]
        more  = fromDigits [m,o,r,e]
        money = fromDigits [m,o,n,e,y]
    guard $ s /= 0 && m /= 0 && send + more == money
    return (send, more, money)
-- /show
main :: IO ()
main = do
    putStrLn "Now this is more like it."
    print sendMoreMoney
```

Finally, something reasonable!
At the third level of the tree, this prunes out all but one branch of each subtree.
That's just about a 90% reduction in size of the search tree.
Not bad.

I can do better.


``` active haskell
-- /show
import Control.Monad.State
import Control.Monad.Trans.List
import Control.Monad.Morph
import Data.List (foldl')

type CS a b = StateT [a] (ListT (State Int)) b

select' :: [a] -> [(a, [a])]
select' [] = []
select' (x:xs) = (x, xs) : [(y, x:ys) | (y, ys) <- select' xs]

select :: CS a a
select = do
    i <- lift . lift $ get
    xs <- get
    lift . lift . put $! i + length xs
    hoist (ListT . return) (StateT select')

runCS :: CS a b -> [a] -> ([b], Int)
runCS a xs = flip runState 0 . runListT $ evalStateT a xs

fromDigits :: [Int] -> Int
fromDigits = foldl' (\x y -> 10 * x + y) 0

-- show
sendMoreMoney :: ([(Int, Int, Int)], Int)
sendMoreMoney = flip runCS [0..9] $ do
    [d,e,y] <- replicateM 3 select
    guard $ (d + e) `mod` 10 == y
    [n,r] <- replicateM 2 select
    guard $ (fromDigits [n,d] + fromDigits [r,e]) `mod` 100 == fromDigits [e,y]
    [s,m,o] <- replicateM 3 select
    let send  = fromDigits [s,e,n,d]
        more  = fromDigits [m,o,r,e]
        money = fromDigits [m,o,n,e,y]
    guard $ s /= 0 && m /= 0 && send + more == money
    return (send, more, money)
-- /show
main :: IO ()
main = do
    putStrLn "Whoa."
    print sendMoreMoney
```

And...  Even better?

``` active haskell
-- /show
import Control.Monad.State
import Control.Monad.Trans.List
import Control.Monad.Morph
import Data.List (foldl')

type CS a b = StateT [a] (ListT (State Int)) b

select' :: [a] -> [(a, [a])]
select' [] = []
select' (x:xs) = (x, xs) : [(y, x:ys) | (y, ys) <- select' xs]

select :: CS a a
select = do
    i <- lift . lift $ get
    xs <- get
    lift . lift . put $! i + length xs
    hoist (ListT . return) (StateT select')

runCS :: CS a b -> [a] -> ([b], Int)
runCS a xs = flip runState 0 . runListT $ evalStateT a xs

fromDigits :: [Int] -> Int
fromDigits = foldl' (\x y -> 10 * x + y) 0

-- show
sendMoreMoney :: ([(Int, Int, Int)], Int)
sendMoreMoney = flip runCS [0..9] $ do
    [d,e,y] <- replicateM 3 select
    guard $ (d + e) `mod` 10 == y
    [n,r] <- replicateM 2 select
    guard $ (fromDigits [n,d] + fromDigits [r,e]) `mod` 100 == fromDigits [e,y]
    o <- select
    guard $ (fromDigits [e,n,d] + fromDigits [o,r,e]) `mod` 1000 == fromDigits [n,e,y]
    [s,m] <- replicateM 2 select
    let send  = fromDigits [s,e,n,d]
        more  = fromDigits [m,o,r,e]
        money = fromDigits [m,o,n,e,y]
    guard $ s /= 0 && m /= 0 && send + more == money
    return (send, more, money)
-- /show
main :: IO ()
main = print sendMoreMoney
```

So yeah.
These optimizations work.
Really well, in fact.

But something's just not right.
They're mechanical, tedious, and ugly to do by hand.
And even worse, they're error-prone.
When I was putting these illustrations together, I kept getting empty result lists because I messed up pattern-matching the `replicateM`s.

Whenever there's a problem that's mechanical, tedious, and error-prone, that suggests automation.
What's the structure of the problem here?
Well, I have a couple ideas, but I haven't got them down into code just yet.
But this is getting awfully long, so it's time to just call this part 1.
I'll write part 2 as soon as I've figured out how to automate this process in an interesting way.
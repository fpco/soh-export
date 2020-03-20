# fold and accumulate

# what is this about?

Recently I played a bit with [threepenny-gui](http://hackage.haskell.org/package/threepenny-gui-0.4.0.1) and soon ran into the situation where I wanted to update a behavior (my state) using the old state and certain events.
Not having to much real functional approach to reactive programming my mind at once cried "give me some fold man" ... but alas: there is nothing that has a close shape to a fold in this library.

So I skimmed through the libraray and could really not get how I could ever get to use "old-state" together with an event ...

After a while I finally saw some "accumulate" functions in there, and as I knew their close relationship with folds (for example in .net/linq fold is named Accumulate) I knew I might have found a way out ... but wait this is the type-signature of the one I finally used:

```haskell
accumB :: MonadIO m => a -> Event (a -> a) -> m (Behavior a)
```

let's rephrase that a bit and you get basically

```haskell
accum :: acc -> [acc -> acc] -> acc
```

It's quite a bit away from
``` haskell
foldl :: (acc -> b -> acc) -> acc -> [b] -> acc
```

And I really struggled to finally see how you could use accum to get foldl.

If you like me don't see how you could possible to this read on - if you find this trivial better close the site now.

# the idea
After 20 minutes or so (searching around for other stuff, coming back, scratching my head, ...) I finally saw how to do it:

Just map the `b`s into functions `acc -> acc` and vóila: done.

# let's implement this

So let's try - here is functional version of the basic accum function:

``` active haskell
accum :: acc -> [acc -> acc] -> acc
accum acc []     = acc
accum acc (f:fs) = accum (f acc) fs

main = do
    putStrLn $ show . accum 0 $ [(+1), (+2), (+3), (+4)]
```

And of course, using `f :: acc -> b -> acc` and given a `b` we map this into `\acc -> f acc b` or `flip f b`:

``` active haskell
-- /show
accum :: acc -> [acc -> acc] -> acc
accum acc []     = acc
accum acc (f:fs) = accum (f acc) fs

-- show
fold :: (acc -> b -> acc) -> acc -> [b] -> acc
fold f acc = accum acc . map (flip f)

main = do
    putStrLn $ show . fold (+) 0 $ [1,2,3,4]
```

Seems to be correct (at least the compiler is happy and the results math).

# a bit eq. reasoning

So let's get a bit further by actually showing that this is correct. Using the definition of `accum` we see that (abusing the syntax highlighter and eq. reasoning here - so `==` is meta-equals instead of a lang. construct):

``` haskell
fold f acc [] == accum acc . map (flip f) $ []
              == accum acc []
              == acc
```

So the base-case is the same as `foldl` - check.

The non-empty list case is not much more:

``` haskell
fold f acc (b:bs) == accum acc . map (flip f) $ (b:bs)
                  == accum acc $ ((\a -> f a b) : map (flip f) bs)
                  == accum ((\a -> f a b) acc) $ map (flip f) bs
                  == accum (f acc b) $ map (flip f) bs
                  == fold f (f acc b) bs
```

That's it - qed ;)

# and vice versa?
Well this is just as simple: having a `fold` like above you can define a `accum` by:

``` active haskell
-- /show
fold :: (acc -> b -> acc) -> acc -> [b] -> acc
fold = foldl

-- show
accum :: acc -> [acc -> acc] -> acc
accum = fold (flip ($))

main = do
    putStrLn $ show . accum 0 $ [(+1), (+2), (+3), (+4)]
```
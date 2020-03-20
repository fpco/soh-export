# vectorBuilder

Often times when working with Haskell in a high performance context, the overhead introduced by a linked list representation can be too high. Having an extra constructor around each value, a constructor for each cons cell, and the indirection introduced by having to follow pointers, can completely kill performance. The most common examples of this are the high speedup you can often achieve by replacing `String` with `Text` (or something `ByteString`), or by using `Vector`s- especially unboxed or storable `Vector`s.

conduit has a similar representation of a stream as a list, including the constructor overheads just mentioned. It's not surprising, therefore, that in a situation that a list would be a poor representation, conduits will often suffer similar performance problems. Like lists, some of this overhead is mitigated by shortcut fusion (a.k.a., rewrite rules). But this isn't always the case.

conduit-combinators provides a helper function which allows us to take back performance, by working with a packed representation instead of creating a bunch of cons cells. It does this by using the vector package's generic mutable interface under the surface, while at a user-facing level providing a simple yield-like function, avoiding the need to muck around with mutable buffers.

This article will cover how to use this function, some implementation details, and comparisons to other approaches.

__NOTE__: At the time of writing, the version of conduit-combinators provided on School of Haskell does not contain the `vectorBuilder` function, and therefore the active code below will not run.

## Motivating use case

Let's start with a simple goal: we have chunks of bytes coming in, and we want to (1) duplicate each successive byte so that, e.g. [1, 2, 3] becomes [1, 1, 2, 2, 3, 3] and (2) rechunk the values into vectors of size 512. The original data could be chunked in any way, so we can rely on any specific incoming chunk size (in this case, a known 256 chunk size would be convenient).

Likely the easiest approach is to convert our stream of chunked values (e.g., `ByteString` or `Vector Word8`) into a stream of elements (e.g., `Word8`), duplicate the individual values, then chunk those back up. Such a solution would look like:

```haskell
rechunk1 = concatC
       =$= concatMapC (\x -> [x, x])
       =$= conduitVector 512
```

This uses the `concatC` combinator to "flatten out" the input stream, `concatMapC` to duplicate each individual `Word8`, and then `conduitVector` to create a stream of 512-sized `Vector`s. In my simple benchmark, this function took 13.06ms.

But as we can probably guess, this falls into the problem zone described in our introduction. So instead of dealing with things on the individual byte level, let's try to use some higher-level functions operating on `Vector`s of values instead. Our new approach will be to first `mapC` over the stream and use vector's `concatMap` to double each value, and then use `takeCE` and `foldC` to extract successive chunks of size 4096. In code:

```haskell
rechunk2 =
    mapC (concatMap $ replicate 2) =$= loop
  where
    loop = do
        x <- takeCE 512 =$= foldC
        unless (null x) $ yield x >> loop
```

In the same benchmark, this performed at 8.83ms, a 32% speedup. While respectable, we can do better.

## Buffer copying

Our first approach is optimal in one way: it avoids needless buffer copying. Each `Word8` is copied precisely once into an output `Vector` by `conduitVector`. Unfortunately, this advantage is killed by the overhead of boxing the `Word8`s and allocating constructors for conduit. Our second approach avoids the boxing and constructors by always operating on `Vector`s, but we end up copying buffers multiple times: from the original `Vector` to the doubled `Vector`, and then when folding together multiple `Vector`s into a single `Vector` of size 512.

What we want to do is to be able to `yield` a `Word8` and have it fill up an output buffer, and once that buffer is filled, `yield` that buffer downstream and start working on a new one. We could do that by directly dealing with mutable `Vector`s, but that's error-prone and tedious. Instead, let's introduce our new combinator function: `vectorBuilder` (or its unqualified name, `vectorBuilderC`).

The idea is simple. `vectorBuilder` will allocate an output buffer for you. It provides you with a special `yield`-like function that fills up this buffer, and when it's full, yields the entire buffer downstream for you.

To use it, we're going to use one other combinator function: `mapM_CE`, which performs an action for every value in a chunked input stream (in our case, for each `Word8` in our input `Vector Word8`s). Altogether, this looks like:

```haskell
rechunk3 = vectorBuilderC 512 $ \yield' ->
    mapM_CE (\x -> yield' x >> yield' x)
```

We call `yield'` twice to double our bytes. `vectorBuilder` ensures that each output buffer is of size 512. `mapM_CE` efficiently traverses the incoming `Vector`s without creating intermediate data structures.

This version benchmarks at 401.12us. This is approximately 95% faster than our previous attempt!

## Avoiding transformers

There's something tricky about the `yield'` function above. Notice how it's not being used in the `Conduit` monad transformer, but is instead living the base monad (e.g., `IO`). This is not accidental. Not only does this allow us to use existing monadic combinators like `mapM_CE`, it also allows for *far* more efficient code. To demonstrate, let's look at two different ways of doing the same thing:

```haskell
bgroup "transformers" $
    let src = return () in
    [ bench "single" $ nfIO $ do
        ref <- newIORef 0
        let incr = modifyIORef ref succ
        src $$ liftIO (replicateM_ 1000 incr)
    , bench "multi" $ nfIO $ do
        ref <- newIORef 0
        let incr = liftIO $ modifyIORef ref succ
        src $$ replicateM_ 1000 incr
    ]
```

Both of these benchmarks use no conduit features. They both create an `IORef`, then increment it 1000 times. The difference is that the first calls `liftIO` once, while the second calls `liftIO` 1000 times. Let's see the difference in benchmark results:

```
benchmarking transformers/single
mean: 4.292891 us, lb 4.285319 us, ub 4.303626 us, ci 0.950
std dev: 45.83832 ns, lb 35.04324 ns, ub 59.43617 ns, ci 0.950

benchmarking transformers/multi
mean: 93.10228 us, lb 92.95708 us, ub 93.30159 us, ci 0.950
std dev: 869.6636 ns, lb 673.8342 ns, ub 1.090044 us, ci 0.950
```

Avoiding extra `liftIO` calls has a profound performance impact. The reason for this is somewhat similar to what we've been discussing up until now about extra cons cells. In our case, it's extra `PipeM` constructors used by conduit's `MonadIO` instance. I don't want to dwell on those details too much right now, as that's a whole separate topic of analysis, involving looking at GHC core output. But let's take it as a given right now.

The question is: how does `vectorBuilder` allow you to live in the base monad, but still `yield` values downstream, which requires access to the `Conduit` transformer? There's a trick here using mutable variables. The implementation essentially works like this:

* Allocate a new, empty mutable vector.
* Allocate a mutable variable holding an empty list.
* Start running the user-supplied `Conduit` function, providing it with a specialized `yield` function.
* The specialized `yield` function- which lives in the base monad- will write values into the mutable vector. Once that mutable vector is filled, the vector is frozen and added to the end of the mutable variable's list, and a new mutable vector is allocated.
* The next time the user's function `await`s for values from upstream, we jump into action. Since we're already forced to be in the `Conduit` transformer at that point, this is our chance to `yield`. We grab all of the frozen vectors from the mutable variable and `yield` them downstream. Once that's done, we `await` for new data from upstream, and provide it to the user's function.
* When the user's function is finished, we freeze the last bit of data from the mutable vector and yield that downstream too.

The upsides of this approach are ease-of-use and performance. There *is* one downside you should be aware of: if you generate a large amount of output without `await`ing for more data from upstream, you can begin to accumulate more memory. You can force the collection of frozen `Vector`s to be flushed using the following helper function:

```haskell
forceFlush :: Monad m => ConduitM i o m ()
forceFlush = await >>= maybe (return ()) leftover
```

This simply `await`s for a value, allowing `vectorBuilder` to clear its cache, and then gives the new value back as a leftover.

Overall, your goal should be to have a decent trade-off between memory and time efficiency. To demonstrate, try playing around with the functions f1, f2, and f3 in the following code snippet:

```haskell active
{-# LANGUAGE NoImplicitPrelude #-}
{-# LANGUAGE FlexibleContexts #-}
import ClassyPrelude.Conduit

forceFlush :: Monad m => ConduitM i o m ()
forceFlush = await >>= maybe (return ()) leftover

-- Memory inefficient, time efficient
f1 :: (Int -> IO ()) -> Sink () IO ()
f1 f = liftIO $ forM_ [1..1000000] f

-- Memory efficient, time inefficient
f2 :: (Int -> Sink () IO ()) -> Sink () IO ()
f2 f = forM_ [1..1000000] $ \i -> do
    f i
    forceFlush

-- Good trade-off
f3 f = forM_ (chunksOf 10000 [1..1000000]) $ \is -> do
    liftIO $ mapM_ f is
    forceFlush
  where
    chunksOf _ [] = []
    chunksOf i x =
        y : chunksOf i z
      where
        (y, z) = splitAt i x

main = vectorBuilderC 4096 f3
    $$ (sinkNull :: Sink (Vector Int) IO ())
```

## ByteString and Vector

It may be surprising to have seen an entire article on packed representations of bytes, and not yet seen `ByteString`. As a matter of fact, the [original use case](http://www.reddit.com/r/haskell/comments/29nvsx/how_to_get_good_performance_when_processing/) I started working on this for had nothing to do with the vector package. However, I decided to focus on `vector` for two reasons:

1. Unlike bytestring, it provides a well developed mutable interface. Not only that, but the mutable interface is optimized for storable, unboxed, and generic vectors, plus existing helper packages like [hybrid-vectors](http://hackage.haskell.org/package/hybrid-vectors). In other words, this is a far more general-purpose solution.
2. It's trivial and highly efficient to convert a `ByteString` to and from a storable `Vector`.

To demonstrate that second point, let's try to read a file, duplicate all of its bytes as we did above, and write it back to a separate file. We'll use the `toByteVector` and `fromByteVector` functions, which I recently added to mono-traversable for just this purpose:

```haskell active
{-# LANGUAGE NoImplicitPrelude #-}
import           ClassyPrelude.Conduit
import           System.IO             (IOMode (ReadMode, WriteMode),
                                        withBinaryFile)

double :: (Word8 -> IO ()) -> Sink (SVector Word8) IO ()
double yield' = mapM_CE $ \w ->
    yield' w >> yield' w

main :: IO ()
main = withBinaryFile "input.txt" ReadMode $ \inH ->
       withBinaryFile "output.txt" WriteMode $ \outH ->
       sourceHandle inH
    $$ mapC toByteVector
    =$ vectorBuilderC 4096 double
    =$ mapC fromByteVector
    =$ sinkHandle outH
```

## Comparison with blaze-builder

There's a strong overlap between what `vectorBuilder` does, and how blaze-builder (and more recently, bytestring's `Builder` type) are intended to be used. I unfortunately can't give any conclusive comparisons between these two techniques right now. What I *can* say is that there are cases where using a `Builder` has proven to be inefficient, and `vectorBuilder` provides a large performance improvement. I can also say that `vectorBuilder` addresses many more use cases that `Builder`. For example, at FP Complete we're planning to use this in financial analyses for creating time series data.

On the other hand, blaze-builder and bytestring's `Builder` have both had far more real-world tuning than `vectorBuilder`. They also have support for things such as copying existing `ByteString`s into the output stream, whereas `vectorBuilder` always works by copying a single element at a time.

So for now, if you have a use case and you're uncertain whether to use `vectorBuilder` to blaze-builder, I recommend either trying both approaches, or discussing it on one of the Haskell mailing lists to get more feedback.

## Complete code

The code for most of the blog post above is below. Sorry that it's a bit messy:

```haskell active
{-# LANGUAGE FlexibleContexts          #-}
{-# LANGUAGE NoImplicitPrelude         #-}
{-# LANGUAGE NoMonomorphismRestriction #-}
import           ClassyPrelude.Conduit
import           Control.Monad.Primitive (PrimMonad)
import           Control.Monad.ST        (runST)
import           Criterion.Main          (bench, bgroup, defaultMain, nfIO,
                                          whnfIO)
import qualified Data.Vector.Generic     as VG
import qualified System.Random.MWC       as MWC
import           Test.Hspec              (hspec, shouldBe)
import           Test.Hspec.QuickCheck   (prop)

rechunk1 :: ( Monad m
            , VG.Vector vector (Element input)
            , PrimMonad base
            , MonadBase base m
            , MonoFoldable input
            )
         => Conduit input m (vector (Element input))
rechunk1 = concatC =$= concatMapC (\x -> [x, x]) =$= conduitVector 512
{-# INLINE rechunk1 #-}

rechunk2 :: (Monad m, IsSequence a) => Conduit a m a
rechunk2 =
    mapC (concatMap $ replicate 2) =$= loop
  where
    loop = do
        x <- takeCE 512 =$= foldC
        unless (null x) $ yield x >> loop
{-# INLINE rechunk2 #-}

rechunk3 :: ( MonadBase base m
            , PrimMonad base
            , MonoFoldable input
            , VG.Vector vector (Element input)
            )
         => Conduit input m (vector (Element input))
rechunk3 = vectorBuilderC 512 $ \yield' ->
    mapM_CE (\x -> yield' x >> yield' x)
{-# INLINE rechunk3 #-}

main :: IO ()
main = do
    hspec $ prop "rechunking" $ \ws -> do
        let src = yield (pack ws :: UVector Word8)
            doubled = concatMap (\w -> [w, w]) ws
            res1 = runST $ src $$ rechunk1 =$ sinkList
            res2 = runST $ src $$ rechunk2 =$ sinkList
            res3 = runST $ src $$ rechunk3 =$ sinkList
        res1 `shouldBe` (res2 :: [UVector Word8])
        (res3 :: [UVector Word8]) `shouldBe` (res2 :: [UVector Word8])
        (unpack $ (mconcat res2 :: UVector Word8)) `shouldBe` (doubled :: [Word8])
        case reverse res1 :: [UVector Word8] of
            [] -> return ()
            x:xs -> do
                (length x <= 512) `shouldBe` True
                all ((== 512) . length) xs `shouldBe` True

    gen <- MWC.createSystemRandom
    bytes <- replicateM 20 $
        MWC.uniformR (12, 1024) gen >>= MWC.uniformVector gen

    defaultMain
        [ bgroup "copy bytes"
            [ bench "rechunk1" $ whnfIO
                $ yieldMany (bytes :: [UVector Word8])
               $$ (rechunk1 :: Conduit (UVector Word8) IO (UVector Word8))
               =$ sinkNull
            , bench "rechunk2" $ whnfIO
                $ yieldMany (bytes :: [UVector Word8])
               $$ (rechunk2 :: Conduit (UVector Word8) IO (UVector Word8))
               =$ sinkNull
            , bench "rechunk3" $ whnfIO
                $ yieldMany (bytes :: [UVector Word8])
               $$ (rechunk3 :: Conduit (UVector Word8) IO (UVector Word8))
               =$ sinkNull
            ]
        , bgroup "transformers" $
            let src = return () in
            [ bench "single" $ nfIO $ do
                ref <- newIORef (0 :: Int)
                let incr = modifyIORef ref succ
                src $$ liftIO (replicateM_ 1000 incr)
            , bench "multi" $ nfIO $ do
                ref <- newIORef (0 :: Int)
                let incr = liftIO $ modifyIORef ref succ
                src $$ replicateM_ 1000 incr
            ]
        ]
```
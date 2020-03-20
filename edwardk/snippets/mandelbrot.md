# Mandelbrot

This short little snippet combines a Mandelbrot set generator using lens based on one by [N. Haas](https://github.com/nandykins) with the fold-based PNG generator from the [second part of my cellular automata series](https://www.fpcomplete.com/user/edwardk/cellular-automata/part-2). His version of the Mandelbrot function was tighter than mine, but I mixed it with the formatting logic.

```active haskell web
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE ExistentialQuantification #-}
{-# LANGUAGE Rank2Types #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TypeFamilies #-}

import Codec.Compression.Zlib
import Control.Lens
import Control.DeepSeq
import Control.Parallel.Strategies
import Data.Bits
import Data.Binary
import Data.Binary.Put
import Data.Complex
import qualified Data.ByteString.Lazy as Lazy
import qualified Data.Vector.Unboxed as Unboxed
import Data.Foldable as F
import Data.Monoid
import Yesod

-- * Mandelbrot

-- show
mandelbrot :: Int -> Int -> Int -> Lazy.ByteString
mandelbrot n w h = png w h $ \r i -> 
 maybe 0 scale $ steps $ (r/.w*3-2) :+ (i/.h*2-1)
 where 
   x /. y = fromIntegral x / fromIntegral y :: Double
   scale k = floor (k /. n * 255)
   diverges (r :+ i) = r^2 + i^2 > 4
   steps c = iterate (\z -> z^2 + c) 0 ^?
     taking n ifolded.filtered diverges.asIndex
-- /show

-- * Folds

data L b a = forall x. L (x -> b -> x) x (x -> a)

more :: Lazy.ByteString -> L Word8 a -> a
more bs (L xbx x xa) = xa (Lazy.foldl' xbx x bs)

-- * CRC32

crc32 :: L Word8 Word32
crc32 = L step 0xffffffff complement where
  step r b = unsafeShiftR r 8 `xor` crcs Unboxed.! fromIntegral (xor r (fromIntegral b) .&. 0xff)

crcs :: Unboxed.Vector Word32
crcs = Unboxed.generate 256 (go.go.go.go.go.go.go.go.fromIntegral) where
  go c = unsafeShiftR c 1 `xor` if c .&. 1 /= 0 then 0xedb88320 else 0

-- * PNG

putChunk :: Lazy.ByteString -> Lazy.ByteString -> Put
putChunk h b = do
  putWord32be $ fromIntegral (Lazy.length b)
  putLazyByteString h
  putLazyByteString b
  putWord32be $ more (h <> b) crc32

putChunks :: Lazy.ByteString -> Lazy.ByteString -> Put
putChunks h b = forM_ (Lazy.toChunks b) (putChunk h . Lazy.fromChunks . return)

png :: Int -> Int -> (Int -> Int -> Word8) -> Lazy.ByteString
png w h p = runPut $ do
  putLazyByteString "\x89PNG\r\n\x1a\n"
  putChunk "IHDR" $ runPut $ do
    putWord32be (fromIntegral w)
    putWord32be (fromIntegral h)
    putWord8 8 -- 8 bit color depth
    putWord8 0 -- greyscale
    putWord8 0
    putWord8 0
    putWord8 1 -- Adam7 interlaced
  putChunks "IDAT" $ 
    compressWith defaultCompressParams { compressLevel = bestSpeed } $ runPut $ do
      pass [0,8 ..h-1] [0,8..w-1]
      pass [0,8 ..h-1] [4,12..w-1]
      pass [4,12..h-1] [0,4..w-1]
      pass [0,4 ..h-1] [2,6..w-1]
      pass [2,6 ..h-1] [0,2..w-1]
      pass [0,2 ..h-1] [1,3..w-1]
      pass [1,3 ..h-1] [0..w-1]    
  putChunk "IEND" mempty
  where
    pass ys xs = forM_ ys $ \y -> do
      putWord8 0
      F.mapM_ put (fmap (p ?? y) xs `using` parListChunk 16 rdeepseq)


-- * Yesod

data App = App
instance Yesod App
mkYesod "App" [parseRoutes| / ImageR GET |]

-- show 
getImageR :: MonadHandler m => m TypedContent
getImageR = sendResponse
          $ toTypedContent (typePng, toContent img)
  where 
    img = mandelbrot 32 600 300
-- /show

main :: IO ()
main = warpEnv App
```

It would be fun to modify the little embedded Yesod web server to permit interactive browsing of the resulting Mandelbrot set.

### [Update: Sept 2, 2013] Just Add Parallelism

In the PNG writer, replacing

```haskell
forM_ [0..w-1] $ \x -> put (p x y)
```

with a little bit of parallelism:

```haskell
F.mapM_ put 
  (fmap (p ?? y) [0..w-1] `using` parListChunk 16 rdeepseq)
```

rather dramatically speeds up rendering when compiled and run locally with the threaded runtime. 

Unfortunately, we don't seem to have the ability to pass RTS flags here on the School of Haskell at this time.

### [Update: Sept 2, 2013] Interlacing and Incrementalization

I also took the liberty of modifying the PNG writer to support directly generating the image
using greyscale to cut down repetition, and to support multiple IDAT blocks and use Adam7 interlacing so you can see the Mandelbrot set as soon as possible.

Greyscaling just involves changing one of the constants in the PNG header.

Getting multiple `IDAT` blocks just involves replacing

```
putChunk "IDAT"
```

with

```haskell
putChunks "IDAT"
```

after we let Haskell pick for us reasonable sounding chunk boundaries:

```haskell
putChunks :: Lazy.ByteString -> Lazy.ByteString -> Put
putChunks h b = forM_ (Lazy.toChunks b) $ 
  putChunk h . Lazy.fromChunks . return
```

Switching to [Adam7](http://en.wikipedia.org/wiki/Adam7_algorithm) is only slightly more involved.

After we factor out the core loop that generates our PNG body:

```haskell
    pass ys xs = forM_ ys $ \y -> do
      putWord8 0
      F.mapM_ put (fmap (p ?? y) xs `using` parListChunk 16 rdeepseq)
```

We wind up replacing:

```haskell
  putChunks "IDAT" $ compress $ runPut $ do
      pass [0..h-1] [0..w-1]
```

with

```haskell
  putChunks "IDAT" $ compress $ runPut $ do
      pass [0,8 ..h-1] [0,8..w-1]
      pass [0,8 ..h-1] [4,12..w-1]
      pass [4,12..h-1] [0,4..w-1]
      pass [0,4 ..h-1] [2,6..w-1]
      pass [2,6 ..h-1] [0,2..w-1]
      pass [0,2 ..h-1] [1,3..w-1]
      pass [1,3 ..h-1] [0..w-1]    
```

and modifying the header to indicate we want it to be interlaced.

This spits out the data in 7 passes refining 8x8 blocks as follows:

```
1 6 4 6 2 6 4 6
7 7 7 7 7 7 7 7
5 6 5 6 5 6 5 6
7 7 7 7 7 7 7 7
3 6 4 6 3 6 4 6
7 7 7 7 7 7 7 7
5 6 5 6 5 6 5 6
7 7 7 7 7 7 7 7
```

Done!

-[Edward Kmett](mailto:ekmett@gmail.com)

September 2, 2013
# Multiple output files with Conduit

```haskell active
{-# LANGUAGE NoImplicitPrelude #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE ViewPatterns      #-}
import           ClassyPrelude.Conduit
import           Safe                  (atMay)

idxSinkFile :: MonadResource m
            => (Int -> FilePath)
            -> Int
            -> Consumer [ByteString] m ()
idxSinkFile mkFP idx =
    concatMapC (flip atMay idx) =$= sinkFile fp
  where
    fp = mkFP idx

sinkMultiFiles :: MonadResource m
               => (Int -> FilePath)
               -> [Int]
               -> Sink [ByteString] m ()
sinkMultiFiles mkFP indices = getZipSink $ otraverse_ (ZipSink . idxSinkFile mkFP) indices

someFunc :: ByteString -> [ByteString]
someFunc (decodeUtf8 -> x) = map encodeUtf8 [x, toUpper x, toLower x]

mkFP :: Int -> FilePath
mkFP 0 = "file0.txt"
mkFP 1 = "file1.txt"
mkFP 2 = "file2.txt"

src :: Monad m => Producer m ByteString
src = yieldMany $ map encodeUtf8 $ words "Hello There World!"

main :: IO ()
main = do
    let indices = [0..2]
    runResourceT $ src $$ mapC someFunc =$ sinkMultiFiles mkFP indices
    forM_ indices $ \idx -> do
        let fp = mkFP idx
        bs <- readFile fp
        print (fp, bs :: ByteString)
```
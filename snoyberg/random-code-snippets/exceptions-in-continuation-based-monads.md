# Exceptions in continuation-based monads

```haskell active
{-# LANGUAGE PackageImports #-}
import           "MonadCatchIO-transformers" Control.Monad.CatchIO  (MonadCatchIO,
                                                                     bracket)
import           Control.Monad.IO.Class (MonadIO, liftIO)
import           Data.ByteString        (ByteString, hPut)
import           Data.Enumerator        (Iteratee, run, ($$))
import           Data.Enumerator.Binary (enumFile, iterHandle)
import           Snap.Iteratee          () -- orphan instance
import qualified System.IO              as IO

iterFile :: (MonadCatchIO m, MonadIO m, Functor m)
         => FilePath -> Iteratee ByteString m ()
iterFile fp = bracket
    (liftIO $ do
        putStrLn $ "opening file for writing: " ++ fp
        IO.openFile fp IO.WriteMode)
    (\h -> liftIO $ do
        putStrLn $ "closing file for writing: " ++ fp
        IO.hClose h)
    iterHandle

main :: IO ()
main = do
    writeFile "exists.txt" "this file exists"
    run (enumFile "exists.txt" $$ iterFile "output1.txt") >>= print
    run (enumFile "does-not-exist.txt" $$ iterFile "output2.txt") >>= print
```
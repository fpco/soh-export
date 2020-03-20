# conduit without ResourceT

```haskell active
{-# START_FILE main.hs #-}
-- show
import Data.Conduit (($$))
import Data.Conduit.Binary (sourceHandle, sinkHandle)
import System.IO (withBinaryFile, IOMode (..))

main = do
    withBinaryFile "input.txt" ReadMode $ \input ->
      withBinaryFile "output.txt" WriteMode $ \output ->
      sourceHandle input $$ sinkHandle output
    readFile "output.txt" >>= putStrLn
{-# START_FILE input.txt #-}
-- show
This is the input file.
```
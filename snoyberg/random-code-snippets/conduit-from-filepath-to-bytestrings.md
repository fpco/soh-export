# Conduit from FilePath to ByteStrings

```haskell active
{-# START_FILE file1.txt #-}
-- show
file1 contents
{-# START_FILE file2.txt #-}
-- show
file3 contents
{-# START_FILE file3.txt #-}
-- show
file3 contents
{-# START_FILE main.hs #-}
-- show
import Data.Conduit
import Data.Conduit.Binary (sourceFile)
import qualified Data.Conduit.List as CL
import Control.Monad.IO.Class (liftIO)
import Data.ByteString (ByteString)

src = mapM_ yield ["file1.txt", "file2.txt", "file3.txt"]

main = runResourceT $ src $$ fileConduit =$ CL.mapM_ (liftIO . print)

fileConduit :: MonadResource m => Conduit FilePath m ByteString
fileConduit = awaitForever sourceFile
```
# traverseTree

```haskell active
{-# START_FILE foo/bar/baz.txt #-}
-- show
This is a file
{-# START_FILE main.hs #-}
-- show
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE OverloadedStrings #-}
import Data.FileEmbed
import Data.Conduit
import Data.Conduit.Filesystem
import qualified Data.Conduit.List as CL
import Control.Monad.IO.Class (liftIO)

main = do
    print $(embedDir "foo")
    runResourceT
       $ traverse False "foo"
      $$ awaitForever sourceFile
      =$ CL.mapM_ (liftIO . print)
```
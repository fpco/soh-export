# Unexpected exit: stderr and exit code

We want to report about some problem via message to `stderr` and after that we have to exit with some not-null code. 

### Solution

Let's check a file existence. We have to install [`directory`](http://hackage.haskell.org/package/directory) package, and after that:


```haskell
module Main where

import System.Directory (doesFileExist)
import System.IO (hPutStrLn, stderr)
import System.Exit

main :: IO ()
main = do
    fileIsHere <- doesFileExist "/Users/dshevchenko/ed" -- No such file...
    if fileIsHere
    then 
        putStrLn "OK"
    else do 
        hPutStrLn stderr "No such file, sorry..."
        exitWith $ ExitFailure 1
```

Result will be:

```bash
No such file, sorry...

shell returned 1
```

That's it.
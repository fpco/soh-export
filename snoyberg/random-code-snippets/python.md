# Python

```haskell active
{-# START_FILE test.py #-}
print "Hello from Python!"
{-# START_FILE main.hs #-}
import System.Cmd
main = do
    putStrLn "Hello from Haskell!"
    rawSystem "python" ["test.py"]
```
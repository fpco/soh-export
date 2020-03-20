# Bad document locations

```haskell active
{-# LANGUAGE OverloadedStrings #-}
import Filesystem

main = Filesystem.readFile "foo" >>= putStrLn
```
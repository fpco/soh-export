# Call a system command

We want to call some system command right in our Haskell code.

### Solution

We have to install [`process`](http://hackage.haskell.org/package/process) package. Now:

```haskell
module Main where

import System.Process 

main :: IO ()
main = system "ls -al" >>= \exitCode -> print exitCode
```

Result will be similar to:

```bash
drwxr-xr-x  5 dshevchenko  staff   170  2 Jan 23:25 .
drwxr-xr-x  8 dshevchenko  staff   272 14 Jan 13:22 ..
-rw-r--r--@ 1 dshevchenko  staff  6148 29 Mar 23:13 .DS_Store
-rw-r--r--  1 dshevchenko  staff   869 12 Maj 17:16 Main.hs
drwxr-xr-x  6 dshevchenko  staff   204 29 Mar 23:14 Utils
ExitSuccess
```

Done.

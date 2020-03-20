# Raw string

We have a text string with a *non-standard* characters. To avoid possible problems with formatting and, in addition, to avoid frequent usage of backslash we will use a raw string. Each symbol in the raw string is a symbol *itself*.

### Solution

We have to install [`raw-strings-qq`](http://hackage.haskell.org/package/raw-strings-qq). Now:

```haskell
{-# LANGUAGE QuasiQuotes #-}

module Main where

import Text.RawString.QQ

helpInfo :: String
helpInfo = [r|

This is raw string, "strange
  -- Very strange
  {-
  '"Yes, very strange\ string"
  -}

But in some cases 
                   it's 
            very convenient...
  |]

main :: IO ()
main = putStr helpInfo
```

And this is our strange result:

```bash
This is raw string, "strange
  -- Very strange
  {-
  '"Yes, very strange\ string"
  -}

But in some cases
                   it's
            very convenient...
```

Done.
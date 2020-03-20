# Snap and Happstack

Snap and Happstack- together with a simple helper library- are now available on the School of Haskell.

## Snap

```haskell active web
{-# LANGUAGE OverloadedStrings #-}
module Main where

import Snap.Http.Server.Env
import Snap.Core

main :: IO ()
main = httpServe defaultConfig $ writeBS "Hello from Snap!"
```

## Happstack

```haskell active web
module Main where

import Happstack.Server.Env

main = simpleHTTP nullConf $ ok "Hello from Happstack"
```
# IO One liner

Arg passed along. No variable in need:


``` active haskell

module Main where

main = putStrLn "Name?" >> getLine >>= putStrLn . ((++) "Hi ")

```

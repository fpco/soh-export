# IO Right to Left, or Left to Right?

Typically the monadic IO goes like this:

``` active haskell
main :: IO ()
main0a = putStrLn =<< getContents
```

If you prefier, an opposite is also possible:


``` active haskell
main :: IO ()
main = getContents >>= putStrLn
```

Choice is yours.
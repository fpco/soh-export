# Learn you a Haskell for Great Good - Ch.4 - snippet 01

``` active haskell
lucky :: (Integral a) => a -> String   
lucky 7 = "LUCKY NUMBER SEVEN!"   
lucky x = "Sorry, you're out of luck, pal!"  
```
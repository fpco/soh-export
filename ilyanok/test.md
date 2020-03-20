# test

``` haskell active
fact 0 = 1
fact x = x * fact (x-1)

main = do
         putStr "Enter a number to be factorized: "
         x <- getLine
         putStrLn $ "The answer is " ++ (show . fact . read) x
```

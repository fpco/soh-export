# Emoji

While it's not possible to use emoji as regular names in Haskell

``` active haskell
{-# START_FILE Main.hs #-}
💩 = "emoji"

main = print emoji
```

They can be used as infix function names

``` active haskell
{-# START_FILE Main.hs #-}
(🍆) = (+)

main = print $ 3 🍆 4
                          
```



``` active haskell
{-# START_FILE Main.hs #-}
infixr 5 👉

(👉) = (:)
(👌) = []

main = putStrLn $ 'e' 👉 'm' 👉 'o' 👉 'j' 👉 'i' 👉 (👌)
                          
```

Can you guess what the ✌ and 📩 functions do? 
          

``` active haskell
{-# START_FILE Main.hs #-}

import Data.Time.Clock.POSIX

getTime = round `fmap` getPOSIXTime

(📩) :: Functor f => (a -> b) -> f a -> f b
(📩) = fmap
(✌) = (* 2)

-- show

main = do
    print $ (✌) 100
    print $ (✌) 📩 [1..5]
    print $ (✌) 📩 Just 10

    time <- getTime
    print time

    punTime <- (✌) 📩 getTime
    print punTime

-- /show
                                                         
```
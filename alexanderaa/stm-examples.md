# STM examples

## Software Transactional Memory examples


[Documentation for STM](https://hackage.haskell.org/package/stm-2.4.4/docs/Control-Monad-STM.html)

[Documentation - TMVar](https://hackage.haskell.org/package/stm-2.4.4/docs/Control-Concurrent-STM-TMVar.html)

[A presentation on Haskell and STM in particular](http://research.microsoft.com/en-us/um/people/simonpj/papers/stm/STMTokyoApr10.pdf)



## Example 1


```active haskell
module Main where

import qualified Control.Concurrent        as CC
import qualified Control.Concurrent.STM    as T
import qualified Control.Monad             as CM

type Account = T.TMVar Float

openAccount :: Float -> T.STM (Account)
openAccount balance = T.newTMVar balance

getBalance :: Account -> T.STM Float
getBalance account = do
    balance <- T.takeTMVar account
    return balance

main :: IO ()
main = do
    accountA <- T.atomically (openAccount 20)
    balanceA <- T.atomically $ getBalance accountA
    print $ balanceA
```


## Example 2


```active haskell
module Main where

import qualified Control.Concurrent.STM    as T

type Account = T.TMVar Float

openAccount :: Float -> T.STM (Account)
openAccount balance = T.newTMVar balance

transfer :: Account -> Account -> Float -> T.STM Float
transfer accountA accountB amount = do
    startingBalanceA <- T.takeTMVar accountA
    startingBalanceB <- T.takeTMVar accountB

    let finalBalanceA = (startingBalanceA - amount)
    let finalBalanceB = (startingBalanceB + amount)

    T.putTMVar accountA finalBalanceA
    T.putTMVar accountB finalBalanceB

    return $ amount

main :: IO ()
main = do
    accountA <- T.atomically (openAccount 20)
    accountB <- T.atomically (openAccount 50)
    amt <- T.atomically (transfer accountA accountB 30)
    print $ amt
```


## Example 3


```active haskell
module Main where


import qualified Control.Concurrent        as CC
import qualified Control.Concurrent.STM    as T
import qualified Control.Monad             as CM

type Account = T.TMVar Float
    
openAccount :: Float -> T.STM (Account)
openAccount balance = T.newTMVar balance

getBalance :: Account -> T.STM Float
getBalance account = do
    balance <- T.takeTMVar account
    return balance

transfer :: Account -> Account -> Float -> T.STM ()
transfer accountA accountB amount = do
    startingBalanceA <- T.takeTMVar accountA
    startingBalanceB <- T.takeTMVar accountB

    let finalBalanceA = (startingBalanceA - amount)
    let finalBalanceB = (startingBalanceB + amount)

    T.check (finalBalanceA >= 0)

    T.putTMVar accountA finalBalanceA
    T.putTMVar accountB finalBalanceB

takeFee :: Account -> Float -> T.STM ()
takeFee account fee = do
    startingBalance <- T.takeTMVar account
    let finalBalance = (startingBalance - fee)
    T.putTMVar account finalBalance

transferOrFee :: Account -> Account -> Float -> Float -> T.STM ()
transferOrFee accountA accountB amount dishonourFee =
    T.orElse (transfer accountA accountB amount) (takeFee accountA dishonourFee)

transferOrFee2 :: Account -> Account -> Float -> Float -> T.STM ()
transferOrFee2 accountA accountB amount dishonourFee =
    T.orElse (
        do
            T.check (amount >= 0)
            transferOrFee accountA accountB amount dishonourFee
    ) (return ())

main :: IO ()
main = do
    accountA <- T.atomically (openAccount 20)
    accountB <- T.atomically (openAccount 70)
    _ <- T.atomically (transferOrFee2 accountA accountB 30 5)
    balA <- T.atomically $ getBalance accountA
    print $ balA
    balB <- T.atomically $ getBalance accountB
    print $ balB

```


# Basic functions

Replace `fixMe` with the appropriate value.

```active haskell
--/show
import Test.HUnit
check p = do
  assertBool "Sorry, try again!" p
  putStrLn "OK"
--show
result = fixMe

main = check (2 + 2 == result)
```

```active haskell
--/show
import Test.HUnit
check p = do
  assertBool "Sorry, try again!" p
  putStrLn "OK"
--show
result = fixMe

main = check (False || result)
```

```active haskell
--/show
import Test.HUnit
check p = do
  assertBool "Sorry, try again!" p
  putStrLn "OK"
--show
result = fixMe

main = check (True && result)
```

```active haskell
--/show
import Test.HUnit
check p = do
  assertBool "Sorry, try again!" p
  putStrLn "OK"
--show
result = fixMe

main = check (read "1377" == result)
```
# pretty-time

This demonstrates how we recommend documenting your projects. Note that this library isn't actually available from Hackage, and the code below will not actually run.

---

Often times you will want to display dates and times to end users. The time package provides very powerful formating abilities, but in many cases you will want to just select from some simple options. The purpose of this library is to make this possible.

Let's suppose you want to display the current date to a user. To do so, you would write:

```haskell active
import Data.Time
import Data.Time.Pretty

main :: IO ()
main = do
    now <- getCurrentTime
    putStrLn $ renderDay now
```

To display the current time in 12-hour format, you could use:

```haskell active
import Data.Time
import Data.Time.Pretty

main :: IO ()
main = do
    now <- getCurrentTime
    putStrLn $ renderTime12 now
```

__Exercise__

Write a function that would render the given value to display both date and 12-hour time.

```haskell active
import Data.Time
import Data.Time.Pretty

main :: IO ()
main = do
    now <- getCurrentTime
    putStrLn $ renderDateTime12 now
    
renderDateTime12 :: UTCTime -> String
renderDateTime12 = ...
```

@@@
```haskell active
import Data.Time
import Data.Time.Pretty

main :: IO ()
main = do
    now <- getCurrentTime
    putStrLn $ renderDateTime12 now
    
renderDateTime12 :: UTCTime -> String
renderDateTime12 x = concat
    [ renderDay x
    , " at "
    , renderTime12 x
    ]
```
@@@
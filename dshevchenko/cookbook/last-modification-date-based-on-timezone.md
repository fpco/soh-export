# Last modification date based on timezone

So, we have to find out the last modification date of the file and display this date based on the current timezone. Timezone name must be specified in the standard three-letter form.

### Solution


We have to install packages [`directory`](http://hackage.haskell.org/package/directory) and [`time`](http://hackage.haskell.org/package/time), and after that:


```haskell
module Main where

import System.Directory (getModificationTime)
import Data.Time.LocalTime (getCurrentTimeZone,
                            utcToLocalTime,
                            timeZoneName)

main :: IO ()
main = do
    utcTime <- getModificationTime "/Users/dshevchenko/my.conf"
    zone <- getCurrentTimeZone
    let localTime = utcToLocalTime zone utcTime
    putStrLn $ show localTime ++ " " ++ timeZoneName zone

```

Result is something like this:

```bash
2014-05-18 21:53:39 MSK
```

That's it.
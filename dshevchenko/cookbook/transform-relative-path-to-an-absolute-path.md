# Transform relative path to an absolute path

So, we have a path in the Unix-like OS, in one from these forms:

```bash
path/to/something
```
```bash
./../../path/to/something
```
```bash
~/path/to/something
```

And we need to convert it to an absolute path.

### Solution

We have to install these packages: [`MissingH`](http://hackage.haskell.org/package/MissingH), [`filepath`](http://hackage.haskell.org/package/filepath) and [`directory`](http://hackage.haskell.org/package/directory). After that:


```haskell
module Main where

import System.Path.NameManip (guess_dotdot, absolute_path)
import System.FilePath (addTrailingPathSeparator, normalise)
import System.Directory (getHomeDirectory)
import Data.Maybe (fromJust)
import Data.List (isPrefixOf)

absolutize :: String -> IO String
absolutize aPath 
    | "~" `isPrefixOf` aPath = do
        homePath <- getHomeDirectory
        return $ normalise $ addTrailingPathSeparator homePath 
                             ++ tail aPath
    | otherwise = do
        pathMaybeWithDots <- absolute_path aPath
        return $ fromJust $ guess_dotdot pathMaybeWithDots

main :: IO ()
main = do
    fromRelative <- absolutize "blog"
    fromDotted <- absolutize "./../../../blog"
    fromTildeBased <- absolutize "~/blog"
    putStrLn $ concat ["First path:  ", fromRelative, "\n",
                       "Second path: ", fromDotted, "\n",
                       "Third path:  ", fromTildeBased]
```

This is a result on my Mac:

```bash
First path:  /Users/dshevchenko/Profession/Learn/Real/src/blog
Second path: /Users/dshevchenko/Profession/blog
Third path:  /Users/dshevchenko/blog   
```

That's all.
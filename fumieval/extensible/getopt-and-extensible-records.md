# GetOpt and extensible records

In the [base](http://hackage.haskell.org/package/base) package, there's a module called [System.Console.GetOpt](http://hackage.haskell.org/package/base-4.10.0.0/docs/System-Console-GetOpt.html). It offers a simple command-line option parser. Here's how this is typically used.

```haskell
{-# LANGUAGE LambdaCase #-}

import System.Console.GetOpt
import System.Environment
import System.Exit

data Options = Options
  { verbose :: Bool
  , extra :: Maybe String
  }

defaultOptions :: Options
defaultOptions = Options
  { verbose = False
  , extra = Nothing
  }

options :: [OptDescr (Options -> Options)]
options = [Option "v" ["verbose"] (NoArg $ \o -> o { verbose = True }) "verbose output"
  , Option "e" ["extra"] (ReqArg (\e o -> o { extra = Just e }) "ARG") "extra argument"]

main = getOpt Permute options <$> getArgs >>= \case
  (fs, _, []) -> do
    let o = foldl (flip id) defaultOptions fs
    putStrLn $ "verbose: " ++ show (verbose o)
    putStrLn $ "extra: " ++ show (extra o)
  (_, _, es) -> do
    name <- getProgName
    die $ unlines es ++ usageInfo name options
```

Not too bad. However you need to write 3 things for each option:

* The type of the option
* The default value for the option
* A record updater

Also `foldl (flip id)` and the code for printing errors are annoying pieces of boilerplate.

In the [latest version of extensible](http://hackage.haskell.org/package/extensible-0.4.6), I added a new module [Data.Extensible.GetOpt](http://hackage.haskell.org/package/extensible-0.4.6/docs/Data-Extensible-GetOpt.html) to get things easier. This is just a wrapper of `System.Console.GetOpt` which returns an extensible record instead of a list of `OptDescr`s.

```haskell
-- | Option without an argument; the result is the total count of this option.
optNoArg :: [Char] -- ^ short option
    -> [String] -- ^ long option
    -> String -- ^ explanation
    -> OptDescr' Int

-- | Option with an argument
optReqArg :: [Char] -- ^ short option
    -> [String] -- ^ long option
    -> String -- ^ placeholder
    -> String -- ^ explanation
    -> OptDescr' [String]
```

A set of options is expressed as an extensible record. Each field is either `optNoArg` or `optReqArg`. `Int` means the total count of option occurrences and `[String]` is the list of arguments for the option.

```haskell
opts :: RecordOf OptDescr' ["verbose" >: Int, "extra" >: [String]]
opts = #verbose @= optNoArg "v" ["verbose"] "verbose"
  <: #extra @= optReqArg "e" ["extra"] "ARG" "extra arguments"
  <: nil
```

`withGetOpt` does what you'd expect; when something is wrong, it writes the errors and the usage to stderr and dies. Otherwise it passes the record of option arguments and the remainder to the function.

```haskell
withGetOpt :: MonadIO m => RecordOf OptDescr' xs
  -> (Record xs -> [String] -> m a) -> m a
```

Putting it all together, we get 13 lines of code. Much tidier!

```haskell
{-# LANGUAGE OverloadedLabels #-}
import Control.Lens
import Data.Extensible
import Data.Extensible.GetOpt

main :: IO ()
main = withGetOpt opts $ \r _args -> do
  putStrLn $ "verbose: " ++ show (r ^. #verbose > 0)
  putStrLn $ "extra: " ++ show (r ^? #extra . folded)
  where
    opts = #verbose @= optNoArg "v" ["verbose"] "verbose"
      <: #extra @= optReqArg "e" ["extra"] "ARG" "extra arguments"
      <: nil
```

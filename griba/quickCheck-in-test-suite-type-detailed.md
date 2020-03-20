# QuickCheck in Test-suite type "detailed"

Cabal 1.20 let you setup  [cabal Test-suite's](https://www.haskell.org/cabal/users-guide/developing-packages.html#test-suites) type **detailed** reflecting individual tests results.

This was problematic with previous versions forcing the use of type *exitcode-stdio* without the failed/success summary message offered in the *detailed* Test-suite type.

But the use of QuickCheck with the *detailed* Test-suite is not straight, since you have to adapt the QuickCheck result type to the *Distribution.TestSuite* result type.

So here is a module template, adapting the QuickCheck result in the Distribution.TestSuite.TestInstance runnable :

```haskell
{-# LANGUAGE PackageImports, NamedFieldPuns #-}
module TestDetailed (tests) where

import qualified Test.QuickCheck as Q
import Distribution.TestSuite as TS

import MyTestableProperties (propCheckP0, propCheckP1)

toTSResult :: Q.Result -> TS.Result
toTSResult Q.Success {} = TS.Pass
toTSResult Q.GaveUp {} = TS.Fail "GaveUp"
toTSResult Q.Failure {Q.reason} = TS.Fail reason

runQuickCheck :: Q.Testable p => p -> IO TS.Progress
runQuickCheck prop = do
        qres <- Q.quickCheckWithResult Q.stdArgs {Q.maxSuccess = 30, 
                                                  Q.maxSize = 20} prop
        return $ (Finished . toTSResult) qres

tests :: IO [Test]

tests = return [Test $ TestInstance (runQuickCheck propCheckP0) 
                                    "propCheckP0" ["tag"] [] undefined,
                Test $ TestInstance (runQuickCheck propCheckP1) 
                                    "propCheckP1" ["tag"] [] undefined
                -- ...
                ]

```

The actual cabal version (1.20) complaints about *detailed-1.0* showing in the message that only *detailed-0.9* is available.

And here is the .cabal Test-suite part:

```
Test-Suite test-my-library
  type:                 detailed-0.9
  Hs-Source-Dirs:       test
  test-module:          TestDetailed
  other-modules:        MyTestableProperties
  Default-Language:     Haskell2010
  Build-Depends:        base, Cabal >= 1.20.0
                      , QuickCheck >= 2.4
                      , random >= 1.0.1
                      , my-library

```


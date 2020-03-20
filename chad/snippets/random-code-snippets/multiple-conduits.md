# Multiple Conduits

The following demonstrates how you can run the same `Conduit` multiple times. Triggered by a [mailing list discussion](http://www.haskell.org/pipermail/haskell-cafe/2013-March/106750.html)

```haskell active
import Data.Conduit
import qualified Data.Conduit.List as CL

accumer :: Monad m => Conduit Int m Int
accumer =
    loop 0
  where
    loop accum =
        await >>= maybe (return ()) (go accum)
    go accum i = do
        let accum' = accum + i
        yield accum'
        loop accum'

breaker :: Monad m => Conduit Int m Int -> Conduit Int m Int
breaker inner = do
    mx <- CL.peek
    case mx of
        Nothing -> return ()
        Just _ -> do
            go =$= inner
            breaker inner
  where
    go = await >>= maybe (return ()) check
    check i
        | i `mod` 5 == 0 = yield i
        | otherwise = yield i >> go

main = mapM_ yield [1..20] $$ breaker accumer =$ CL.mapM_ print
```
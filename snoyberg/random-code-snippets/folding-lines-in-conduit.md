# Folding lines in conduit

```haskell active
{-# START_FILE input.txt #-}
This is the first line
Here's the second
Don't forget the third
Goodbye!
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}
import           Control.Monad       (void)
import           Data.Conduit
import qualified Data.Conduit.Binary as CB
import qualified Data.Conduit.List   as CL
import qualified Data.Conduit.Text   as CT
import           Data.Text           (Text)
import qualified Data.Text           as T
import           System.IO           (stdout)

-- show
-- Ignoring CRs for simplicity, easy to add support.
foldLines :: Monad m
          => (a -> ConduitM Text o m a)
          -> a
          -> ConduitM Text o m a
foldLines f =
    start
  where
    start a = CL.peek >>= maybe (return a) (const $ loop $ f a)

    loop consumer = do
        a <- takeWhileText (/= '\n') =$= do
            a <- consumer
            CL.sinkNull
            return a
        dropText 1
        start a

-- Here's the userland piece of the code
type LineNumber = Int
type CharCount = Int
data LineStat = LineStat !LineNumber !CharCount

myFunc :: Monad m => LineNumber -> ConduitM Text LineStat m LineNumber
myFunc number' = do
    count <- CL.fold (\count t -> count + T.length t) 0
    yield $ LineStat number count
    return number
  where
    number = number' + 1

showLineStat :: LineStat -> Text
showLineStat (LineStat number count) = T.concat
    [ "Line number "
    , T.pack $ show number
    , " has character count of: "
    , T.pack $ show count
    ]

main :: IO ()
main = runResourceT
     $ CB.sourceFile "input.txt"
    $$ CT.decode CT.utf8
    =$ void (foldLines myFunc 0)
    =$ CL.map showLineStat
    =$ unlinesText
    =$ CT.encode CT.utf8
    =$ CB.sinkHandle stdout
-- /show

-- Helper functions that should really be defined in Data.Conduit.Text
unlinesText :: Monad m => Conduit Text m Text
unlinesText = awaitForever $ \x -> yield x >> yield "\n"

takeWhileText :: Monad m
              => (Char -> Bool)
              -> Conduit Text m Text
takeWhileText p =
    loop
  where
    loop = await >>= maybe (return ()) go
    go t =
        case T.span p t of
            (x, y)
                | T.null y -> yield x >> loop
                | otherwise -> yield x >> leftover y

dropText :: Monad m => Int -> Consumer Text m ()
dropText =
    loop
  where
    loop i = await >>= maybe (return ()) (go i)
    go i t
        | diff == 0 = return ()
        | diff < 0 = leftover $ T.drop i t
        | otherwise = loop diff
      where
        diff = i - T.length t
```
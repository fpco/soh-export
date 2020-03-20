# Simple conduit parser

```haskell active
{-# LANGUAGE DeriveDataTypeable #-}
{-# LANGUAGE OverloadedStrings  #-}

import           Control.Exception         (Exception)
import           Control.Monad.Trans.Class
import           Data.Conduit
import qualified Data.Conduit.List         as CL
import qualified Data.Text                 as T
import           Data.Typeable             (Typeable)

-- show
parser :: MonadThrow m
       => (A -> m res1)
       -> (res1 -> Sink B m res2)
       -> (res2 -> C -> m res3)
       -> (res3 -> Sink D m res4)
       -> Sink T.Text m res4
parser getA sinkB getC sinkD = do
    res1 <- parseA >>= lift . getA
    res2 <- parseBs =$ consumeAll (sinkB res1)
    res3 <- parseC >>= lift . getC res2
    parseDs =$ consumeAll (sinkD res3)
  where
    consumeAll sink = do
        res <- sink
        CL.sinkNull
        return res

sink :: Sink T.Text IO Int
sink = parser
    (\A -> putStrLn "Got an A" >> return 1)
    (\i -> CL.mapM_ (\B -> putStrLn "Got a B") >> return (i + 1))
    (\i C -> putStrLn "Got a C" >> return (i + 1))
    (\i -> CL.mapM_ (\D -> putStrLn "Got a D") >> return (i + 1))
-- /show

data A = A
data B = B
data C = C
data D = D

data Exc = EndOfInput | UnexpectedChar Char
    deriving (Show, Typeable)
instance Exception Exc

parse1 :: MonadThrow m => (Char -> Maybe a) -> Sink T.Text m a
parse1 pred =
    await >>= maybe (monadThrow EndOfInput) go
  where
    go t =
        case T.uncons t of
            Nothing -> parse1 pred
            Just (c, t') -> do
                leftover t'
                case pred c of
                    Nothing -> monadThrow $ UnexpectedChar c
                    Just x -> return x

parseMany :: MonadThrow m => (Char -> Maybe a) -> Conduit T.Text m a
parseMany pred =
    await >>= maybe (return ()) go
  where
    go t =
        case T.uncons t of
            Nothing -> parseMany pred
            Just (c, t') -> do
                case pred c of
                    Nothing -> leftover t
                    Just x -> yield x >> go t'

parseA :: MonadThrow m => Sink T.Text m A
parseA = parse1 (\c -> if c == 'a' then Just A else Nothing)

parseC :: MonadThrow m => Sink T.Text m C
parseC = parse1 (\c -> if c == 'c' then Just C else Nothing)

parseBs :: MonadThrow m => Conduit T.Text m B
parseBs = parseMany (\c -> if c == 'b' then Just B else Nothing)

parseDs :: MonadThrow m => Conduit T.Text m D
parseDs = parseMany (\c -> if c == 'd' then Just D else Nothing)

main :: IO ()
main = do
    (yield "abbbbbcddd" $$ sink) >>= print
    (yield "acddd" $$ sink) >>= print
    (yield "bbbcddd" $$ sink) >>= print
```
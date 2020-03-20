# Number classifier

```active haskell
import Data.Functor ((<$))
import Control.Applicative ((<*), (*>), (<|>), many, optional, some)
import Text.Parsec (char, digit, eof, hexDigit, octDigit, parse, satisfy)

($>) = flip (<$)

-- show
data NumberType = Hexadecimal | Octal | Binary | Int | Float deriving (Eq, Show)

classifyString =
  sign *> (  char '0' *> (  char 'x' *> some hexDigit *> eof $> Hexadecimal
                        <|> char 'b' *> some binDigit *> eof $> Binary
                        <|>             some octDigit *> eof $> Octal
                        <|>                     float *> eof $> Float
                        <|>                              eof $> Int)
         <|> some digit              *> (       float *> eof $> Float
                                       <|>               eof $> Int))

  where

    float    =  char '.' *> many digit <* optional expPart
            <|> expPart

    expPart  =  char 'e' *> sign *> some digit

    sign     =  optional (char '-')

    binDigit =  satisfy (`elem` "01")

main = do
  runTests
  putStrLn "input a number:"
  line <- getLine
  case parse classifyString "" line of
    Left  _          -> putStrLn "invalid number"
    Right numberType -> print numberType
-- /show

runTests = do
  check "0xabc"    (Just Hexadecimal)
  check "-0x123ef" (Just Hexadecimal)
  check "0123"     (Just Octal)
  check "-0567"    (Just Octal)
  check "0b010"    (Just Binary)
  check "-0b1101"  (Just Binary)
  check "123"      (Just Int)
  check "-456"     (Just Int)
  check "12.2"     (Just Float)
  check "-14.56"   (Just Float)
  check "2e10"     (Just Float)
  check "-2.34e-5" (Just Float)
  check "2.4e5.6"  Nothing
  check "0xxy"     Nothing
  check "0789"     Nothing
  check "0b123"    Nothing
  check "a12"      Nothing
  check "123:4"    Nothing
  where check  s x | classify s == x = return ()
                   | otherwise       = ioError . userError $ s ++
                                       " returned " ++ show (classify s) ++
                                       " instead of " ++ show x
        classify s = case parse classifyString "" s of
          Left  _ -> Nothing
          Right t -> Just t
```

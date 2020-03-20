# Scheme in 48 hours

this is a tutorial of writing scheme in haskell

``` active haskell
 
module Main where 

import Data.Complex
import Data.Ratio
import Numeric 
import Control.Monad
import System.Environment
import Text.Parsec hiding (spaces)
import Text.Parsec.String 

main :: IO ()
main = do args <- getLine
          putStrLn (readExpr ([args] !! 0))

symbol :: Parser Char
symbol = oneOf "!$%&|*+-/:<=>?@^_~"

readExpr :: String -> String
readExpr input = case parse parseExpr "lisp" input of
    Left err -> "No match: " ++ show err
    Right val -> "Found value"

spaces :: Parser ()
spaces = skipMany1 space

toDouble :: LispVal -> Double
toDouble(Float f) = f
toDouble(Number n) = fromIntegral n 

 
data LispVal = Atom String
             | List [LispVal]
             | DottedList [LispVal] LispVal
             | Number Integer
             | String String
             | Bool Bool
             | Character Char
             | Float Double
             | Ratio Rational
             | Complex (Complex Double)


escapedChars :: Parser Char
escapedChars = do char '\\' -- a backslash
                  x <- oneOf "\\\"nrt" -- either backslash or doublequote or n,r, or t
                  return $ case x of -- return the escaped character
                    '\\' -> x
                    '"'  -> x
                    'n'  ->'\n'
                    'r'  ->'\r'
                    't'  ->'\t'

parseCharacter :: Parser LispVal 
parseCharacter = do try $ string "#\\"
                    value <- try (string "newline" <|> string "space") <|> do { x <- anyChar; notFollowedBy alphaNum ; return [x] }
                    return $ Character $ case value of
                      "space" -> ' '
                      "newline" -> '\n'
                      otherwise -> (value !! 0)


parseBool :: Parser LispVal 
parseBool = do char '#'
               (char 't' >> return (Bool True)) <|> (char 'f' >> return (Bool False))

parseString :: Parser LispVal
parseString = do char '"'
                 x <- many $ escapedChars <|> noneOf ['\34', '\92'] -- double-quote, backslash
                 char '"'
                 return $ String x

parseAtom :: Parser LispVal
parseAtom = do first <- letter <|> symbol
               rest <- many (letter <|> digit <|> symbol)
               let atom = [first] ++ rest
               return $ case atom of 
                          "#t" -> Bool True
                          "#f" -> Bool False
                          otherwise -> Atom atom

--was used before parseDigital came along
--parseNumber :: Parser LispVal
--parseNumber = liftM (Number . read) $ many1 digit
--parseNumber = many1 digit >>= \x -> (return. Number. read) x

parseNumber :: Parser LispVal 
parseNumber = parseDigital1 <|> parseDigital2 <|> parseHex <|> parseOct <|> parseBin

parseFloat :: Parser LispVal
parseFloat = do x <- many1 digit
                char '.' 
                y <- many1 digit
                return $ Float (fst.head $ readFloat (x ++ "." ++y))

parseComplex :: Parser LispVal
parseComplex = do x <- (try parseFloat <|> parseDigital1)
                  char '+'
                  y <- (try parseFloat <|> parseDigital1)
                  char 'i'
                  return $ Complex (toDouble x :+ toDouble y)

parseRatio :: Parser LispVal
parseRatio = do x <- many1 digit
                char '/'
                y <- many1 digit
                return $ Ratio ((read x ) % (read y))

parseDigital1 :: Parser LispVal
parseDigital1 = many1 digit >>= (return . Number. read)

parseDigital2 :: Parser LispVal
parseDigital2 = do try $ string "#d"
                   x <-many1 digit
                   (return . Number . read) x

parseHex :: Parser LispVal
parseHex = do try $ string "#x"
              x <- many1 hexDigit 
              return  $ Number (hex2dig x)

parseOct :: Parser LispVal
parseOct = do try $ string "#o"
              x <- many1 octDigit
              return $ Number (oct2dig x)

parseBin :: Parser LispVal
parseBin = do try $ string "#b"
              x <- many1 (oneOf "10")
              return $ Number (bin2dig x)



oct2dig x = fst $ readOct x !! 0
hex2dig x = fst $ readHex x !! 0
bin2dig = bin2dig' 0
bin2dig' digint "" = digint
bin2dig' digint (x:xs) = let old = 2 * digint + ( if x == '0' then 0 else 1) in 
                         bin2dig' old xs

parseExpr ::Parser LispVal
parseExpr = parseAtom
        
  <|> parseString
  <|> parseRatio
  <|> parseComplex
  <|> parseFloat
  <|> try parseNumber --we need try because they can all start with hash char
  <|> try parseBool
  <|> try parseCharacter
```
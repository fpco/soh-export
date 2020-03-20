# 1. Symbolic Calculator

An example of a session:

![Calculator session](http://www.bartosz.com/images/SoH/Calc.png)

Try it!

``` active haskell
import Text.Parsec
import Text.Parsec.String
import Text.Parsec.Token
import Text.Parsec.Expr
import Text.Parsec.Language
import qualified Data.Map as M
import qualified Control.Monad.State as S
import Control.Monad.Error
import Control.Monad.Identity

-- Lexer

def = emptyDef { identStart  = letter
               , identLetter = alphaNum 
               , opStart     = oneOf "+-*/="
               , opLetter    = oneOf "+-*/="
               }

lexer :: TokenParser ()
lexer = makeTokenParser def

-- Expression tree

data Expression = Constant Double
                | Identifier String
                | Addition Expression Expression
                | Subtraction Expression Expression
                | Multiplication Expression Expression
                | Division Expression Expression
                | Negation Expression
                | Assignment Expression Expression
                deriving Show

-- Parser

parseNumber :: Parser Expression
parseNumber = do
    v <- naturalOrFloat lexer
    case v of
        Left  i -> return $ Constant $ fromIntegral i
        Right n -> return $ Constant n

parseIdentifier :: Parser Expression
parseIdentifier = do
   i <- identifier lexer
   return $ Identifier i
   
parseExpression :: Parser Expression
parseExpression = (flip buildExpressionParser) parseTerm [
   [ Prefix (reservedOp lexer "-" >> return Negation)
   , Prefix (reservedOp lexer "+" >> return id) 
   ]
 , [ Infix (reservedOp lexer "*" >> return Multiplication) AssocLeft
   , Infix (reservedOp lexer "/" >> return Division) AssocLeft
   ]
 , [ Infix (reservedOp lexer "+" >> return Addition) AssocLeft
   , Infix (reservedOp lexer "-" >> return Subtraction) AssocLeft 
   ]
 , [ Infix (reservedOp lexer "=" >> return Assignment) AssocRight
   ]
 ]

parseTerm :: Parser Expression
parseTerm = parens lexer parseExpression 
        <|> parseNumber
        <|> parseIdentifier

parseInput :: Parser Expression
parseInput = do
    whiteSpace lexer
    ex <- parseExpression
    eof
    return ex

-- Evaluator

type SymTab = M.Map String Double

type Evaluator a = S.StateT SymTab (ErrorT String Identity) a

runEvaluator :: Evaluator Double -> SymTab -> Either String (Double, SymTab)
runEvaluator calc symTab = runIdentity $ runErrorT $ S.runStateT calc symTab

eval :: Expression -> Evaluator Double

eval (Constant x) = return x

eval (Identifier i) = do
    symtab <- S.get
    case M.lookup i symtab of
        Nothing -> fail $ "Undefined variable " ++ i
        Just e  -> return e

eval (Addition eLeft eRight) = do
    lft <- eval eLeft
    rgt <- eval eRight
    return $ lft + rgt

eval (Subtraction eLeft eRight) = do
    lft <- eval eLeft
    rgt <- eval eRight
    return $ lft - rgt

eval (Multiplication eLeft eRight) = do
    lft <- eval eLeft
    rgt <- eval eRight
    return $ lft * rgt

eval (Division eLeft eRight) = do
    lft <- eval eLeft
    rgt <- eval eRight
    return $ lft / rgt

eval (Negation e) = do
    val <- eval e
    return $ -val

eval (Assignment (Identifier i) e) = do
    val <- eval e
    S.modify (M.insert i val)
    return val

eval (Assignment _ _) =
    fail "Left of assignment must be an identifier"

defaultVars :: M.Map String Double
defaultVars = M.fromList
   [ ("e", exp 1)
   , ("pi", pi)
   ]
   
--runEvaluator returns Either String (Double, SymTab Double)

calculate :: SymTab -> String -> (String, SymTab)
calculate symTab s = 
    case parse parseInput "" s of
    Left  err -> ("error: " ++ (show err), symTab)
    Right exp -> case runEvaluator (eval exp) symTab of
                 Left  err              -> ("error: " ++ err, symTab)
                 Right (val, newSymTab) -> (show val, newSymTab)

loop :: SymTab -> IO ()
loop symTab = do
    line <- getLine
    if null line
    then return ()
    else do
        let (result, symTab') = calculate symTab line
        putStrLn result
        loop symTab'
   
main = loop defaultVars
-- show 
-- Enter expressions, one per line. Empty line to quit --
```

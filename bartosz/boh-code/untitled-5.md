# 6. Monads

**Ex 1** Define a Monad instance for May (an instance for Maybe is already defined in the Prelude). Define >>=, return, and fail.

``` active haskell
data May a = Noth | Jst a
  deriving Show

instance Monad May where
   ...

look :: [(String, Int)] -> String -> May Int
look dict str = case lookup str dict of
                  Nothing -> Noth
                  Just v  -> Jst v

test :: [(String, Int)] -> String -> String -> May Int
test dict str str' = do
    v  <- look dict str
    v' <- look dict str'
    return $ v + v'


testDict = [("x", 2), ("y", 3)]

main = do
   print $ test testDict "x" "y"
   print $ test testDict "x" "z"
   print $ test testDict "z" "y"
   print $ test testDict "z" "v"
```

@@@ Show solution

``` active haskell
data May a = Noth | Jst a
  deriving Show

instance Monad May where
    mv >>= k = case mv of
               Noth   -> Noth
               Jst x  -> k x
    return x = Jst x

look :: [(String, Int)] -> String -> May Int
look dict str = case lookup str dict of
                  Nothing -> Noth
                  Just v  -> Jst v

test :: [(String, Int)] -> String -> String -> May Int
test dict str str' = do
    v  <- look dict str
    v' <- look dict str'
    return $ v + v'


testDict = [("x", 2), ("y", 3)]

main = do
   print $ test testDict "x" "y"
   print $ test testDict "x" "z"
   print $ test testDict "z" "y"
   print $ test testDict "z" "v"
```
@@@

## Monadic Parser

**Ex 1.** Implement Monad instance for Parser. It should deal both with state (token list) and errors (Either). 

``` active haskell
import Data.Char
import qualified Data.Map as M

data Operator = Plus | Minus | Times | Div
    deriving (Show, Eq)

data Token = TokOp Operator
           | TokAssign
           | TokLParen
           | TokRParen
           | TokIdent String
           | TokNum Double
           | TokEnd
    deriving (Show, Eq)

operator :: Char -> Operator
operator c | c == '+' = Plus
           | c == '-' = Minus
           | c == '*' = Times
           | c == '/' = Div

tokenize [] = []
tokenize (c : cs) 
    | elem c "+-*/" = TokOp (operator c) : tokenize cs
    | c == '='  = TokAssign : tokenize cs
    | c == '('  = TokLParen : tokenize cs
    | c == ')'  = TokRParen : tokenize cs
    | isDigit c = number c cs
    | isAlpha c = identifier c cs
    | isSpace c = tokenize cs
    | otherwise = error $ "Cannot tokenize " ++ [c]

identifier :: Char -> String -> [Token]
identifier c cs = let (name, cs') = span isAlphaNum cs in
                  TokIdent (c:name) : tokenize cs'

number :: Char -> String -> [Token]
number c cs = 
   let (digs, cs') = span isDigit cs in
   TokNum (read (c : digs)) : tokenize cs'

---- parser ----
-- show
data Tree = SumNode Operator Tree Tree
          | ProdNode Operator Tree Tree
          | AssignNode String Tree
          | UnaryNode Operator Tree
          | NumNode Double
          | VarNode String
    deriving Show

newtype Parser a = P ([Token] -> Either String (a, [Token]))
   
instance Monad Parser where
   ???


lookAhead :: Parser Token
lookAhead = P $ \toks ->
    case toks of
    []     -> Right (TokEnd, [])
    (t:ts) -> Right (t, t:ts)

accept :: Parser ()
accept = P $ \toks ->
    case toks of
    []     -> Left "Nothing to accept"
    (t:ts) -> Right ((), ts)

expression :: Parser Tree
expression = do
   termTree <- term
   tok <- lookAhead
   case tok of
       (TokOp op) | elem op [Plus, Minus] -> do
            accept
            exTree <- expression
            return $ SumNode op termTree exTree
       TokAssign ->
            case termTree of
               VarNode str -> do
                  accept
                  exTree <- expression
                  return $ AssignNode str exTree
               _ -> fail "Only variables can be assigned to"
       _ -> return termTree

term :: Parser Tree
term = do
   facTree <- factor
   tok <- lookAhead
   case tok of
         (TokOp op) | elem op [Times, Div] -> do
            accept
            termTree <- term
            return $ ProdNode op facTree termTree
         _ -> return facTree

factor :: Parser Tree
factor = do
   tok <- lookAhead
   case tok of
      (TokNum x) -> do
          accept
          return $ NumNode x
      (TokIdent str) -> do
          accept
          return $ VarNode str
      (TokOp op) | elem op [Plus, Minus] -> do
          accept
          facTree <- factor
          return $ UnaryNode op facTree
      TokLParen -> do
         accept
         expTree <- expression
         tok' <- lookAhead
         if tok' /= TokRParen 
            then fail "Missing right parenthesis"
            else do
                accept
                return expTree
      _ -> fail $ "Token: " ++ show tok

parse :: [Token] -> Either String Tree
parse toks = 
    let P act = expression
        result = act toks
    in
        case result of
        Left msg -> Left msg
        Right (tree, toks') ->
          if null toks' 
          then Right tree
          else Left $ "Leftover tokens: " ++ show toks'
-- /show
---- evaluator ----

type SymTab = M.Map String Double

newtype Evaluator a = Ev (SymTab -> Either String (a, SymTab))

-- k : a -> Ev (SymTab -> Either String (b, SymTab))
instance Monad Evaluator where
    (Ev act) >>= k = Ev $
        \symTab -> 
            case act symTab of
            Left str -> Left str
            Right (x, symTab') ->
                let Ev act' = k x 
                in act' symTab'
    return x = Ev (\symTab -> Right (x, symTab))
    fail str = Ev (\_ -> Left str)

lookUp :: String -> Evaluator Double
lookUp str = Ev $ \symTab ->
    case M.lookup str symTab of
      Just v  -> Right (v, symTab)
      Nothing -> Left $ "Undefined variable: " ++ str

addSymbol :: String -> Double -> Evaluator Double
addSymbol str val = Ev $ \symTab ->
    let symTab' = M.insert str val symTab
    in  Right (val, symTab')

evaluate :: Tree -> Evaluator Double

evaluate (SumNode op left right) = do
    lft <- evaluate left
    rgt <- evaluate right
    case op of
       Plus  -> return $ lft + rgt
       Minus -> return $ lft - rgt

evaluate (ProdNode op left right) = do
    lft <- evaluate left
    rgt <- evaluate right
    case op of
       Times -> return $ lft * rgt
       Div   -> return $ lft / rgt

evaluate (UnaryNode op tree) = do
    x <- evaluate tree 
    case op of
       Plus  -> return x
       Minus -> return (-x)

evaluate (NumNode x) = return x

evaluate (VarNode str) = lookUp str

evaluate (AssignNode str tree) = do
    v <- evaluate tree
    addSymbol str v
-- show
main = do
   loop (M.fromList [("pi", pi)])
-- /show
loop symTab = do
   str <- getLine
   if null str
   then
      return ()
   else
      let toks = tokenize str
          eTree = parse toks
      in
          case eTree of
          Left msg -> do 
              print $ "Parse error: " ++ msg
              loop symTab
          Right tree ->
              let Ev act = evaluate tree
              in
                  case act symTab of
                  Left str -> do
                      putStrLn $ "Error: " ++ str
                      loop symTab
                  Right (val, symTab') -> do
                      print val
                      loop symTab'
```

@@@ Show solution

``` haskell
instance Monad Parser where
    (P act) >>= k = P $
        \toks -> 
            case act toks of
            Left str -> Left str
            Right (x, toks') ->
                let P act' = k x 
                in act' toks' 
    return x = P (\toks -> Right (x, toks))
    fail str = P (\_ -> Left str)
```
@@@

### Parser Combinators

**eX 1.** Reimplement `tokProd` using `token`.
``` haskell
tokProd = do
    tok <- lookAhead
    case tok of
      TokOp op | elem op [Times, Div] -> do
          accept
          return op
      _ -> fail "not a product"
```

``` active haskell
data Operator = Plus | Minus | Times | Div
    deriving (Show, Eq)

data Token = TokOp Operator
           | TokAssign
           | TokLParen
           | TokRParen
           | TokIdent String
           | TokNum Double
           | TokEnd
    deriving (Show, Eq)

data Tree = SumNode Operator Tree Tree
          | ProdNode Operator Tree Tree
          | AssignNode String Tree
          | UnaryNode Operator Tree
          | NumNode Double
          | VarNode String
    deriving Show

newtype Parser a = P ([Token] -> Either String (a, [Token]))
   
instance Monad Parser where
    (P act) >>= k = P $
        \toks -> 
            case act toks of
            Left str -> Left str
            Right (x, toks') ->
                let P act' = k x 
                in act' toks' 
    return x = P (\toks -> Right (x, toks))
    fail str = P (\_ -> Left str)

lookAhead :: Parser Token
lookAhead = P $ \toks ->
    case toks of
    []     -> Right (TokEnd, [])
    (t:ts) -> Right (t, t:ts)

accept :: Parser ()
accept = P $ \toks ->
    case toks of
    []     -> Left "Nothing to accept"
    (t:ts) -> Right ((), ts)

-- show
infixr 2 <|>  -- infix, right associative, precedence 2

(<|>) :: Parser a -> Parser a -> Parser a
(P act) <|> (P act') =
   P (\toks ->
      case act toks of         -- act may “consume” tokens
         Left _ -> act' toks   -- but here we “roll back”
         Right (x, toks') -> Right (x, toks'))

token :: Token -> Parser Token
token tok = do
   t <- lookAhead
   if t == tok 
   then do
       accept
       return t
   else
       fail ""

tokProd = ???

parse toks = 
    let (P act) = tokProd
    in
        case act toks of 
        Left _         -> "Failed"
        Right (tok, _) -> "Matched " ++ show tok
    
main = do
    print $ parse [TokOp Div]
    print $ parse [TokOp Plus]
```

@@@ Show solution

``` active haskell
data Operator = Plus | Minus | Times | Div
    deriving (Show, Eq)

data Token = TokOp Operator
           | TokAssign
           | TokLParen
           | TokRParen
           | TokIdent String
           | TokNum Double
           | TokEnd
    deriving (Show, Eq)

data Tree = SumNode Operator Tree Tree
          | ProdNode Operator Tree Tree
          | AssignNode String Tree
          | UnaryNode Operator Tree
          | NumNode Double
          | VarNode String
    deriving Show

newtype Parser a = P ([Token] -> Either String (a, [Token]))
   
instance Monad Parser where
    (P act) >>= k = P $
        \toks -> 
            case act toks of
            Left str -> Left str
            Right (x, toks') ->
                let P act' = k x 
                in act' toks' 
    return x = P (\toks -> Right (x, toks))
    fail str = P (\_ -> Left str)

lookAhead :: Parser Token
lookAhead = P $ \toks ->
    case toks of
    []     -> Right (TokEnd, [])
    (t:ts) -> Right (t, t:ts)

accept :: Parser ()
accept = P $ \toks ->
    case toks of
    []     -> Left "Nothing to accept"
    (t:ts) -> Right ((), ts)

-- show
infixr 2 <|>  -- infix, right associative, precedence 2

(<|>) :: Parser a -> Parser a -> Parser a
(P act) <|> (P act') =
   P (\toks ->
      case act toks of         -- act may “consume” tokens
         Left _ -> act' toks   -- but here we “roll back”
         Right (x, toks') -> Right (x, toks'))

token :: Token -> Parser Token
token tok = do
   t <- lookAhead
   if t == tok 
   then do
       accept
       return t
   else
       fail ""

tokProd = token (TokOp Times) <|> token (TokOp Div)

parse toks = 
    let (P act) = tokProd
    in
        case act toks of 
        Left _         -> "Failed"
        Right (tok, _) -> "Matched " ++ show tok
    
main = do
    print $ parse [TokOp Div]
    print $ parse [TokOp Plus]
```

@@@

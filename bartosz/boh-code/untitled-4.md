# 5. Evaluator

## Evaluator

**Ex 1.** Implement `evaluate` for `UnaryNode`

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
 
tokenize :: String -> [Token]
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

data Tree = SumNode Operator Tree Tree
          | ProdNode Operator Tree Tree
          | AssignNode String Tree
          | UnaryNode Operator Tree
          | NumNode Double
          | VarNode String
    deriving Show

lookAhead :: [Token] -> Token
lookAhead [] = TokEnd
lookAhead (t:ts) = t

accept :: [Token] -> [Token]
accept [] = error "Nothing to accept"
accept (t:ts) = ts

expression :: [Token] -> (Tree, [Token])
expression toks = 
   let (termTree, toks') = term toks
   in
      case lookAhead toks' of
         (TokOp op) | elem op [Plus, Minus] -> 
            let (exTree, toks'') = expression (accept toks') 
            in (SumNode op termTree exTree, toks'')
         TokAssign ->
            case termTree of
               VarNode str -> 
                  let (exTree, toks'') = expression (accept toks') 
                  in (AssignNode str exTree, toks'')
               _ -> error "Only variables can be assigned to"
         _ -> (termTree, toks')

term :: [Token] -> (Tree, [Token])
term toks = 
   let (facTree, toks') = factor toks
   in
      case lookAhead toks' of
         (TokOp op) | elem op [Times, Div] ->
            let (termTree, toks'') = term (accept toks') 
            in (ProdNode op facTree termTree, toks'')
         _ -> (facTree, toks')

factor :: [Token] -> (Tree, [Token])
factor toks = 
   case lookAhead toks of
      (TokNum x)     -> (NumNode x, accept toks)
      (TokIdent str) -> (VarNode str, accept toks)
      (TokOp op) | elem op [Plus, Minus] -> 
            let (facTree, toks') = factor (accept toks) 
            in (UnaryNode op facTree, toks')
      TokLParen      -> 
         let (expTree, toks') = expression (accept toks)
         in
            if lookAhead toks' /= TokRParen 
            then error "Missing right parenthesis"
            else (expTree, accept toks')
      _ -> error $ "Parse error on token: " ++ show toks

parse :: [Token] -> Tree
parse toks = let (tree, toks') = expression toks
             in
               if null toks' 
               then tree
               else error $ "Leftover tokens: " ++ show toks'

---- evaluator ----
{-
data Tree = SumNode Operator Tree Tree
          | ProdNode Operator Tree Tree
          | AssignNode String Tree
          | UnaryNode Operator Tree
          | NumNode Double
          | VarNode String
-}

-- show

type SymTab = M.Map String Double

lookUp :: SymTab -> String -> (Double, SymTab)
lookUp symTab str = 
    case M.lookup str symTab of
      Just v -> (v, symTab)
      Nothing -> error $ "Undefined variable " ++ str

addSymbol :: SymTab -> String -> Double -> (Double, SymTab)
addSymbol symTab str val = 
    let symTab' = M.insert str val symTab
    in (val, symTab')

evaluate :: SymTab -> Tree -> (Double, SymTab)

evaluate symTab (SumNode op left right) = 
    let (lft, symTab') = evaluate symTab left
        (rgt, symTab'') = evaluate symTab' right
    in
        case op of
          Plus  -> (lft + rgt, symTab'')
          Minus -> (lft - rgt, symTab'')

evaluate symTab (ProdNode op left right) = 
    let (lft, symTab') = evaluate symTab left
        (rgt, symTab'') = evaluate symTab right
    in
        case op of
          Times -> (lft * rgt, symTab)
          Div   -> (lft / rgt, symTab)

-- show
evaluate symTab (UnaryNode op tree) = undefined
-- /show
evaluate symTab (NumNode x) = (x, symTab)

evaluate symTab (VarNode str) = lookUp symTab str

evaluate symTab (AssignNode str tree) = 
    let (v, symTab')  = evaluate symTab tree
        (_, symTab'') = addSymbol symTab' str v
    in (v, symTab'')

main = do
   loop (M.fromList [("pi", pi)])

loop symTab = do
   str <- getLine
   if null str
   then
      return ()
   else
      let toks = tokenize str
          tree = parse toks
          (val, symTab') = evaluate symTab tree
      in do
          print val
          loop symTab'
```

@@@ Show solution
``` haskell
evaluate symTab (UnaryNode op tree) =
    let (x, symTab') = evaluate symTab tree 
    in case op of
         Plus  -> (x, symTab')
         Minus -> (-x, symTab')
```
@@@

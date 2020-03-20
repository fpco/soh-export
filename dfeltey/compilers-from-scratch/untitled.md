# Desugaring

Write a function to transform SExpr expressions into Expr expressions.

```haskell
type Name = String

data SExpr = AppS SExpr SExpr
          | LambdaS Name SExpr
          | BinopS Op SExpr SExpr
          | VarS Name
          | ValS Integer
          | BoolS Bool
          | IFS SExpr SExpr SExpr
          | LetS (Name,SExpr) SExpr 
          | PrintS SExpr
          deriving(Show)

data Expr = App Expr Expr
          | Lambda Name Expr
          | Binop Op Expr Expr
          | Var Name
          | Val Integer
          | BoolE Bool
          | IF Expr Expr Expr
          | PrintE Expr 
          deriving(Show)
          
data Op = Add
        | Sub
        | Mul
        | Div
        | Mod
        | Eq 
        | Lt
        | Gt
        | Leq
        | Geq 
        | OR
        | AND
        deriving(Show)  
```



```active haskell
-- show

desugar :: SExpr -> Expr
desugar = undefined

test = LetS ("x",ValS 5) (BinopS Add (VarS "x") (ValS 2))

-- /show

main = putStrLn $ show $ desugar test

type Name = String

data SExpr = AppS SExpr SExpr
          | LambdaS Name SExpr
          | BinopS Op SExpr SExpr
          | VarS Name
          | ValS Integer
          | BoolS Bool
          | IFS SExpr SExpr SExpr
          | LetS (Name,SExpr) SExpr 
          | PrintS SExpr
          deriving(Show)

data Expr = App Expr Expr
          | Lambda Name Expr
          | Binop Op Expr Expr
          | Var Name
          | Val Integer
          | BoolE Bool
          | IF Expr Expr Expr
          | PrintE Expr 
          deriving(Show)
          
data Op = Add
        | Sub
        | Mul
        | Div
        | Mod
        | Eq 
        | Lt
        | Gt
        | Leq
        | Geq 
        | OR
        | AND
        deriving(Show) 
```

 

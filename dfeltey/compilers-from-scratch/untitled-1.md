# Parsing

The Parse module contains the following definitions:

```haskell
newtype Parser s t = P ([s] -> [(t,[s])])
unP (P p) = p

pSym :: Eq s => s -> Parser s s
pSym a = P $ \inp -> case inp of 
                     (s:ss) | s == a -> [(s,ss)]
                     otherwise -> []

pReturn :: a -> Parser s a
pReturn a = P $ \inp -> [(a,inp)]

pFail :: Parser s t
pFail = P $ const []

infixl 5 <*>
(<*>) :: Parser s (b -> a) -> Parser s b -> Parser s a
P p1 <*> P p2 = P $ \inp -> [ (v1 v2, ss2) | (v1,ss1) <- p1 inp
                                           , (v2, ss2) <- p2 ss1]

infixr 3 <|>
(<|>) :: Parser s a -> Parser s a -> Parser s a
P p1 <|> P p2 = P $ \inp -> p1 inp ++ p2 inp

pChoice :: [Parser s a] -> Parser s a
pChoice ps = foldr (<|>) pFail ps

infix 7 <$>
(<$>) :: (b -> a) -> Parser s b -> Parser s a
f <$> p = pReturn f <*> p


infixl 3 `opt`
infixl 5 <*, *>
infixl 7 <$

f <$ p = const <$> pReturn f <*> p
p <* q = const <$> p         <*> q
p *> q = id    <$  p         <*> q

opt :: Parser s a -> a -> Parser s a
p `opt` v = p <|> pReturn v


pSatisfy :: (s -> Bool) -> Parser s s
pSatisfy p = P $ \inp -> case inp of
                        (x:xs) | p x -> [(x,xs)]
                        otherwise    -> []

pMany :: Parser s a -> Parser s [a]
pMany p = (:) <$> p <*> pMany p `opt` []

pMany1 :: Parser s a -> Parser s [a]
pMany1 p = (:) <$> p <*> pMany p

applyAll :: a -> [a -> a] -> a
applyAll x [] = x
applyAll x (f:fs) = applyAll (f x) fs

pChainL :: Parser s (a -> a -> a) -> Parser s a -> Parser s a
pChainL op p = applyAll <$> p <*> pMany (flip <$> op <*> p)
```

Recall our SExpr data type:

```haskell
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
                 

```

Complete the following specification of the parser for SExprs. Feel free to modify the token definition if you have a different set of tokens.

```active haskell

newtype Parser s t = P ([s] -> [(t,[s])])
unP (P p) = p

pSym :: Eq s => s -> Parser s s
pSym a = P $ \inp -> case inp of 
                     (s:ss) | s == a -> [(s,ss)]
                     otherwise -> []

pReturn :: a -> Parser s a
pReturn a = P $ \inp -> [(a,inp)]

pFail :: Parser s t
pFail = P $ const []

infixl 5 <*>
(<*>) :: Parser s (b -> a) -> Parser s b -> Parser s a
P p1 <*> P p2 = P $ \inp -> [ (v1 v2, ss2) | (v1,ss1) <- p1 inp
                                           , (v2, ss2) <- p2 ss1]

infixr 3 <|>
(<|>) :: Parser s a -> Parser s a -> Parser s a
P p1 <|> P p2 = P $ \inp -> p1 inp ++ p2 inp

pChoice :: [Parser s a] -> Parser s a
pChoice ps = foldr (<|>) pFail ps

infix 7 <$>
(<$>) :: (b -> a) -> Parser s b -> Parser s a
f <$> p = pReturn f <*> p


infixl 3 `opt`
infixl 5 <*, *>
infixl 7 <$

f <$ p = const <$> pReturn f <*> p
p <* q = const <$> p         <*> q
p *> q = id    <$  p         <*> q

opt :: Parser s a -> a -> Parser s a
p `opt` v = p <|> pReturn v


pSatisfy :: (s -> Bool) -> Parser s s
pSatisfy p = P $ \inp -> case inp of
                        (x:xs) | p x -> [(x,xs)]
                        otherwise    -> []

pMany :: Parser s a -> Parser s [a]
pMany p = (:) <$> p <*> pMany p `opt` []

pMany1 :: Parser s a -> Parser s [a]
pMany1 p = (:) <$> p <*> pMany p

applyAll :: a -> [a -> a] -> a
applyAll x [] = x
applyAll x (f:fs) = applyAll (f x) fs

pChainL :: Parser s (a -> a -> a) -> Parser s a -> Parser s a
pChainL op p = applyAll <$> p <*> pMany (flip <$> op <*> p)

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

type Name = String
-- show

-- Token list corresponding to the  "(let (x 5) (+ x 3))"  program
-- Adjust this if you used different tokens
tokens = [LParenT,LetT,SpaceT,LParenT,VarT "x",SpaceT,IntT 5,RParenT,SpaceT,LParenT,AddT,SpaceT,VarT "x",SpaceT,IntT 3,RParenT,RParenT]


-- The token definitions
data Token  = LParenT
			| RParenT
			| SpaceT
			| LambdaT
			| AddT
			| MulT
			| DivT
			| SubT
			| ModT
			| EqT
			| LtT
			| GtT
			| LeqT
			| GeqT
			| LetT
			| ORT
			| ANDT
			| IFT
			| IntT Integer
			| BoolT Bool
			| VarT Name
			| PrintT
			deriving(Show,Eq)

parseExpr :: Parser Token SExpr
parseExpr = parseInt

{-
parseExpr =  parseApp
		 <|> parseLambda
		 <|> parseBinop
		 <|> parseVar
		 <|> parseBool
		 <|> parseInt
		 <|> parseIF
		 <|> parseLet
		 <|> parsePrint
         
-}         


parseApp = undefined
parseLambda = undefined
parseBinop = undefined
parseVar = undefined
parseBool = undefined
parseIF = undefined
parseLet = undefined
parsePrint = undefined

parseInt :: Parser Token SExpr
parseInt = (\(IntT n) -> ValS n) <$> pSatisfy isIntTok where
	isIntTok t = case t of
		IntT _ -> True
		otherwise -> False

 
main = do
    -- putStrLn $ show $ (unP parseExpr) tokens
    putStrLn $ show $ (unP parseExpr) [IntT 53]
-- /show

```




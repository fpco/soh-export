# Lexing

```active haskell

import Prelude hiding (seq)
import Data.Char
import Control.Monad.Error

data RegEx c = Null
             | Eps
             | Sym (c -> Bool)
             | Alt (RegEx c) (RegEx c)
             | Seq (RegEx c) (RegEx c)
             | Star (RegEx c)

notNull :: RegEx c -> Bool
notNull Null = False
notNull _ = True
-- show

-- Some helpful utility functions 

word :: String -> RegEx Char 
word s = foldr seq Eps (map symC s) 

symC :: (Eq c) => c -> RegEx c
symC c = Sym (==c)

alt :: RegEx c -> RegEx c -> RegEx c
alt Null r = r
alt r Null = r
alt r1 r2 = Alt r1 r2

seq :: RegEx c -> RegEx c -> RegEx c
seq Null _ = Null
seq _ Null = Null
seq Eps r = r
seq r Eps = r
seq r1 r2 = Seq r1 r2

star :: RegEx c -> RegEx c 
star Null = Eps
star r = Star r

-- /show

empty :: RegEx c -> Bool
empty Null = False
empty Eps = True
empty (Sym _) = False
empty (Alt r1 r2) = empty r1 || empty r2
empty (Seq r1 r2) = empty r1 && empty r2
empty (Star _) = True

fromBool :: Bool -> RegEx c
fromBool True = Eps
fromBool False = Null

derivative :: c -> RegEx c -> RegEx c
derivative _ Null = Null
derivative _ Eps = Null
derivative c (Sym f) = if f c then Eps else Null
derivative c (Alt r1 r2) = alt (derivative c r1) (derivative c r2)
derivative c (Seq r1 r2) = alt (seq (fromBool $ empty r1) (derivative c r2)) 
                               (seq (derivative c r1) (r2))
derivative c (Star r) = seq (derivative c r) (star r)

match :: RegEx Char -> String -> Bool
match r [] = empty r
match r (c:cs) = match (derivative c r) cs

type Regex = RegEx Char

mapFst :: (a->t) -> [(a,b)] -> [(t,b)]
mapFst f ps = map (\(x,y) ->(f x, y)) ps 

lexS :: String -> Either String [Token]
lexS s = lexer rules s

lexer :: [(Regex,String -> t)] -> String -> Either String [t]
lexer rules s = lexAt 0 0 rules s where
              lexAt line char rules s = do
                      (t,s',line',char') <- tokenize line char rules s
                      case s' of
                        [] -> return [t]
                        str -> do
                            rest <- lexAt line' char' rules str
                            return $ t:rest

tok :: [(Regex,String->t)] -> String -> Either String (t,String,Int,Int)
tok = tokenize 0 0 


tokenize :: Int -> Int -> [(Regex,String->t)] -> String -> Either String (t,String,Int,Int)
tokenize line char rules string = go line char rules string [] where
                                  go line char [] _ _ = Left "Error: Empty rule list."
                                  go line char rules [] s = let rules' = filter ((\(a,b) -> empty a)) rules in
                                                                        case rules' of
                                                                            [] -> Left $ "Line: " ++ show line ++ " Char: " ++ show char ++ "\nError: Empty string unexpected"
                                                                            (r:rs) -> Right (snd r $ reverse s,[],line,char)
                                  go line char rules (c:cs) s = let char' = char + 1
                                                                    line' = line + if c == '\n' then 1 else 0
                                                                    step = mapFst (derivative c) rules in
                                                                        case filter (\(a,b) -> notNull a) step of
                                                                            (r:rs) -> go line' char' step cs (c:s)
                                                                            [] -> case filter (\(a,b) -> empty a) rules of
                                                                                [] -> Left $ "Line: " ++ show line ++ " Char: " ++ show char ++ "\nError: No matching regular expressions."
                                                                                (r:rs) -> Right $ (snd r $ reverse s,(c:cs),line,char)


-- show
data Token  = LParenT
			| RParenT
            | SpaceT
            deriving(Show)

rules = [  (Sym $ isSpace, const SpaceT)
		 , (symC '(', const LParenT)
		 , (symC ')', const RParenT)
         ]
         
         
program = "(let (x 5) (+ x 3))" 
-- /show

main = putStrLn $ show $ lexS program

```
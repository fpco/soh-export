# M-words in Linguistics

# An introduction to formal analysis of the truth of a sentence

In the scripts below users can change parts of the program.
Places where changes can be made are denoted by "<-- CHANGE".

When setting alphabet:

- users can give any list as an alphabet, eg. _([4,7,2,157]::[Int])_ or _['a'..'z']_. The type _::[Int]_ is necessary to prevent ambiguity (the same list could be a list of type Float, Double,...).
- the order of elements in the list gives the ordering of letters in the alphabet
- Char type alphabet assumes all letters are lowercase; uppercase letters represent two consecutive identical lowercase ones
- when using non-Char type alphabet the consecutive letters in output will be separated with the symbol "-"

When setting words:

- make sure the words contain an even number of letters
- make sure the words contain only letters from the alphabet, Eg. if the alphabet is _abc=lista2abeceda ([4,7,2,157]::[Int])_ then a word from it could be _x=lista2rijec ([4,4,2,157,4,2]::[Int])_


## Example 1

For an alphabet _{a, b, c, d, e, f, g}_, and an order _a < b < c < d < e < f < g_  we can conclude:

- the starting symbol is _a_, the ending symbol is _g_, while the center of the alphabet is _d_;
- the base consists of 49 atoms: _{aa, ab, ac,..., ag, ba, bb, bc,..., bg,...,ga,gb,..., gg}_;
- the M-word  _x = abbcdddc = aBcDdc_ is the concatenation of 4 atoms: _x = ab · bc · dd · dc_;
- dual word of _x_ is _x'=gffeddde_;
- starting and ending symbols of _x_ are _l(x)= a_ and _r(x)=c_,  and of its dual are _l'(x)= g_ and _r'(x) = e_;
- shell: _q(x) = ac_;
- the length of _x_ is given by _λ(abbcdddc) = -2_;
- compression: as _λ(ac) = -2_, it follows _λ(abbcdddc)= λ(ac)_, so _x = abbcdddc = ac_;
 - infimum and supremum: _d ↓ f = d,    f ↑ g = g,    l'(x) ↓ r'(x) = e_.

 

``` active haskell
{-# START_FILE Linguistic2.hs #-}
-- show
-- /show
{-# LANGUAGE UndecidableInstances, FlexibleInstances, OverlappingInstances, IncoherentInstances #-}
module Linguistic2 where

import           Data.Char  (chr, isLower, isUpper, ord, toLower, toUpper)
import           Data.List  (elemIndex, nub)
import           Data.Maybe (fromJust)

-- alphabet type definition and constructors
data Slovo a = Slovo a deriving (Eq, Show)
data Abeceda a = Abeceda {unAbeceda :: [Slovo a]} deriving (Eq, Show)
type Rijec a = [Slovo a]

-- |for a list the function constructs the alphabet containing unique elements of the list
lista2abeceda :: Eq a => [a] -> Abeceda a
lista2abeceda xs=Abeceda [Slovo x|x<-nub xs]

-- |for 2 Enum types (like Char) the function constructs the alphabet from first to second one
enum2abeceda :: (Eq a, Enum a) => a -> a -> Abeceda a
enum2abeceda a z=lista2abeceda [a..z]

-- |for a list the function constructs the word containing letters in the list
lista2rijec :: (Show a,Eq a) => [a] -> Rijec a
lista2rijec xs| even (length xs)= [Slovo x|x<-xs]
              | otherwise= error("The word "++show xs++" doesn't have an even number of letters.")

-- |for a word the function returns the list of letters in the word
rijec2lista :: Eq a => Rijec a -> [a]
rijec2lista xs= [x|Slovo x<-xs]

elemAbc :: Eq a => Slovo a -> Abeceda a -> Bool
elemAbc s (Abeceda a) = elem s a

elemIndexAbc :: Eq a => Slovo a -> Abeceda a -> Maybe Int
elemIndexAbc s (Abeceda a) = elemIndex s a

sup :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
sup a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) > fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

inf :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
inf a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) < fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

-- |function converting two consecutive lowercase letters to an uppercase letter
zapisu :: String->String
zapisu []=[]
zapisu [x]=[x]
zapisu (x:y:xs)|isLower x && x==y = toUpper x:zapisu xs
               |otherwise=x:zapisu (y:xs)

-- |function converting uppercase letter to two consecutive lowercase letters
zapisl :: String->String
zapisl []=[]
zapisl (x:xs)|isUpper x =x':x':zapisl xs
             |otherwise =x:zapisl xs
             where x'=toLower x

-- string output of a word
class Show a => Ispisiv a where
    ispis :: Show a => Rijec a -> String
    ispis []=[]
    ispis [Slovo x]=show x
    ispis (Slovo x:xs)=show x++"-"++ispis xs

instance Show a => Ispisiv a
instance Ispisiv Char where
    ispis=zapisu.ispis'

ispis' :: Rijec Char -> String
ispis' []=[]
ispis' [Slovo x]=[x]
ispis' (Slovo x:xs)=x:ispis' xs

-- functions returning left and right letters of a word
l :: [Slovo a] -> Slovo a
l = head
d :: [Slovo a] -> Slovo a
d = last

-- |function returning a dual of a letter in an alphabet
dual :: Eq a => Abeceda a -> Slovo a -> Slovo a
dual (Abeceda []) s = error("Nema duala - slovo nije u abecedi.")
dual (Abeceda [c]) s | s==c = c
                     | otherwise = error("No dual - Letter is not from the alphabet.")
dual (Abeceda cs) s | s==lijevi = desni
                    | s==desni = lijevi
                    | otherwise = dual (Abeceda (tail (init cs))) s
                     where lijevi= l cs
                           desni= d cs

-- functions returning duals of left/right letters of a word
l' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
l' a= (dual a).l
d' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
d' a= (dual a).d

ljuska :: [Slovo a]->[Slovo a]
ljuska xs=[l xs,d xs]


-- |distance between letters in an alphabet
dd :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Int
dd a x y| m==Nothing || n==Nothing = error("Cannot calculate distance. Letter is not from the alphabet.")
        | otherwise = abs (fromJust m-fromJust n)
  where m=x `elemIndexAbc` a
        n=y `elemIndexAbc` a

-- |word length
duljina :: Eq a => Abeceda a -> Rijec a -> Int
duljina a []=0
duljina a [c]=0
duljina a (c1:c2:cs)= - dd a c1 c2 - duljina a (c2:cs)

-- word shortening
krati1 :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function shortening a 4 letter word
krati1 a [p,r,s,t] | duljina a [p,r,s,t]==duljina a [p,t] = [p,t]
                 | otherwise = [p,r,s,t]
krati1 _ cs = cs

krati2 :: Eq a => Abeceda a -> Rijec a -> Rijec a     -- function shortening a 4 letter sub-word
krati2 _ [] = []
krati2 a(c:cs) | length kr==2 = kr ++ drop 3 cs
               | otherwise = c : krati2 a cs
               where kr = ((krati1 a).(take 4)) (c:cs)

krati :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function fully shortening a word
krati a cs | cs/=kr = krati a kr
           | otherwise = cs
           where kr = krati2 a cs

-- symmetries
opD :: Eq a => Abeceda a -> Rijec a -> Rijec a
opD _ []=[]
opD a (s:t:ys)=dual a s:dual a t:opD a ys

opE :: Eq a => Abeceda a -> Rijec a -> Rijec a
opE _ []=[]
opE a xs=t:s:opE a ys
        where n=length xs
              t=d xs
              s=xs !! (n-2)
              ys=take (n-2) xs
--opE a (s:t:ys)=opE a ys++[t,s]    --kraće, ali nije po formuli iz teksta

opF :: Eq a => Abeceda a -> Rijec a -> Rijec a
opF a=(opD a).(opE a)

-- permutations
opI :: Eq a => Abeceda a -> Rijec a -> Rijec a
opI a xs=l xs:xs++[d xs]

-- |negation operator
opK :: Eq a => Abeceda a -> Rijec a -> Rijec a
opK a xs=l' a xs:xs++[d' a xs]

opG :: Eq a => Abeceda a -> Rijec a -> Rijec a
opG a xs=d xs:xs++[l xs]

opH :: Eq a => Abeceda a -> Rijec a -> Rijec a
opH a xs=d' a xs:xs++[l' a xs]

-- serial and parallel conjecture operators
opS :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opS a xs ys = inf a (l xs) (l ys) : inf a (l xs) (l ys) : xs ++
              [sup a (d xs) (l ys),inf a (d xs) (l ys)] ++ ys ++
              [sup a (d xs) (d ys),sup a (d xs) (d ys)]

opP :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opP a xs ys = sup a (l xs) (l ys) : sup a (l xs) (l ys) : xs ++
              [inf a (d xs) (l ys),sup a (d xs) (l ys)] ++ ys ++
              [inf a (d xs) (d ys),inf a (d xs) (d ys)]

-- |partitions
particija :: Eq a => Abeceda a -> Rijec a -> Char
particija a xs | lijevi>=*lijevi' && desni<=*desni' && lijevi>*desni ='T'
               | lijevi<=*lijevi' && desni<=*desni' && not (lijevi==lijevi' || desni==desni') ='A'
               | lijevi==lijevi' && desni==desni' ='0'
               | lijevi>=*lijevi' && desni>=*desni' && not (lijevi==lijevi' || desni==desni')  ='Z'
               | lijevi<=*lijevi' && desni>=*desni' && lijevi<*desni  ='F'
         where lijevi=l xs
               desni=d xs
               lijevi'=l' a xs
               desni'=d' a xs
               x >* y| m==Nothing || n==Nothing =error("Letter not in alphabet.")
                     | otherwise = (fromJust m) > (fromJust n)
                      where m=x `elemIndexAbc` a
                            n=y `elemIndexAbc` a
               x >=* y = x >* y || x==y
               x <* y = not (x >=* y)
               x <=* y = not (x >* y)

-- implications
pimp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pimp a x y=(opK a x) `opP'` y
    where opP'=opP a

simp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
simp a x y=(opK a x) `opS'` y
    where opS'=opS a

-- equivalencies
pekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pekv a x y=(x `pimp'` y) `opS'` (y `pimp'` x)
    where opS'=opS a
          pimp'=pimp a

sekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
sekv a x y=(x `simp'` y) `opP'` (y `simp'` x)
    where opP'=opP a
          simp'=simp a

-- |tautologies and rules
taut :: Eq a => Abeceda a -> Rijec a ->Rijec a -> Rijec a -> [(String,Char)]
taut a x y z= zip
  ["1.ZAKON ISKLJUCENJA TRECEG",
  "2.DUAL ZAKONA ISKLJUCENJA TRECEG",
  "3.TRANZITIVNOST IMPLIKACIJE",
  "4.DUAL TRANZITIVNOSTI IMPLIKACIJE",
  "5.ZAKON DVOSTRUKE NEGACIJE",
  "6.ZAKON KONTRAPOZICIJE",
  "7.DUAL ZAKONA KONTRAPOZICIJE",
  "8.ZAKON APSORPCIJE ILI UPIJANJA",
  "9.DUAL ZAKONA APSORPCIJE ILI UPIJANJA",
  "10.MODUS PONENDO PONENS",
  "11.DUAL MODUSA PONENDO PONENS",
  "12.MODUS TOLENDO TOLLENS",
  "13.DOKAZ NABRAJANJEM",
  "14.DOVODJENJE DO PROTURJECJA"]
  (map (particija a) [
    x `opP'` (neg x),
    x `opS'` (neg x),
    ((x `pimp'` y) `opS'` (y `pimp'` z)) `pimp'` (x `pimp'` z),
    ((x `simp'` y) `opP'` (y `simp'` z)) `simp'` (x `simp'` z),
    (neg (neg x)) `pekv'` x,
    (x `pimp'` y) `pimp'` ((neg y) `pimp'` (neg x)),
    (x `simp'` y) `simp'` ((neg y) `simp'` (neg x)),
    (x `opP'` (y `opP'` x)) `pimp'` x,
    (x `opS'` (y `opS'` x)) `simp'` x,
    (x `opS'` (x `pimp'` y)) `pimp'` y,
    (x `opP'` (x `simp'` y)) `simp'` y,
    ((neg x) `opS'` ((neg x) `pimp'` (neg y))) `pimp'` (neg y),
    ((x `pimp'` z) `opS'` (y `pimp'` z)) `pimp'` ((x `opP'` y) `pimp'` z),
    ((x `pimp'` y) `opS'` (x `pimp'` (neg y))) `pimp'` (neg x)
    ]
  )
  where opP'=opP a
        opS'=opS a
        pimp'=pimp a
        simp'=simp a
        pekv'=pekv a
        sekv'=sekv a
        neg=opK a
        
{-# START_FILE Main.hs #-}
module Main where
import Linguistic2

main :: IO ()

-- show Alphabet
main=do let {abc=lista2abeceda "abcdefg"}         -- <-- CHANGE
-- /show


-- show Word x
        let {x=lista2rijec.zapisl $ "aBcDdc"}     -- <-- CHANGE
-- In the previous line the part '.zapisl $' is added only for Char type words to convert uppercase letters to lowercase.
-- /show


-- show Dual
        putStrLn ("x'= "++ ispis (map (dual abc) x))
-- /show


-- show Starting and ending symbols
        putStr ("l(x)= "++ ispis [l x]++", ")
        putStr ("r(x)= "++ ispis [d x]++", ")
        putStr ("l'(x)= "++ ispis [l' abc x]++", ")
        putStrLn ("r'(x)= "++ ispis [d' abc x])
-- /show


-- show Shell
        putStrLn ("q(x)= "++ ispis (ljuska x))
-- /show


-- show Length
        putStrLn ("λ(x)= "++ show (duljina abc x))
-- /show


-- show Compression
        putStrLn (ispis x ++" =zip= "++ ispis (krati abc x))
-- /show


-- show Infimum and supremum
        putStrLn ("inf(l'(x),r'(x))= "++ ispis [inf abc (l' abc x) (d' abc x)])
        putStrLn ("sup(l'(x),r'(x))= "++ ispis [sup abc (l' abc x) (d' abc x)])
-- /show
```

### Example 1 with integer alphabet

``` active haskell
{-# START_FILE Linguistic2.hs #-}
-- show
-- /show
{-# LANGUAGE UndecidableInstances, FlexibleInstances, OverlappingInstances, IncoherentInstances #-}
module Linguistic2 where

import           Data.Char  (chr, isLower, isUpper, ord, toLower, toUpper)
import           Data.List  (elemIndex, nub)
import           Data.Maybe (fromJust)

-- alphabet type definition and constructors
data Slovo a = Slovo a deriving (Eq, Show)
data Abeceda a = Abeceda {unAbeceda :: [Slovo a]} deriving (Eq, Show)
type Rijec a = [Slovo a]

-- |for a list the function constructs the alphabet containing unique elements of the list
lista2abeceda :: Eq a => [a] -> Abeceda a
lista2abeceda xs=Abeceda [Slovo x|x<-nub xs]

-- |for 2 Enum types (like Char) the function constructs the alphabet from first to second one
enum2abeceda :: (Eq a, Enum a) => a -> a -> Abeceda a
enum2abeceda a z=lista2abeceda [a..z]

-- |for a list the function constructs the word containing letters in the list
lista2rijec :: (Show a,Eq a) => [a] -> Rijec a
lista2rijec xs| even (length xs)= [Slovo x|x<-xs]
              | otherwise= error("The word "++show xs++" doesn't have an even number of letters.")

-- |for a word the function returns the list of letters in the word
rijec2lista :: Eq a => Rijec a -> [a]
rijec2lista xs= [x|Slovo x<-xs]

elemAbc :: Eq a => Slovo a -> Abeceda a -> Bool
elemAbc s (Abeceda a) = elem s a

elemIndexAbc :: Eq a => Slovo a -> Abeceda a -> Maybe Int
elemIndexAbc s (Abeceda a) = elemIndex s a

sup :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
sup a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) > fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

inf :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
inf a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) < fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

-- |function converting two consecutive lowercase letters to an uppercase letter
zapisu :: String->String
zapisu []=[]
zapisu [x]=[x]
zapisu (x:y:xs)|isLower x && x==y = toUpper x:zapisu xs
               |otherwise=x:zapisu (y:xs)

-- |function converting uppercase letter to two consecutive lowercase letters
zapisl :: String->String
zapisl []=[]
zapisl (x:xs)|isUpper x =x':x':zapisl xs
             |otherwise =x:zapisl xs
             where x'=toLower x

-- string output of a word
class Show a => Ispisiv a where
    ispis :: Show a => Rijec a -> String
    ispis []=[]
    ispis [Slovo x]=show x
    ispis (Slovo x:xs)=show x++"-"++ispis xs

instance Show a => Ispisiv a
instance Ispisiv Char where
    ispis=zapisu.ispis'

ispis' :: Rijec Char -> String
ispis' []=[]
ispis' [Slovo x]=[x]
ispis' (Slovo x:xs)=x:ispis' xs

-- functions returning left and right letters of a word
l :: [Slovo a] -> Slovo a
l = head
d :: [Slovo a] -> Slovo a
d = last

-- |function returning a dual of a letter in an alphabet
dual :: Eq a => Abeceda a -> Slovo a -> Slovo a
dual (Abeceda []) s = error("Nema duala - slovo nije u abecedi.")
dual (Abeceda [c]) s | s==c = c
                     | otherwise = error("No dual - Letter is not from the alphabet.")
dual (Abeceda cs) s | s==lijevi = desni
                    | s==desni = lijevi
                    | otherwise = dual (Abeceda (tail (init cs))) s
                     where lijevi= l cs
                           desni= d cs

-- functions returning duals of left/right letters of a word
l' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
l' a= (dual a).l
d' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
d' a= (dual a).d

ljuska :: [Slovo a]->[Slovo a]
ljuska xs=[l xs,d xs]


-- |distance between letters in an alphabet
dd :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Int
dd a x y| m==Nothing || n==Nothing = error("Cannot calculate distance. Letter is not from the alphabet.")
        | otherwise = abs (fromJust m-fromJust n)
  where m=x `elemIndexAbc` a
        n=y `elemIndexAbc` a

-- |word length
duljina :: Eq a => Abeceda a -> Rijec a -> Int
duljina a []=0
duljina a [c]=0
duljina a (c1:c2:cs)= - dd a c1 c2 - duljina a (c2:cs)

-- word shortening
krati1 :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function shortening a 4 letter word
krati1 a [p,r,s,t] | duljina a [p,r,s,t]==duljina a [p,t] = [p,t]
                 | otherwise = [p,r,s,t]
krati1 _ cs = cs

krati2 :: Eq a => Abeceda a -> Rijec a -> Rijec a     -- function shortening a 4 letter sub-word
krati2 _ [] = []
krati2 a(c:cs) | length kr==2 = kr ++ drop 3 cs
               | otherwise = c : krati2 a cs
               where kr = ((krati1 a).(take 4)) (c:cs)

krati :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function fully shortening a word
krati a cs | cs/=kr = krati a kr
           | otherwise = cs
           where kr = krati2 a cs

-- symmetries
opD :: Eq a => Abeceda a -> Rijec a -> Rijec a
opD _ []=[]
opD a (s:t:ys)=dual a s:dual a t:opD a ys

opE :: Eq a => Abeceda a -> Rijec a -> Rijec a
opE _ []=[]
opE a xs=t:s:opE a ys
        where n=length xs
              t=d xs
              s=xs !! (n-2)
              ys=take (n-2) xs
--opE a (s:t:ys)=opE a ys++[t,s]    --kraće, ali nije po formuli iz teksta

opF :: Eq a => Abeceda a -> Rijec a -> Rijec a
opF a=(opD a).(opE a)

-- permutations
opI :: Eq a => Abeceda a -> Rijec a -> Rijec a
opI a xs=l xs:xs++[d xs]

-- |negation operator
opK :: Eq a => Abeceda a -> Rijec a -> Rijec a
opK a xs=l' a xs:xs++[d' a xs]

opG :: Eq a => Abeceda a -> Rijec a -> Rijec a
opG a xs=d xs:xs++[l xs]

opH :: Eq a => Abeceda a -> Rijec a -> Rijec a
opH a xs=d' a xs:xs++[l' a xs]

-- serial and parallel conjecture operators
opS :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opS a xs ys = inf a (l xs) (l ys) : inf a (l xs) (l ys) : xs ++
              [sup a (d xs) (l ys),inf a (d xs) (l ys)] ++ ys ++
              [sup a (d xs) (d ys),sup a (d xs) (d ys)]

opP :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opP a xs ys = sup a (l xs) (l ys) : sup a (l xs) (l ys) : xs ++
              [inf a (d xs) (l ys),sup a (d xs) (l ys)] ++ ys ++
              [inf a (d xs) (d ys),inf a (d xs) (d ys)]

-- |partitions
particija :: Eq a => Abeceda a -> Rijec a -> Char
particija a xs | lijevi>=*lijevi' && desni<=*desni' && lijevi>*desni ='T'
               | lijevi<=*lijevi' && desni<=*desni' && not (lijevi==lijevi' || desni==desni') ='A'
               | lijevi==lijevi' && desni==desni' ='0'
               | lijevi>=*lijevi' && desni>=*desni' && not (lijevi==lijevi' || desni==desni')  ='Z'
               | lijevi<=*lijevi' && desni>=*desni' && lijevi<*desni  ='F'
         where lijevi=l xs
               desni=d xs
               lijevi'=l' a xs
               desni'=d' a xs
               x >* y| m==Nothing || n==Nothing =error("Letter not in alphabet.")
                     | otherwise = (fromJust m) > (fromJust n)
                      where m=x `elemIndexAbc` a
                            n=y `elemIndexAbc` a
               x >=* y = x >* y || x==y
               x <* y = not (x >=* y)
               x <=* y = not (x >* y)

-- implications
pimp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pimp a x y=(opK a x) `opP'` y
    where opP'=opP a

simp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
simp a x y=(opK a x) `opS'` y
    where opS'=opS a

-- equivalencies
pekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pekv a x y=(x `pimp'` y) `opS'` (y `pimp'` x)
    where opS'=opS a
          pimp'=pimp a

sekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
sekv a x y=(x `simp'` y) `opP'` (y `simp'` x)
    where opP'=opP a
          simp'=simp a

-- |tautologies and rules
taut :: Eq a => Abeceda a -> Rijec a ->Rijec a -> Rijec a -> [(String,Char)]
taut a x y z= zip
  ["1.ZAKON ISKLJUCENJA TRECEG",
  "2.DUAL ZAKONA ISKLJUCENJA TRECEG",
  "3.TRANZITIVNOST IMPLIKACIJE",
  "4.DUAL TRANZITIVNOSTI IMPLIKACIJE",
  "5.ZAKON DVOSTRUKE NEGACIJE",
  "6.ZAKON KONTRAPOZICIJE",
  "7.DUAL ZAKONA KONTRAPOZICIJE",
  "8.ZAKON APSORPCIJE ILI UPIJANJA",
  "9.DUAL ZAKONA APSORPCIJE ILI UPIJANJA",
  "10.MODUS PONENDO PONENS",
  "11.DUAL MODUSA PONENDO PONENS",
  "12.MODUS TOLENDO TOLLENS",
  "13.DOKAZ NABRAJANJEM",
  "14.DOVODJENJE DO PROTURJECJA"]
  (map (particija a) [
    x `opP'` (neg x),
    x `opS'` (neg x),
    ((x `pimp'` y) `opS'` (y `pimp'` z)) `pimp'` (x `pimp'` z),
    ((x `simp'` y) `opP'` (y `simp'` z)) `simp'` (x `simp'` z),
    (neg (neg x)) `pekv'` x,
    (x `pimp'` y) `pimp'` ((neg y) `pimp'` (neg x)),
    (x `simp'` y) `simp'` ((neg y) `simp'` (neg x)),
    (x `opP'` (y `opP'` x)) `pimp'` x,
    (x `opS'` (y `opS'` x)) `simp'` x,
    (x `opS'` (x `pimp'` y)) `pimp'` y,
    (x `opP'` (x `simp'` y)) `simp'` y,
    ((neg x) `opS'` ((neg x) `pimp'` (neg y))) `pimp'` (neg y),
    ((x `pimp'` z) `opS'` (y `pimp'` z)) `pimp'` ((x `opP'` y) `pimp'` z),
    ((x `pimp'` y) `opS'` (x `pimp'` (neg y))) `pimp'` (neg x)
    ]
  )
  where opP'=opP a
        opS'=opS a
        pimp'=pimp a
        simp'=simp a
        pekv'=pekv a
        sekv'=sekv a
        neg=opK a
        
{-# START_FILE Main.hs #-}
module Main where
import Linguistic2

main :: IO ()

-- show Alphabet
main=do let {abc=lista2abeceda ([4,7,2,157]::[Int])}     -- <-- CHANGE
-- /show


-- show Word x
        let {x=lista2rijec ([4,4,2,157,4,2]::[Int])}     -- <-- CHANGE
-- /show


-- show Dual
        putStrLn ("x'= "++ ispis (map (dual abc) x))
-- /show


-- show Starting and ending symbols
        putStr ("l(x)= "++ ispis [l x]++", ")
        putStr ("r(x)= "++ ispis [d x]++", ")
        putStr ("l'(x)= "++ ispis [l' abc x]++", ")
        putStrLn ("r'(x)= "++ ispis [d' abc x])
-- /show


-- show Shell
        putStrLn ("q(x)= "++ ispis (ljuska x))
-- /show


-- show Length
        putStrLn ("λ(x)= "++ show (duljina abc x))
-- /show


-- show Compression
        putStrLn (ispis x ++" =zip= "++ ispis (krati abc x))
-- /show


-- show Infimum and supremum
        putStrLn ("inf(l'(x),r'(x))= "++ ispis [inf abc (l' abc x) (d' abc x)])
        putStrLn ("sup(l'(x),r'(x))= "++ ispis [sup abc (l' abc x) (d' abc x)])
-- /show
```

## Example 2
To visualize the symmetries, we define an alphabet _{⊤, |, ⊥}_. In the logic algebra these symbols correspond to: truth, uncertainty and falsity, but here their role is to help to visualize that the truth and false symbols are symmetric with respect to the horizontal axis, and the uncertainty symbol with respect to the vertical axis. Operators __D, E__ and __ F __ act on M-words over this alphabet as follows:

``` active haskell
{-# START_FILE Linguistic2.hs #-}
-- show
-- /show
{-# LANGUAGE UndecidableInstances, FlexibleInstances, OverlappingInstances, IncoherentInstances #-}
module Linguistic2 where

import           Data.Char  (chr, isLower, isUpper, ord, toLower, toUpper)
import           Data.List  (elemIndex, nub)
import           Data.Maybe (fromJust)

-- alphabet type definition and constructors
data Slovo a = Slovo a deriving (Eq, Show)
data Abeceda a = Abeceda {unAbeceda :: [Slovo a]} deriving (Eq, Show)
type Rijec a = [Slovo a]

-- |for a list the function constructs the alphabet containing unique elements of the list
lista2abeceda :: Eq a => [a] -> Abeceda a
lista2abeceda xs=Abeceda [Slovo x|x<-nub xs]

-- |for 2 Enum types (like Char) the function constructs the alphabet from first to second one
enum2abeceda :: (Eq a, Enum a) => a -> a -> Abeceda a
enum2abeceda a z=lista2abeceda [a..z]

-- |for a list the function constructs the word containing letters in the list
lista2rijec :: (Show a,Eq a) => [a] -> Rijec a
lista2rijec xs| even (length xs)= [Slovo x|x<-xs]
              | otherwise= error("The word "++show xs++" doesn't have an even number of letters.")

-- |for a word the function returns the list of letters in the word
rijec2lista :: Eq a => Rijec a -> [a]
rijec2lista xs= [x|Slovo x<-xs]

elemAbc :: Eq a => Slovo a -> Abeceda a -> Bool
elemAbc s (Abeceda a) = elem s a

elemIndexAbc :: Eq a => Slovo a -> Abeceda a -> Maybe Int
elemIndexAbc s (Abeceda a) = elemIndex s a

sup :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
sup a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) > fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

inf :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
inf a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) < fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

-- |function converting two consecutive lowercase letters to an uppercase letter
zapisu :: String->String
zapisu []=[]
zapisu [x]=[x]
zapisu (x:y:xs)|isLower x && x==y = toUpper x:zapisu xs
               |otherwise=x:zapisu (y:xs)

-- |function converting uppercase letter to two consecutive lowercase letters
zapisl :: String->String
zapisl []=[]
zapisl (x:xs)|isUpper x =x':x':zapisl xs
             |otherwise =x:zapisl xs
             where x'=toLower x

-- string output of a word
class Show a => Ispisiv a where
    ispis :: Show a => Rijec a -> String
    ispis []=[]
    ispis [Slovo x]=show x
    ispis (Slovo x:xs)=show x++"-"++ispis xs

instance Show a => Ispisiv a
instance Ispisiv Char where
    ispis=zapisu.ispis'

ispis' :: Rijec Char -> String
ispis' []=[]
ispis' [Slovo x]=[x]
ispis' (Slovo x:xs)=x:ispis' xs

-- functions returning left and right letters of a word
l :: [Slovo a] -> Slovo a
l = head
d :: [Slovo a] -> Slovo a
d = last

-- |function returning a dual of a letter in an alphabet
dual :: Eq a => Abeceda a -> Slovo a -> Slovo a
dual (Abeceda []) s = error("Nema duala - slovo nije u abecedi.")
dual (Abeceda [c]) s | s==c = c
                     | otherwise = error("No dual - Letter is not from the alphabet.")
dual (Abeceda cs) s | s==lijevi = desni
                    | s==desni = lijevi
                    | otherwise = dual (Abeceda (tail (init cs))) s
                     where lijevi= l cs
                           desni= d cs

-- functions returning duals of left/right letters of a word
l' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
l' a= (dual a).l
d' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
d' a= (dual a).d

ljuska :: [Slovo a]->[Slovo a]
ljuska xs=[l xs,d xs]


-- |distance between letters in an alphabet
dd :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Int
dd a x y| m==Nothing || n==Nothing = error("Cannot calculate distance. Letter is not from the alphabet.")
        | otherwise = abs (fromJust m-fromJust n)
  where m=x `elemIndexAbc` a
        n=y `elemIndexAbc` a

-- |word length
duljina :: Eq a => Abeceda a -> Rijec a -> Int
duljina a []=0
duljina a [c]=0
duljina a (c1:c2:cs)= - dd a c1 c2 - duljina a (c2:cs)

-- word shortening
krati1 :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function shortening a 4 letter word
krati1 a [p,r,s,t] | duljina a [p,r,s,t]==duljina a [p,t] = [p,t]
                 | otherwise = [p,r,s,t]
krati1 _ cs = cs

krati2 :: Eq a => Abeceda a -> Rijec a -> Rijec a     -- function shortening a 4 letter sub-word
krati2 _ [] = []
krati2 a(c:cs) | length kr==2 = kr ++ drop 3 cs
               | otherwise = c : krati2 a cs
               where kr = ((krati1 a).(take 4)) (c:cs)

krati :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function fully shortening a word
krati a cs | cs/=kr = krati a kr
           | otherwise = cs
           where kr = krati2 a cs

-- symmetries
opD :: Eq a => Abeceda a -> Rijec a -> Rijec a
opD _ []=[]
opD a (s:t:ys)=dual a s:dual a t:opD a ys

opE :: Eq a => Abeceda a -> Rijec a -> Rijec a
opE _ []=[]
opE a xs=t:s:opE a ys
        where n=length xs
              t=d xs
              s=xs !! (n-2)
              ys=take (n-2) xs
--opE a (s:t:ys)=opE a ys++[t,s]    --kraće, ali nije po formuli iz teksta

opF :: Eq a => Abeceda a -> Rijec a -> Rijec a
opF a=(opD a).(opE a)

-- permutations
opI :: Eq a => Abeceda a -> Rijec a -> Rijec a
opI a xs=l xs:xs++[d xs]

-- |negation operator
opK :: Eq a => Abeceda a -> Rijec a -> Rijec a
opK a xs=l' a xs:xs++[d' a xs]

opG :: Eq a => Abeceda a -> Rijec a -> Rijec a
opG a xs=d xs:xs++[l xs]

opH :: Eq a => Abeceda a -> Rijec a -> Rijec a
opH a xs=d' a xs:xs++[l' a xs]

-- serial and parallel conjecture operators
opS :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opS a xs ys = inf a (l xs) (l ys) : inf a (l xs) (l ys) : xs ++
              [sup a (d xs) (l ys),inf a (d xs) (l ys)] ++ ys ++
              [sup a (d xs) (d ys),sup a (d xs) (d ys)]

opP :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opP a xs ys = sup a (l xs) (l ys) : sup a (l xs) (l ys) : xs ++
              [inf a (d xs) (l ys),sup a (d xs) (l ys)] ++ ys ++
              [inf a (d xs) (d ys),inf a (d xs) (d ys)]

-- |partitions
particija :: Eq a => Abeceda a -> Rijec a -> Char
particija a xs | lijevi>=*lijevi' && desni<=*desni' && lijevi>*desni ='T'
               | lijevi<=*lijevi' && desni<=*desni' && not (lijevi==lijevi' || desni==desni') ='A'
               | lijevi==lijevi' && desni==desni' ='0'
               | lijevi>=*lijevi' && desni>=*desni' && not (lijevi==lijevi' || desni==desni')  ='Z'
               | lijevi<=*lijevi' && desni>=*desni' && lijevi<*desni  ='F'
         where lijevi=l xs
               desni=d xs
               lijevi'=l' a xs
               desni'=d' a xs
               x >* y| m==Nothing || n==Nothing =error("Letter not in alphabet.")
                     | otherwise = (fromJust m) > (fromJust n)
                      where m=x `elemIndexAbc` a
                            n=y `elemIndexAbc` a
               x >=* y = x >* y || x==y
               x <* y = not (x >=* y)
               x <=* y = not (x >* y)

-- implications
pimp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pimp a x y=(opK a x) `opP'` y
    where opP'=opP a

simp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
simp a x y=(opK a x) `opS'` y
    where opS'=opS a

-- equivalencies
pekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pekv a x y=(x `pimp'` y) `opS'` (y `pimp'` x)
    where opS'=opS a
          pimp'=pimp a

sekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
sekv a x y=(x `simp'` y) `opP'` (y `simp'` x)
    where opP'=opP a
          simp'=simp a

-- |tautologies and rules
taut :: Eq a => Abeceda a -> Rijec a ->Rijec a -> Rijec a -> [(String,Char)]
taut a x y z= zip
  ["1.ZAKON ISKLJUCENJA TRECEG",
  "2.DUAL ZAKONA ISKLJUCENJA TRECEG",
  "3.TRANZITIVNOST IMPLIKACIJE",
  "4.DUAL TRANZITIVNOSTI IMPLIKACIJE",
  "5.ZAKON DVOSTRUKE NEGACIJE",
  "6.ZAKON KONTRAPOZICIJE",
  "7.DUAL ZAKONA KONTRAPOZICIJE",
  "8.ZAKON APSORPCIJE ILI UPIJANJA",
  "9.DUAL ZAKONA APSORPCIJE ILI UPIJANJA",
  "10.MODUS PONENDO PONENS",
  "11.DUAL MODUSA PONENDO PONENS",
  "12.MODUS TOLENDO TOLLENS",
  "13.DOKAZ NABRAJANJEM",
  "14.DOVODJENJE DO PROTURJECJA"]
  (map (particija a) [
    x `opP'` (neg x),
    x `opS'` (neg x),
    ((x `pimp'` y) `opS'` (y `pimp'` z)) `pimp'` (x `pimp'` z),
    ((x `simp'` y) `opP'` (y `simp'` z)) `simp'` (x `simp'` z),
    (neg (neg x)) `pekv'` x,
    (x `pimp'` y) `pimp'` ((neg y) `pimp'` (neg x)),
    (x `simp'` y) `simp'` ((neg y) `simp'` (neg x)),
    (x `opP'` (y `opP'` x)) `pimp'` x,
    (x `opS'` (y `opS'` x)) `simp'` x,
    (x `opS'` (x `pimp'` y)) `pimp'` y,
    (x `opP'` (x `simp'` y)) `simp'` y,
    ((neg x) `opS'` ((neg x) `pimp'` (neg y))) `pimp'` (neg y),
    ((x `pimp'` z) `opS'` (y `pimp'` z)) `pimp'` ((x `opP'` y) `pimp'` z),
    ((x `pimp'` y) `opS'` (x `pimp'` (neg y))) `pimp'` (neg x)
    ]
  )
  where opP'=opP a
        opS'=opS a
        pimp'=pimp a
        simp'=simp a
        pekv'=pekv a
        sekv'=sekv a
        neg=opK a
        
{-# START_FILE Main.hs #-}
module Main where
import Linguistic2

main :: IO ()

-- show Alphabet
main=do let {abc=lista2abeceda "⊤|⊥"}         -- <-- CHANGE
-- /show


-- show Words
        let {x=lista2rijec "⊤⊤||⊥⊥||";
             y=lista2rijec "⊥⊥||⊤⊤||";        -- <-- CHANGE
             z=lista2rijec "||⊤⊤||⊥⊥"}
-- /show


-- show Symmetry operators D, E and F
        putStrLn ("D("++ispis x++")= "++ ispis (opD abc x))
        putStrLn ("E("++ispis y++")= "++ ispis (opE abc y))
        putStrLn ("F("++ispis z++")= "++ ispis (opF abc z))
-- /show
```

## Example 3
Let the alphabet be the English alphabet _{a,... ,z}_.
Let _x = croatian = croian_ and  _y=language = le_.
Unary operators __D, E, F, I, K, G, H__ applied to these words give the following results:

``` active haskell
{-# START_FILE Linguistic2.hs #-}
-- show
-- /show
{-# LANGUAGE UndecidableInstances, FlexibleInstances, OverlappingInstances, IncoherentInstances #-}
module Linguistic2 where

import           Data.Char  (chr, isLower, isUpper, ord, toLower, toUpper)
import           Data.List  (elemIndex, nub)
import           Data.Maybe (fromJust)

-- alphabet type definition and constructors
data Slovo a = Slovo a deriving (Eq, Show)
data Abeceda a = Abeceda {unAbeceda :: [Slovo a]} deriving (Eq, Show)
type Rijec a = [Slovo a]

-- |for a list the function constructs the alphabet containing unique elements of the list
lista2abeceda :: Eq a => [a] -> Abeceda a
lista2abeceda xs=Abeceda [Slovo x|x<-nub xs]

-- |for 2 Enum types (like Char) the function constructs the alphabet from first to second one
enum2abeceda :: (Eq a, Enum a) => a -> a -> Abeceda a
enum2abeceda a z=lista2abeceda [a..z]

-- |for a list the function constructs the word containing letters in the list
lista2rijec :: (Show a,Eq a) => [a] -> Rijec a
lista2rijec xs| even (length xs)= [Slovo x|x<-xs]
              | otherwise= error("The word "++show xs++" doesn't have an even number of letters.")

-- |for a word the function returns the list of letters in the word
rijec2lista :: Eq a => Rijec a -> [a]
rijec2lista xs= [x|Slovo x<-xs]

elemAbc :: Eq a => Slovo a -> Abeceda a -> Bool
elemAbc s (Abeceda a) = elem s a

elemIndexAbc :: Eq a => Slovo a -> Abeceda a -> Maybe Int
elemIndexAbc s (Abeceda a) = elemIndex s a

sup :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
sup a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) > fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

inf :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
inf a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) < fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

-- |function converting two consecutive lowercase letters to an uppercase letter
zapisu :: String->String
zapisu []=[]
zapisu [x]=[x]
zapisu (x:y:xs)|isLower x && x==y = toUpper x:zapisu xs
               |otherwise=x:zapisu (y:xs)

-- |function converting uppercase letter to two consecutive lowercase letters
zapisl :: String->String
zapisl []=[]
zapisl (x:xs)|isUpper x =x':x':zapisl xs
             |otherwise =x:zapisl xs
             where x'=toLower x

-- string output of a word
class Show a => Ispisiv a where
    ispis :: Show a => Rijec a -> String
    ispis []=[]
    ispis [Slovo x]=show x
    ispis (Slovo x:xs)=show x++"-"++ispis xs

instance Show a => Ispisiv a
instance Ispisiv Char where
    ispis=zapisu.ispis'

ispis' :: Rijec Char -> String
ispis' []=[]
ispis' [Slovo x]=[x]
ispis' (Slovo x:xs)=x:ispis' xs

-- functions returning left and right letters of a word
l :: [Slovo a] -> Slovo a
l = head
d :: [Slovo a] -> Slovo a
d = last

-- |function returning a dual of a letter in an alphabet
dual :: Eq a => Abeceda a -> Slovo a -> Slovo a
dual (Abeceda []) s = error("Nema duala - slovo nije u abecedi.")
dual (Abeceda [c]) s | s==c = c
                     | otherwise = error("No dual - Letter is not from the alphabet.")
dual (Abeceda cs) s | s==lijevi = desni
                    | s==desni = lijevi
                    | otherwise = dual (Abeceda (tail (init cs))) s
                     where lijevi= l cs
                           desni= d cs

-- functions returning duals of left/right letters of a word
l' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
l' a= (dual a).l
d' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
d' a= (dual a).d

ljuska :: [Slovo a]->[Slovo a]
ljuska xs=[l xs,d xs]


-- |distance between letters in an alphabet
dd :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Int
dd a x y| m==Nothing || n==Nothing = error("Cannot calculate distance. Letter is not from the alphabet.")
        | otherwise = abs (fromJust m-fromJust n)
  where m=x `elemIndexAbc` a
        n=y `elemIndexAbc` a

-- |word length
duljina :: Eq a => Abeceda a -> Rijec a -> Int
duljina a []=0
duljina a [c]=0
duljina a (c1:c2:cs)= - dd a c1 c2 - duljina a (c2:cs)

-- word shortening
krati1 :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function shortening a 4 letter word
krati1 a [p,r,s,t] | duljina a [p,r,s,t]==duljina a [p,t] = [p,t]
                 | otherwise = [p,r,s,t]
krati1 _ cs = cs

krati2 :: Eq a => Abeceda a -> Rijec a -> Rijec a     -- function shortening a 4 letter sub-word
krati2 _ [] = []
krati2 a(c:cs) | length kr==2 = kr ++ drop 3 cs
               | otherwise = c : krati2 a cs
               where kr = ((krati1 a).(take 4)) (c:cs)

krati :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function fully shortening a word
krati a cs | cs/=kr = krati a kr
           | otherwise = cs
           where kr = krati2 a cs

-- symmetries
opD :: Eq a => Abeceda a -> Rijec a -> Rijec a
opD _ []=[]
opD a (s:t:ys)=dual a s:dual a t:opD a ys

opE :: Eq a => Abeceda a -> Rijec a -> Rijec a
opE _ []=[]
opE a xs=t:s:opE a ys
        where n=length xs
              t=d xs
              s=xs !! (n-2)
              ys=take (n-2) xs
--opE a (s:t:ys)=opE a ys++[t,s]    --kraće, ali nije po formuli iz teksta

opF :: Eq a => Abeceda a -> Rijec a -> Rijec a
opF a=(opD a).(opE a)

-- permutations
opI :: Eq a => Abeceda a -> Rijec a -> Rijec a
opI a xs=l xs:xs++[d xs]

-- |negation operator
opK :: Eq a => Abeceda a -> Rijec a -> Rijec a
opK a xs=l' a xs:xs++[d' a xs]

opG :: Eq a => Abeceda a -> Rijec a -> Rijec a
opG a xs=d xs:xs++[l xs]

opH :: Eq a => Abeceda a -> Rijec a -> Rijec a
opH a xs=d' a xs:xs++[l' a xs]

-- serial and parallel conjecture operators
opS :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opS a xs ys = inf a (l xs) (l ys) : inf a (l xs) (l ys) : xs ++
              [sup a (d xs) (l ys),inf a (d xs) (l ys)] ++ ys ++
              [sup a (d xs) (d ys),sup a (d xs) (d ys)]

opP :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opP a xs ys = sup a (l xs) (l ys) : sup a (l xs) (l ys) : xs ++
              [inf a (d xs) (l ys),sup a (d xs) (l ys)] ++ ys ++
              [inf a (d xs) (d ys),inf a (d xs) (d ys)]

-- |partitions
particija :: Eq a => Abeceda a -> Rijec a -> Char
particija a xs | lijevi>=*lijevi' && desni<=*desni' && lijevi>*desni ='T'
               | lijevi<=*lijevi' && desni<=*desni' && not (lijevi==lijevi' || desni==desni') ='A'
               | lijevi==lijevi' && desni==desni' ='0'
               | lijevi>=*lijevi' && desni>=*desni' && not (lijevi==lijevi' || desni==desni')  ='Z'
               | lijevi<=*lijevi' && desni>=*desni' && lijevi<*desni  ='F'
         where lijevi=l xs
               desni=d xs
               lijevi'=l' a xs
               desni'=d' a xs
               x >* y| m==Nothing || n==Nothing =error("Letter not in alphabet.")
                     | otherwise = (fromJust m) > (fromJust n)
                      where m=x `elemIndexAbc` a
                            n=y `elemIndexAbc` a
               x >=* y = x >* y || x==y
               x <* y = not (x >=* y)
               x <=* y = not (x >* y)

-- implications
pimp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pimp a x y=(opK a x) `opP'` y
    where opP'=opP a

simp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
simp a x y=(opK a x) `opS'` y
    where opS'=opS a

-- equivalencies
pekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pekv a x y=(x `pimp'` y) `opS'` (y `pimp'` x)
    where opS'=opS a
          pimp'=pimp a

sekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
sekv a x y=(x `simp'` y) `opP'` (y `simp'` x)
    where opP'=opP a
          simp'=simp a

-- |tautologies and rules
taut :: Eq a => Abeceda a -> Rijec a ->Rijec a -> Rijec a -> [(String,Char)]
taut a x y z= zip
  ["1.ZAKON ISKLJUCENJA TRECEG",
  "2.DUAL ZAKONA ISKLJUCENJA TRECEG",
  "3.TRANZITIVNOST IMPLIKACIJE",
  "4.DUAL TRANZITIVNOSTI IMPLIKACIJE",
  "5.ZAKON DVOSTRUKE NEGACIJE",
  "6.ZAKON KONTRAPOZICIJE",
  "7.DUAL ZAKONA KONTRAPOZICIJE",
  "8.ZAKON APSORPCIJE ILI UPIJANJA",
  "9.DUAL ZAKONA APSORPCIJE ILI UPIJANJA",
  "10.MODUS PONENDO PONENS",
  "11.DUAL MODUSA PONENDO PONENS",
  "12.MODUS TOLENDO TOLLENS",
  "13.DOKAZ NABRAJANJEM",
  "14.DOVODJENJE DO PROTURJECJA"]
  (map (particija a) [
    x `opP'` (neg x),
    x `opS'` (neg x),
    ((x `pimp'` y) `opS'` (y `pimp'` z)) `pimp'` (x `pimp'` z),
    ((x `simp'` y) `opP'` (y `simp'` z)) `simp'` (x `simp'` z),
    (neg (neg x)) `pekv'` x,
    (x `pimp'` y) `pimp'` ((neg y) `pimp'` (neg x)),
    (x `simp'` y) `simp'` ((neg y) `simp'` (neg x)),
    (x `opP'` (y `opP'` x)) `pimp'` x,
    (x `opS'` (y `opS'` x)) `simp'` x,
    (x `opS'` (x `pimp'` y)) `pimp'` y,
    (x `opP'` (x `simp'` y)) `simp'` y,
    ((neg x) `opS'` ((neg x) `pimp'` (neg y))) `pimp'` (neg y),
    ((x `pimp'` z) `opS'` (y `pimp'` z)) `pimp'` ((x `opP'` y) `pimp'` z),
    ((x `pimp'` y) `opS'` (x `pimp'` (neg y))) `pimp'` (neg x)
    ]
  )
  where opP'=opP a
        opS'=opS a
        pimp'=pimp a
        simp'=simp a
        pekv'=pekv a
        sekv'=sekv a
        neg=opK a
        
{-# START_FILE Main.hs #-}
module Main where
import Linguistic2

main :: IO ()

-- show Alphabet
main=do let {abc=lista2abeceda ['a'..'z']}         -- <-- CHANGE
-- /show


-- show Words
        let {x=lista2rijec "croatian"; y=lista2rijec "language"}  -- <-- CHANGE
-- /show


-- show Symmetry operators D, E and F
        putStr ("D("++ispis x++")= "++ ispis (opD abc x))
        putStrLn (" =zip= "++ (ispis.(krati abc)) (opD abc x))
        putStr ("E("++ispis x++")= "++ ispis (opE abc x))
        putStrLn (" =zip= "++ (ispis.(krati abc)) (opE abc x))
        putStr ("E("++ispis y++")= "++ ispis (opE abc y))
        putStrLn (" =zip= "++ (ispis.(krati abc)) (opE abc y))
        putStr ("F("++ispis x++")= "++ ispis (opF abc x))
        putStrLn (" =zip= "++ (ispis.(krati abc)) (opF abc x))
-- /show


-- show Symmetry operator E applied to words concatenation
        putStr ("E("++ispis (x++y)++")= "++ ispis (opE abc (x++y)))
        putStrLn (" =zip= "++ (ispis.(krati abc)) (opE abc (x++y)))
-- /show


-- show permutation operators I, K, G and H
        putStr ("I("++ispis x++")= "++ ispis (opI abc x))
        putStrLn (" =zip= "++ (ispis.(krati abc)) (opI abc x))
        putStr ("K("++ispis x++")= "++ ispis (opK abc x))
        putStrLn (" =zip= "++ (ispis.(krati abc)) (opK abc x))
        putStr ("G("++ispis x++")= "++ ispis (opG abc x))
        putStrLn (" =zip= "++ (ispis.(krati abc)) (opG abc x))
        putStr ("H("++ispis x++")= "++ ispis (opH abc x))
        putStrLn (" =zip= "++ (ispis.(krati abc)) (opH abc x))
-- /show


--  0-inverzivnost
--      putStr ("E(G("++ispis y++"))= "++ ispis (opE abc (opG abc y)))
--      putStrLn (" =zip= "++ (ispis.(krati abc)) (opE abc (opG abc y)))
--      putStr ("G(E("++ispis y++"))= "++ ispis (opG abc (opE abc y)))
--      putStrLn (" =zip= "++ (ispis.(krati abc)) (opG abc (opE abc y)))
```

## Example 4
Let the set of symbols be the English alphabet _{a,..., z}_.
Let _x = croatian = croian_  and _y=language = le_. Then the serial and parallel connections are given by:

``` active haskell
{-# START_FILE Linguistic2.hs #-}
-- show
-- /show
{-# LANGUAGE UndecidableInstances, FlexibleInstances, OverlappingInstances, IncoherentInstances #-}
module Linguistic2 where

import           Data.Char  (chr, isLower, isUpper, ord, toLower, toUpper)
import           Data.List  (elemIndex, nub)
import           Data.Maybe (fromJust)

-- alphabet type definition and constructors
data Slovo a = Slovo a deriving (Eq, Show)
data Abeceda a = Abeceda {unAbeceda :: [Slovo a]} deriving (Eq, Show)
type Rijec a = [Slovo a]

-- |for a list the function constructs the alphabet containing unique elements of the list
lista2abeceda :: Eq a => [a] -> Abeceda a
lista2abeceda xs=Abeceda [Slovo x|x<-nub xs]

-- |for 2 Enum types (like Char) the function constructs the alphabet from first to second one
enum2abeceda :: (Eq a, Enum a) => a -> a -> Abeceda a
enum2abeceda a z=lista2abeceda [a..z]

-- |for a list the function constructs the word containing letters in the list
lista2rijec :: (Show a,Eq a) => [a] -> Rijec a
lista2rijec xs| even (length xs)= [Slovo x|x<-xs]
              | otherwise= error("The word "++show xs++" doesn't have an even number of letters.")

-- |for a word the function returns the list of letters in the word
rijec2lista :: Eq a => Rijec a -> [a]
rijec2lista xs= [x|Slovo x<-xs]

elemAbc :: Eq a => Slovo a -> Abeceda a -> Bool
elemAbc s (Abeceda a) = elem s a

elemIndexAbc :: Eq a => Slovo a -> Abeceda a -> Maybe Int
elemIndexAbc s (Abeceda a) = elemIndex s a

sup :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
sup a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) > fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

inf :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
inf a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) < fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

-- |function converting two consecutive lowercase letters to an uppercase letter
zapisu :: String->String
zapisu []=[]
zapisu [x]=[x]
zapisu (x:y:xs)|isLower x && x==y = toUpper x:zapisu xs
               |otherwise=x:zapisu (y:xs)

-- |function converting uppercase letter to two consecutive lowercase letters
zapisl :: String->String
zapisl []=[]
zapisl (x:xs)|isUpper x =x':x':zapisl xs
             |otherwise =x:zapisl xs
             where x'=toLower x

-- string output of a word
class Show a => Ispisiv a where
    ispis :: Show a => Rijec a -> String
    ispis []=[]
    ispis [Slovo x]=show x
    ispis (Slovo x:xs)=show x++"-"++ispis xs

instance Show a => Ispisiv a
instance Ispisiv Char where
    ispis=zapisu.ispis'

ispis' :: Rijec Char -> String
ispis' []=[]
ispis' [Slovo x]=[x]
ispis' (Slovo x:xs)=x:ispis' xs

-- functions returning left and right letters of a word
l :: [Slovo a] -> Slovo a
l = head
d :: [Slovo a] -> Slovo a
d = last

-- |function returning a dual of a letter in an alphabet
dual :: Eq a => Abeceda a -> Slovo a -> Slovo a
dual (Abeceda []) s = error("Nema duala - slovo nije u abecedi.")
dual (Abeceda [c]) s | s==c = c
                     | otherwise = error("No dual - Letter is not from the alphabet.")
dual (Abeceda cs) s | s==lijevi = desni
                    | s==desni = lijevi
                    | otherwise = dual (Abeceda (tail (init cs))) s
                     where lijevi= l cs
                           desni= d cs

-- functions returning duals of left/right letters of a word
l' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
l' a= (dual a).l
d' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
d' a= (dual a).d

ljuska :: [Slovo a]->[Slovo a]
ljuska xs=[l xs,d xs]


-- |distance between letters in an alphabet
dd :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Int
dd a x y| m==Nothing || n==Nothing = error("Cannot calculate distance. Letter is not from the alphabet.")
        | otherwise = abs (fromJust m-fromJust n)
  where m=x `elemIndexAbc` a
        n=y `elemIndexAbc` a

-- |word length
duljina :: Eq a => Abeceda a -> Rijec a -> Int
duljina a []=0
duljina a [c]=0
duljina a (c1:c2:cs)= - dd a c1 c2 - duljina a (c2:cs)

-- word shortening
krati1 :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function shortening a 4 letter word
krati1 a [p,r,s,t] | duljina a [p,r,s,t]==duljina a [p,t] = [p,t]
                 | otherwise = [p,r,s,t]
krati1 _ cs = cs

krati2 :: Eq a => Abeceda a -> Rijec a -> Rijec a     -- function shortening a 4 letter sub-word
krati2 _ [] = []
krati2 a(c:cs) | length kr==2 = kr ++ drop 3 cs
               | otherwise = c : krati2 a cs
               where kr = ((krati1 a).(take 4)) (c:cs)

krati :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function fully shortening a word
krati a cs | cs/=kr = krati a kr
           | otherwise = cs
           where kr = krati2 a cs

-- symmetries
opD :: Eq a => Abeceda a -> Rijec a -> Rijec a
opD _ []=[]
opD a (s:t:ys)=dual a s:dual a t:opD a ys

opE :: Eq a => Abeceda a -> Rijec a -> Rijec a
opE _ []=[]
opE a xs=t:s:opE a ys
        where n=length xs
              t=d xs
              s=xs !! (n-2)
              ys=take (n-2) xs
--opE a (s:t:ys)=opE a ys++[t,s]    --kraće, ali nije po formuli iz teksta

opF :: Eq a => Abeceda a -> Rijec a -> Rijec a
opF a=(opD a).(opE a)

-- permutations
opI :: Eq a => Abeceda a -> Rijec a -> Rijec a
opI a xs=l xs:xs++[d xs]

-- |negation operator
opK :: Eq a => Abeceda a -> Rijec a -> Rijec a
opK a xs=l' a xs:xs++[d' a xs]

opG :: Eq a => Abeceda a -> Rijec a -> Rijec a
opG a xs=d xs:xs++[l xs]

opH :: Eq a => Abeceda a -> Rijec a -> Rijec a
opH a xs=d' a xs:xs++[l' a xs]

-- serial and parallel conjecture operators
opS :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opS a xs ys = inf a (l xs) (l ys) : inf a (l xs) (l ys) : xs ++
              [sup a (d xs) (l ys),inf a (d xs) (l ys)] ++ ys ++
              [sup a (d xs) (d ys),sup a (d xs) (d ys)]

opP :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opP a xs ys = sup a (l xs) (l ys) : sup a (l xs) (l ys) : xs ++
              [inf a (d xs) (l ys),sup a (d xs) (l ys)] ++ ys ++
              [inf a (d xs) (d ys),inf a (d xs) (d ys)]

-- |partitions
particija :: Eq a => Abeceda a -> Rijec a -> Char
particija a xs | lijevi>=*lijevi' && desni<=*desni' && lijevi>*desni ='T'
               | lijevi<=*lijevi' && desni<=*desni' && not (lijevi==lijevi' || desni==desni') ='A'
               | lijevi==lijevi' && desni==desni' ='0'
               | lijevi>=*lijevi' && desni>=*desni' && not (lijevi==lijevi' || desni==desni')  ='Z'
               | lijevi<=*lijevi' && desni>=*desni' && lijevi<*desni  ='F'
         where lijevi=l xs
               desni=d xs
               lijevi'=l' a xs
               desni'=d' a xs
               x >* y| m==Nothing || n==Nothing =error("Letter not in alphabet.")
                     | otherwise = (fromJust m) > (fromJust n)
                      where m=x `elemIndexAbc` a
                            n=y `elemIndexAbc` a
               x >=* y = x >* y || x==y
               x <* y = not (x >=* y)
               x <=* y = not (x >* y)

-- implications
pimp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pimp a x y=(opK a x) `opP'` y
    where opP'=opP a

simp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
simp a x y=(opK a x) `opS'` y
    where opS'=opS a

-- equivalencies
pekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pekv a x y=(x `pimp'` y) `opS'` (y `pimp'` x)
    where opS'=opS a
          pimp'=pimp a

sekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
sekv a x y=(x `simp'` y) `opP'` (y `simp'` x)
    where opP'=opP a
          simp'=simp a

-- |tautologies and rules
taut :: Eq a => Abeceda a -> Rijec a ->Rijec a -> Rijec a -> [(String,Char)]
taut a x y z= zip
  ["1.ZAKON ISKLJUCENJA TRECEG",
  "2.DUAL ZAKONA ISKLJUCENJA TRECEG",
  "3.TRANZITIVNOST IMPLIKACIJE",
  "4.DUAL TRANZITIVNOSTI IMPLIKACIJE",
  "5.ZAKON DVOSTRUKE NEGACIJE",
  "6.ZAKON KONTRAPOZICIJE",
  "7.DUAL ZAKONA KONTRAPOZICIJE",
  "8.ZAKON APSORPCIJE ILI UPIJANJA",
  "9.DUAL ZAKONA APSORPCIJE ILI UPIJANJA",
  "10.MODUS PONENDO PONENS",
  "11.DUAL MODUSA PONENDO PONENS",
  "12.MODUS TOLENDO TOLLENS",
  "13.DOKAZ NABRAJANJEM",
  "14.DOVODJENJE DO PROTURJECJA"]
  (map (particija a) [
    x `opP'` (neg x),
    x `opS'` (neg x),
    ((x `pimp'` y) `opS'` (y `pimp'` z)) `pimp'` (x `pimp'` z),
    ((x `simp'` y) `opP'` (y `simp'` z)) `simp'` (x `simp'` z),
    (neg (neg x)) `pekv'` x,
    (x `pimp'` y) `pimp'` ((neg y) `pimp'` (neg x)),
    (x `simp'` y) `simp'` ((neg y) `simp'` (neg x)),
    (x `opP'` (y `opP'` x)) `pimp'` x,
    (x `opS'` (y `opS'` x)) `simp'` x,
    (x `opS'` (x `pimp'` y)) `pimp'` y,
    (x `opP'` (x `simp'` y)) `simp'` y,
    ((neg x) `opS'` ((neg x) `pimp'` (neg y))) `pimp'` (neg y),
    ((x `pimp'` z) `opS'` (y `pimp'` z)) `pimp'` ((x `opP'` y) `pimp'` z),
    ((x `pimp'` y) `opS'` (x `pimp'` (neg y))) `pimp'` (neg x)
    ]
  )
  where opP'=opP a
        opS'=opS a
        pimp'=pimp a
        simp'=simp a
        pekv'=pekv a
        sekv'=sekv a
        neg=opK a
        
{-# START_FILE Main.hs #-}
module Main where
import Linguistic2

main :: IO ()

-- show Alphabet
main=do let {abc=lista2abeceda ['a'..'z']}         -- <-- CHANGE
-- /show


-- show Words
        let {x=lista2rijec "croatian"; y=lista2rijec "language"}   -- <-- CHANGE
-- /show

        let opPa=opP abc; opSa=opS abc

-- show M-conjunction (serial connection, `AND`)
        putStr (ispis x++" `AND` "++ ispis y)
        putStr (" = "++ ispis (x `opSa` y))
        putStrLn (" =zip= "++ (ispis.(krati abc)) (x `opSa` y))
        putStr (ispis y++" `AND` "++ ispis x)
        putStr (" = "++ ispis (y `opSa` x))
        putStrLn (" =zip= "++ (ispis.(krati abc)) (y `opSa` x))
-- /show


-- show M-disjunction (parallel connection, `OR`)
        putStr (ispis x++" `OR` "++ ispis y)
        putStr (" = "++ ispis (x `opPa` y))
        putStr (" =zip= "++ (ispis.(krati abc)) (x `opPa` y))
        putStrLn (" =len= "++ show (duljina abc (x `opPa` y)))
        putStr (ispis y++" `OR` "++ ispis x)
        putStr (" = "++ ispis (y `opPa` x))
        putStr (" =zip= "++ (ispis.(krati abc)) (y `opPa` x))
        putStrLn (" =len= "++ show (duljina abc (y `opPa` x)))
-- /show
```

## Example 5
For alphabet _{1,2,3,4,5}_ partition of base has the following form:

``` active haskell
{-# START_FILE Linguistic2.hs #-}
-- show
-- /show
{-# LANGUAGE UndecidableInstances, FlexibleInstances, OverlappingInstances, IncoherentInstances #-}
module Linguistic2 where

import           Data.Char  (chr, isLower, isUpper, ord, toLower, toUpper)
import           Data.List  (elemIndex, nub)
import           Data.Maybe (fromJust)

-- alphabet type definition and constructors
data Slovo a = Slovo a deriving (Eq, Show)
data Abeceda a = Abeceda {unAbeceda :: [Slovo a]} deriving (Eq, Show)
type Rijec a = [Slovo a]

-- |for a list the function constructs the alphabet containing unique elements of the list
lista2abeceda :: Eq a => [a] -> Abeceda a
lista2abeceda xs=Abeceda [Slovo x|x<-nub xs]

-- |for 2 Enum types (like Char) the function constructs the alphabet from first to second one
enum2abeceda :: (Eq a, Enum a) => a -> a -> Abeceda a
enum2abeceda a z=lista2abeceda [a..z]

-- |for a list the function constructs the word containing letters in the list
lista2rijec :: (Show a,Eq a) => [a] -> Rijec a
lista2rijec xs| even (length xs)= [Slovo x|x<-xs]
              | otherwise= error("The word "++show xs++" doesn't have an even number of letters.")

-- |for a word the function returns the list of letters in the word
rijec2lista :: Eq a => Rijec a -> [a]
rijec2lista xs= [x|Slovo x<-xs]

elemAbc :: Eq a => Slovo a -> Abeceda a -> Bool
elemAbc s (Abeceda a) = elem s a

elemIndexAbc :: Eq a => Slovo a -> Abeceda a -> Maybe Int
elemIndexAbc s (Abeceda a) = elemIndex s a

sup :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
sup a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) > fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

inf :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
inf a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) < fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

-- |function converting two consecutive lowercase letters to an uppercase letter
zapisu :: String->String
zapisu []=[]
zapisu [x]=[x]
zapisu (x:y:xs)|isLower x && x==y = toUpper x:zapisu xs
               |otherwise=x:zapisu (y:xs)

-- |function converting uppercase letter to two consecutive lowercase letters
zapisl :: String->String
zapisl []=[]
zapisl (x:xs)|isUpper x =x':x':zapisl xs
             |otherwise =x:zapisl xs
             where x'=toLower x

-- string output of a word
class Show a => Ispisiv a where
    ispis :: Show a => Rijec a -> String
    ispis []=[]
    ispis [Slovo x]=show x
    ispis (Slovo x:xs)=show x++"-"++ispis xs

instance Show a => Ispisiv a
instance Ispisiv Char where
    ispis=zapisu.ispis'

ispis' :: Rijec Char -> String
ispis' []=[]
ispis' [Slovo x]=[x]
ispis' (Slovo x:xs)=x:ispis' xs

-- functions returning left and right letters of a word
l :: [Slovo a] -> Slovo a
l = head
d :: [Slovo a] -> Slovo a
d = last

-- |function returning a dual of a letter in an alphabet
dual :: Eq a => Abeceda a -> Slovo a -> Slovo a
dual (Abeceda []) s = error("Nema duala - slovo nije u abecedi.")
dual (Abeceda [c]) s | s==c = c
                     | otherwise = error("No dual - Letter is not from the alphabet.")
dual (Abeceda cs) s | s==lijevi = desni
                    | s==desni = lijevi
                    | otherwise = dual (Abeceda (tail (init cs))) s
                     where lijevi= l cs
                           desni= d cs

-- functions returning duals of left/right letters of a word
l' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
l' a= (dual a).l
d' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
d' a= (dual a).d

ljuska :: [Slovo a]->[Slovo a]
ljuska xs=[l xs,d xs]


-- |distance between letters in an alphabet
dd :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Int
dd a x y| m==Nothing || n==Nothing = error("Cannot calculate distance. Letter is not from the alphabet.")
        | otherwise = abs (fromJust m-fromJust n)
  where m=x `elemIndexAbc` a
        n=y `elemIndexAbc` a

-- |word length
duljina :: Eq a => Abeceda a -> Rijec a -> Int
duljina a []=0
duljina a [c]=0
duljina a (c1:c2:cs)= - dd a c1 c2 - duljina a (c2:cs)

-- word shortening
krati1 :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function shortening a 4 letter word
krati1 a [p,r,s,t] | duljina a [p,r,s,t]==duljina a [p,t] = [p,t]
                 | otherwise = [p,r,s,t]
krati1 _ cs = cs

krati2 :: Eq a => Abeceda a -> Rijec a -> Rijec a     -- function shortening a 4 letter sub-word
krati2 _ [] = []
krati2 a(c:cs) | length kr==2 = kr ++ drop 3 cs
               | otherwise = c : krati2 a cs
               where kr = ((krati1 a).(take 4)) (c:cs)

krati :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function fully shortening a word
krati a cs | cs/=kr = krati a kr
           | otherwise = cs
           where kr = krati2 a cs

-- symmetries
opD :: Eq a => Abeceda a -> Rijec a -> Rijec a
opD _ []=[]
opD a (s:t:ys)=dual a s:dual a t:opD a ys

opE :: Eq a => Abeceda a -> Rijec a -> Rijec a
opE _ []=[]
opE a xs=t:s:opE a ys
        where n=length xs
              t=d xs
              s=xs !! (n-2)
              ys=take (n-2) xs
--opE a (s:t:ys)=opE a ys++[t,s]    --kraće, ali nije po formuli iz teksta

opF :: Eq a => Abeceda a -> Rijec a -> Rijec a
opF a=(opD a).(opE a)

-- permutations
opI :: Eq a => Abeceda a -> Rijec a -> Rijec a
opI a xs=l xs:xs++[d xs]

-- |negation operator
opK :: Eq a => Abeceda a -> Rijec a -> Rijec a
opK a xs=l' a xs:xs++[d' a xs]

opG :: Eq a => Abeceda a -> Rijec a -> Rijec a
opG a xs=d xs:xs++[l xs]

opH :: Eq a => Abeceda a -> Rijec a -> Rijec a
opH a xs=d' a xs:xs++[l' a xs]

-- serial and parallel conjecture operators
opS :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opS a xs ys = inf a (l xs) (l ys) : inf a (l xs) (l ys) : xs ++
              [sup a (d xs) (l ys),inf a (d xs) (l ys)] ++ ys ++
              [sup a (d xs) (d ys),sup a (d xs) (d ys)]

opP :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opP a xs ys = sup a (l xs) (l ys) : sup a (l xs) (l ys) : xs ++
              [inf a (d xs) (l ys),sup a (d xs) (l ys)] ++ ys ++
              [inf a (d xs) (d ys),inf a (d xs) (d ys)]

-- |partitions
particija :: Eq a => Abeceda a -> Rijec a -> Char
particija a xs | lijevi>=*lijevi' && desni<=*desni' && lijevi>*desni ='T'
               | lijevi<=*lijevi' && desni<=*desni' && not (lijevi==lijevi' || desni==desni') ='A'
               | lijevi==lijevi' && desni==desni' ='0'
               | lijevi>=*lijevi' && desni>=*desni' && not (lijevi==lijevi' || desni==desni')  ='Z'
               | lijevi<=*lijevi' && desni>=*desni' && lijevi<*desni  ='F'
         where lijevi=l xs
               desni=d xs
               lijevi'=l' a xs
               desni'=d' a xs
               x >* y| m==Nothing || n==Nothing =error("Letter not in alphabet.")
                     | otherwise = (fromJust m) > (fromJust n)
                      where m=x `elemIndexAbc` a
                            n=y `elemIndexAbc` a
               x >=* y = x >* y || x==y
               x <* y = not (x >=* y)
               x <=* y = not (x >* y)

-- implications
pimp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pimp a x y=(opK a x) `opP'` y
    where opP'=opP a

simp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
simp a x y=(opK a x) `opS'` y
    where opS'=opS a

-- equivalencies
pekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pekv a x y=(x `pimp'` y) `opS'` (y `pimp'` x)
    where opS'=opS a
          pimp'=pimp a

sekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
sekv a x y=(x `simp'` y) `opP'` (y `simp'` x)
    where opP'=opP a
          simp'=simp a

-- |tautologies and rules
taut :: Eq a => Abeceda a -> Rijec a ->Rijec a -> Rijec a -> [(String,Char)]
taut a x y z= zip
  ["1.ZAKON ISKLJUCENJA TRECEG",
  "2.DUAL ZAKONA ISKLJUCENJA TRECEG",
  "3.TRANZITIVNOST IMPLIKACIJE",
  "4.DUAL TRANZITIVNOSTI IMPLIKACIJE",
  "5.ZAKON DVOSTRUKE NEGACIJE",
  "6.ZAKON KONTRAPOZICIJE",
  "7.DUAL ZAKONA KONTRAPOZICIJE",
  "8.ZAKON APSORPCIJE ILI UPIJANJA",
  "9.DUAL ZAKONA APSORPCIJE ILI UPIJANJA",
  "10.MODUS PONENDO PONENS",
  "11.DUAL MODUSA PONENDO PONENS",
  "12.MODUS TOLENDO TOLLENS",
  "13.DOKAZ NABRAJANJEM",
  "14.DOVODJENJE DO PROTURJECJA"]
  (map (particija a) [
    x `opP'` (neg x),
    x `opS'` (neg x),
    ((x `pimp'` y) `opS'` (y `pimp'` z)) `pimp'` (x `pimp'` z),
    ((x `simp'` y) `opP'` (y `simp'` z)) `simp'` (x `simp'` z),
    (neg (neg x)) `pekv'` x,
    (x `pimp'` y) `pimp'` ((neg y) `pimp'` (neg x)),
    (x `simp'` y) `simp'` ((neg y) `simp'` (neg x)),
    (x `opP'` (y `opP'` x)) `pimp'` x,
    (x `opS'` (y `opS'` x)) `simp'` x,
    (x `opS'` (x `pimp'` y)) `pimp'` y,
    (x `opP'` (x `simp'` y)) `simp'` y,
    ((neg x) `opS'` ((neg x) `pimp'` (neg y))) `pimp'` (neg y),
    ((x `pimp'` z) `opS'` (y `pimp'` z)) `pimp'` ((x `opP'` y) `pimp'` z),
    ((x `pimp'` y) `opS'` (x `pimp'` (neg y))) `pimp'` (neg x)
    ]
  )
  where opP'=opP a
        opS'=opS a
        pimp'=pimp a
        simp'=simp a
        pekv'=pekv a
        sekv'=sekv a
        neg=opK a
        
{-# START_FILE Main.hs #-}
module Main where
import Linguistic2

main :: IO ()

-- show Alphabet
main=do let {abc=lista2abeceda ([1..5]::[Int]);}   -- <-- CHANGE
-- /show


-- show Base
        let {tab=[([x,y],particija abc [x,y])|x <- unAbeceda abc,y <- unAbeceda abc];}
-- /show


-- show Partitioning of base
        putStrLn (filter (/='"') (concat [p:": "++show [ispis x|(x,y)<-tab,y==p]++"\n"|p<-['T','A','0','Z','F']]))
-- /show
```

## Example 6
Let _x=robin_, _y=chicken_, _z=penguin_, _w=cow_ be animal species, where the first three come from bird species and the last one from mammals. By the evidence of their belonging to the species of birds, we could assign _ca_ to _x_, _ba_ to _y_, _aa_ to _z_, while for _w_, we can assign the biggest untruth _ac_ from 3-generator M-system.
The truths of certain compound statements are:

``` active haskell
{-# START_FILE Linguistic2.hs #-}
-- show
-- /show
{-# LANGUAGE UndecidableInstances, FlexibleInstances, OverlappingInstances, IncoherentInstances #-}
module Linguistic2 where

import           Data.Char  (chr, isLower, isUpper, ord, toLower, toUpper)
import           Data.List  (elemIndex, nub)
import           Data.Maybe (fromJust)

-- alphabet type definition and constructors
data Slovo a = Slovo {unSlovo :: a} deriving (Eq, Show)
data Abeceda a = Abeceda {unAbeceda :: [Slovo a]} deriving (Eq, Show)
type Rijec a = [Slovo a]

-- |for a list the function constructs the alphabet containing unique elements of the list
lista2abeceda :: Eq a => [a] -> Abeceda a
lista2abeceda xs=Abeceda [Slovo x|x<-nub xs]

-- |for 2 Enum types (like Char) the function constructs the alphabet from first to second one
enum2abeceda :: (Eq a, Enum a) => a -> a -> Abeceda a
enum2abeceda a z=lista2abeceda [a..z]

-- |for a list the function constructs the word containing letters in the list
lista2rijec :: (Show a,Eq a) => [a] -> Rijec a
lista2rijec xs| even (length xs)= [Slovo x|x<-xs]
              | otherwise= error("The word "++show xs++" doesn't have an even number of letters.")

-- |for a word the function returns the list of letters in the word
rijec2lista :: Eq a => Rijec a -> [a]
rijec2lista xs= [x|Slovo x<-xs]

elemAbc :: Eq a => Slovo a -> Abeceda a -> Bool
elemAbc s (Abeceda a) = elem s a

elemIndexAbc :: Eq a => Slovo a -> Abeceda a -> Maybe Int
elemIndexAbc s (Abeceda a) = elemIndex s a

sup :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
sup a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) > fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

inf :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
inf a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) < fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

-- |function converting two consecutive lowercase letters to an uppercase letter
zapisu :: String->String
zapisu []=[]
zapisu [x]=[x]
zapisu (x:y:xs)|isLower x && x==y = toUpper x:zapisu xs
               |otherwise=x:zapisu (y:xs)

-- |function converting uppercase letter to two consecutive lowercase letters
zapisl :: String->String
zapisl []=[]
zapisl (x:xs)|isUpper x =x':x':zapisl xs
             |otherwise =x:zapisl xs
             where x'=toLower x

-- string output of a word
class Show a => Ispisiv a where
    ispis :: Show a => Rijec a -> String
    ispis []=[]
    ispis [Slovo x]=show x
    ispis (Slovo x:xs)=show x++"-"++ispis xs

instance Show a => Ispisiv a
instance Ispisiv Char where
    ispis=zapisu.ispis'

ispis' :: Rijec Char -> String
ispis' []=[]
ispis' [Slovo x]=[x]
ispis' (Slovo x:xs)=x:ispis' xs

-- functions returning left and right letters of a word
l :: [Slovo a] -> Slovo a
l = head
d :: [Slovo a] -> Slovo a
d = last

-- |function returning a dual of a letter in an alphabet
dual :: Eq a => Abeceda a -> Slovo a -> Slovo a
dual (Abeceda []) s = error("Nema duala - slovo nije u abecedi.")
dual (Abeceda [c]) s | s==c = c
                     | otherwise = error("No dual - Letter is not from the alphabet.")
dual (Abeceda cs) s | s==lijevi = desni
                    | s==desni = lijevi
                    | otherwise = dual (Abeceda (tail (init cs))) s
                     where lijevi= l cs
                           desni= d cs

-- functions returning duals of left/right letters of a word
l' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
l' a= (dual a).l
d' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
d' a= (dual a).d

ljuska :: [Slovo a]->[Slovo a]
ljuska xs=[l xs,d xs]


-- |distance between letters in an alphabet
dd :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Int
dd a x y| m==Nothing || n==Nothing = error("Cannot calculate distance. Letter is not from the alphabet.")
        | otherwise = abs (fromJust m-fromJust n)
  where m=x `elemIndexAbc` a
        n=y `elemIndexAbc` a

-- |word length
duljina :: Eq a => Abeceda a -> Rijec a -> Int
duljina a []=0
duljina a [c]=0
duljina a (c1:c2:cs)= - dd a c1 c2 - duljina a (c2:cs)

-- word shortening
krati1 :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function shortening a 4 letter word
krati1 a [p,r,s,t] | duljina a [p,r,s,t]==duljina a [p,t] = [p,t]
                 | otherwise = [p,r,s,t]
krati1 _ cs = cs

krati2 :: Eq a => Abeceda a -> Rijec a -> Rijec a     -- function shortening a 4 letter sub-word
krati2 _ [] = []
krati2 a(c:cs) | length kr==2 = kr ++ drop 3 cs
               | otherwise = c : krati2 a cs
               where kr = ((krati1 a).(take 4)) (c:cs)

krati :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function fully shortening a word
krati a cs | cs/=kr = krati a kr
           | otherwise = cs
           where kr = krati2 a cs

-- symmetries
opD :: Eq a => Abeceda a -> Rijec a -> Rijec a
opD _ []=[]
opD a (s:t:ys)=dual a s:dual a t:opD a ys

opE :: Eq a => Abeceda a -> Rijec a -> Rijec a
opE _ []=[]
opE a xs=t:s:opE a ys
        where n=length xs
              t=d xs
              s=xs !! (n-2)
              ys=take (n-2) xs
--opE a (s:t:ys)=opE a ys++[t,s]    --kraće, ali nije po formuli iz teksta

opF :: Eq a => Abeceda a -> Rijec a -> Rijec a
opF a=(opD a).(opE a)

-- permutations
opI :: Eq a => Abeceda a -> Rijec a -> Rijec a
opI a xs=l xs:xs++[d xs]

-- |negation operator
opK :: Eq a => Abeceda a -> Rijec a -> Rijec a
opK a xs=l' a xs:xs++[d' a xs]

opG :: Eq a => Abeceda a -> Rijec a -> Rijec a
opG a xs=d xs:xs++[l xs]

opH :: Eq a => Abeceda a -> Rijec a -> Rijec a
opH a xs=d' a xs:xs++[l' a xs]

-- serial and parallel conjecture operators
opS :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opS a xs ys = inf a (l xs) (l ys) : inf a (l xs) (l ys) : xs ++
              [sup a (d xs) (l ys),inf a (d xs) (l ys)] ++ ys ++
              [sup a (d xs) (d ys),sup a (d xs) (d ys)]

opP :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opP a xs ys = sup a (l xs) (l ys) : sup a (l xs) (l ys) : xs ++
              [inf a (d xs) (l ys),sup a (d xs) (l ys)] ++ ys ++
              [inf a (d xs) (d ys),inf a (d xs) (d ys)]

-- |partitions
particija :: Eq a => Abeceda a -> Rijec a -> Char
particija a xs | lijevi>=*lijevi' && desni<=*desni' && lijevi>*desni ='T'
               | lijevi<=*lijevi' && desni<=*desni' && not (lijevi==lijevi' || desni==desni') ='A'
               | lijevi==lijevi' && desni==desni' ='0'
               | lijevi>=*lijevi' && desni>=*desni' && not (lijevi==lijevi' || desni==desni')  ='Z'
               | lijevi<=*lijevi' && desni>=*desni' && lijevi<*desni  ='F'
         where lijevi=l xs
               desni=d xs
               lijevi'=l' a xs
               desni'=d' a xs
               x >* y| m==Nothing || n==Nothing =error("Letter not in alphabet.")
                     | otherwise = (fromJust m) > (fromJust n)
                      where m=x `elemIndexAbc` a
                            n=y `elemIndexAbc` a
               x >=* y = x >* y || x==y
               x <* y = not (x >=* y)
               x <=* y = not (x >* y)

-- implications
pimp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pimp a x y=(opK a x) `opP'` y
    where opP'=opP a

simp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
simp a x y=(opK a x) `opS'` y
    where opS'=opS a

-- equivalencies
pekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pekv a x y=(x `pimp'` y) `opS'` (y `pimp'` x)
    where opS'=opS a
          pimp'=pimp a

sekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
sekv a x y=(x `simp'` y) `opP'` (y `simp'` x)
    where opP'=opP a
          simp'=simp a

-- |tautologies and rules
taut :: Eq a => Abeceda a -> Rijec a ->Rijec a -> Rijec a -> [(String,Char)]
taut a x y z= zip
  ["1.ZAKON ISKLJUCENJA TRECEG",
  "2.DUAL ZAKONA ISKLJUCENJA TRECEG",
  "3.TRANZITIVNOST IMPLIKACIJE",
  "4.DUAL TRANZITIVNOSTI IMPLIKACIJE",
  "5.ZAKON DVOSTRUKE NEGACIJE",
  "6.ZAKON KONTRAPOZICIJE",
  "7.DUAL ZAKONA KONTRAPOZICIJE",
  "8.ZAKON APSORPCIJE ILI UPIJANJA",
  "9.DUAL ZAKONA APSORPCIJE ILI UPIJANJA",
  "10.MODUS PONENDO PONENS",
  "11.DUAL MODUSA PONENDO PONENS",
  "12.MODUS TOLENDO TOLLENS",
  "13.DOKAZ NABRAJANJEM",
  "14.DOVODJENJE DO PROTURJECJA"]
  (map (particija a) [
    x `opP'` (neg x),
    x `opS'` (neg x),
    ((x `pimp'` y) `opS'` (y `pimp'` z)) `pimp'` (x `pimp'` z),
    ((x `simp'` y) `opP'` (y `simp'` z)) `simp'` (x `simp'` z),
    (neg (neg x)) `pekv'` x,
    (x `pimp'` y) `pimp'` ((neg y) `pimp'` (neg x)),
    (x `simp'` y) `simp'` ((neg y) `simp'` (neg x)),
    (x `opP'` (y `opP'` x)) `pimp'` x,
    (x `opS'` (y `opS'` x)) `simp'` x,
    (x `opS'` (x `pimp'` y)) `pimp'` y,
    (x `opP'` (x `simp'` y)) `simp'` y,
    ((neg x) `opS'` ((neg x) `pimp'` (neg y))) `pimp'` (neg y),
    ((x `pimp'` z) `opS'` (y `pimp'` z)) `pimp'` ((x `opP'` y) `pimp'` z),
    ((x `pimp'` y) `opS'` (x `pimp'` (neg y))) `pimp'` (neg x)
    ]
  )
  where opP'=opP a
        opS'=opS a
        pimp'=pimp a
        simp'=simp a
        pekv'=pekv a
        sekv'=sekv a
        neg=opK a
        
{-# START_FILE Main.hs #-}
module Main where
import Linguistic2
import Data.Char (toUpper)

main :: IO ()

-- show Alphabet
main=do let {abc=lista2abeceda ['a'..'c']}         -- <-- CHANGE
-- /show


-- show Words
        let {x= "robin";
             y= "chicken";             -- <-- CHANGE
             z= "penguin";
             w= "cow";
             xx=lista2rijec "ca";
             yy=lista2rijec "ba";             -- <-- CHANGE
             zz=lista2rijec "aa";
             ww=lista2rijec "ac";}
-- /show

        let {opPa=opP abc; opSa=opS abc;
             xxx=head xx:(lista2rijec.zapisl) (map toUpper x)++[last xx];
             yyy=head yy:(lista2rijec.zapisl) (map toUpper y)++[last yy];
             zzz=head zz:(lista2rijec.zapisl) (map toUpper z)++[last zz];
             www=head ww:(lista2rijec.zapisl) (map toUpper w)++[last ww];}

-- show istinitost riječi
        putStrLn ("τ("++x++")= τ("++ispis xx++")= "++[particija abc xx])
        putStrLn ("τ("++y++")= τ("++ispis yy++")= "++[particija abc yy])
        putStrLn ("τ("++z++")= τ("++ispis zz++")= "++[particija abc zz])
        putStrLn ("τ("++w++")= τ("++ispis ww++")= "++[particija abc ww])
-- /show


-- show istinitost logičkih funkcija
        putStrLn ("τ("++x++"`AND`"++y++"`OR`"++w++")= τ("++ispis xx++"`AND`"++ispis yy++"`OR`"++ispis ww++")= τ("++(ispis.(krati abc)) (xx `opSa` yy)++"`OR`"++ispis ww++")= τ("++(ispis.(krati abc)) (xx `opSa` yy `opPa` ww)++")= "++[particija abc (xx `opSa` yy `opPa` ww)])
        putStrLn ("τ("++z++"`OR`"++y++"`AND`"++w++")= τ("++ispis zz++"`OR`"++ispis yy++"`AND`"++ispis ww++")= τ("++(ispis.(krati abc)) (zz `opPa` yy)++"`AND`"++ispis ww++")= τ("++(ispis.(krati abc)) (zz `opPa` yy `opSa` ww)++")= "++[particija abc (zz `opPa` yy `opSa` ww)])
-- /show
```


## Example 7
Let us observe the following three statements written in three languages:

- Croatian: SVI LJUDI SU SMRTNI
- English: ALL MEN ARE MORTAL
- French: TOUS LES HOMMES SONT MORTELS

<hr/>

- Croatian: SOKRAT JE ČOVJEK
- English: SOCRATES IS A MAN
- French: SOCRATE EST UN HOMME

<hr/>

- Croatian: SOKRAT JE SMRTAN
- English: SOCRATES IS MORTAL
- French: SOCRATE EST MORTEL


Processing using  M-system have the following form:

``` active haskell
{-# START_FILE Linguistic2.hs #-}
-- show
-- /show
{-# LANGUAGE UndecidableInstances, FlexibleInstances, OverlappingInstances, IncoherentInstances #-}
module Linguistic2 where

import           Data.Char  (chr, isLower, isUpper, ord, toLower, toUpper)
import           Data.List  (elemIndex, nub)
import           Data.Maybe (fromJust)

-- alphabet type definition and constructors
data Slovo a = Slovo a deriving (Eq, Show)
data Abeceda a = Abeceda {unAbeceda :: [Slovo a]} deriving (Eq, Show)
type Rijec a = [Slovo a]

-- |for a list the function constructs the alphabet containing unique elements of the list
lista2abeceda :: Eq a => [a] -> Abeceda a
lista2abeceda xs=Abeceda [Slovo x|x<-nub xs]

-- |for 2 Enum types (like Char) the function constructs the alphabet from first to second one
enum2abeceda :: (Eq a, Enum a) => a -> a -> Abeceda a
enum2abeceda a z=lista2abeceda [a..z]

-- |for a list the function constructs the word containing letters in the list
lista2rijec :: (Show a,Eq a) => [a] -> Rijec a
lista2rijec xs| even (length xs)= [Slovo x|x<-xs]
              | otherwise= error("The word "++show xs++" doesn't have an even number of letters.")

-- |for a word the function returns the list of letters in the word
rijec2lista :: Eq a => Rijec a -> [a]
rijec2lista xs= [x|Slovo x<-xs]

elemAbc :: Eq a => Slovo a -> Abeceda a -> Bool
elemAbc s (Abeceda a) = elem s a

elemIndexAbc :: Eq a => Slovo a -> Abeceda a -> Maybe Int
elemIndexAbc s (Abeceda a) = elemIndex s a

sup :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
sup a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) > fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

inf :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
inf a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) < fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

-- |function converting two consecutive lowercase letters to an uppercase letter
zapisu :: String->String
zapisu []=[]
zapisu [x]=[x]
zapisu (x:y:xs)|isLower x && x==y = toUpper x:zapisu xs
               |otherwise=x:zapisu (y:xs)

-- |function converting uppercase letter to two consecutive lowercase letters
zapisl :: String->String
zapisl []=[]
zapisl (x:xs)|isUpper x =x':x':zapisl xs
             |otherwise =x:zapisl xs
             where x'=toLower x

-- string output of a word
class Show a => Ispisiv a where
    ispis :: Show a => Rijec a -> String
    ispis []=[]
    ispis [Slovo x]=show x
    ispis (Slovo x:xs)=show x++"-"++ispis xs

instance Show a => Ispisiv a
instance Ispisiv Char where
    ispis=zapisu.ispis'

ispis' :: Rijec Char -> String
ispis' []=[]
ispis' [Slovo x]=[x]
ispis' (Slovo x:xs)=x:ispis' xs

-- functions returning left and right letters of a word
l :: [Slovo a] -> Slovo a
l = head
d :: [Slovo a] -> Slovo a
d = last

-- |function returning a dual of a letter in an alphabet
dual :: Eq a => Abeceda a -> Slovo a -> Slovo a
dual (Abeceda []) s = error("Nema duala - slovo nije u abecedi.")
dual (Abeceda [c]) s | s==c = c
                     | otherwise = error("No dual - Letter is not from the alphabet.")
dual (Abeceda cs) s | s==lijevi = desni
                    | s==desni = lijevi
                    | otherwise = dual (Abeceda (tail (init cs))) s
                     where lijevi= l cs
                           desni= d cs

-- functions returning duals of left/right letters of a word
l' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
l' a= (dual a).l
d' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
d' a= (dual a).d

ljuska :: [Slovo a]->[Slovo a]
ljuska xs=[l xs,d xs]


-- |distance between letters in an alphabet
dd :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Int
dd a x y| m==Nothing || n==Nothing = error("Cannot calculate distance. Letter is not from the alphabet.")
        | otherwise = abs (fromJust m-fromJust n)
  where m=x `elemIndexAbc` a
        n=y `elemIndexAbc` a

-- |word length
duljina :: Eq a => Abeceda a -> Rijec a -> Int
duljina a []=0
duljina a [c]=0
duljina a (c1:c2:cs)= - dd a c1 c2 - duljina a (c2:cs)

-- word shortening
krati1 :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function shortening a 4 letter word
krati1 a [p,r,s,t] | duljina a [p,r,s,t]==duljina a [p,t] = [p,t]
                 | otherwise = [p,r,s,t]
krati1 _ cs = cs

krati2 :: Eq a => Abeceda a -> Rijec a -> Rijec a     -- function shortening a 4 letter sub-word
krati2 _ [] = []
krati2 a(c:cs) | length kr==2 = kr ++ drop 3 cs
               | otherwise = c : krati2 a cs
               where kr = ((krati1 a).(take 4)) (c:cs)

krati :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function fully shortening a word
krati a cs | cs/=kr = krati a kr
           | otherwise = cs
           where kr = krati2 a cs

-- symmetries
opD :: Eq a => Abeceda a -> Rijec a -> Rijec a
opD _ []=[]
opD a (s:t:ys)=dual a s:dual a t:opD a ys

opE :: Eq a => Abeceda a -> Rijec a -> Rijec a
opE _ []=[]
opE a xs=t:s:opE a ys
        where n=length xs
              t=d xs
              s=xs !! (n-2)
              ys=take (n-2) xs
--opE a (s:t:ys)=opE a ys++[t,s]    --kraće, ali nije po formuli iz teksta

opF :: Eq a => Abeceda a -> Rijec a -> Rijec a
opF a=(opD a).(opE a)

-- permutations
opI :: Eq a => Abeceda a -> Rijec a -> Rijec a
opI a xs=l xs:xs++[d xs]

-- |negation operator
opK :: Eq a => Abeceda a -> Rijec a -> Rijec a
opK a xs=l' a xs:xs++[d' a xs]

opG :: Eq a => Abeceda a -> Rijec a -> Rijec a
opG a xs=d xs:xs++[l xs]

opH :: Eq a => Abeceda a -> Rijec a -> Rijec a
opH a xs=d' a xs:xs++[l' a xs]

-- serial and parallel conjecture operators
opS :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opS a xs ys = inf a (l xs) (l ys) : inf a (l xs) (l ys) : xs ++
              [sup a (d xs) (l ys),inf a (d xs) (l ys)] ++ ys ++
              [sup a (d xs) (d ys),sup a (d xs) (d ys)]

opP :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opP a xs ys = sup a (l xs) (l ys) : sup a (l xs) (l ys) : xs ++
              [inf a (d xs) (l ys),sup a (d xs) (l ys)] ++ ys ++
              [inf a (d xs) (d ys),inf a (d xs) (d ys)]

-- |partitions
particija :: Eq a => Abeceda a -> Rijec a -> Char
particija a xs | lijevi>=*lijevi' && desni<=*desni' && lijevi>*desni ='T'
               | lijevi<=*lijevi' && desni<=*desni' && not (lijevi==lijevi' || desni==desni') ='A'
               | lijevi==lijevi' && desni==desni' ='0'
               | lijevi>=*lijevi' && desni>=*desni' && not (lijevi==lijevi' || desni==desni')  ='Z'
               | lijevi<=*lijevi' && desni>=*desni' && lijevi<*desni  ='F'
         where lijevi=l xs
               desni=d xs
               lijevi'=l' a xs
               desni'=d' a xs
               x >* y| m==Nothing || n==Nothing =error("Letter not in alphabet.")
                     | otherwise = (fromJust m) > (fromJust n)
                      where m=x `elemIndexAbc` a
                            n=y `elemIndexAbc` a
               x >=* y = x >* y || x==y
               x <* y = not (x >=* y)
               x <=* y = not (x >* y)

-- implications
pimp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pimp a x y=(opK a x) `opP'` y
    where opP'=opP a

simp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
simp a x y=(opK a x) `opS'` y
    where opS'=opS a

-- equivalencies
pekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pekv a x y=(x `pimp'` y) `opS'` (y `pimp'` x)
    where opS'=opS a
          pimp'=pimp a

sekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
sekv a x y=(x `simp'` y) `opP'` (y `simp'` x)
    where opP'=opP a
          simp'=simp a

-- |tautologies and rules
taut :: Eq a => Abeceda a -> Rijec a ->Rijec a -> Rijec a -> [(String,Char)]
taut a x y z= zip
  ["1.The law of  excluded middle",
  "2.Dual law of excluded middle",
  "3.Transitivity of implication",
  "4.Dual law of transitivity of implication",
  "5.Double negation law",
  "6.Contraposition law",
  "7.Dual contraposition law",
  "8.Absorption law",
  "9.Dual absorption Law",
  "10.Modus ponendo ponens",
  "11.Dual modus ponendo ponens",
  "12.Modus tollendo tollens",
  "13.Demonstratio per enumerationem",
  "14.Proof by contradiction"]
  (map (particija a) [
    x `opP'` (neg x),
    x `opS'` (neg x),
    ((x `pimp'` y) `opS'` (y `pimp'` z)) `pimp'` (x `pimp'` z),
    ((x `simp'` y) `opP'` (y `simp'` z)) `simp'` (x `simp'` z),
    (neg (neg x)) `pekv'` x,
    (x `pimp'` y) `pimp'` ((neg y) `pimp'` (neg x)),
    (x `simp'` y) `simp'` ((neg y) `simp'` (neg x)),
    (x `opP'` (y `opS'` x)) `pimp'` x,
    (x `opS'` (y `opP'` x)) `simp'` x,
    (x `opS'` (x `pimp'` y)) `pimp'` y,
    (x `opP'` (x `simp'` y)) `simp'` y,
    ((neg x) `opS'` ((neg x) `pimp'` (neg y))) `pimp'` (neg y),
    ((x `pimp'` z) `opS'` (y `pimp'` z)) `pimp'` ((x `opP'` y) `pimp'` z),
    ((x `pimp'` y) `opS'` (x `pimp'` (neg y))) `pimp'` (neg x)
    ]
  )
  where opP'=opP a
        opS'=opS a
        pimp'=pimp a
        simp'=simp a
        pekv'=pekv a
        sekv'=sekv a
        neg=opK a
        
{-# START_FILE Main.hs #-}
module Main where
import Linguistic2

main :: IO ()

-- show Alphabet
main=do let {abc=lista2abeceda ['a'..'z'];}         -- <-- CHANGE
-- /show


-- show Words
        let {hr=rijeci ("SOKRAT","COVJEK","SMRTAN");
             en=rijeci ("SOCRATES","MAN","MORTALS");    -- <-- CHANGE
             fr=rijeci ("SOCRATE","HOMME","MORTELS");}
-- /show

        let opPa=opP abc; opSa=opS abc; pimpa=pimp abc; simpa=simp abc;

-- show Word partition
        putStrLn ("τ"++show (fNaTri ispis hr)++" = "++show (fNaTri (particija abc) hr))
        putStrLn ("τ"++show (fNaTri ispis en)++" = "++show (fNaTri (particija abc) en))
        putStrLn ("τ"++show (fNaTri ispis fr)++" = "++show (fNaTri (particija abc) fr))
-- /show


        putStrLn "\nThe rule of syllogism"


-- show The rule of syllogism
        putStrLn "Croatian:"
        obrada abc hr
        putStrLn "\nEnglish:"
        obrada abc en
        putStrLn "\nFrench:"
        obrada abc fr
-- /show


        putStrLn "\nFirst fallacy"


-- show Fallacy
        putStrLn "Croatian:"
        obrada2 abc hr
        putStrLn "\nEnglish:"
        obrada2 abc en
        putStrLn "\nFrench:"
        obrada2 abc fr
-- /show


        putStrLn "\nSecond fallacy"


-- show Fallacy
        putStrLn "Croatian:"
        obrada3 abc hr
        putStrLn "\nEnglish:"
        obrada3 abc en
        putStrLn "\nFrench:"
        obrada3 abc fr
-- /show


        putStrLn "\nThird fallacy"


-- show Fallacy
        putStrLn "Croatian:"
        obrada4 abc hr
        putStrLn "\nEnglish:"
        obrada4 abc en
        putStrLn "\nFrench:"
        obrada4 abc fr
-- /show


        putStrLn "\nFourth fallacy"


-- show Fallacy
        putStrLn "Croatian:"
        obrada5 abc hr
        putStrLn "\nEnglish:"
        obrada5 abc en
        putStrLn "\nFrench:"
        obrada5 abc fr
-- /show

-- show Tautologies
        putStrLn ("\nAll tautologies and rules for (x,y,z)="++show (fNaTri ispis hr)++":")
        print ((\a (x,y,z)->taut a x y z) abc hr)
-- /show


fNaTri::(a->b)->(a,a,a)->(b,b,b)
fNaTri f (x,y,z)=(f x,f y,f z)

rijeci::(String,String,String)->(Rijec Char,Rijec Char,Rijec Char)
rijeci x=fNaTri (lista2rijec.zapisl) x

obrada :: Abeceda Char->(Rijec Char,Rijec Char,Rijec Char)->IO ()
obrada abc (y,z,x)= do let {pimpa=pimp abc;opSa=opS abc;ispiskr=(ispis.(krati abc));}
                       putStrLn ("τ((("++ispis y++" -p-> "++ispis z++") `AND` ("++ispis z++" -p-> "++ispis x++")) -p-> ("++ispis y++" -p-> "++ispis x++"))\n =τ(("
                         ++ispiskr (y `pimpa` z)++" `AND` "++ispiskr (z `pimpa` x)++") -p-> "++ispiskr (y `pimpa` x)++")\n =τ("
                         ++ispiskr ((y `pimpa` z) `opSa` (z `pimpa` x))++" -p-> "++ispiskr (y `pimpa` x)++")\n =τ("
                         ++ispiskr (((y `pimpa` z) `opSa` (z `pimpa` x)) `pimpa` (y `pimpa` x))++")\n ="
                         ++[particija abc (((y `pimpa` z) `opSa` (z `pimpa` x)) `pimpa` (y `pimpa` x))])

obrada2 :: Abeceda Char->(Rijec Char,Rijec Char,Rijec Char)->IO ()
obrada2 abc (y,z,x)= do let {pimpa=pimp abc;simpa=simp abc;opSa=opS abc;ispiskr=(ispis.(krati abc));}
                        putStrLn ("τ((("++ispis y++" -p-> "++ispis z++") `AND` ("++ispis z++" -p-> "++ispis x++")) -s-> ("++ispis y++" -p-> "++ispis x++"))\n =τ(("
                          ++ispiskr (y `pimpa` z)++" `AND` "++ispiskr (z `pimpa` x)++") -s-> "++ispiskr (y `pimpa` x)++")\n =τ("
                          ++ispiskr ((y `pimpa` z) `opSa` (z `pimpa` x))++" -s-> "++ispiskr (y `pimpa` x)++")\n =τ("
                          ++ispiskr (((y `pimpa` z) `opSa` (z `pimpa` x)) `simpa` (y `pimpa` x))++")\n ="
                          ++[particija abc (((y `pimpa` z) `opSa` (z `pimpa` x)) `simpa` (y `pimpa` x))])

obrada3 :: Abeceda Char->(Rijec Char,Rijec Char,Rijec Char)->IO ()
obrada3 abc (y,z,x)= do let {pimpa=pimp abc;simpa=simp abc;opSa=opS abc;ispiskr=(ispis.(krati abc));}
                        putStrLn ("τ((("++ispis y++" -p-> "++ispis z++") `AND` ("++ispis z++" -p-> "++ispis x++")) -p-> ("++ispis y++" -s-> "++ispis x++"))\n =τ(("
                          ++ispiskr (y `pimpa` z)++" `AND` "++ispiskr (z `pimpa` x)++") -p-> "++ispiskr (y `simpa` x)++")\n =τ("
                          ++ispiskr ((y `pimpa` z) `opSa` (z `pimpa` x))++" -p-> "++ispiskr (y `simpa` x)++")\n =τ("
                          ++ispiskr (((y `pimpa` z) `opSa` (z `pimpa` x)) `pimpa` (y `simpa` x))++")\n ="
                          ++[particija abc (((y `pimpa` z) `opSa` (z `pimpa` x)) `pimpa` (y `simpa` x))])

obrada4 :: Abeceda Char->(Rijec Char,Rijec Char,Rijec Char)->IO ()
obrada4 abc (y,z,x)= do let {pimpa=pimp abc;simpa=simp abc;opSa=opS abc;nega=opK abc;ispiskr=(ispis.(krati abc));}
                        putStrLn ("τ((("++ispis y++" -p-> "++ispis z++") `AND` ("++ispis z++" -p-> "++ispis x++")) -p-> ¬("++ispis y++" -p-> "++ispis x++"))\n =τ(("
                          ++ispiskr (y `pimpa` z)++" `AND` "++ispiskr (z `pimpa` x)++") -p-> ¬"++ispiskr (y `pimpa` x)++")\n =τ("
                          ++ispiskr ((y `pimpa` z) `opSa` (z `pimpa` x))++" -p-> "++ispiskr (nega (y `pimpa` x))++")\n =τ("
                          ++ispiskr (((y `pimpa` z) `opSa` (z `pimpa` x)) `pimpa` (nega (y `pimpa` x)))++")\n ="
                          ++[particija abc (((y `pimpa` z) `opSa` (z `pimpa` x)) `pimpa` (nega (y `pimpa` x)))])

obrada5 :: Abeceda Char->(Rijec Char,Rijec Char,Rijec Char)->IO ()
obrada5 abc (y,z,x)= do let {pimpa=pimp abc;simpa=simp abc;opSa=opS abc;nega=opK abc;ispiskr=(ispis.(krati abc));}
                        putStrLn ("τ((("++ispis y++" -p-> "++ispis z++") `AND` ("++ispis z++" -p-> "++ispis x++")) -p-> ("++ispis y++" -p-> ¬("++ispis x++")))\n =τ(("
                          ++ispiskr (y `pimpa` z)++" `AND` "++ispiskr (z `pimpa` x)++") -p-> "++ispiskr (y `pimpa` (nega x))++")\n =τ("
                          ++ispiskr ((y `pimpa` z) `opSa` (z `pimpa` x))++" -p-> "++ispiskr (y `pimpa` (nega x))++")\n =τ("
                          ++ispiskr (((y `pimpa` z) `opSa` (z `pimpa` x)) `pimpa` (y `pimpa` (nega x)))++")\n ="
                          ++[particija abc (((y `pimpa` z) `opSa` (z `pimpa` x)) `pimpa` (y `pimpa` (nega x)))])
```

## The rules of inference and tautologies

``` active haskell
{-# START_FILE Linguistic2.hs #-}
-- show
-- /show
{-# LANGUAGE UndecidableInstances, FlexibleInstances, OverlappingInstances, IncoherentInstances #-}
module Linguistic2 where

import           Data.Char  (chr, isLower, isUpper, ord, toLower, toUpper)
import           Data.List  (elemIndex, nub)
import           Data.Maybe (fromJust)

-- alphabet type definition and constructors
data Slovo a = Slovo a deriving (Eq, Show)
data Abeceda a = Abeceda {unAbeceda :: [Slovo a]} deriving (Eq, Show)
type Rijec a = [Slovo a]

-- |for a list the function constructs the alphabet containing unique elements of the list
lista2abeceda :: Eq a => [a] -> Abeceda a
lista2abeceda xs=Abeceda [Slovo x|x<-nub xs]

-- |for 2 Enum types (like Char) the function constructs the alphabet from first to second one
enum2abeceda :: (Eq a, Enum a) => a -> a -> Abeceda a
enum2abeceda a z=lista2abeceda [a..z]

-- |for a list the function constructs the word containing letters in the list
lista2rijec :: (Show a,Eq a) => [a] -> Rijec a
lista2rijec xs| even (length xs)= [Slovo x|x<-xs]
              | otherwise= error("The word "++show xs++" doesn't have an even number of letters.")

-- |for a word the function returns the list of letters in the word
rijec2lista :: Eq a => Rijec a -> [a]
rijec2lista xs= [x|Slovo x<-xs]

elemAbc :: Eq a => Slovo a -> Abeceda a -> Bool
elemAbc s (Abeceda a) = elem s a

elemIndexAbc :: Eq a => Slovo a -> Abeceda a -> Maybe Int
elemIndexAbc s (Abeceda a) = elemIndex s a

sup :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
sup a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) > fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

inf :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
inf a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) < fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

-- |function converting two consecutive lowercase letters to an uppercase letter
zapisu :: String->String
zapisu []=[]
zapisu [x]=[x]
zapisu (x:y:xs)|isLower x && x==y = toUpper x:zapisu xs
               |otherwise=x:zapisu (y:xs)

-- |function converting uppercase letter to two consecutive lowercase letters
zapisl :: String->String
zapisl []=[]
zapisl (x:xs)|isUpper x =x':x':zapisl xs
             |otherwise =x:zapisl xs
             where x'=toLower x

-- string output of a word
class Show a => Ispisiv a where
    ispis :: Show a => Rijec a -> String
    ispis []=[]
    ispis [Slovo x]=show x
    ispis (Slovo x:xs)=show x++"-"++ispis xs

instance Show a => Ispisiv a
instance Ispisiv Char where
    ispis=zapisu.ispis'

ispis' :: Rijec Char -> String
ispis' []=[]
ispis' [Slovo x]=[x]
ispis' (Slovo x:xs)=x:ispis' xs

-- functions returning left and right letters of a word
l :: [Slovo a] -> Slovo a
l = head
d :: [Slovo a] -> Slovo a
d = last

-- |function returning a dual of a letter in an alphabet
dual :: Eq a => Abeceda a -> Slovo a -> Slovo a
dual (Abeceda []) s = error("Nema duala - slovo nije u abecedi.")
dual (Abeceda [c]) s | s==c = c
                     | otherwise = error("No dual - Letter is not from the alphabet.")
dual (Abeceda cs) s | s==lijevi = desni
                    | s==desni = lijevi
                    | otherwise = dual (Abeceda (tail (init cs))) s
                     where lijevi= l cs
                           desni= d cs

-- functions returning duals of left/right letters of a word
l' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
l' a= (dual a).l
d' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
d' a= (dual a).d

ljuska :: [Slovo a]->[Slovo a]
ljuska xs=[l xs,d xs]


-- |distance between letters in an alphabet
dd :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Int
dd a x y| m==Nothing || n==Nothing = error("Cannot calculate distance. Letter is not from the alphabet.")
        | otherwise = abs (fromJust m-fromJust n)
  where m=x `elemIndexAbc` a
        n=y `elemIndexAbc` a

-- |word length
duljina :: Eq a => Abeceda a -> Rijec a -> Int
duljina a []=0
duljina a [c]=0
duljina a (c1:c2:cs)= - dd a c1 c2 - duljina a (c2:cs)

-- word shortening
krati1 :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function shortening a 4 letter word
krati1 a [p,r,s,t] | duljina a [p,r,s,t]==duljina a [p,t] = [p,t]
                 | otherwise = [p,r,s,t]
krati1 _ cs = cs

krati2 :: Eq a => Abeceda a -> Rijec a -> Rijec a     -- function shortening a 4 letter sub-word
krati2 _ [] = []
krati2 a(c:cs) | length kr==2 = kr ++ drop 3 cs
               | otherwise = c : krati2 a cs
               where kr = ((krati1 a).(take 4)) (c:cs)

krati :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function fully shortening a word
krati a cs | cs/=kr = krati a kr
           | otherwise = cs
           where kr = krati2 a cs

-- symmetries
opD :: Eq a => Abeceda a -> Rijec a -> Rijec a
opD _ []=[]
opD a (s:t:ys)=dual a s:dual a t:opD a ys

opE :: Eq a => Abeceda a -> Rijec a -> Rijec a
opE _ []=[]
opE a xs=t:s:opE a ys
        where n=length xs
              t=d xs
              s=xs !! (n-2)
              ys=take (n-2) xs
--opE a (s:t:ys)=opE a ys++[t,s]    --kraće, ali nije po formuli iz teksta

opF :: Eq a => Abeceda a -> Rijec a -> Rijec a
opF a=(opD a).(opE a)

-- permutations
opI :: Eq a => Abeceda a -> Rijec a -> Rijec a
opI a xs=l xs:xs++[d xs]

-- |negation operator
opK :: Eq a => Abeceda a -> Rijec a -> Rijec a
opK a xs=l' a xs:xs++[d' a xs]

opG :: Eq a => Abeceda a -> Rijec a -> Rijec a
opG a xs=d xs:xs++[l xs]

opH :: Eq a => Abeceda a -> Rijec a -> Rijec a
opH a xs=d' a xs:xs++[l' a xs]

-- serial and parallel conjecture operators
opS :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opS a xs ys = inf a (l xs) (l ys) : inf a (l xs) (l ys) : xs ++
              [sup a (d xs) (l ys),inf a (d xs) (l ys)] ++ ys ++
              [sup a (d xs) (d ys),sup a (d xs) (d ys)]

opP :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opP a xs ys = sup a (l xs) (l ys) : sup a (l xs) (l ys) : xs ++
              [inf a (d xs) (l ys),sup a (d xs) (l ys)] ++ ys ++
              [inf a (d xs) (d ys),inf a (d xs) (d ys)]

-- |partitions
particija :: Eq a => Abeceda a -> Rijec a -> Char
particija a xs | lijevi>=*lijevi' && desni<=*desni' && lijevi>*desni ='T'
               | lijevi<=*lijevi' && desni<=*desni' && not (lijevi==lijevi' || desni==desni') ='A'
               | lijevi==lijevi' && desni==desni' ='0'
               | lijevi>=*lijevi' && desni>=*desni' && not (lijevi==lijevi' || desni==desni')  ='Z'
               | lijevi<=*lijevi' && desni>=*desni' && lijevi<*desni  ='F'
         where lijevi=l xs
               desni=d xs
               lijevi'=l' a xs
               desni'=d' a xs
               x >* y| m==Nothing || n==Nothing =error("Letter not in alphabet.")
                     | otherwise = (fromJust m) > (fromJust n)
                      where m=x `elemIndexAbc` a
                            n=y `elemIndexAbc` a
               x >=* y = x >* y || x==y
               x <* y = not (x >=* y)
               x <=* y = not (x >* y)

-- implications
pimp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pimp a x y=(opK a x) `opP'` y
    where opP'=opP a

simp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
simp a x y=(opK a x) `opS'` y
    where opS'=opS a

-- equivalencies
pekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pekv a x y=(x `pimp'` y) `opS'` (y `pimp'` x)
    where opS'=opS a
          pimp'=pimp a

sekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
sekv a x y=(x `simp'` y) `opP'` (y `simp'` x)
    where opP'=opP a
          simp'=simp a

-- |tautologies and rules
taut :: Eq a => Abeceda a -> Rijec a ->Rijec a -> Rijec a -> [(String,Char)]
taut a x y z= zip
  ["1.The law of excluded middle (Tertium non datur)",
  "2.Dual law of excluded middle (The law of non-contradiction)",
  "3.Transitivity of implication (The rule of syllogism)",
  "4.Dual low of transitivity of implication",
  "5.Double negation row",
  "6.Contraposition law",
  "7.Dual contraposition law",
  "8.Absorption law",
  "9.Dual absorption Law",
  "10.Modus ponendo ponens",
  "11.Dual modus ponendo ponens",
  "12.Modus tollendo tollens",
  "13.Demonstratio per enumerationem",
  "14.Proof by contradiction (Reductio ad apsurdum)"]
  (map (particija a) [
    x `opP'` (neg x),
    x `opS'` (neg x),
    ((x `pimp'` y) `opS'` (y `pimp'` z)) `pimp'` (x `pimp'` z),
    ((x `simp'` y) `opP'` (y `simp'` z)) `simp'` (x `simp'` z),
    (neg (neg x)) `pekv'` x,
    (x `pimp'` y) `pimp'` ((neg y) `pimp'` (neg x)),
    (x `simp'` y) `simp'` ((neg y) `simp'` (neg x)),
    (x `opP'` (y `opP'` x)) `pimp'` x,
    (x `opS'` (y `opS'` x)) `simp'` x,
    (x `opS'` (x `pimp'` y)) `pimp'` y,
    (x `opP'` (x `simp'` y)) `simp'` y,
    ((neg x) `opS'` ((neg x) `pimp'` (neg y))) `pimp'` (neg y),
    ((x `pimp'` z) `opS'` (y `pimp'` z)) `pimp'` ((x `opP'` y) `pimp'` z),
    ((x `pimp'` y) `opS'` (x `pimp'` (neg y))) `pimp'` (neg x)
    ]
  )
  where opP'=opP a
        opS'=opS a
        pimp'=pimp a
        simp'=simp a
        pekv'=pekv a
        sekv'=sekv a
        neg=opK a
        
{-# START_FILE Main.hs #-}
module Main where
import Linguistic2

main :: IO ()

-- show Alphabet
main=do let {abc=lista2abeceda ['a'..'z']}         -- <-- CHANGE
-- /show


-- show Words
        let {x=lista2rijec (zapisl "BIRD");
             y=lista2rijec (zapisl "PINGU");   -- <-- CHANGE
             z=lista2rijec (zapisl "PENGUIN");
             w=lista2rijec (zapisl "CHICKEN");}
-- /show

        let {opPa=opP abc; opSa=opS abc; pimpa=pimp abc; simpa=simp abc; pekva=pekv abc; nega=opK abc; ispiskr=(ispis.(krati abc));}

-- show Word partition
        putStrLn ("τ("++ispis x++")= "++[particija abc x])
        putStrLn ("τ("++ispis y++")= "++[particija abc y])
        putStrLn ("τ("++ispis z++")= "++[particija abc z])
        putStrLn ("τ("++ispis w++")= "++[particija abc w])
-- /show


-- show 1. The law of excluded middle
        putStrLn "The law of excluded middle"
        putStrLn ("τ("++ispis x++" `OR` ¬("++ispis x++"))\n =τ("
          ++ispis x++" `OR` "++ispiskr (nega x)++")\n =τ("
          ++ispiskr (x `opPa` (nega x))++")\n ="
          ++[particija abc (x `opPa` (nega x))])
-- /show


-- show 2. Dual law of excluded middle
        putStrLn "Dual law of excluded middle"
        putStrLn ("τ("++ispis x++" `AND` ¬("++ispis x++"))\n =τ("
          ++ispis x++" `AND` "++ispiskr (nega x)++")\n =τ("
          ++ispiskr (x `opSa` (nega x))++")\n ="
          ++[particija abc (x `opSa` (nega x))])
-- /show


-- show 3. Transitivity of implication
        putStrLn "Transitivity of implication"
        putStrLn ("τ((("++ispis y++" -p-> "++ispis z++") `AND` ("++ispis z++" -p-> "++ispis x++")) -p-> ("++ispis y++" -p-> "++ispis x++"))\n =τ(("
          ++ispiskr (y `pimpa` z)++" `AND` "++ispiskr (z `pimpa` x)++") -p-> "++ispiskr (y `pimpa` x)++")\n =τ("
          ++ispiskr ((y `pimpa` z) `opSa` (z `pimpa` x))++" -p-> "++ispiskr (y `pimpa` x)++")\n =τ("
          ++ispiskr (((y `pimpa` z) `opSa` (z `pimpa` x)) `pimpa` (y `pimpa` x))++")\n ="
          ++[particija abc (((y `pimpa` z) `opSa` (z `pimpa` x)) `pimpa` (y `pimpa` x))])
-- /show


-- show 4. Dual law of transitivity of implication
        putStrLn "Dual law of transitivity of implication"
        putStrLn ("τ((("++ispis y++" -s-> "++ispis z++") `OR` ("++ispis z++" -s-> "++ispis x++")) -s-> ("++ispis y++" -s-> "++ispis x++"))\n =τ(("
          ++ispiskr (y `simpa` z)++" `OR` "++ispiskr (z `simpa` x)++") -s-> "++ispiskr (y `simpa` x)++")\n =τ("
          ++ispiskr ((y `simpa` z) `opPa` (z `simpa` x))++" -s-> "++ispiskr (y `simpa` x)++")\n =τ("
          ++ispiskr (((y `simpa` z) `opPa` (z `simpa` x)) `pimpa` (y `pimpa` x))++")\n ="
          ++[particija abc (((y `simpa` z) `opPa` (z `simpa` x)) `simpa` (y `simpa` x))])
-- /show


-- show 5. Double negation law
        putStrLn "Double negation law"
        putStrLn ("τ(¬(¬("++ispis x++")) <=p=> "++ispis x++")\n =τ(¬("
          ++ispiskr (nega x)++") <=p=> "++ispis x++")\n =τ("
          ++ispiskr (nega (nega x))++" <=p=> "++ispis x++")\n =τ("
          ++ispiskr ((nega (nega x)) `pekva` x)++")\n ="
          ++[particija abc ((nega (nega x)) `pekva` x)])
-- /show


-- show 6. Contraposition law
        putStrLn "Contraposition law"
        putStrLn ("τ(("++ispis w++" -p-> "++ispis x++") -p-> (¬("++ispis x++") -p-> ¬("++ispis w++")))\n =τ("
          ++ispiskr (w `pimpa` x)++" -p-> ("++ispis (nega x)++" -p-> "++ispis (nega w)++"))\n =τ("
          ++ispiskr (w `pimpa` x)++" -p-> "++ispiskr ((nega x) `pimpa` (nega w))++")\n =τ("
          ++ispiskr ((w `pimpa` x) `pimpa` ((nega x) `pimpa` (nega w)))++")\n ="
          ++[particija abc ((w `pimpa` x) `pimpa` ((nega x) `pimpa` (nega w)))])
-- /show


-- show 7. Dual contraposition law
        putStrLn "Dual contraposition law"
        putStrLn ("τ(("++ispis w++" -s-> "++ispis x++") -s-> (¬("++ispis x++") -s-> ¬("++ispis w++")))\n =τ("
          ++ispiskr (w `simpa` x)++" -s-> ("++ispis (nega x)++" -s-> "++ispis (nega w)++"))\n =τ("
          ++ispiskr (w `simpa` x)++" -s-> "++ispiskr ((nega x) `simpa` (nega w))++")\n =τ("
          ++ispiskr ((w `simpa` x) `simpa` ((nega x) `simpa` (nega w)))++")\n ="
          ++[particija abc ((w `simpa` x) `simpa` ((nega x) `simpa` (nega w)))])
-- /show


-- show 8. Absorption law
        putStrLn "Absorption law"
        putStrLn ("τ(("++ispis x++" `OR` ("++ispis y++" `AND` "++ispis x++")) -p-> "++ispis x++")\n =τ(("
          ++ispis x++" `OR` "++ispiskr (y `opSa` x)++") -p-> "++ispis x++")\n =τ("
          ++ispiskr (x `opPa` (y `opSa` x))++") -p-> "++ispis x++")\n =τ("
          ++ispiskr ((x `opPa` (y `opSa` x)) `pimpa` x)++")\n ="
          ++[particija abc ((x `opPa` (y `opSa` x)) `pimpa` x)])
-- /show


-- show 9. Dual absorption law
        putStrLn ("τ(("++ispis x++" `AND` ("++ispis y++" `OR` "++ispis x++")) -s-> "++ispis x++")\n =τ(("
          ++ispis x++" `AND` "++ispiskr (y `opPa` x)++") -s-> "++ispis x++")\n =τ("
          ++ispiskr (x `opSa` (y `opPa` x))++") -s-> "++ispis x++")\n =τ("
          ++ispiskr ((x `opSa` (y `opPa` x)) `simpa` x)++")\n ="
          ++[particija abc ((x `opSa` (y `opPa` x)) `simpa` x)])
-- /show


-- show 10. Modus ponendo ponens
        putStrLn "Modus ponendo ponens"
        putStrLn ("τ(("++ispis w++" `AND` ("++ispis w++" -p-> "++ispis x++")) -p-> "++ispis x++")\n =τ(("
          ++ispis w++" `AND` "++ispiskr (w `pimpa` x)++") -p-> "++ispis x++")\n =τ(("
          ++ispiskr (w `opSa` (w `pimpa` x))++") -p-> "++ispis x++")\n =τ("
          ++ispiskr ((w `opSa` (w `pimpa` x)) `pimpa` x)++")\n ="
          ++[particija abc ((w `opSa` (w `pimpa` x)) `pimpa` x)])
-- /show


-- show 11. Dual modus ponendo ponens
        putStrLn "Dual modus ponendo ponens"
        putStrLn ("τ(("++ispis w++" `OR` ("++ispis w++" -s-> "++ispis x++")) -s-> "++ispis x++")\n =τ(("
          ++ispis w++" `OR` "++ispiskr (w `simpa` x)++") -s-> "++ispis x++")\n =τ(("
          ++ispiskr (w `opPa` (w `simpa` x))++") -s-> "++ispis x++")\n =τ("
          ++ispiskr ((w `opPa` (w `simpa` x)) `simpa` x)++")\n ="
          ++[particija abc ((w `opPa` (w `simpa` x)) `simpa` x)])
-- /show


-- show 12. Modus tollendo tollens
        putStrLn "Modus tollendo tollens"
        putStrLn ("τ((¬("++ispis w++") `AND` (¬("++ispis w++") -p-> ¬("++ispis x++"))) -p-> ¬("++ispis x++"))\n =τ(("
                  ++ispis (nega w)++" `AND` ("++ispis (nega w)++" -p-> "++ispis (nega x)++")) -p-> "++ispis (nega x)++")\n =τ(("
          ++ispis (nega w)++" `AND` "++ispiskr ((nega w) `pimpa` (nega x))++") -p-> "++ispis (nega x)++")\n =τ(("
          ++ispiskr ((nega w) `opSa` ((nega w) `pimpa` (nega x)))++") -p-> "++ispis (nega x)++")\n =τ("
          ++ispiskr (((nega w) `opSa` ((nega w) `pimpa` (nega x))) `pimpa` (nega x))++")\n ="
          ++[particija abc (((nega w) `opSa` ((nega w) `pimpa` (nega x))) `pimpa` (nega x))])
-- /show


-- show 13. Demonstratio per enumerationem
        putStrLn "Demonstratio per enumerationem"
        putStrLn ("τ((("++ispis z++" -p-> "++ispis x++") `AND` ("++ispis w++" -p-> "++ispis x++")) -p-> (("++ispis z++" `OR` "++ispis w++") -p-> "++ispis x++"))\n =τ(("
          ++ispiskr (z `pimpa` x)++" `AND` "++ispiskr (w `pimpa` x)++") -p-> ("++ispiskr (z `opPa` w)++" -p-> "++ispis x++"))\n =τ("
          ++ispiskr ((z `pimpa` x) `opSa` (w `pimpa` x))++" -p-> "++ispiskr ((z `opPa` w) `pimpa` x)++")\n =τ("
          ++ispiskr (((z `pimpa` x) `opSa` (w `pimpa` x)) `pimpa` ((z `opPa` w) `pimpa` x))++")\n ="
          ++[particija abc (((z `pimpa` x) `opSa` (w `pimpa` x)) `pimpa` ((z `opPa` w) `pimpa` x))])
-- /show


-- show 14. Proof by contradiction
        putStrLn "Proof by contradiction"
        putStrLn ("τ((("++ispis x++" -p-> "++ispis y++") `AND` ("++ispis x++" -p-> ¬("++ispis y++"))) -p-> ¬("++ispis x++"))\n =τ((("
          ++ispis x++" -p-> "++ispis y++") `AND` ("++ispis x++" -p-> "++ispis (nega y)++")) -p-> ¬("++ispis x++"))\n =τ(("
          ++ispiskr (x `pimpa` y)++") `AND` ("++ispiskr (x `pimpa` (nega y))++")) -p-> "++ispis (nega x)++")\n =τ("
          ++ispiskr ((x `pimpa` y) `opSa` (x `pimpa` (nega y)))++")) -p-> "++ispis (nega x)++")\n =τ("
                  ++ispiskr (((x `pimpa` y) `opSa` (x `pimpa` (nega y))) `pimpa` (nega x))++")\n ="
          ++[particija abc (((x `pimpa` y) `opSa` (x `pimpa` (nega y))) `pimpa` (nega x))])
-- /show
```


## Appendix

### Operator tables
``` active haskell
{-# START_FILE Linguistic2.hs #-}
-- show
-- /show
{-# LANGUAGE UndecidableInstances, FlexibleInstances, OverlappingInstances, IncoherentInstances #-}
module Linguistic2 where

import           Data.Char  (chr, isLower, isUpper, ord, toLower, toUpper)
import           Data.List  (elemIndex, nub)
import           Data.Maybe (fromJust)

-- alphabet type definition and constructors
data Slovo a = Slovo a deriving (Eq, Show)
data Abeceda a = Abeceda {unAbeceda :: [Slovo a]} deriving (Eq, Show)
type Rijec a = [Slovo a]

-- |for a list the function constructs the alphabet containing unique elements of the list
lista2abeceda :: Eq a => [a] -> Abeceda a
lista2abeceda xs=Abeceda [Slovo x|x<-nub xs]

-- |for 2 Enum types (like Char) the function constructs the alphabet from first to second one
enum2abeceda :: (Eq a, Enum a) => a -> a -> Abeceda a
enum2abeceda a z=lista2abeceda [a..z]

-- |for a list the function constructs the word containing letters in the list
lista2rijec :: (Show a,Eq a) => [a] -> Rijec a
lista2rijec xs| even (length xs)= [Slovo x|x<-xs]
              | otherwise= error("The word "++show xs++" doesn't have an even number of letters.")

-- |for a word the function returns the list of letters in the word
rijec2lista :: Eq a => Rijec a -> [a]
rijec2lista xs= [x|Slovo x<-xs]

elemAbc :: Eq a => Slovo a -> Abeceda a -> Bool
elemAbc s (Abeceda a) = elem s a

elemIndexAbc :: Eq a => Slovo a -> Abeceda a -> Maybe Int
elemIndexAbc s (Abeceda a) = elemIndex s a

sup :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
sup a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) > fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

inf :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
inf a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) < fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

-- |function converting two consecutive lowercase letters to an uppercase letter
zapisu :: String->String
zapisu []=[]
zapisu [x]=[x]
zapisu (x:y:xs)|isLower x && x==y = toUpper x:zapisu xs
               |otherwise=x:zapisu (y:xs)

-- |function converting uppercase letter to two consecutive lowercase letters
zapisl :: String->String
zapisl []=[]
zapisl (x:xs)|isUpper x =x':x':zapisl xs
             |otherwise =x:zapisl xs
             where x'=toLower x

-- string output of a word
class Show a => Ispisiv a where
    ispis :: Show a => Rijec a -> String
    ispis []=[]
    ispis [Slovo x]=show x
    ispis (Slovo x:xs)=show x++"-"++ispis xs

instance Show a => Ispisiv a
instance Ispisiv Char where
    ispis=zapisu.ispis'

ispis' :: Rijec Char -> String
ispis' []=[]
ispis' [Slovo x]=[x]
ispis' (Slovo x:xs)=x:ispis' xs

-- functions returning left and right letters of a word
l :: [Slovo a] -> Slovo a
l = head
d :: [Slovo a] -> Slovo a
d = last

-- |function returning a dual of a letter in an alphabet
dual :: Eq a => Abeceda a -> Slovo a -> Slovo a
dual (Abeceda []) s = error("Nema duala - slovo nije u abecedi.")
dual (Abeceda [c]) s | s==c = c
                     | otherwise = error("No dual - Letter is not from the alphabet.")
dual (Abeceda cs) s | s==lijevi = desni
                    | s==desni = lijevi
                    | otherwise = dual (Abeceda (tail (init cs))) s
                     where lijevi= l cs
                           desni= d cs

-- functions returning duals of left/right letters of a word
l' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
l' a= (dual a).l
d' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
d' a= (dual a).d

ljuska :: [Slovo a]->[Slovo a]
ljuska xs=[l xs,d xs]


-- |distance between letters in an alphabet
dd :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Int
dd a x y| m==Nothing || n==Nothing = error("Cannot calculate distance. Letter is not from the alphabet.")
        | otherwise = abs (fromJust m-fromJust n)
  where m=x `elemIndexAbc` a
        n=y `elemIndexAbc` a

-- |word length
duljina :: Eq a => Abeceda a -> Rijec a -> Int
duljina a []=0
duljina a [c]=0
duljina a (c1:c2:cs)= - dd a c1 c2 - duljina a (c2:cs)

-- word shortening
krati1 :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function shortening a 4 letter word
krati1 a [p,r,s,t] | duljina a [p,r,s,t]==duljina a [p,t] = [p,t]
                 | otherwise = [p,r,s,t]
krati1 _ cs = cs

krati2 :: Eq a => Abeceda a -> Rijec a -> Rijec a     -- function shortening a 4 letter sub-word
krati2 _ [] = []
krati2 a(c:cs) | length kr==2 = kr ++ drop 3 cs
               | otherwise = c : krati2 a cs
               where kr = ((krati1 a).(take 4)) (c:cs)

krati :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function fully shortening a word
krati a cs | cs/=kr = krati a kr
           | otherwise = cs
           where kr = krati2 a cs

-- symmetries
opD :: Eq a => Abeceda a -> Rijec a -> Rijec a
opD _ []=[]
opD a (s:t:ys)=dual a s:dual a t:opD a ys

opE :: Eq a => Abeceda a -> Rijec a -> Rijec a
opE _ []=[]
opE a xs=t:s:opE a ys
        where n=length xs
              t=d xs
              s=xs !! (n-2)
              ys=take (n-2) xs
--opE a (s:t:ys)=opE a ys++[t,s]    --kraće, ali nije po formuli iz teksta

opF :: Eq a => Abeceda a -> Rijec a -> Rijec a
opF a=(opD a).(opE a)

-- permutations
opI :: Eq a => Abeceda a -> Rijec a -> Rijec a
opI a xs=l xs:xs++[d xs]

-- |negation operator
opK :: Eq a => Abeceda a -> Rijec a -> Rijec a
opK a xs=l' a xs:xs++[d' a xs]

opG :: Eq a => Abeceda a -> Rijec a -> Rijec a
opG a xs=d xs:xs++[l xs]

opH :: Eq a => Abeceda a -> Rijec a -> Rijec a
opH a xs=d' a xs:xs++[l' a xs]

-- serial and parallel conjecture operators
opS :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opS a xs ys = inf a (l xs) (l ys) : inf a (l xs) (l ys) : xs ++
              [sup a (d xs) (l ys),inf a (d xs) (l ys)] ++ ys ++
              [sup a (d xs) (d ys),sup a (d xs) (d ys)]

opP :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opP a xs ys = sup a (l xs) (l ys) : sup a (l xs) (l ys) : xs ++
              [inf a (d xs) (l ys),sup a (d xs) (l ys)] ++ ys ++
              [inf a (d xs) (d ys),inf a (d xs) (d ys)]

-- |partitions
particija :: Eq a => Abeceda a -> Rijec a -> Char
particija a xs | lijevi>=*lijevi' && desni<=*desni' && lijevi>*desni ='T'
               | lijevi<=*lijevi' && desni<=*desni' && not (lijevi==lijevi' || desni==desni') ='A'
               | lijevi==lijevi' && desni==desni' ='0'
               | lijevi>=*lijevi' && desni>=*desni' && not (lijevi==lijevi' || desni==desni')  ='Z'
               | lijevi<=*lijevi' && desni>=*desni' && lijevi<*desni  ='F'
         where lijevi=l xs
               desni=d xs
               lijevi'=l' a xs
               desni'=d' a xs
               x >* y| m==Nothing || n==Nothing =error("Letter not in alphabet.")
                     | otherwise = (fromJust m) > (fromJust n)
                      where m=x `elemIndexAbc` a
                            n=y `elemIndexAbc` a
               x >=* y = x >* y || x==y
               x <* y = not (x >=* y)
               x <=* y = not (x >* y)

-- implications
pimp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pimp a x y=(opK a x) `opP'` y
    where opP'=opP a

simp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
simp a x y=(opK a x) `opS'` y
    where opS'=opS a

-- equivalencies
pekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pekv a x y=(x `pimp'` y) `opS'` (y `pimp'` x)
    where opS'=opS a
          pimp'=pimp a

sekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
sekv a x y=(x `simp'` y) `opP'` (y `simp'` x)
    where opP'=opP a
          simp'=simp a

-- |tautologies and rules
taut :: Eq a => Abeceda a -> Rijec a ->Rijec a -> Rijec a -> [(String,Char)]
taut a x y z= zip
  ["1.ZAKON ISKLJUCENJA TRECEG",
  "2.DUAL ZAKONA ISKLJUCENJA TRECEG",
  "3.TRANZITIVNOST IMPLIKACIJE",
  "4.DUAL TRANZITIVNOSTI IMPLIKACIJE",
  "5.ZAKON DVOSTRUKE NEGACIJE",
  "6.ZAKON KONTRAPOZICIJE",
  "7.DUAL ZAKONA KONTRAPOZICIJE",
  "8.ZAKON APSORPCIJE ILI UPIJANJA",
  "9.DUAL ZAKONA APSORPCIJE ILI UPIJANJA",
  "10.MODUS PONENDO PONENS",
  "11.DUAL MODUSA PONENDO PONENS",
  "12.MODUS TOLENDO TOLLENS",
  "13.DOKAZ NABRAJANJEM",
  "14.DOVODJENJE DO PROTURJECJA"]
  (map (particija a) [
    x `opP'` (neg x),
    x `opS'` (neg x),
    ((x `pimp'` y) `opS'` (y `pimp'` z)) `pimp'` (x `pimp'` z),
    ((x `simp'` y) `opP'` (y `simp'` z)) `simp'` (x `simp'` z),
    (neg (neg x)) `pekv'` x,
    (x `pimp'` y) `pimp'` ((neg y) `pimp'` (neg x)),
    (x `simp'` y) `simp'` ((neg y) `simp'` (neg x)),
    (x `opP'` (y `opP'` x)) `pimp'` x,
    (x `opS'` (y `opS'` x)) `simp'` x,
    (x `opS'` (x `pimp'` y)) `pimp'` y,
    (x `opP'` (x `simp'` y)) `simp'` y,
    ((neg x) `opS'` ((neg x) `pimp'` (neg y))) `pimp'` (neg y),
    ((x `pimp'` z) `opS'` (y `pimp'` z)) `pimp'` ((x `opP'` y) `pimp'` z),
    ((x `pimp'` y) `opS'` (x `pimp'` (neg y))) `pimp'` (neg x)
    ]
  )
  where opP'=opP a
        opS'=opS a
        pimp'=pimp a
        simp'=simp a
        pekv'=pekv a
        sekv'=sekv a
        neg=opK a
        
{-# START_FILE Main.hs #-}
{-# LANGUAGE UndecidableInstances, FlexibleInstances, OverlappingInstances, IncoherentInstances #-}
module Main where
import Linguistic2

main :: IO ()

-- show Set alphabet, row and column headers and the binary operator for the table.
main=do let {abc = lista2abeceda ['a'..'c'];                  -- <-- CHANGE
             red = map lista2rijec ["aa","bb","cc"];         -- <-- CHANGE
             stupac = map lista2rijec ["aa","bb","cc"];      -- <-- CHANGE
             opName = "OP";
             operator = (\a x y->let opDa=opD a; opEa=opE a; opFa=opF a; neg=opK a; opPa=opP a; opSa=opS a; pimpa=pimp a; simpa=simp a; pekva=pekv a; sekva=sekv a; in 
                                        ((x `opSa` y) `simpa` y)   -- <-- CHANGE
                                        );}
-- /show


-- show Formatting (latex -> regular table or LaTeX; printShell -> show just shell or the whole result)
        let {latex=False;            -- <-- CHANGE
             printShell=True;}      -- <-- CHANGE
-- /show

        let {operator'=(if printShell then (opLjuska operator) else operator);
             n=length stupac;
             prije=(if latex then "\\begin{tabular}{|c|"++replicate n 'c'++"|}\n\\hline\n" else "");
             sep=(if latex then "& " else " ");
             sepRed=(if latex then "\\hline\n" else (replicate ((n+1)*11) '-'++"\n"));
             eol=(if latex then "\\\\\n" else "\n");
             poslije=(if latex then "\\hline\n\\end{tabular}" else "");
             tablica=[(zapisl.ispis) x:[(zapisl.ispis.(krati abc)) (operator' abc x y)|y<-stupac]|x<-red];
             ispisKraj=prije++redIspis (sep,eol) (opName:(map (zapisl.ispis) stupac))++sepRed++concat [redIspis (sep,eol) x|x<-tablica]++poslije;
             }
        putStrLn ispisKraj

opLjuska op=(\a x y->ljuska (op a x y)) -- funkcija koja na operator još primjenjuje ljusku
-- opBroj op=(fst op, (\a x y->perm2broj a (ljuska ((snd op) a x y)))) -- funkcija koja na operator još primjenjuje ljusku i zamjenjuje elemente brojevima

perm :: Abeceda a -> [Rijec a]
perm abc = [[x,y]|x<-a,y<-a]
           where a=unAbeceda abc

perm' :: Abeceda a -> [Rijec a]
perm' abc = [[x,y]|x<-reverse a,y<-a]
           where a=unAbeceda abc

-- perm2broj :: Eq a => Abeceda a->Rijec a->String
-- perm2broj a xs=head [c|(b,c)<-zip (perm a) [show n|n<-[1..]],b==xs]

-- |for a given separator, end of line string and list returns a line string
redIspis::(String,String)->[String]->String
redIspis (_,eol) [xs]=make10 xs++eol
redIspis (sep,eol) (xs:xss)=make10 xs ++ sep ++ redIspis (sep,eol) xss

make10::String->String
make10 xs| n<10      =xs++replicate (10-n) ' '
         | otherwise =xs
  where n=length xs
```


### Example
This example shows partitions of a 2-generator alphabet and truth tables for the set operator

``` active haskell
{-# START_FILE Linguistic2.hs #-}
-- show
-- /show
{-# LANGUAGE UndecidableInstances, FlexibleInstances, OverlappingInstances, IncoherentInstances #-}
module Linguistic2 where

import           Data.Char  (chr, isLower, isUpper, ord, toLower, toUpper)
import           Data.List  (elemIndex, nub)
import           Data.Maybe (fromJust)

-- alphabet type definition and constructors
data Slovo a = Slovo a deriving (Eq, Show)
data Abeceda a = Abeceda {unAbeceda :: [Slovo a]} deriving (Eq, Show)
type Rijec a = [Slovo a]

-- |for a list the function constructs the alphabet containing unique elements of the list
lista2abeceda :: Eq a => [a] -> Abeceda a
lista2abeceda xs=Abeceda [Slovo x|x<-nub xs]

-- |for 2 Enum types (like Char) the function constructs the alphabet from first to second one
enum2abeceda :: (Eq a, Enum a) => a -> a -> Abeceda a
enum2abeceda a z=lista2abeceda [a..z]

-- |for a list the function constructs the word containing letters in the list
lista2rijec :: (Show a,Eq a) => [a] -> Rijec a
lista2rijec xs| even (length xs)= [Slovo x|x<-xs]
              | otherwise= error("The word "++show xs++" doesn't have an even number of letters.")

-- |for a word the function returns the list of letters in the word
rijec2lista :: Eq a => Rijec a -> [a]
rijec2lista xs= [x|Slovo x<-xs]

elemAbc :: Eq a => Slovo a -> Abeceda a -> Bool
elemAbc s (Abeceda a) = elem s a

elemIndexAbc :: Eq a => Slovo a -> Abeceda a -> Maybe Int
elemIndexAbc s (Abeceda a) = elemIndex s a

sup :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
sup a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) > fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

inf :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
inf a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) < fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

-- |function converting two consecutive lowercase letters to an uppercase letter
zapisu :: String->String
zapisu []=[]
zapisu [x]=[x]
zapisu (x:y:xs)|isLower x && x==y = toUpper x:zapisu xs
               |otherwise=x:zapisu (y:xs)

-- |function converting uppercase letter to two consecutive lowercase letters
zapisl :: String->String
zapisl []=[]
zapisl (x:xs)|isUpper x =x':x':zapisl xs
             |otherwise =x:zapisl xs
             where x'=toLower x

-- string output of a word
class Show a => Ispisiv a where
    ispis :: Show a => Rijec a -> String
    ispis []=[]
    ispis [Slovo x]=show x
    ispis (Slovo x:xs)=show x++"-"++ispis xs

instance Show a => Ispisiv a
instance Ispisiv Char where
    ispis=zapisu.ispis'

ispis' :: Rijec Char -> String
ispis' []=[]
ispis' [Slovo x]=[x]
ispis' (Slovo x:xs)=x:ispis' xs

-- functions returning left and right letters of a word
l :: [Slovo a] -> Slovo a
l = head
d :: [Slovo a] -> Slovo a
d = last

-- |function returning a dual of a letter in an alphabet
dual :: Eq a => Abeceda a -> Slovo a -> Slovo a
dual (Abeceda []) s = error("Nema duala - slovo nije u abecedi.")
dual (Abeceda [c]) s | s==c = c
                     | otherwise = error("No dual - Letter is not from the alphabet.")
dual (Abeceda cs) s | s==lijevi = desni
                    | s==desni = lijevi
                    | otherwise = dual (Abeceda (tail (init cs))) s
                     where lijevi= l cs
                           desni= d cs

-- functions returning duals of left/right letters of a word
l' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
l' a= (dual a).l
d' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
d' a= (dual a).d

ljuska :: [Slovo a]->[Slovo a]
ljuska xs=[l xs,d xs]


-- |distance between letters in an alphabet
dd :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Int
dd a x y| m==Nothing || n==Nothing = error("Cannot calculate distance. Letter is not from the alphabet.")
        | otherwise = abs (fromJust m-fromJust n)
  where m=x `elemIndexAbc` a
        n=y `elemIndexAbc` a

-- |word length
duljina :: Eq a => Abeceda a -> Rijec a -> Int
duljina a []=0
duljina a [c]=0
duljina a (c1:c2:cs)= - dd a c1 c2 - duljina a (c2:cs)

-- word shortening
krati1 :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function shortening a 4 letter word
krati1 a [p,r,s,t] | duljina a [p,r,s,t]==duljina a [p,t] = [p,t]
                 | otherwise = [p,r,s,t]
krati1 _ cs = cs

krati2 :: Eq a => Abeceda a -> Rijec a -> Rijec a     -- function shortening a 4 letter sub-word
krati2 _ [] = []
krati2 a(c:cs) | length kr==2 = kr ++ drop 3 cs
               | otherwise = c : krati2 a cs
               where kr = ((krati1 a).(take 4)) (c:cs)

krati :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function fully shortening a word
krati a cs | cs/=kr = krati a kr
           | otherwise = cs
           where kr = krati2 a cs

-- symmetries
opD :: Eq a => Abeceda a -> Rijec a -> Rijec a
opD _ []=[]
opD a (s:t:ys)=dual a s:dual a t:opD a ys

opE :: Eq a => Abeceda a -> Rijec a -> Rijec a
opE _ []=[]
opE a xs=t:s:opE a ys
        where n=length xs
              t=d xs
              s=xs !! (n-2)
              ys=take (n-2) xs
--opE a (s:t:ys)=opE a ys++[t,s]    --kraće, ali nije po formuli iz teksta

opF :: Eq a => Abeceda a -> Rijec a -> Rijec a
opF a=(opD a).(opE a)

-- permutations
opI :: Eq a => Abeceda a -> Rijec a -> Rijec a
opI a xs=l xs:xs++[d xs]

-- |negation operator
opK :: Eq a => Abeceda a -> Rijec a -> Rijec a
opK a xs=l' a xs:xs++[d' a xs]

opG :: Eq a => Abeceda a -> Rijec a -> Rijec a
opG a xs=d xs:xs++[l xs]

opH :: Eq a => Abeceda a -> Rijec a -> Rijec a
opH a xs=d' a xs:xs++[l' a xs]

-- serial and parallel conjecture operators
opS :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opS a xs ys = inf a (l xs) (l ys) : inf a (l xs) (l ys) : xs ++
              [sup a (d xs) (l ys),inf a (d xs) (l ys)] ++ ys ++
              [sup a (d xs) (d ys),sup a (d xs) (d ys)]

opP :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opP a xs ys = sup a (l xs) (l ys) : sup a (l xs) (l ys) : xs ++
              [inf a (d xs) (l ys),sup a (d xs) (l ys)] ++ ys ++
              [inf a (d xs) (d ys),inf a (d xs) (d ys)]

-- |partitions
particija :: Eq a => Abeceda a -> Rijec a -> Char
particija a xs | lijevi>=*lijevi' && desni<=*desni' && lijevi>*desni ='T'
               | lijevi<=*lijevi' && desni<=*desni' && not (lijevi==lijevi' || desni==desni') ='A'
               | lijevi==lijevi' && desni==desni' ='0'
               | lijevi>=*lijevi' && desni>=*desni' && not (lijevi==lijevi' || desni==desni')  ='Z'
               | lijevi<=*lijevi' && desni>=*desni' && lijevi<*desni  ='F'
         where lijevi=l xs
               desni=d xs
               lijevi'=l' a xs
               desni'=d' a xs
               x >* y| m==Nothing || n==Nothing =error("Letter not in alphabet.")
                     | otherwise = (fromJust m) > (fromJust n)
                      where m=x `elemIndexAbc` a
                            n=y `elemIndexAbc` a
               x >=* y = x >* y || x==y
               x <* y = not (x >=* y)
               x <=* y = not (x >* y)

-- implications
pimp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pimp a x y=(opK a x) `opP'` y
    where opP'=opP a

simp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
simp a x y=(opK a x) `opS'` y
    where opS'=opS a

-- equivalencies
pekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pekv a x y=(x `pimp'` y) `opS'` (y `pimp'` x)
    where opS'=opS a
          pimp'=pimp a

sekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
sekv a x y=(x `simp'` y) `opP'` (y `simp'` x)
    where opP'=opP a
          simp'=simp a

-- |tautologies and rules
taut :: Eq a => Abeceda a -> Rijec a ->Rijec a -> Rijec a -> [(String,Char)]
taut a x y z= zip
  ["1.ZAKON ISKLJUCENJA TRECEG",
  "2.DUAL ZAKONA ISKLJUCENJA TRECEG",
  "3.TRANZITIVNOST IMPLIKACIJE",
  "4.DUAL TRANZITIVNOSTI IMPLIKACIJE",
  "5.ZAKON DVOSTRUKE NEGACIJE",
  "6.ZAKON KONTRAPOZICIJE",
  "7.DUAL ZAKONA KONTRAPOZICIJE",
  "8.ZAKON APSORPCIJE ILI UPIJANJA",
  "9.DUAL ZAKONA APSORPCIJE ILI UPIJANJA",
  "10.MODUS PONENDO PONENS",
  "11.DUAL MODUSA PONENDO PONENS",
  "12.MODUS TOLENDO TOLLENS",
  "13.DOKAZ NABRAJANJEM",
  "14.DOVODJENJE DO PROTURJECJA"]
  (map (particija a) [
    x `opP'` (neg x),
    x `opS'` (neg x),
    ((x `pimp'` y) `opS'` (y `pimp'` z)) `pimp'` (x `pimp'` z),
    ((x `simp'` y) `opP'` (y `simp'` z)) `simp'` (x `simp'` z),
    (neg (neg x)) `pekv'` x,
    (x `pimp'` y) `pimp'` ((neg y) `pimp'` (neg x)),
    (x `simp'` y) `simp'` ((neg y) `simp'` (neg x)),
    (x `opP'` (y `opP'` x)) `pimp'` x,
    (x `opS'` (y `opS'` x)) `simp'` x,
    (x `opS'` (x `pimp'` y)) `pimp'` y,
    (x `opP'` (x `simp'` y)) `simp'` y,
    ((neg x) `opS'` ((neg x) `pimp'` (neg y))) `pimp'` (neg y),
    ((x `pimp'` z) `opS'` (y `pimp'` z)) `pimp'` ((x `opP'` y) `pimp'` z),
    ((x `pimp'` y) `opS'` (x `pimp'` (neg y))) `pimp'` (neg x)
    ]
  )
  where opP'=opP a
        opS'=opS a
        pimp'=pimp a
        simp'=simp a
        pekv'=pekv a
        sekv'=sekv a
        neg=opK a
        
{-# START_FILE Main.hs #-}
{-# LANGUAGE UndecidableInstances, FlexibleInstances, OverlappingInstances, IncoherentInstances #-}
module Main where
import Linguistic2

main :: IO ()

-- show Set alphabet, row and column headers and the binary operator for the table.
main=do let {abc = lista2abeceda ['a'..'b'];                  -- <-- CHANGE
             red = [map lista2rijec x| x<-[["ba"],["aa","bb"],["ab"]]];         -- <-- CHANGE
             stupac = red;       -- <-- CHANGE
             opName = "AND";
             operator = (\a x y->let opDa=opD a; opEa=opE a; opFa=opF a; neg=opK a; opPa=opP a; opSa=opS a; pimpa=pimp a; simpa=simp a; pekva=pekv a; sekva=sekv a; in 
                                        (x `opSa` y)   -- <-- CHANGE
                                        );}
-- /show


-- show Formatting (latex -> regular table or LaTeX; printShell -> show just shell or the whole result)
        let {latex=False;            -- <-- CHANGE
             printShell=True;        -- <-- CHANGE
             printPartition=False;}  -- <-- CHANGE
-- /show

        let {tab=[([x,y],particija abc [x,y])|x <- unAbeceda abc,y <- unAbeceda abc];
             operator'=(if printShell then (opLjuska operator) else operator);
             n=length (concat stupac);
             prije=(if latex then "%% requires 'tabu' package\n\\begin{tabu}{c|"++concatWith "|[on 2pt off 2pt]" [replicate (length ygr) 'l'|ygr<-stupac]++"}\n" else "");
             sepCol=(if latex then "& " else " ");
             sepColGr=(if latex then "& " else "| ");
             eol=(if latex then "\\\\\n" else "\n");
             sepHeadRow=(if latex then "\\tabucline{-}\n" else (replicate ((n+1)*11) '-'++"\n"));
             sepRowGr=(if latex then "\\tabucline[on 2pt off 2pt]{-}\n" else (replicate ((n+1)*11) '-'++"\n"));
             poslije=(if latex then "\\end{tabu}" else "");
             printPartFunc a x=(if printPartition then (":"++[particija a x]) else "");
             tablica=[[[(zapisl.ispis) x++(printPartFunc abc x)]:[[(zapisl.ispis.(krati abc)) (operator' abc x y)++(printPartFunc abc (operator' abc x y))|y<-ygr]|ygr<-stupac]|x<-xgr]|xgr<-red];
             ispisKraj=prije++redIspis (sepCol,sepColGr,eol) ([opName]:[map (\x->(zapisl.ispis) x++(printPartFunc  abc x)) ygr|ygr<-stupac])++sepHeadRow++concatWith sepRowGr [concat [redIspis (sepCol,sepColGr,eol) x|x<-xgr]|xgr<-tablica]++poslije;
             --ispisKraj=prije++redIspis (sepCol,eol) (opName:(map (\x->(zapisl.ispis) x++(printPartFunc abc x)) (concat stupac)))++sepHeadRow++concat [redIspis (sepCol,eol) x|x<-tablica]++poslije;
             }
        putStrLn (filter (/='"') (concat [p:": "++show [ispis x|(x,y)<-tab,y==p]++"\n"|p<-['T','A','0','Z','F']]))
        putStrLn ispisKraj

opLjuska op=(\a x y->ljuska (op a x y)) -- funkcija koja na operator još primjenjuje ljusku
-- opBroj op=(fst op, (\a x y->perm2broj a (ljuska ((snd op) a x y)))) -- funkcija koja na operator još primjenjuje ljusku i zamjenjuje elemente brojevima

perm :: Abeceda a -> [Rijec a]
perm abc = [[x,y]|x<-a,y<-a]
           where a=unAbeceda abc

perm' :: Abeceda a -> [Rijec a]
perm' abc = [[x,y]|x<-reverse a,y<-a]
           where a=unAbeceda abc

-- perm2broj :: Eq a => Abeceda a->Rijec a->String
-- perm2broj a xs=head [c|(b,c)<-zip (perm a) [show n|n<-[1..]],b==xs]

-- |for a given separator, end of line string and list returns a line string
redIspis::(String,String,String)->[[String]]->String
redIspis (sepCol,_,eol) [xss]=concatWith sepCol [make10 xs|xs<-xss]++eol
redIspis (sepCol,sepColGr,eol) (xss:xsss)=(concatWith sepCol [make10 xs|xs<-xss]) ++ sepColGr ++ redIspis (sepCol,sepColGr,eol) xsss

concatWith::String->[String]->String
concatWith _ [x]=x
concatWith s (x:xs)=x++s++concatWith s xs

make10::String->String
make10 xs| n<10      =xs++replicate (10-n) ' '
         | otherwise =xs
  where n=length xs
```

### Integer type example
``` active haskell
{-# START_FILE Linguistic2.hs #-}
-- show
-- /show
{-# LANGUAGE UndecidableInstances, FlexibleInstances, OverlappingInstances, IncoherentInstances #-}
module Linguistic2 where

import           Data.Char  (chr, isLower, isUpper, ord, toLower, toUpper)
import           Data.List  (elemIndex, nub)
import           Data.Maybe (fromJust)

-- alphabet type definition and constructors
data Slovo a = Slovo a deriving (Eq, Show)
data Abeceda a = Abeceda {unAbeceda :: [Slovo a]} deriving (Eq, Show)
type Rijec a = [Slovo a]

-- |for a list the function constructs the alphabet containing unique elements of the list
lista2abeceda :: Eq a => [a] -> Abeceda a
lista2abeceda xs=Abeceda [Slovo x|x<-nub xs]

-- |for 2 Enum types (like Char) the function constructs the alphabet from first to second one
enum2abeceda :: (Eq a, Enum a) => a -> a -> Abeceda a
enum2abeceda a z=lista2abeceda [a..z]

-- |for a list the function constructs the word containing letters in the list
lista2rijec :: (Show a,Eq a) => [a] -> Rijec a
lista2rijec xs| even (length xs)= [Slovo x|x<-xs]
              | otherwise= error("The word "++show xs++" doesn't have an even number of letters.")

-- |for a word the function returns the list of letters in the word
rijec2lista :: Eq a => Rijec a -> [a]
rijec2lista xs= [x|Slovo x<-xs]

elemAbc :: Eq a => Slovo a -> Abeceda a -> Bool
elemAbc s (Abeceda a) = elem s a

elemIndexAbc :: Eq a => Slovo a -> Abeceda a -> Maybe Int
elemIndexAbc s (Abeceda a) = elemIndex s a

sup :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
sup a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) > fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

inf :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
inf a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
         | fromJust (x `elemIndexAbc` a) < fromJust(y `elemIndexAbc` a) = x
         | otherwise = y

-- |function converting two consecutive lowercase letters to an uppercase letter
zapisu :: String->String
zapisu []=[]
zapisu [x]=[x]
zapisu (x:y:xs)|isLower x && x==y = toUpper x:zapisu xs
               |otherwise=x:zapisu (y:xs)

-- |function converting uppercase letter to two consecutive lowercase letters
zapisl :: String->String
zapisl []=[]
zapisl (x:xs)|isUpper x =x':x':zapisl xs
             |otherwise =x:zapisl xs
             where x'=toLower x

-- string output of a word
class Show a => Ispisiv a where
    ispis :: Show a => Rijec a -> String
    ispis []=[]
    ispis [Slovo x]=show x
    ispis (Slovo x:xs)=show x++"-"++ispis xs

instance Show a => Ispisiv a
instance Ispisiv Char where
    ispis=zapisu.ispis'

ispis' :: Rijec Char -> String
ispis' []=[]
ispis' [Slovo x]=[x]
ispis' (Slovo x:xs)=x:ispis' xs

-- functions returning left and right letters of a word
l :: [Slovo a] -> Slovo a
l = head
d :: [Slovo a] -> Slovo a
d = last

-- |function returning a dual of a letter in an alphabet
dual :: Eq a => Abeceda a -> Slovo a -> Slovo a
dual (Abeceda []) s = error("Nema duala - slovo nije u abecedi.")
dual (Abeceda [c]) s | s==c = c
                     | otherwise = error("No dual - Letter is not from the alphabet.")
dual (Abeceda cs) s | s==lijevi = desni
                    | s==desni = lijevi
                    | otherwise = dual (Abeceda (tail (init cs))) s
                     where lijevi= l cs
                           desni= d cs

-- functions returning duals of left/right letters of a word
l' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
l' a= (dual a).l
d' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
d' a= (dual a).d

ljuska :: [Slovo a]->[Slovo a]
ljuska xs=[l xs,d xs]


-- |distance between letters in an alphabet
dd :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Int
dd a x y| m==Nothing || n==Nothing = error("Cannot calculate distance. Letter is not from the alphabet.")
        | otherwise = abs (fromJust m-fromJust n)
  where m=x `elemIndexAbc` a
        n=y `elemIndexAbc` a

-- |word length
duljina :: Eq a => Abeceda a -> Rijec a -> Int
duljina a []=0
duljina a [c]=0
duljina a (c1:c2:cs)= - dd a c1 c2 - duljina a (c2:cs)

-- word shortening
krati1 :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function shortening a 4 letter word
krati1 a [p,r,s,t] | duljina a [p,r,s,t]==duljina a [p,t] = [p,t]
                 | otherwise = [p,r,s,t]
krati1 _ cs = cs

krati2 :: Eq a => Abeceda a -> Rijec a -> Rijec a     -- function shortening a 4 letter sub-word
krati2 _ [] = []
krati2 a(c:cs) | length kr==2 = kr ++ drop 3 cs
               | otherwise = c : krati2 a cs
               where kr = ((krati1 a).(take 4)) (c:cs)

krati :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function fully shortening a word
krati a cs | cs/=kr = krati a kr
           | otherwise = cs
           where kr = krati2 a cs

-- symmetries
opD :: Eq a => Abeceda a -> Rijec a -> Rijec a
opD _ []=[]
opD a (s:t:ys)=dual a s:dual a t:opD a ys

opE :: Eq a => Abeceda a -> Rijec a -> Rijec a
opE _ []=[]
opE a xs=t:s:opE a ys
        where n=length xs
              t=d xs
              s=xs !! (n-2)
              ys=take (n-2) xs
--opE a (s:t:ys)=opE a ys++[t,s]    --kraće, ali nije po formuli iz teksta

opF :: Eq a => Abeceda a -> Rijec a -> Rijec a
opF a=(opD a).(opE a)

-- permutations
opI :: Eq a => Abeceda a -> Rijec a -> Rijec a
opI a xs=l xs:xs++[d xs]

-- |negation operator
opK :: Eq a => Abeceda a -> Rijec a -> Rijec a
opK a xs=l' a xs:xs++[d' a xs]

opG :: Eq a => Abeceda a -> Rijec a -> Rijec a
opG a xs=d xs:xs++[l xs]

opH :: Eq a => Abeceda a -> Rijec a -> Rijec a
opH a xs=d' a xs:xs++[l' a xs]

-- serial and parallel conjecture operators
opS :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opS a xs ys = inf a (l xs) (l ys) : inf a (l xs) (l ys) : xs ++
              [sup a (d xs) (l ys),inf a (d xs) (l ys)] ++ ys ++
              [sup a (d xs) (d ys),sup a (d xs) (d ys)]

opP :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opP a xs ys = sup a (l xs) (l ys) : sup a (l xs) (l ys) : xs ++
              [inf a (d xs) (l ys),sup a (d xs) (l ys)] ++ ys ++
              [inf a (d xs) (d ys),inf a (d xs) (d ys)]

-- |partitions
particija :: Eq a => Abeceda a -> Rijec a -> Char
particija a xs | lijevi>=*lijevi' && desni<=*desni' && lijevi>*desni ='T'
               | lijevi<=*lijevi' && desni<=*desni' && not (lijevi==lijevi' || desni==desni') ='A'
               | lijevi==lijevi' && desni==desni' ='0'
               | lijevi>=*lijevi' && desni>=*desni' && not (lijevi==lijevi' || desni==desni')  ='Z'
               | lijevi<=*lijevi' && desni>=*desni' && lijevi<*desni  ='F'
         where lijevi=l xs
               desni=d xs
               lijevi'=l' a xs
               desni'=d' a xs
               x >* y| m==Nothing || n==Nothing =error("Letter not in alphabet.")
                     | otherwise = (fromJust m) > (fromJust n)
                      where m=x `elemIndexAbc` a
                            n=y `elemIndexAbc` a
               x >=* y = x >* y || x==y
               x <* y = not (x >=* y)
               x <=* y = not (x >* y)

-- implications
pimp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pimp a x y=(opK a x) `opP'` y
    where opP'=opP a

simp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
simp a x y=(opK a x) `opS'` y
    where opS'=opS a

-- equivalencies
pekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pekv a x y=(x `pimp'` y) `opS'` (y `pimp'` x)
    where opS'=opS a
          pimp'=pimp a

sekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
sekv a x y=(x `simp'` y) `opP'` (y `simp'` x)
    where opP'=opP a
          simp'=simp a

-- |tautologies and rules
taut :: Eq a => Abeceda a -> Rijec a ->Rijec a -> Rijec a -> [(String,Char)]
taut a x y z= zip
  ["1.ZAKON ISKLJUCENJA TRECEG",
  "2.DUAL ZAKONA ISKLJUCENJA TRECEG",
  "3.TRANZITIVNOST IMPLIKACIJE",
  "4.DUAL TRANZITIVNOSTI IMPLIKACIJE",
  "5.ZAKON DVOSTRUKE NEGACIJE",
  "6.ZAKON KONTRAPOZICIJE",
  "7.DUAL ZAKONA KONTRAPOZICIJE",
  "8.ZAKON APSORPCIJE ILI UPIJANJA",
  "9.DUAL ZAKONA APSORPCIJE ILI UPIJANJA",
  "10.MODUS PONENDO PONENS",
  "11.DUAL MODUSA PONENDO PONENS",
  "12.MODUS TOLENDO TOLLENS",
  "13.DOKAZ NABRAJANJEM",
  "14.DOVODJENJE DO PROTURJECJA"]
  (map (particija a) [
    x `opP'` (neg x),
    x `opS'` (neg x),
    ((x `pimp'` y) `opS'` (y `pimp'` z)) `pimp'` (x `pimp'` z),
    ((x `simp'` y) `opP'` (y `simp'` z)) `simp'` (x `simp'` z),
    (neg (neg x)) `pekv'` x,
    (x `pimp'` y) `pimp'` ((neg y) `pimp'` (neg x)),
    (x `simp'` y) `simp'` ((neg y) `simp'` (neg x)),
    (x `opP'` (y `opP'` x)) `pimp'` x,
    (x `opS'` (y `opS'` x)) `simp'` x,
    (x `opS'` (x `pimp'` y)) `pimp'` y,
    (x `opP'` (x `simp'` y)) `simp'` y,
    ((neg x) `opS'` ((neg x) `pimp'` (neg y))) `pimp'` (neg y),
    ((x `pimp'` z) `opS'` (y `pimp'` z)) `pimp'` ((x `opP'` y) `pimp'` z),
    ((x `pimp'` y) `opS'` (x `pimp'` (neg y))) `pimp'` (neg x)
    ]
  )
  where opP'=opP a
        opS'=opS a
        pimp'=pimp a
        simp'=simp a
        pekv'=pekv a
        sekv'=sekv a
        neg=opK a
        
{-# START_FILE Main.hs #-}
{-# LANGUAGE UndecidableInstances, FlexibleInstances, OverlappingInstances, IncoherentInstances #-}
module Main where
import Linguistic2

main :: IO ()

-- show Set alphabet, row and column headers and the binary operator for the table.
main=do let {abc = lista2abeceda ([1..2]::[Int]);                  -- <-- CHANGE
             red = [map lista2rijec x| x<-([[[2,1],[2,2]],[[1,1],[1,2]]]::[[[Int]]])];         -- <-- CHANGE
             stupac = [map lista2rijec x| x<-([[[2,1],[2,2]],[[1,1],[1,2]]]::[[[Int]]])];      -- <-- CHANGE
             opName = "OR";
             operator = (\a x y->let opDa=opD a; opEa=opE a; opFa=opF a; neg=opK a; opPa=opP a; opSa=opS a; pimpa=pimp a; simpa=simp a; pekva=pekv a; sekva=sekv a; in 
                                        (x `opPa` y)   -- <-- CHANGE
                                        );}
-- /show


-- show Formatting (latex -> regular table or LaTeX; printShell -> show just shell or the whole result)
        let {latex=False;            -- <-- CHANGE
             printShell=True;        -- <-- CHANGE
             printPartition=False;}  -- <-- CHANGE
-- /show

        let {tab=[([x,y],particija abc [x,y])|x <- unAbeceda abc,y <- unAbeceda abc];
             operator'=(if printShell then (opLjuska operator) else operator);
             n=length (concat stupac);
             prije=(if latex then "%% requires 'tabu' package\n\\begin{tabu}{c|"++concatWith "|[on 2pt off 2pt]" [replicate (length ygr) 'l'|ygr<-stupac]++"}\n" else "");
             sepCol=(if latex then "& " else " ");
             sepColGr=(if latex then "& " else "| ");
             eol=(if latex then "\\\\\n" else "\n");
             sepHeadRow=(if latex then "\\tabucline{-}\n" else (replicate ((n+1)*11) '-'++"\n"));
             sepRowGr=(if latex then "\\tabucline[on 2pt off 2pt]{-}\n" else (replicate ((n+1)*11) '-'++"\n"));
             poslije=(if latex then "\\end{tabu}" else "");
             printPartFunc a x=(if printPartition then (":"++[particija a x]) else "");
             tablica=[[[(zapisl.ispis) x++(printPartFunc abc x)]:[[(zapisl.ispis.(krati abc)) (operator' abc x y)++(printPartFunc abc (operator' abc x y))|y<-ygr]|ygr<-stupac]|x<-xgr]|xgr<-red];
             ispisKraj=prije++redIspis (sepCol,sepColGr,eol) ([opName]:[map (\x->(zapisl.ispis) x++(printPartFunc  abc x)) ygr|ygr<-stupac])++sepHeadRow++concatWith sepRowGr [concat [redIspis (sepCol,sepColGr,eol) x|x<-xgr]|xgr<-tablica]++poslije;
             --ispisKraj=prije++redIspis (sepCol,eol) (opName:(map (\x->(zapisl.ispis) x++(printPartFunc abc x)) (concat stupac)))++sepHeadRow++concat [redIspis (sepCol,eol) x|x<-tablica]++poslije;
             }
        putStrLn (filter (/='"') (concat [p:": "++show [ispis x|(x,y)<-tab,y==p]++"\n"|p<-['T','A','0','Z','F']]))
        putStrLn ispisKraj

opLjuska op=(\a x y->ljuska (op a x y)) -- funkcija koja na operator još primjenjuje ljusku
-- opBroj op=(fst op, (\a x y->perm2broj a (ljuska ((snd op) a x y)))) -- funkcija koja na operator još primjenjuje ljusku i zamjenjuje elemente brojevima

perm :: Abeceda a -> [Rijec a]
perm abc = [[x,y]|x<-a,y<-a]
           where a=unAbeceda abc

perm' :: Abeceda a -> [Rijec a]
perm' abc = [[x,y]|x<-reverse a,y<-a]
           where a=unAbeceda abc

-- perm2broj :: Eq a => Abeceda a->Rijec a->String
-- perm2broj a xs=head [c|(b,c)<-zip (perm a) [show n|n<-[1..]],b==xs]

-- |for a given separator, end of line string and list returns a line string
redIspis::(String,String,String)->[[String]]->String
redIspis (sepCol,_,eol) [xss]=concatWith sepCol [make10 xs|xs<-xss]++eol
redIspis (sepCol,sepColGr,eol) (xss:xsss)=(concatWith sepCol [make10 xs|xs<-xss]) ++ sepColGr ++ redIspis (sepCol,sepColGr,eol) xsss

concatWith::String->[String]->String
concatWith _ [x]=x
concatWith s (x:xs)=x++s++concatWith s xs

make10::String->String
make10 xs| n<10      =xs++replicate (10-n) ' '
         | otherwise =xs
  where n=length xs
```

### Expression calculator
``` active haskell
{-# START_FILE Linguistic2.hs #-}
-- show
-- /show
{-# LANGUAGE UndecidableInstances, FlexibleInstances, OverlappingInstances, IncoherentInstances #-}
module Linguistic2 where

import           Data.Char  (chr, isLower, isUpper, ord, toLower, toUpper)
import           Data.List  (elemIndex, nub)
import           Data.Maybe (fromJust)

-- alphabet type definition and constructors
data Slovo a = Slovo a deriving (Eq, Show)
data Abeceda a = Abeceda {unAbeceda :: [Slovo a]} deriving (Eq, Show)
type Rijec a = [Slovo a]

-- |for a list the function constructs the alphabet containing unique elements of the list
lista2abeceda :: Eq a => [a] -> Abeceda a
lista2abeceda xs=Abeceda [Slovo x|x<-nub xs]

-- |for 2 Enum types (like Char) the function constructs the alphabet from first to second one
enum2abeceda :: (Eq a, Enum a) => a -> a -> Abeceda a
enum2abeceda a z=lista2abeceda [a..z]

-- |for a list the function constructs the word containing letters in the list
lista2rijec :: (Show a,Eq a) => [a] -> Rijec a
lista2rijec xs| even (length xs)= [Slovo x|x<-xs]
              | otherwise= error("The word "++show xs++" doesn't have an even number of letters.")

-- |for a word the function returns the list of letters in the word
rijec2lista :: Eq a => Rijec a -> [a]
rijec2lista xs= [x|Slovo x<-xs]

elemAbc :: Eq a => Slovo a -> Abeceda a -> Bool
elemAbc s (Abeceda a) = elem s a

elemIndexAbc :: Eq a => Slovo a -> Abeceda a -> Maybe Int
elemIndexAbc s (Abeceda a) = elemIndex s a

ggeq :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Bool
ggeq a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
          | fromJust (x `elemIndexAbc` a) >= fromJust(y `elemIndexAbc` a) = True
          | otherwise = False

gleq :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Bool
gleq a x y| x `elemIndexAbc` a==Nothing || y `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
          | fromJust (x `elemIndexAbc` a) <= fromJust(y `elemIndexAbc` a) = True
          | otherwise = False

geq :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Bool
geq a x y=(ggeq a x y) && (gleq a x y)

sup :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
sup a x y| ggeq a x y = x
         | otherwise = y

inf :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Slovo a
inf a x y| gleq a x y = x
         | otherwise = y

-- |function converting two consecutive lowercase letters to an uppercase letter
zapisu :: String->String
zapisu []=[]
zapisu [x]=[x]
zapisu (x:y:xs)|isLower x && x==y = toUpper x:zapisu xs
               |otherwise=x:zapisu (y:xs)

-- |function converting uppercase letter to two consecutive lowercase letters
zapisl :: String->String
zapisl []=[]
zapisl (x:xs)|isUpper x =x':x':zapisl xs
             |otherwise =x:zapisl xs
             where x'=toLower x

-- string output of a word
class Show a => Ispisiv a where
    ispis :: Show a => Rijec a -> String
    ispis []=[]
    ispis [Slovo x]=show x
    ispis (Slovo x:xs)=show x++"-"++ispis xs

instance Show a => Ispisiv a
instance Ispisiv Char where
    ispis=zapisu.ispis'

ispis' :: Rijec Char -> String
ispis' []=[]
ispis' [Slovo x]=[x]
ispis' (Slovo x:xs)=x:ispis' xs

-- functions returning left and right letters of a word
l :: [Slovo a] -> Slovo a
l = head
d :: [Slovo a] -> Slovo a
d = last

-- |function returning a dual of a letter in an alphabet
dual :: Eq a => Abeceda a -> Slovo a -> Slovo a
dual (Abeceda []) s = error("Nema duala - slovo nije u abecedi.")
dual (Abeceda [c]) s | s==c = c
                     | otherwise = error("No dual - Letter is not from the alphabet.")
dual (Abeceda cs) s | s==lijevi = desni
                    | s==desni = lijevi
                    | otherwise = dual (Abeceda (tail (init cs))) s
                     where lijevi= l cs
                           desni= d cs

-- functions returning duals of left/right letters of a word
l' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
l' a= (dual a).l
d' :: Eq a => Abeceda a -> [Slovo a] -> Slovo a
d' a= (dual a).d

ljuska :: [Slovo a]->[Slovo a]
ljuska xs=[l xs,d xs]


-- |distance between letters in an alphabet
dd :: Eq a => Abeceda a -> Slovo a -> Slovo a -> Int
dd a x y| m==Nothing || n==Nothing = error("Cannot calculate distance. Letter is not from the alphabet.")
        | otherwise = abs (fromJust m-fromJust n)
  where m=x `elemIndexAbc` a
        n=y `elemIndexAbc` a

-- |word length
duljina :: Eq a => Abeceda a -> Rijec a -> Int
duljina a []=0
duljina a [c]=0
duljina a (c1:c2:cs)= - dd a c1 c2 - duljina a (c2:cs)

defekt :: Eq a => Abeceda a -> Rijec a -> Int
defekt a x| (d x) `elemIndexAbc` a==Nothing || (l x) `elemIndexAbc` a==Nothing = error("Letter is not from the alphabet.")
          | otherwise = fromJust ((d x) `elemIndexAbc` a) - fromJust((l x) `elemIndexAbc` a)

sDuljina :: Eq a => Abeceda a -> Rijec a -> Rational
sDuljina a x=(fromIntegral (duljina a x)-fromIntegral (defekt a x))/2

pDuljina :: Eq a => Abeceda a -> Rijec a -> Rational
pDuljina a x=(fromIntegral (duljina a x)+fromIntegral (defekt a x))/2

-- word shortening
krati1 :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function shortening a 4 letter word
krati1 a [p,r,s,t] | duljina a [p,r,s,t]==duljina a [p,t] = [p,t]
                 | otherwise = [p,r,s,t]
krati1 _ cs = cs

krati2 :: Eq a => Abeceda a -> Rijec a -> Rijec a     -- function shortening a 4 letter sub-word
krati2 _ [] = []
krati2 a(c:cs) | length kr==2 = kr ++ drop 3 cs
               | otherwise = c : krati2 a cs
               where kr = ((krati1 a).(take 4)) (c:cs)

krati :: Eq a => Abeceda a -> Rijec a -> Rijec a    -- function fully shortening a word
krati a cs | cs/=kr = krati a kr
           | otherwise = cs
           where kr = krati2 a cs

oBaza :: Abeceda a -> [Rijec a]
oBaza abc = [[x,y]|x <- unAbeceda abc,y <- unAbeceda abc]

-- quasi-order
qleq :: Eq a => Abeceda a->Rijec a-> Rijec a->Bool
qleq a x y |(ggeq a (l x) (l y)) && (gleq a (d x) (d y)) = True
           | otherwise = False

qgeq :: Eq a => Abeceda a->Rijec a-> Rijec a->Bool
qgeq a x y |(gleq a (l x) (l y)) && (ggeq a (d x) (d y)) = True
           | otherwise = False

qeq :: Eq a => Abeceda a->Rijec a-> Rijec a->Bool
qeq a x y |(qleq a x y) && (qleq a y x) = True
          | otherwise = False

sNula :: Abeceda a->Rijec a
sNula (Abeceda a)=[d a,l a]

pNula :: Abeceda a->Rijec a
pNula (Abeceda a)=[l a,d a]

-- jezgre
opJs :: Abeceda a->Rijec a->Rijec a
opJs a x=sNula a++x++sNula a

opJp :: Abeceda a->Rijec a->Rijec a
opJp a x=pNula a++x++pNula a

-- ljuske
sLjuska :: Eq a => Abeceda a->Rijec a->Rijec a
sLjuska a x=l x:(inf a (l x) (d x)):(sup a (l x) (d x)):d x:[]

pLjuska :: Eq a => Abeceda a->Rijec a->Rijec a
pLjuska a x=l x:(sup a (l x) (d x)):(inf a (l x) (d x)):d x:[]

-- operatori inverzivnosti
opIs :: Eq a => Abeceda a->Rijec a->Rijec a
opIs a x=l x:infx:infx:x++supx:infx:infx:supx:d x:[]
           where infx=(inf a (l x) (d x))
                 supx=(sup a (l x) (d x))

opIp :: Eq a => Abeceda a->Rijec a->Rijec a
opIp a x=l x:supx:supx:x++infx:supx:supx:infx:d x:[]
           where infx=(inf a (l x) (d x))
                 supx=(sup a (l x) (d x))

opSminus :: Eq a => Abeceda a->Rijec a->Rijec a->Rijec a
opSminus a x y=opS a x (opIs a y)

opPminus :: Eq a => Abeceda a->Rijec a->Rijec a->Rijec a
opPminus a x y=opP a x (opIp a y)

-- operatori komplementarnosti
opKs :: Eq a => Abeceda a -> Rijec a -> Rijec a
opKs (Abeceda a) xs=l a:xs++[d a]

opKp :: Eq a => Abeceda a -> Rijec a -> Rijec a
opKp (Abeceda a) xs=d a:xs++[l a]  

-- symmetries
opD :: Eq a => Abeceda a -> Rijec a -> Rijec a
opD _ []=[]
opD a (s:t:ys)=dual a s:dual a t:opD a ys

opE :: Eq a => Abeceda a -> Rijec a -> Rijec a
opE _ []=[]
opE a xs=t:s:opE a ys
        where n=length xs
              t=d xs
              s=xs !! (n-2)
              ys=take (n-2) xs
--opE a (s:t:ys)=opE a ys++[t,s]    --kraće, ali nije po formuli iz teksta

opF :: Eq a => Abeceda a -> Rijec a -> Rijec a
opF a=(opD a).(opE a)

-- permutations
opI :: Eq a => Abeceda a -> Rijec a -> Rijec a
opI a xs=l xs:xs++[d xs]

-- |negation operator
opK :: Eq a => Abeceda a -> Rijec a -> Rijec a
opK a xs=l' a xs:xs++[d' a xs]

opG :: Eq a => Abeceda a -> Rijec a -> Rijec a
opG a xs=d xs:xs++[l xs]

opH :: Eq a => Abeceda a -> Rijec a -> Rijec a
opH a xs=d' a xs:xs++[l' a xs]

-- serial (union) and parallel (intersection) conjecture operators
opS :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opS a xs ys = inf a (l xs) (l ys) : inf a (l xs) (l ys) : xs ++
              [sup a (d xs) (l ys),inf a (d xs) (l ys)] ++ ys ++
              [sup a (d xs) (d ys),sup a (d xs) (d ys)]

opP :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
opP a xs ys = sup a (l xs) (l ys) : sup a (l xs) (l ys) : xs ++
              [inf a (d xs) (l ys),sup a (d xs) (l ys)] ++ ys ++
              [inf a (d xs) (d ys),inf a (d xs) (d ys)]

-- |partitions
particija :: Eq a => Abeceda a -> Rijec a -> Char
particija a xs | lijevi>=*lijevi' && desni<=*desni' && lijevi>*desni ='T'
               | lijevi<=*lijevi' && desni<=*desni' && not (lijevi==lijevi' || desni==desni') ='A'
               | lijevi==lijevi' && desni==desni' ='0'
               | lijevi>=*lijevi' && desni>=*desni' && not (lijevi==lijevi' || desni==desni')  ='Z'
               | lijevi<=*lijevi' && desni>=*desni' && lijevi<*desni  ='F'
         where lijevi=l xs
               desni=d xs
               lijevi'=l' a xs
               desni'=d' a xs
               x >* y| m==Nothing || n==Nothing =error("Letter not in alphabet.")
                     | otherwise = (fromJust m) > (fromJust n)
                      where m=x `elemIndexAbc` a
                            n=y `elemIndexAbc` a
               x >=* y = x >* y || x==y
               x <* y = not (x >=* y)
               x <=* y = not (x >* y)

-- implications
pimp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pimp a x y=(opK a x) `opP'` y
    where opP'=opP a

simp :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
simp a x y=(opK a x) `opS'` y
    where opS'=opS a

-- equivalencies
pekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
pekv a x y=(x `pimp'` y) `opS'` (y `pimp'` x)
    where opS'=opS a
          pimp'=pimp a

sekv :: Eq a => Abeceda a -> Rijec a -> Rijec a -> Rijec a
sekv a x y=(x `simp'` y) `opP'` (y `simp'` x)
    where opP'=opP a
          simp'=simp a

-- |tautologies and rules
taut :: Eq a => Abeceda a -> Rijec a ->Rijec a -> Rijec a -> [(String,Char)]
taut a x y z= zip
  ["1.ZAKON ISKLJUCENJA TRECEG",
  "2.DUAL ZAKONA ISKLJUCENJA TRECEG",
  "3.TRANZITIVNOST IMPLIKACIJE",
  "4.DUAL TRANZITIVNOSTI IMPLIKACIJE",
  "5.ZAKON DVOSTRUKE NEGACIJE",
  "6.ZAKON KONTRAPOZICIJE",
  "7.DUAL ZAKONA KONTRAPOZICIJE",
  "8.ZAKON APSORPCIJE ILI UPIJANJA",
  "9.DUAL ZAKONA APSORPCIJE ILI UPIJANJA",
  "10.MODUS PONENDO PONENS",
  "11.DUAL MODUSA PONENDO PONENS",
  "12.MODUS TOLENDO TOLLENS",
  "13.DOKAZ NABRAJANJEM",
  "14.DOVODJENJE DO PROTURJECJA"]
  (map (particija a) [
    x `opP'` (neg x),
    x `opS'` (neg x),
    ((x `pimp'` y) `opS'` (y `pimp'` z)) `pimp'` (x `pimp'` z),
    ((x `simp'` y) `opP'` (y `simp'` z)) `simp'` (x `simp'` z),
    (neg (neg x)) `pekv'` x,
    (x `pimp'` y) `pimp'` ((neg y) `pimp'` (neg x)),
    (x `simp'` y) `simp'` ((neg y) `simp'` (neg x)),
    (x `opP'` (y `opP'` x)) `pimp'` x,
    (x `opS'` (y `opS'` x)) `simp'` x,
    (x `opS'` (x `pimp'` y)) `pimp'` y,
    (x `opP'` (x `simp'` y)) `simp'` y,
    ((neg x) `opS'` ((neg x) `pimp'` (neg y))) `pimp'` (neg y),
    ((x `pimp'` z) `opS'` (y `pimp'` z)) `pimp'` ((x `opP'` y) `pimp'` z),
    ((x `pimp'` y) `opS'` (x `pimp'` (neg y))) `pimp'` (neg x)
    ]
  )
  where opP'=opP a
        opS'=opS a
        pimp'=pimp a
        simp'=simp a
        pekv'=pekv a
        sekv'=sekv a
        neg=opK a
        
{-# START_FILE Main.hs #-}
{-# LANGUAGE UndecidableInstances, FlexibleInstances, OverlappingInstances, IncoherentInstances #-}
module Main where
import Linguistic2

main :: IO ()

-- show Set alphabet and variables.
main=do let {a = lista2abeceda ['a'..'z'];                  -- <-- CHANGE
             x = lista2rijec "ante";                      -- <-- CHANGE
             y = lista2rijec "koba";                        -- <-- CHANGE
             -- ADD MORE VARIABLES IF NECESSARY
             }
-- /show


-- show Operators that can be used in the expression
        let opDa =opD a;         -- unary
            opEa =opE a;         -- unary
            opFa =opF a;         -- unary
            opIa =opI a;         -- unary
            opGa =opG a;         -- unary
            opHa =opH a;         -- unary
            neg =opK a;          -- unary
            opKsa =opKs a;       -- unary
            opKpa =opKp a;       -- unary
            opJsa =opJs a;       -- unary
            opJpa =opJp a;       -- unary
            sLjuskaa =sLjuska a; -- unary
            pLjuskaa =pLjuska a; -- unary
            opIsa =opIs a;       -- unary
            opIpa =opIp a;       -- unary
            opSa =opS a;           -- binary
            opPa =opP a;           -- binary
            opSminusa =opSminus a; -- binary
            opPminusa =opPminus a; -- binary
            pimpa =pimp a;         -- binary
            simpa =simp a;         -- binary
            pekva =pekv a;         -- binary
            sekva =sekv a;         -- binary
            kratia =krati a;     -- unary
-- /show

        let izraz=

-- show Set the expression
              opIsa y                        -- <-- CHANGE
-- /show

        putStrLn (ispis izraz);
        putStrLn ("Length:"++show (duljina a izraz));
        putStrLn ("Defect:"++show (defekt a izraz));
        putStrLn ("s-length:"++show (sDuljina a izraz));
        putStrLn ("p-length:"++show (pDuljina a izraz));
```
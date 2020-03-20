# Haskell en 15 minutes

# Introduction

Haskell est un langage __fonctionnel__, __pur__, __non-strict__, __typé statiquement__.

## Fonctionnel

Contrairement à un langage impératif, on ne définit pas une séquence d'instructions à exécuter, mais une expression à évaluer.

Les fonctions sont des valeurs comme les autres (fonctions d'ordre supérieur) : les fonctions peuvent prendre comme argument d'autres fonctions et renvoyer une nouvelle fonction.

@@@
Avoir des fonctions d'ordre supérieur, des fonctions anonymes (lambda), et pouvoir appliquer partiellement ces fonctions permet, de manière très élégante et concise, de combiner des fonctions simples (et donc simples à écrire et tester) en fonctions plus complexes, qui peuvent à leur tour être combinées à d'autres fonctions. Ceci contribue à rendre les programmes Haskell (nettement) plus courts que des programmes impératifs équivalents. Exemples :
``` haskell
import Data.List(sort)
-- show
reversedSort = reverse . sort -- on commence par appliquer la fonction sort aux
                              -- arguments de sortReversed, puis on applique la fonction
                              -- reverse au résultat obtenu
foo = map (\x -> 3 * x + 5)   -- map f xs applique la fonction f à chaque élément de la
                              -- liste xs ; map est une fonction comme une autre
                              -- (contrairement aux boucles for ou while de la plupart
                              -- des langages)
```
@@@

@@@
Par défaut, Haskell est un langage fonctionnel, mais il offre également un mode impératif.
Voir [Beautiful concurrency](http://research.microsoft.com/pubs/74063/beautiful.pdf):  _Haskell is, first and foremost, a functional language. Nevertheless, I think that
it is also the world’s most beautiful imperative language. [Simon Peyton Jones]_
@@@

## Pur

Pas d'effet de bord : données immuables (les "variables" sont constantes), pas d'I/O caché (pas de communication avec le monde extérieur : écran, clavier, disque, réseau...).

@@@
Les fonctions pures sont plus faciles à écrire, plus faciles à tester, plus faciles à utiliser : un appel de fonction ne peut pas modifier les valeurs existantes ni modifier l'état du système. Une fonction pure appelée plusieurs fois avec les mêmes arguments renvoie toujours la même valeur : on parle de _referential transparency_. Cela permet l'évaluation lazy des valeurs et rend la composition de fonctions plus simple. Cela permet aussi une parallélisation des calculs.
@@@

@@@
Un programme sans I/O n'a aucun intérêt (à part consommer de la mémoire et charger le CPU). Haskell offre donc évidemment un moyen de communiquer avec le reste du système : les _actions `IO`_. Grâce au système de type, le code pur est séparé du code impur (basé sur ces actions `IO`) de manière très claire et très propre.
@@@

## Non-strict

Lazy : les calculs sont effectués uniquement lorsque leurs résultats sont nécessaires.

@@@
L'évaluation lazy permet de travailler avec des structures de donnée (listes, arbres...) infinies. Elle permet aussi de définir des fonctions qui court-circuitent (comme l'opérateur `||` par exemple).
@@@

@@@
Par défaut, Haskell est lazy, mais il est possible de forcer l'évaluation d'une valeur lorsque cela s'avère nécessaire.
@@@

## Typé statiquement

Tous les types sont connus lors de la compilation. Il n'y a pas de coercion implicite (entier non signé casté en entier signé, entier interprété comme une valeur booléenne...).

Grâce au système d'inférence de types (le compilateur est capable d'inférer le type des valeurs utilisées), le code peut être aussi concis que dans un langage typé dynamiquement, mais avec la sécurité d'un typage statique en plus.

@@@
Le système de types de Haskell est très strict : écrire du code qui compile est plus difficile en Haskell que dans les autres langages (y compris les autres langages typés statiquement). Cela peut sembler gênant, mais c'est une bonne chose : il vaut mieux avoir une erreur à la compilation qu'une erreur lors de l'exécution du programme.
On dit souvent qu'un programme Haskell qui compile est un programme qui fonctionne.
@@@

Le type (la signature) d'une fonction permet de savoir dans une certaine mesure ce qu'elle fait, et (ce qui est plus important) ce qu'elle ne fait pas.

@@@
Que peuvent faire les fonctions dont les signatures sont les suivantes : <hoogle>[a] -> Int</hoogle> ? <hoogle>(a -> b) -> [a] -> [b]</hoogle> ? <hoogle>FilePath -> IO String</hoogle> ?
@@@

Les fonctions et les types peuvent être polymorphiques, ce qui favorise la ré-utilisation de code.

@@@
Par défaut (sans signature explicite), Haskell infère le type le plus générique possible pour les valeurs, en toute sécurité. Par exemple, le type de la fonction `sum` est `Num a => [a] -> a`, ce qui signifie qu'elle s'applique à des listes d'éléments qui se comportent comme des nombres. Si on essaye de l'appliquer à une liste de caractères, on aura une erreur lors de la compilation.
Avec une signature explicite, il est possible de restreindre la portée d'une fonction, mais il est impossible de l'étendre. Si on oublie une containte, le compilateur nous le dit.
@@@


# Outils

- `ghc` : le compilateur Haskell le plus répandu.
- `ghci` : interpréteur Haskell.
- `runhaskell` : pour exécuter des programmes Haskell sans les compiler.
- `cabal` : système de build et de gestion de packages.
- [hackage](http://hackage.haskell.org) : collection de packages Haskell (un package contient un ou plusieurs programmes/librairies).
- [hoogle](http://www.haskell.org/hoogle) : moteur de recherche de packages/librairies/fonctions/types Haskell.
- [Haskell Platform](http://www.haskell.org/platform) : rassemble le compilateur `ghc` (y compris `ghci` et `runhaskell`), `cabal` et une sélection de packages.


# Syntaxe

La syntaxe Haskell est élégante et concise, mais fort différente de celles de langages plus répandus.

## Quelques types de base

``` haskell
x1 = 34      -- nombre entier
x2 = 1.6     -- nombre fractionnel
x3 = 'z'     -- caractère
x4 = False   -- booléen
```

## Listes

``` haskell
-- Il y a deux moyens de créer une liste :
-- 1. création d'une liste vide
emptyList = []
-- 2. création d'une liste à partir d'un élément et d'une liste
prefixedList = 'b' : emptyList
-- Syntactic sugar
s123 = [1, 2, 3] -- équivalent à 1:2:3:[]
sabc = "abc" -- équivalent à 'a':'b':'c':[]
```

## Tuples

``` haskell
t1 = (1, 2)            -- paire d'entiers
t2 = (3, 'a', "alpha") -- triplet comprenant un entier, un caractère et une string
```

## Fonctions

``` haskell
-- fonctions unaires
plus2 x = x + 2

-- fonctions binaires
sumOfSquares x y = x * x + y * y

-- application de fonctions (pas de parenthèses) :
c = sumOfSquares 2.3 6.5
```

## Pattern matching et pattern guards

``` haskell
contains []     e             = False
contains (x:xs) e | x == e    = True
                  | otherwise = contains xs e
```

## Let ... in

``` haskell
-- l'indentation est importante :
f x y = let a = 3.2
            b = 1.9
        in a * x - b * y
```

## ... where

``` haskell
f' x y = a * x - b * y where  -- l'apostrophe est autorisée dans les noms
    a = 3.2                   -- de variables
    b = 1.9
```

## if ... then ... else

``` haskell
contains' []     _ = False                           -- _ est un wildcard
contains' (x:xs) e = if x == e then True
                               else contains' xs e
```

## Fonctions et opérateurs

Un opérateur (`+` par exemple) est une fonction comme une autre, avec deux petites différences :

- leur nom est composé des caractères suivants : `# $ % & * + . / < = > ? @ \ ^ | - ~`
- par défaut, ils sont appliqués entre le premier et le deuxième argument ; pour utiliser la syntaxe préfixe (c'est-à-dire la syntaxe d'application de fonction), il faut les entourer de parenthèses.
``` haskell
a  = 1.2 + 3.4
a' = (+) 1.2 3.4
```

Les fonctions peuvent aussi être appliquées entre le premier et le deuxième argument ; il suffit d'entourer leur noms de backticks (`` `) :
``` haskell
found  = contains [1..7] 5
found' = [1..7] `contains` 5
```

## Type annotations

``` haskell
-- x1 est un Int
x1 :: Int
x1 = 34

-- plus2 est une fonction (polymorphique) de a vers a
-- le type a doit faire partie de la classe Num
plus2 :: Num a => a -> a
plus2 x = x + 2

-- contains est une fonction qui prend comme arguments
-- 1) une liste de a (une liste d'éléments de type a)
-- 2) un a (une valeur de type a)
-- et renvoie un booléen
-- le type a doit faire partie de la classe Eq
contains :: Eq a => [a] -> a -> Bool
contains []     _             = False
contains (x:xs) e | x == e    = True
                  | otherwise = contains xs e
```

@@@
`ghci` peut aider à se faire une idée des types :
```
$ ghci h15m.hs
GHCi, version 7.4.2: http://www.haskell.org/ghc/  :? for help
Loading package ghc-prim ... linking ... done.
Loading package integer-gmp ... linking ... done.
Loading package base ... linking ... done.
[1 of 1] Compiling Main             ( h15m.hs, interpreted )
Ok, modules loaded: Main.
*Main> :t plus2
plus2 :: Num a => a -> a
*Main> :t contains
contains :: Eq a => [a] -> a -> Bool
```
@@@


# Un programme d'exemple

Voici un exemple complet qui illustre également des éléments de syntaxe ignorés dans la section précédente (`import`, `do`, `<-`, `case` ... `of`) ainsi que le type `Maybe`.
``` haskell active
import Data.Char (toUpper, toLower) -- importe les fonctions toUpper et
                                    -- toLower du module Data.Char
main :: IO ()
main = do               -- do marque le début d'une séquence d'opérations
    input <- getLine    -- <- associe à input le résultat de getLine
    putStrLn (caesar input)
    main

caesar :: String -> String
caesar = map (toUpper . rot13 . toLower)

rot13 :: Char -> Char
rot13 x = case lookup x table of
    Nothing -> x
    Just r  -> r
  where
    table = zip alphabet alphabet13
    alphabet = ['a'..'z']
    alphabet13 = drop 13 alphabet ++ take 13 alphabet
```

@@@
- Utiliser <hoogle>($)</hoogle> pour supprimer les parenthèses.
- Utiliser <hoogle>forever</hoogle> au lieu d'une récursion explicite.
- Utiliser <hoogle>fromMaybe</hoogle> au lieu du `case` ... `of`.
- Utiliser <hoogle>(>>=)</hoogle> pour éviter la notation `do`.
@@@

# Haskell en plus de 15 minutes

- [Learn You a Haskell for Great Good!](http://learnyouahaskell.com/chapters)
- [School of Haskell](https://www.fpcomplete.com/school)
- [Real World Haskell](http://book.realworldhaskell.org/read/)

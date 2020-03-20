# Skew-Monoidal Normalization

# Pass it to the right... 
Skew-monoidal category normalization

## Introduction
While trying to normalize arrow expression produced by a custom quasiquoter I asked around (thanks edwardk) and was pointed toward [Coherence for Skew-Monoidal Categories](http://arxiv.org/pdf/1406.2064v1.pdf). I do not have a rigorous background in category theory and notation, so I took this as the chance to implement and learn. You can follow along with the paper as I tried to keep as close as possible to its order of presenetation.

## Skew
[Standard monoidal categories](http://hackage.haskell.org/package/categories-1.0.7/docs/Control-Category-Monoidal.html#t:Monoidal) include many functions that are *balanced* in that when there is a bifunctor (a type like (a,b) or Either a b) there is no preference for one or the other component of it. This lack of preference leads to some ambiguity and inability to define a clear path to a normal form. There are multiple ways to create to any form and multiple ways to get to them. So to resolve this Tarmo Uustalu, the author, picks a preferred direction.

## Setup
At first I started with trying to defined a `class SkewMonoidal`, but it turns out to be not needed yet. Though I believe it would be helpful to make this a usable normalization procedure. Instead let's start on pg. 3 section 3. 
First the author defines a free symetric monoidal category with Objects from a set Var.

![Untitled.png](https://www.fpcomplete.com/media/6cd546d5-a577-4fe1-a8f9-b765aea4de00.png)
``` haskell
data Tm = X' Var | I | Tm :-: Tm
```
There is also a map between two objects by a set of rule expressions. This tells us that we will build this datatype from primitive rules and some combinators:

![Untitled.png](https://www.fpcomplete.com/media/d626b1a0-cd55-4846-b522-0659435d2755.png)
``` haskell
data Rule = IdRule | Dot Rule Rule | Cross Rule Rule | La | Rh | Asc deriving Show

-- | Interpretation of rules and their composition.
evalRule :: Rule -> Tm -> Tm
evalRule IdRule = id
evalRule (Dot a b) = evalRule a . evalRule b
evalRule (Cross a b) = \(c :-: d) -> evalRule a c :-: evalRule b d
evalRule La = \(I :-: a) -> a
evalRule Rh = \a -> a :-: I
evalRule Asc = \case
    ((a :-: b) :-: c) -> (a :-: (b :-: c))
    a -> error $ show a
-- These lambdas are NOT total! Bad parens later on caused some errors.
```

## Normal forms.
Next we define a normal form to be isomorphic to a list of Terms:

![Untitled.png](https://www.fpcomplete.com/media/f88ee3d2-2a1d-438d-b590-88f623094cac.png)
``` haskell
data Nf = J | Var :.: Nf deriving Show
```
Next we write a function to move from a the normal form back to the original Terms:

![Untitled.png](https://www.fpcomplete.com/media/f05d04a1-7640-419a-a275-f28580bcbe2d.png)
``` haskell
emb :: Nf -> Tm
emb J = I
emb (a :.: n) = X' a :-: emb n
```
We will need a helper function which walks along a structure of terms and folds it into a `Nf`. I could not figure out a way to get outfix operators in Haskell, so I callted it splay.

![Untitled.png](https://www.fpcomplete.com/media/70285558-8bcd-4f12-91a2-fb8b57193fee.png)
``` haskell
splay :: Tm -> Nf -> Nf
splay (X' x) n = x :.: n
splay I n = n
splay (a :-: b) n = splay a (splay b n)

-- | Every object expression is assigned a normal form using the helper.
nf :: Tm -> Nf
nf a = splay a J
```

## Home Stretch
By starting with Identity in `Nf`, the `nf` function creates the normal form for the expression via manually unpacking the structure of the expression. Next we define a new operator. Don't ask me how to create this from scratch, I merely translated:

![Untitled.png](https://www.fpcomplete.com/media/766622fa-81c2-4687-93f3-7174a907f313.png)
``` haskell
splat :: Tm -> Nf -> Rule
splat (X' _) _ = IdRule
splat I _ = La
splat (a :-: b) n = splat a (splay b n)
                    `Dot` (IdRule `Cross` splat b n `Dot` Asc)
```
That is the helper function which creates the rule expression. It is used in the normalizing map expression function defined next; `nm`. The `J` and `Rh` are to allow `splat` to do its job, but then to remove it.

![Untitled.png](https://www.fpcomplete.com/media/f238ba5f-56ed-435c-9367-3b333e8e6592.png)
``` haskell
nm :: Tm -> Rule
nm a = splat a J `Dot` Rh
```
The idea is that `nm` and `nf` should produce the same normal form. This is proved by the author elsewhere. 5 laws are required to be satisfied by the skew-monoidal category, so I should write those down in a nice Haskell format soon.  With this, we are done! The `nm` function will produce a rule expression which can be evaluated with `evalRule` along with a Term to produce a new Term.

## Testing
Let's give it a shot
``` active haskell
{-# LANGUAGE LambdaCase #-}
{-# LANGUAGE GADTs #-}
{-# LANGUAGE TypeOperators #-}
import Prelude hiding (id,(.))
import Control.Category

-- show
s,t,u,v,w :: Tm
s = X' A
t = X' A :-: I :-: X' B
u = I :-: I :-: (X' A :-: I) :-: X' B
v = I :-: u
w = t :-: t

main :: IO ()
main = do
    printTest s
    printTest t
    printTest u
    printTest v
    printTest w

-- /show
    
printTest :: Tm -> IO ()
printTest a = do
    let b = nf a
        c = nm a
        d = evalRule c a
    putStrLn $ "Original Object:  " ++ show a
    putStrLn $ "Normal Form    :  " ++ show b
    putStrLn $ "Rewrite Rules  :  " ++ if length (show c) > 70 then take 65 (show c) ++ " ..." else show c
    putStrLn $ "Rewriten Object:  " ++ show d
    putStrLn ""

-- A Skew-monoidal category is a category `k` together with a
-- distinguished object `Id`, a functor `bimap`, and three
-- natural transformations, `lambda`, rho, and disassociate.
-- It is half of an Associative, Monoidal, Cartesian

-- | Objects of free symetric monoidal category with Objects from Var
data Var = A | B | C deriving Show
data Tm = I | X' Var  | Tm :-: Tm deriving Show

-- | Maps between two objects.
data Rule = IdRule | Dot Rule Rule | Cross Rule Rule | La | Rh | Asc deriving Show

-- | Interpretation of rules and their composition.
evalRule :: Rule -> Tm -> Tm
evalRule IdRule = id
evalRule (Dot a b) = evalRule a . evalRule b
evalRule (Cross a b) = \(c :-: d) -> evalRule a c :-: evalRule b d
evalRule La = \(I :-: a) -> a
evalRule Rh = \a -> a :-: I
evalRule Asc = \case
    ((a :-: b) :-: c) -> (a :-: (b :-: c))
    a -> error $ show a

-- Do we need to encode all the rules on page 4? Or are those just expected laws?

-- | We define "normal forms" of object expression as Nf
data Nf = J | Var :.: Nf deriving Show

-- | Normal forms embed into object expressions.
emb :: Nf -> Tm
emb J = I
emb (a :.: n) = X' a :-: emb n

-- | Let ||-|| be a function ... i'm calling it splay
splay :: Tm -> Nf -> Nf
splay (X' x) n = x :.: n
splay I n = n
splay (a :-: b) n = splay a (splay b n)

-- | Every object expression is assigned a normal form.
nf :: Tm -> Nf
nf a = splay a J

-- | In Lemma 3, Let now `<<->>` ... i'm calling it splat
splat :: Tm -> Nf -> Rule
splat (X' _) _ = IdRule
splat I _ = La
splat (a :-: b) n = splat a (splay b n)
                    `Dot` (IdRule `Cross` splat b n `Dot` Asc)

-- | "Normalizing" map expression.
nm :: Tm -> Rule
nm a = splat a J `Dot` Rh
```
We can see that the normal form and rewritten form match. We got this result by applying our rewrite rules, the skewed portions of the monoidal category.

## Ending note
This was a bit of pleasant translation. Is this construct useful, should I include it in Hackage? Any corrections comments or improvements are very welcome.

Entire code:
@@@
``` active haskell
{-# START_FILE Skew.hs #-}
{-# LANGUAGE LambdaCase #-}
{-# LANGUAGE GADTs #-}
{-# LANGUAGE TypeOperators #-}
{- |
Module      :  Control.Category.Monoidal.Skew
Description :  skew-monoidal
Copyright   :  (c) 2015 Thomas Bereknyei
License     :  BSD3
Maintainer  :  Thomas Bereknyei <tomberek@gmail.com>
Stability   :  unstable
Portability :  GADTs,LambdaCase

Implements normalization from: <http://arxiv.org/pdf/1406.2064v1.pdf>

The following is almost directly pulled out from the paper.
This served as a good exercise in translating category theory
notation into Haskell code. I welcome any corrections or
suggestions for improvement.
-}
module Skew where

import Prelude hiding (id,(.))
import Control.Category

-- A Skew-monoidal category is a category `k` together with a
-- distinguished object `Id`, a functor `bimap`, and three
-- natural transformations, `lambda`, rho, and disassociate.
-- It is half of an Associative, Monoidal, Cartesian

-- | Objects of free symetric monoidal category with Objects from Var
data Var = A | B | C deriving Show
data Tm = I | X' Var  | Tm :-: Tm deriving Show

-- | Maps between two objects.
data Rule = IdRule | Dot Rule Rule | Cross Rule Rule | La | Rh | Asc deriving Show

-- | Interpretation of rules and their composition.
evalRule :: Rule -> Tm -> Tm
evalRule IdRule = id
evalRule (Dot a b) = evalRule a . evalRule b
evalRule (Cross a b) = \(c :-: d) -> evalRule a c :-: evalRule b d
evalRule La = \(I :-: a) -> a
evalRule Rh = \a -> a :-: I
evalRule Asc = \case
    ((a :-: b) :-: c) -> (a :-: (b :-: c))
    a -> error $ show a

-- Do we need to encode all the rules on page 4? Or are those just expected laws?

-- | We define "normal forms" of object expression as Nf
data Nf = J | Var :.: Nf deriving Show

-- | Normal forms embed into object expressions.
emb :: Nf -> Tm
emb J = I
emb (a :.: n) = X' a :-: emb n

-- | Let ||-|| be a function ... i'm calling it splay
splay :: Tm -> Nf -> Nf
splay (X' x) n = x :.: n
splay I n = n
splay (a :-: b) n = splay a (splay b n)

-- | Every object expression is assigned a normal form.
nf :: Tm -> Nf
nf a = splay a J

-- | In Lemma 3, Let now `<<->>` ... i'm calling it splat
splat :: Tm -> Nf -> Rule
splat (X' _) _ = IdRule
splat I _ = La
splat (a :-: b) n = splat a (splay b n)
                    `Dot` (IdRule `Cross` splat b n `Dot` Asc)

-- | "Normalizing" map expression.
nm :: Tm -> Rule
nm a = splat a J `Dot` Rh

{-  Would this be useful? Can translate into our Tm as an intermediate, normalize, translate back?
import Control.Categorical.Bifunctor

class Bifunctor p k k k => SkewMonoidal (k :: * -> * -> *) (p :: * -> * -> *) where
    type Id (k :: * -> * -> *) (p :: * -> * -> *) :: *
    lam :: (Id k p `p` a) `k` a
    rho :: a `k` (a `p` Id k p)
    dis :: ((a `p` b) `p` c) `k` (a `p` (b `p` c))

-- Free Skew-monoidal Category includes embedding for
-- normalization.
data SkewF f where
    Id :: SkewF f
    XXX :: f -> f -> SkewF f
    Lam :: f -> SkewF f
    Rho :: f -> SkewF f
    Dis :: f -> SkewF f
    Lift :: Var -> SkewF f
    deriving Show

newtype Fix f = Fix (f (Fix f))
type Skew = Fix SkewF

-- Smart Constructors
i :: Skew
i = Fix Id
x :: (f -> SkewF Skew) -> f -> Skew
x a b = Fix $ a b
a *.* b = Fix (a `XXX` b)
l,r,d :: Skew -> Skew
l = x Lam
r = x Rho
d = x Dis
lift :: Var -> Skew
lift = Fix . Lift

instance (Show (f (Fix f))) => Show (Fix f) where
    showsPrec p (Fix x) = showParen (p >= 11) (showString "Fix " . showsPrec 11 x)
---}
```
@@@
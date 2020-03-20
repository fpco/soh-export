# 3 Adding Regular Lambdas

## Adding Regular Lambdas 

I will now add regular lambdas to my LLC. Regular lambdas behave just like Haskell functions, and can be directly represented as such; however a little care needs to be taken to ensure that regular lambdas are not given linear arguments. The exact behavior of regular lambdas in LLC is spelled out in the following typing rules; for a more in-depth presentation, see Frank Pfenning's [notes](http://www.cs.cmu.edu/~fp/courses/15816-f01/handouts/linlam.pdf).

I'll begin with a new typing judgement which allows for regular hypotheses and linear hypotheses:

```
G ; Di \ Do  |-  m :: A
```
where `G`, the regular context, is a set of typed variables, and `Di` and `Do` are the linear input and output contexts as before. The following typing rules describe the LLC with regular lambdas. The rules for linear functions are essentially unchanged.
```
-------------------------------------- (lvar) 
G ; D, x::A, D' \ D, _, D'  |-  x :: A
```
Note that `x` can appear anywhere in the linear context.
```
G ; x::A, Di \ _, Do  |-  m :: B
------------------------------------ (-<>I) 
G ; Di \ Do  |-  \x -<> m :: A -<> B


G ; Di \ D  |-  m :: A -<> B     G ; D \ Do  |-  n :: A
------------------------------------------------------- (-<>E)
G ; Di \ Do  |-  m ^ n :: B
```
The rules for regular functions are essentially the standard typing rules for functions with the caveat that arguments to regular functions can't depend upon any linear variables.
```
   x::A in G                 
-------------------- (var)
G ; D \ D |-  x :: A      
```
Note the effective linear context must be empty.

```
x::A, G ; Di \ Do  |-  m :: B
---------------------------------- (->I)
G ; Di \ Do  |-  \x -> m :: A -> B


G ; Di \ Do  |-  m :: A -> B     G ;  \  |-  n :: A
--------------------------------------------------- (->E)
G ; Di \ Do  |-  m $ n :: B
```
Note regular argument, `n`, cannot depend upon any linear variables.

It is straightforward to add regular functions to our
interpreter. First we add them to the syntax and semantics of the
language. 

```active haskell
-- /show
{-# LANGUAGE 
  DataKinds,
  FlexibleContexts,
  FlexibleInstances,
  FunctionalDependencies,
  KindSignatures,
  MultiParamTypeClasses,
  NoMonomorphismRestriction,
  OverlappingInstances,
  RankNTypes, 
  TypeFamilies,
  TypeOperators,
  UndecidableInstances
 #-}
--
-- Type level Nats
--
data Nat = Z | S Nat

--
-- Type level Nat equality
--
class EQ (x::Nat) (y::Nat) (b::Bool) | x y -> b
instance EQ x x True
instance (b ~ False) => EQ x y b

--
-- Type level machinery for consuming a variable in a list of variables.
--
class Consume (v::Nat) (i::[Maybe Nat]) (o::[Maybe Nat]) | v i -> o
class Consume1 (b::Bool) (v::Nat) (x::Nat) (i::[Maybe Nat]) (o::[Maybe Nat]) | b v x i -> o
instance (Consume v i o) => Consume v (Nothing ': i) (Nothing ': o)
instance (EQ v x b, Consume1 b v x i o) => Consume v (Just x ': i) o
instance Consume1 True v x i (Nothing ': i)
instance (Consume v i o) => Consume1 False v x i (Just x ': o)

type LinVar repr (vid::Nat) a = forall (v::Nat) (i::[Maybe Nat]) (o::[Maybe Nat]). (Consume vid i o) => repr v i o a
-- show Adding regular lambdas.
type RegVar repr a = forall (v::Nat) (h::[Maybe Nat]) . repr v h h a
-- /show
-- to distinguish linear arrow from regular arrow
newtype a :-<>: b = Lolli {unLolli :: a -> b}

-- show
class Lin (repr :: Nat -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where

    llam :: (LinVar repr vid a -> repr (S vid) (Just vid ': hi) (Nothing ': ho) b) -> 
            repr vid hi ho (a :-<>: b)
    (<^>) :: repr vid hi h (a :-<>: b) -> repr vid h ho a -> repr vid hi ho b

    lam :: (RegVar repr a -> repr vid hi ho b) -> 
           repr vid hi ho (a -> b)
    (<$>) :: repr vid hi ho (a -> b) -> repr vid ho ho a -> repr vid hi ho b

-- /show
type Defn a = forall repr vid h . (Lin repr) => repr vid h h a

defn :: Defn a -> Defn a
defn x = x

-- show
--t0 :: Defn ((a :-<>: b) :-<>: (a :-<>: b))
t0 = defn $ llam $ \f -> llam $ \x -> f <^> x

-- following won't type check
--t1 :: Defn ((a :-<>: (a :-<>: b)) :-<>: (a :-<>: b))
--t1 = defn $ llam $ \f -> llam $ \x -> f <^> x <^> x

-- following is ok since x is regular variable
-- t2 :: Defn ((a :-<>: (a :-<>: b)) :-<>: (a -> b))
t2 = defn $ llam $ \f -> lam $ \x -> f <^> x <^> x

main = putStrLn "ok"
```

The evaluator `R` is also easily extended to deal with regular functions.

```active haskell
-- /show
{-# LANGUAGE 
  DataKinds,
  FlexibleContexts,
  FlexibleInstances,
  FunctionalDependencies,
  KindSignatures,
  MultiParamTypeClasses,
  NoMonomorphismRestriction,
  OverlappingInstances,
  RankNTypes, 
  TypeFamilies,
  TypeOperators,
  UndecidableInstances
 #-}
--
-- Type level Nats
--
data Nat = Z | S Nat

--
-- Type level Nat equality
--
class EQ (x::Nat) (y::Nat) (b::Bool) | x y -> b
instance EQ x x True
instance (b ~ False) => EQ x y b

--
-- Type level machinery for consuming a variable in a list of variables.
--
class Consume (v::Nat) (i::[Maybe Nat]) (o::[Maybe Nat]) | v i -> o
class Consume1 (b::Bool) (v::Nat) (x::Nat) (i::[Maybe Nat]) (o::[Maybe Nat]) | b v x i -> o
instance (Consume v i o) => Consume v (Nothing ': i) (Nothing ': o)
instance (EQ v x b, Consume1 b v x i o) => Consume v (Just x ': i) o
instance Consume1 True v x i (Nothing ': i)
instance (Consume v i o) => Consume1 False v x i (Just x ': o)

type LinVar repr (vid::Nat) a = forall (v::Nat) (i::[Maybe Nat]) (o::[Maybe Nat]). (Consume vid i o) => repr v i o a

type RegVar repr a = forall (v::Nat) (h::[Maybe Nat]) . repr v h h a

-- to distinguish linear arrow from regular arrow
newtype a :-<>: b = Lolli {unLolli :: a -> b}

class Lin (repr :: Nat -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where

    llam :: (LinVar repr vid a -> repr (S vid) (Just vid ': hi) (Nothing ': ho) b) -> 
            repr vid hi ho (a :-<>: b)
    (<^>) :: repr vid hi h (a :-<>: b) -> repr vid h ho a -> repr vid hi ho b

    lam :: (RegVar repr a -> repr vid hi ho b) -> 
           repr vid hi ho (a -> b)
    (<$>) :: repr vid hi ho (a -> b) -> repr vid ho ho a -> repr vid hi ho b


type Defn a = forall repr vid h . (Lin repr) => repr vid h h a

defn :: Defn a -> Defn a
defn x = x

-- show
newtype R (vid::Nat) (hi::[Maybe Nat]) (ho::[Maybe Nat]) a = R {unR :: a}

instance Lin R where
  llam f = R $ Lolli $ \x -> unR $ f $ R x
  f <^> x = R $ unLolli (unR f) (unR x)

  lam f = R $ \x -> unR $ f $ R x
  f <$> x = R $ unR f (unR x)

eval :: R Z '[] '[] a -> a
eval = unR

--t0 :: Defn ((a :-<>: b) :-<>: (a :-<>: b))
t0 = defn $ llam $ \f -> llam $ \x -> f <^> x

-- following won't type check
--t1 :: Defn ((a :-<>: (a :-<>: b)) :-<>: (a :-<>: b))
--t1 = defn $ llam $ \f -> llam $ \x -> f <^> x <^> x

-- following is ok since x is regular variable
-- t2 :: Defn ((a -> (a :-<>: b)) :-<>: (a -> b))
t2 = defn $ llam $ \f -> lam $ \x -> f <$> x <^> x

main = do
    putStrLn $ (eval $ t2 <^> (lam $ \y -> llam $ \x -> x)) "I was passed to a real function."
    putStrLn "ok"
```

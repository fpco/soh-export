# 2 HOAS Typed Tagless-Final Interpreter

## HOAS Based Interpreter

The preceding deBruijn index based interpreter is impressive, and a great example of the expressive power of the typed tagless-final approach. However, using deBruijn notation can be quite cumbersome, so I will now show how to make an interpreter for a HOAS based version of LLC, where there are named variables which are represented by actual Haskell variables.
```haskell
-- show Standard typed HOAS lambda calculus signature
lam :: (repr a -> repr b) -> repr (a -> b)
app :: repr (a -> b) -> repr a -> repr b
```
A complete example of such an interpreter can be found [here](http://okmij.org/ftp/tagless-final/course/TTF.hs).

The basic idea for interpreting a HOAS LLC is to semantically separate the linear context into two components: one for storing variables and their types, which acts the same as, and can be represented by, the Haskell context (linearity does not affect substitution); and another for enforcing linearity, which must be explicitly represented and manipulated since the Haskell context does not enforce linearity. My abstract linear contexts will be type level lists of `Maybe Nat`; the actual type of each variable is unnecssary for checking linearity. Thus, I will label each variable as it is introduced and add that label to the abstract linear context to enforce linearity. To accomplish this, I will have the representation type carry around a `Nat` to use for the next linear variable: 
```haskell
repr (vid::Nat) (hi::[Maybe Nat]) (ho::[Maybe Nat]) (a::*)
```
where `vid` represents the number of linear variables in scope (and also gives the next linear variable "name" to use), `hi` and `ho` again represent abstractions of the input and output linear contexts; the length of `hi` and `ho` should equal `vid`.

Since I am pushing all of the work of checking linearity into the type level, I will try to use type families and promoted data kinds for this code. I will also introduce a linear function type, <span title='enclosing : not needed in GHC 7.6'>`:-<>:`</span>, in anticipation of adding regular lambdas in the next section. Here is the heart of the HOAS interpreter:
```haskell
newtype a :-<>: b = Lolli {unLolli :: a -> b}

class Lin (repr :: Nat -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where

    llam :: (LinVar repr vid a -> repr (S vid) (Just vid ': hi) (Nothing ': ho) b) -> 
            repr vid hi ho (a :-<>: b)

    (<^>) :: repr vid hi h (a :-<>: b) -> repr vid h ho a -> repr vid hi ho b
```
The semantics (i.e. type) of linear application, `<^>`, are a direct transcription of the `-<>E` typing rule; and identical to the previous first-order version. The semantics of the linear lambda directly mirror the `-<>I` rule except that they add a representation of the variable, with its type, (i.e. `LinVar repr vid a` which I will explain below) to the Haskell context, and they specify, via the `Nothing` in the output context, that this variable, labelled `vid`, must be used in the body of the lambda.

The Haskell type checker needs to enforce that each variable of type `LinVar repr vid a` is only used once; which translates into checking that `Just vid` is in the input linear context and placing `Nothing`, in its place, in the output linear context. To do this, I rely on the following type families:
```haskell
-- show Type level Nat equality
type family EQ (x::Nat) (y::Nat) :: Bool
type instance EQ Z Z = True
type instance EQ Z (S y) = False
type instance EQ (S x) Z = False
type instance EQ (S x) (S y) = EQ x y
-- show Type level machinery for consuming a variable in a list of variables.
type family Consume (vid::Nat) (i::[Maybe Nat]) :: [Maybe Nat]
type instance Consume vid (Nothing ': vs) = (Nothing ': Consume vid vs)
type instance Consume vid (Just v ': vs) = Consume1 (EQ vid v) vid v vs

type family Consume1 (b::Bool) (vid::Nat) (v::Nat) (vs::[Maybe Nat]) :: [Maybe Nat]
type instance Consume1 True vid v vs = Nothing ': vs
type instance Consume1 False vid v vs = Just v ': Consume vid vs
```
With the above, I can write `LinVar` as follows:
```haskell
type LinVar (repr :: Nat -> [Maybe Nat] -> [Maybe Nat] -> * -> *) vid a = 
    forall v h . repr v h (Consume vid h) a
```
Note the explicit use of `forall` which is necessary to allow this type to work no matter what context variable `vid` occurs in.

I can now put everything together and try to type check some terms:
```active haskell
-- /show
{-# LANGUAGE 
  DataKinds,
  FlexibleInstances,
  KindSignatures,
  MultiParamTypeClasses,
  NoMonomorphismRestriction,
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
type family EQ (x::Nat) (y::Nat) :: Bool
type instance EQ Z Z = True
type instance EQ Z (S y) = False
type instance EQ (S x) Z = False
type instance EQ (S x) (S y) = EQ x y

--
-- Type level machinery for consuming a variable in a list of variables.
--
type family Consume (vid::Nat) (i::[Maybe Nat]) :: [Maybe Nat]
type family Consume1 (b::Bool) (vid::Nat) (v::Nat) (vs::[Maybe Nat]) :: [Maybe Nat]
type instance Consume vid (Nothing ': vs) = (Nothing ': Consume vid vs)
type instance Consume vid (Just v ': vs) = Consume1 (EQ vid v) vid v vs
type instance Consume1 True vid v vs = Nothing ': vs
type instance Consume1 False vid v vs = Just v ': Consume vid vs
-- show
type LinVar (repr :: Nat -> [Maybe Nat] -> [Maybe Nat] -> * -> *) vid a = 
     forall v i . repr v i (Consume vid i) a

-- to distinguish linear arrow from regular arrow
newtype a :-<>: b = Lolli {unLolli :: a -> b}

class Lin (repr :: Nat -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where
    llam :: (LinVar repr vid a -> repr (S vid) (Just vid ': hi) (Nothing ': ho) b) -> 
            repr vid hi ho (a :-<>: b)
    (<^>) :: repr vid hi h (a :-<>: b) -> repr vid h ho a -> repr vid hi ho b
    
good = llam $ \f -> llam $ \x -> f <^> x

bad = llam $ \f -> llam $ \x -> f <^> x <^> x
    
main = putStrLn "ok"
```
Unfortunately, both `good` and `bad` above type. The generated type for `good` (something like):
```haskell
llam $ \f -> llam $ \x -> f <^> x
  :: (Lin repr,
      Consume
        ('S vid)
        (Consume1
           {-hi-}(EQ vid ('S vid)){-/hi-}
           vid
           ('S vid)
           ((':) (Maybe Nat) ('Just Nat vid) hi))
      ~ (':)
          (Maybe Nat) ('Nothing Nat) ((':) (Maybe Nat) ('Nothing Nat) ho)) =>
     repr vid hi ho ((a :-<>: b) :-<>: (a :-<>: b))
```
shows that the type checker didn't really do anything; it just generated a big constraint involving the `Consume` type family. In order to make the type checker do some work, I'll have to force a specific `vid` (e.g. `Z`) and possibly also `hi` and `ho`; or I'll have to teach the type checker how to deal with the more general types. I will opt for the latter since I'll want to be able to use my definitions in any context.

Looking closely at the above constraint shows that the problem lies in `EQ`-- `EQ vid (S vid)` did not rewrite to `False`. Of course the specification of `EQ` doesn't explain what to do with open variables, but it seems reasonable that `EQ vid (S vid)` should rewrite to `False`. In order to get this behavior, I really need `EQ` to check whether its arguments can be unified; since there will only be one uninstantiated `Nat` type variable unification will really be sufficient to check `Nat` equality. Fortunately, there is a way to determine whether two types, containing at most 1 type variable, unify, though I can't do it with type families:
```haskell
-- show An EQ which tests unifiability
class EQ (x::Nat) (y::Nat) (b::Bool) | x y -> b
instance EQ x x True
instance (b ~ False) => EQ x y b
```
By making the second instance strictly more general than the first, GHC will try the first instance and, if the two arguments unify, succeed with `True` or fail and succeed on the second instance with `False` (this of course requires overlapping instances and a few other extensions; this should also be possible with type families when overlapping family instances are allowed).

Now I have to turn my `Consume` function into a class:
```haskell
class Consume (v::Nat) (i::[Maybe Nat]) (o::[Maybe Nat]) | v i -> o
class Consume1 (b::Bool) (v::Nat) (x::Nat) (i::[Maybe Nat]) (o::[Maybe Nat]) | b v x i -> o

instance (Consume v i o) => Consume v (Nothing ': i) (Nothing ': o)
instance (EQ v x b, Consume1 b v x i o) => Consume v (Just x ': i) o

instance Consume1 True v x i (Nothing ': i)
instance (Consume v i o) => Consume1 False v x i (Just x ': o)
```

I can now create a type synonym for my definitions and, in order to show that GHC can actually infer my linear types, a convenience function to set the type of an LLC term:
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

-- to distinguish linear arrow from regular arrow
newtype a :-<>: b = Lolli {unLolli :: a -> b}

class Lin (repr :: Nat -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where
    llam :: (LinVar repr vid a -> repr (S vid) (Just vid ': hi) (Nothing ': ho) b) -> 
            repr vid hi ho (a :-<>: b)
    (<^>) :: repr vid hi h (a :-<>: b) -> repr vid h ho a -> repr vid hi ho b
    
-- show
type Defn a = forall repr v h . (Lin repr) => repr v h h a

defn :: Defn a -> Defn a
defn x = x

good :: Defn ((a :-<>: b) :-<>: (a :-<>: b))
good = llam $ \f -> llam $ \x -> f <^> x

-- GHC can actually infer the linear types.
-- Main*> :t defn $ llam $ \f -> llam $ \x -> f <^> x
-- defn $ llam $ \f -> llam $ \x -> f <^> x
--   :: (Lin repr) => repr vid h h ((a :-<>: b) :-<>: (a :-<>: b))
--
good' = defn $ llam $ \f -> llam $ \x -> f <^> x

--bad = defn $ llam $ \f -> llam $ \x -> f <^> x <^> x
    
main = putStrLn "ok"
```
The above code correctly fails to typecheck `bad`. Also, as in the deBruijn index based interpreter, all of the linearity checking is done by the type checker, i.e. I have not yet given any instance for `Lin`. Since I've used HOAS, a type to realize evaluation of LLC terms and an eval function are trivial.
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

-- to distinguish linear arrow from regular arrow
newtype a :-<>: b = Lolli {unLolli :: a -> b}

class Lin (repr :: Nat -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where
    llam :: (LinVar repr vid a -> repr (S vid) (Just vid ': hi) (Nothing ': ho) b) -> 
            repr vid hi ho (a :-<>: b)
    (<^>) :: repr vid hi h (a :-<>: b) -> repr vid h ho a -> repr vid hi ho b
    
type Defn a = forall repr v h . (Lin repr) => repr v h h a

defn :: Defn a -> Defn a
defn x = x
-- show
newtype R (vid::Nat) (hi::[Maybe Nat]) (ho::[Maybe Nat]) a = R {unR :: a}

instance Lin R where
  llam f = R $ Lolli $ \x -> unR $ f $ R x
  f <^> x = R $ unLolli (unR f) (unR x)

eval :: R Z '[] '[] a -> a
eval = unR

good = defn $ llam $ \f -> llam $ \x -> f <^> x

main = do
    putStrLn $ unLolli (eval $ good <^> llam (\x -> x)) "I was passed to a real function."
    putStrLn "ok"
```
One interesting thing to note is that the `llam` rule needs to be eta-expanded (as written above) and cannot be written as
```haskell
  llam f = R $ Lolli $ unR . f . R 
```


# 5 Efficient Resource Management

## Efficient Resource Management

As can be seen in the preceding section's `TI` and `0E` rules, additive units complicate the (relatively) efficient typing rules by allowing an arbitrary subset of the linear context to be consumed. Simply trying all possible subsets is not a feasible strategy. However, it is possible to reformulate these rules so that the choice of which resouces the additive units consume is lazily determined. 

The main idea is to sometimes treat the linear context as affine, where not all variables need to be consumed (i.e. an affine context permits ignoring hypotheses but not copying hypotheses). The switch from linear to affine is triggered by the presence of an additive unit; however, the need to coordinate resource usage between the branches of the `&I` and `+E` rules can trigger a switch back from affine to linear. This allows the additive unit typing rules to simply pass the entire input context through, making no choices about what to consume, and just switching from linear to affine. The determination of what an additive unit consumes is pushed into the `&I` and `+E` typing rules.
This idea comes from work on [resource management](http://www.cs.cmu.edu/~fp/papers/erm97.pdf) for linear logic programming; Frank Pfenning also has some [notes](http://www.cs.cmu.edu/~fp/courses/15816-f01/handouts/linfp.pdf) giving a more thorough presentation of the following typing rules.

I will attach a flag, `v`, to the previous typing judgement to indicate whether the linear context has changed to being affine.
```
G ; Di \ Do  |-v  m :: A
```
`v` will be either true, `t`, or false, `f`. The actual linear context used in a derivation will be `Di - Do'` where `Do' == Do` when `v == f` and `Do' subset Do` otherwise. I will write `v1 || v2` and `v1 && v2` for the boolean disjunction and conjunction of `v1` and `v2`. The modified typing rules follow.

The linear variable rule requires the context be linear.
```
-------------------------------------- (lvar) 
G ; D, x::A, D' \ D, _, D' |-f  x :: A
```

The `-<>I` rule does not affect whether the context has become affine.
```
G ; x::A, Di \ xVar, Do  |-v  m :: B      where  v == t  or  xVar == _    
---------------------------------------------------------------------- (-<>I) 
G ; Di \ Do  |-v  \x -<> m :: A -<> B
```
However, if the context has become affine, then it doesn't matter if variable `x` was consumed.

The `-<>E` rule reflects the fact that the linear variables are shared between the two subderivations:
```
G ; Di \ D  |-v1  m :: A -<> B     G ; D \ Do  |-v2  n :: A 
----------------------------------------------------------- (-<>E)
G ; Di \ Do  |-(v1 || v2)  m ^ n :: B
```
If linear context becomes affine in either subderivation of `-<>E` (or any multiplicative rule) then it becomes affine in the whole derivation-- I am free to split the context however I want; in particular in a way that causes all unused resources in `Do` to flow through the subderivation with the additive unit.

The rules for regular variables and functions are almost unchanged from the previous algorithmic typing rules.
```
   x::A in G                 
--------------------- (var)
G ; D \ D |-f  x :: A      


x::A, G ; Di \ Do  |-v  m :: B
----------------------------------- (->I)
G ; Di \ Do  |-v  \x -> m :: A -> B


G ; Di \ Do  |-v  m :: A -> B     G ;  \  |-v'  n :: A
------------------------------------------------------ (->E)
G ; Di \ Do  |-v  m ^ n :: B
```
In the `->E` rule, whether the context is linear depends only on what happened in the derivation of the function; the argument can't depend on any linear variables.  

The `TI` rule simply switches the context to being affine:
```
----------------------- (TI)
G ; D \ D  |-t  () :: T
```

The `&I` rule requires that both conjuncts consume the same linear variables, therefore the context can only be affine if both subderivations have an affine context. 
```
G ; Di \ D1  |-v1  m :: A     G ; Di \ D2  |-v2  n :: B 
-------------------------------------------------------- (&I)
G ; Di \ (D1 intersect D2)  |-(v1 && v2)  (m,n) :: A & B
```
The `&I` rule must additionally satisfy the following conditions:

 * `v1 == f` and `v2 == f` implies `D1 == D2`
 * `v1 == t` and `v2 == f` implies `D2 subset D1` 
 * `v1 == f` and `v2 == t` implies `D1 subset D2`
 
These conditions ensure that a variable consumed by either conjunct will necessarily be consumed by the whole term.

The rest of the rules require no new concepts and their Haskell encodings can be seen in the code of the following section. 

### Representation in Haskell

The typing rules in the previous section can be directly written in Haskell. The representation type will be a direct transcription of the judgement, which is just the previous representation type extended with a boolean flag:
```haskell
repr (vid::Nat) (tf::Bool) (hi::[Maybe Nat]) (ho::[Maybe Nat]) (a::*)
```
I can encode the `lvar` and `var` rules as
```haskell
type LinVar repr (vid::Nat) a = 
     forall (v::Nat) (i::[Maybe Nat]) (o::[Maybe Nat]) . (Consume vid i o) => repr v False i o a

type RegVar repr a = 
     forall (v::Nat) (i::[Maybe Nat]) . repr v False i i a
```
Note that, as this is HOAS, there is no explicit representation of a variable (there is no `var` or `lvar` member of the `Lin` class); however, there is an explicit representation of the type of a variable. The types I use for linear and regular variables specify that neither turns the context affine, and only the linear variables consume something from the context.

Before I can write down any more typing rules, I need some type level supporting machinery to carry out all the ancillary operations used by the typing rules. In addition to `Nat`, `EQ`, `AnyNat`, and `Consume` from before, I'll need boolean disjunction
```haskell
type family Or (x::Bool) (y::Bool) :: Bool
type instance Or False False = False
type instance Or True False = True
type instance Or False True = True
type instance Or True True = True
```
as well as a class to check that a variable was consumed properly:
```haskell
class VarOk (tf :: Bool) (v :: Maybe Nat)
instance VarOk True (Just v)    -- affine context
instance VarOk True Nothing     -- affine context
instance VarOk False Nothing    -- linear context
```
in other words, the variable only has to be explicitly consumed if the context is linear.
This is a class, rather than a type family, since it will be used as a constraint on (the representation of) a typing rule; it does not compute a new type. 

I can now write the `-<>I` and `-<>E` rules:
```haskell
class Lin (repr :: Nat -> Bool -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where

    llam :: (VarOk tf var) => 
            (LinVar repr vid a -> repr (S vid) tf (Just vid ': hi) (var ': ho) b) -> 
            repr vid tf hi ho (a :-<>: b)

    (<^>) :: repr vid tf0 hi h (a :-<>: b) -> repr vid tf1 h ho a -> 
             repr vid (Or tf0 tf1) hi ho b
```
Note that, the same as before, `llam` takes a Haskell function and thus adds a variable of type `LinVar repr vid a` to the Haskell context; it is the definition of `LinVar` and the `VarOk` constraint which enforce linearity for us.

All of the other multiplicative rules follow the same basic pattern.

In order to write the additive rules (`&I` and `+E`), I will need a subset relation on linear contexts:
```haskell
type family (:<:) (x::[Maybe Nat]) (y::[Maybe Nat]) :: Bool 
type instance '[] :<: '[] = True
type instance (Nothing ': xs) :<: (Nothing ': ys) = xs :<: ys
type instance (Nothing ': xs) :<: (Just y ': ys) = xs :<: ys
type instance (Just x ': xs) :<: (Nothing ': ys) =  False
type instance (Just x ': xs) :<: (Just x ': ys) =  xs :<: ys
```
and an intersection function on linear contexts: 
```haskell
type family Intersect (xs::[Maybe Nat]) (ys::[Maybe Nat]) :: [Maybe Nat]
type instance Intersect '[] '[] = '[]
type instance Intersect (Nothing ': xs) (Nothing ': ys) = (Nothing ': Intersect xs ys)
type instance Intersect (Nothing ': xs) (Just y ': ys) = (Nothing ': Intersect xs ys)
type instance Intersect (Just x ': xs) (Nothing ': ys) = (Nothing ': Intersect xs ys)
type instance Intersect (Just x ': xs) (Just x ': ys) = (Just x ': Intersect xs ys)
```
Additionally, I need a way to express the conditions on the `&I` (and `+E`) rule. The astute reader will also notice that I neglected to provide a boolean conjunction type family. I have opted to fold the conditions into the definition of boolean conjunction: 
```haskell
type family AdditiveAnd (tf0::Bool) (tf1::Bool) (h0::[Maybe Nat]) (h1::[Maybe Nat]) :: Bool
type instance AdditiveAnd False False h h = False 
type instance AdditiveAnd False True h0 h1 = When (h0 :<: h1) False
type instance AdditiveAnd True False h0 h1 = When (h1 :<: h0) False 
type instance AdditiveAnd True True h0 h1 = True 

type family When (x::Bool) (y::Bool) :: Bool
type instance When True y = y
```
`AdditiveAnd` is computing the boolean conjunction of the two `Bool` arguments, except that the appropriate subtyping relation must hold in order for `AdditiveAnd` to compute; i.e. there is no instance for `When False`.

I can now write the additive rules; here are `TI` and `&I`:
```haskell
type a :&: b = (a, b)
type Top = ()

class Lin (repr :: Nat -> Bool -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where

    top :: repr vid True h h Top

    (<&>) :: repr vid tf0 hi h0 a -> repr vid tf1 hi h1 b -> 
             repr vid (AdditiveAnd tf0 tf1 h0 h1) hi (Intersect h0 h1) (a :&: b)
```
Note that `top` makes the context affine, while `<&>` relies on the `AdditiveAnd` calculation to determine whether the context should be affine.

Here are the complete typing rules for LLC.
```haskell
--
-- Linear types
--
newtype a :-<>: b = Lolli {unLolli :: a -> b}
data a :*: b = Tensor a b
data One = One
type a :&: b = (a, b)
type Top = ()
data a :+: b = Inl a | Inr b
data Zero 
newtype Bang a = Bang {unBang :: a}


type LinVar repr (vid::Nat) a = 
     forall (v::Nat) (i::[Maybe Nat]) (o::[Maybe Nat]) . (Consume vid i o) => repr v False i o a

type RegVar repr a = 
     forall (v::Nat) (i::[Maybe Nat]) . repr v False i i a

class Lin (repr :: Nat -> Bool -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where
    llam :: (VarOk tf var) => 
            (LinVar repr vid a -> repr (S vid) tf (Just vid ': hi) (var ': ho) b) -> 
            repr vid tf hi ho (a :-<>: b)
    (<^>) :: repr vid tf0 hi h (a :-<>: b) -> repr vid tf1 h ho a -> 
             repr vid (Or tf0 tf1) hi ho b

    (!) :: repr vid tf h h a -> repr vid False h h (Bang a)
    letBang :: repr vid tf0 hi h (Bang a) -> 
               (RegVar repr a -> repr vid tf1 h ho b) -> 
               repr vid (Or tf0 tf1) hi ho b

    lam :: (RegVar repr a -> repr vid tf hi ho b) -> repr vid tf hi ho (a -> b)
    (<$>) :: repr vid tf0 hi ho (a -> b) -> repr vid tf1 ho ho a -> repr vid tf0 hi ho b

    one :: repr vid False h h One
    letOne :: repr vid tf0 hi h One -> repr vid tf1 h ho a -> 
              repr vid (Or tf0 tf1) hi ho a

    (<*>) :: repr vid tf0 hi h a -> repr vid tf1 h ho b -> 
             repr vid (Or tf0 tf1) hi ho (a :*: b)
    letStar :: (VarOk tf1 var0, VarOk tf1 var1) => 
               repr vid tf0 hi h (a :*: b) -> 
               (LinVar repr vid a -> 
                  LinVar repr (S vid) b -> 
                    repr (S (S vid)) 
                         tf1 
                         (Just vid ': Just (S vid) ': h) 
                         (var0 ': var1 ': ho) 
                         c
               ) -> 
               repr vid (Or tf0 tf1) hi ho c
                    
    top :: repr vid True h h Top

    (<&>) :: repr vid tf0 hi h0 a -> repr vid tf1 hi h1 b -> 
             repr vid (AdditiveAnd tf0 tf1 h0 h1) hi (Intersect h0 h1) (a :&: b)
    pi1 :: repr vid tf hi ho (a :&: b) -> repr vid tf hi ho a 
    pi2 :: repr vid tf hi ho (a :&: b) -> repr vid tf hi ho b 

    inl :: repr vid tf hi ho a -> repr vid tf hi ho (a :+: b)
    inr :: repr vid tf hi ho b -> repr vid tf hi ho (a :+: b)
    letPlus :: (VarOk tf1 var1, VarOk tf2 var2) => 
               repr vid tf0 hi h (a :+: b) ->
               (LinVar repr vid a -> repr (S vid) tf1 (Just vid ': h) (var1 ': ho1) c) ->
               (LinVar repr vid b -> repr (S vid) tf2 (Just vid ': h) (var2 ': ho2) c) ->
               repr vid (Or tf0 (AdditiveAnd tf1 tf2 ho1 ho2)) hi (Intersect ho1 ho2) c

    abort :: repr vid tf hi ho Zero -> repr vid True hi ho a
```

As before, I can add a type synonym to give Haskell a little help with inferring types. This time I will also need to include the fact that the intersection of a context with itself is itself, and that a context is a subset of itself.
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

-- module Linear where

{---------------------------------------------------------------------

Type level machinery

---------------------------------------------------------------------}

--
-- Type level Or
--
type family Or (x::Bool) (y::Bool) :: Bool
type instance Or False False = False
type instance Or True False = True
type instance Or False True = True
type instance Or True True = True

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

--
-- Machinery needed to check/enforce conditions for additive conjunction/disjunction to correctly deal with Top.
--
class VarOk (tf :: Bool) (v :: Maybe Nat)
instance VarOk True (Just v)
instance VarOk True Nothing 
instance VarOk False Nothing

type family (:<:) (x::[Maybe Nat]) (y::[Maybe Nat]) :: Bool 
type instance '[] :<: '[] = True
type instance (Nothing ': xs) :<: (Nothing ': ys) = xs :<: ys
type instance (Nothing ': xs) :<: (Just y ': ys) = xs :<: ys
type instance (Just x ': xs) :<: (Nothing ': ys) =  False
type instance (Just x ': xs) :<: (Just x ': ys) =  xs :<: ys

type family Intersect (xs::[Maybe Nat]) (ys::[Maybe Nat]) :: [Maybe Nat]
type instance Intersect '[] '[] = '[]
type instance Intersect (Nothing ': xs) (Nothing ': ys) = (Nothing ': Intersect xs ys)
type instance Intersect (Nothing ': xs) (Just y ': ys) = (Nothing ': Intersect xs ys)
type instance Intersect (Just x ': xs) (Nothing ': ys) = (Nothing ': Intersect xs ys)
type instance Intersect (Just x ': xs) (Just x ': ys) = (Just x ': Intersect xs ys)

type family When (x::Bool) (y::Bool) :: Bool
type instance When True y = y

type family AdditiveAnd (tf0::Bool) (tf1::Bool) (h0::[Maybe Nat]) (h1::[Maybe Nat]) :: Bool
type instance AdditiveAnd False False h h = False 
type instance AdditiveAnd False True h0 h1 = When (h0 :<: h1) False
type instance AdditiveAnd True False h0 h1 = When (h1 :<: h0) False 
type instance AdditiveAnd True True h0 h1 = True 


{---------------------------------------------------------------------

Final tagless HOAS linear lambda calculus.

---------------------------------------------------------------------}

--
-- Linear types
--
newtype a :-<>: b = Lolli {unLolli :: a -> b}
data a :*: b = Tensor a b
data One = One
type a :&: b = (a, b)
type Top = ()
data a :+: b = Inl a | Inr b
data Zero 
newtype Bang a = Bang {unBang :: a}


type LinVar repr (vid::Nat) a = 
     forall (v::Nat) (i::[Maybe Nat]) (o::[Maybe Nat]) . (Consume vid i o) => 
     repr v False i o a

type RegVar repr a = 
     forall (v::Nat) (i::[Maybe Nat]) . repr v False i i a

class Lin (repr :: Nat -> Bool -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where
    llam :: (VarOk tf var) => 
            (LinVar repr vid a -> repr (S vid) tf (Just vid ': hi) (var ': ho) b) -> 
            repr vid tf hi ho (a :-<>: b)
    (<^>) :: repr vid tf0 hi h (a :-<>: b) -> repr vid tf1 h ho a -> 
             repr vid (Or tf0 tf1) hi ho b

    (!) :: repr vid tf h h a -> repr vid False h h (Bang a)
    letBang :: repr vid tf0 hi h (Bang a) -> 
               (RegVar repr a -> repr vid tf1 h ho b) -> 
               repr vid (Or tf0 tf1) hi ho b

    lam :: (RegVar repr a -> repr vid tf hi ho b) -> repr vid tf hi ho (a -> b)
    (<$>) :: repr vid tf0 hi ho (a -> b) -> repr vid tf1 ho ho a -> repr vid tf0 hi ho b

    one :: repr vid False h h One
    letOne :: repr vid tf0 hi h One -> repr vid tf1 h ho a -> 
              repr vid (Or tf0 tf1) hi ho a

    (<*>) :: repr vid tf0 hi h a -> repr vid tf1 h ho b -> 
             repr vid (Or tf0 tf1) hi ho (a :*: b)
    letStar :: (VarOk tf1 var0, VarOk tf1 var1) => 
               repr vid tf0 hi h (a :*: b) -> 
               (LinVar repr vid a -> 
                  LinVar repr (S vid) b -> 
                    repr (S (S vid)) 
                         tf1 
                         (Just vid ': Just (S vid) ': h) 
                         (var0 ': var1 ': ho) c
               ) -> 
               repr vid (Or tf0 tf1) hi ho c
                    
    top :: repr vid True h h Top

    (<&>) :: repr vid tf0 hi h0 a -> repr vid tf1 hi h1 b -> 
             repr vid (AdditiveAnd tf0 tf1 h0 h1) hi (Intersect h0 h1) (a :&: b)
    pi1 :: repr vid tf hi ho (a :&: b) -> repr vid tf hi ho a 
    pi2 :: repr vid tf hi ho (a :&: b) -> repr vid tf hi ho b 

    inl :: repr vid tf hi ho a -> repr vid tf hi ho (a :+: b)
    inr :: repr vid tf hi ho b -> repr vid tf hi ho (a :+: b)
    letPlus :: (VarOk tf1 var1, VarOk tf2 var2) => 
               repr vid tf0 hi h (a :+: b) ->
               (LinVar repr vid a -> repr (S vid) tf1 (Just vid ': h) (var1 ': ho1) c) ->
               (LinVar repr vid b -> repr (S vid) tf2 (Just vid ': h) (var2 ': ho2) c) ->
               repr vid (Or tf0 (AdditiveAnd tf1 tf2 ho1 ho2)) hi (Intersect ho1 ho2) c

    abort :: repr vid tf hi ho Zero -> repr vid True hi ho a
-- show Full LLC Examples
type Defn tf a = forall repr vid h . 
                 (Lin repr, Intersect h h ~ h, (h :<: h) ~ True) => 
                 repr vid tf h h a
defn :: Defn tf a -> Defn tf a
defn x = x

type MyBool = One :+: One

cond :: Defn False (MyBool :-<>: ((a :&: a) :-<>: a))
cond = llam $ \b -> llam $ \tf -> letPlus b (\x -> letOne x (pi1 tf)) (\x -> letOne x (pi2 tf))

cond1 = defn $ llam $ \b -> llam $ \tf -> letPlus b (\x -> letOne x (pi1 tf)) (\x -> letOne x (pi2 tf))
-- *Main> :t cond1
-- cond1
--   :: (Lin repr, AnyNat vid, Intersect h h ~ h) =>
--      repr vid 'False h h ((One :<+>: One) :-<>: ((a :<&>: a) :-<>: a))

--bad :: Defn False ((a :-<>: (a :-<>: b)) :-<>: (a :-<>: b))
-- bad = defn $ llam $ \f -> llam $ \x -> f <^> x <^> x

--good :: Defn False ((a :-<>: (a :-<>: b)) :-<>: (a -> b))
good = defn $ llam $ \f -> lam $ \x -> f <^> x <^> x

--goodBang :: Defn False (a :-<>: (a :-<>: b)) :-<>: (Bang a :-<>: b))
goodBang = defn $ llam $ \f -> llam $ \y -> (llam $ \x -> letBang x (\x -> f <^> x <^> x)) <^> y

w1 = defn $ llam $ \x -> llam $ \y -> llam $ \z -> 
            top <*> ((x <*> y) <&> (y <*> x))

w2 = defn $ llam $ \x -> llam $ \y -> llam $ \z -> 
            top <*> ((x <*> y) <&> (top <*> x))

-- fails since z is not used in first conjunct.
--w3 = defn $ llam $ \x -> llam $ \y -> llam $ \z -> 
--            (x <*> y) <&> (top <*> x)

main = putStrLn "ok"
```
I did not need to add a class and instances to let the type checker solve `Intersection` and `:<:` for each possible instance of `h` since an uninstantiated context `h` will only be intersected and subset-ed with itself.

Note, once again, that the Haskell type checker is enforcing linearity (including coordination of resources among additive subderivations) as there is no instance of `Lin` yet defined.

Also as before, it is trivial to add a concrete instance to allow evaluation.

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

-- module Linear where

{---------------------------------------------------------------------

Type level machinery

---------------------------------------------------------------------}

--
-- Type level Or
--
type family Or (x::Bool) (y::Bool) :: Bool
type instance Or False False = False
type instance Or True False = True
type instance Or False True = True
type instance Or True True = True

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

--
-- Machinery needed to check/enforce conditions for additive conjunction/disjunction to correctly deal with Top.
--
class VarOk (tf :: Bool) (v :: Maybe Nat)
instance VarOk True (Just v)
instance VarOk True Nothing 
instance VarOk False Nothing

type family (:<:) (x::[Maybe Nat]) (y::[Maybe Nat]) :: Bool 
type instance '[] :<: '[] = True
type instance (Nothing ': xs) :<: (Nothing ': ys) = xs :<: ys
type instance (Nothing ': xs) :<: (Just y ': ys) = xs :<: ys
type instance (Just x ': xs) :<: (Nothing ': ys) =  False
type instance (Just x ': xs) :<: (Just x ': ys) =  xs :<: ys

type family Intersect (xs::[Maybe Nat]) (ys::[Maybe Nat]) :: [Maybe Nat]
type instance Intersect '[] '[] = '[]
type instance Intersect (Nothing ': xs) (Nothing ': ys) = (Nothing ': Intersect xs ys)
type instance Intersect (Nothing ': xs) (Just y ': ys) = (Nothing ': Intersect xs ys)
type instance Intersect (Just x ': xs) (Nothing ': ys) = (Nothing ': Intersect xs ys)
type instance Intersect (Just x ': xs) (Just x ': ys) = (Just x ': Intersect xs ys)

type family When (x::Bool) (y::Bool) :: Bool
type instance When True y = y

type family AdditiveAnd (tf0::Bool) (tf1::Bool) (h0::[Maybe Nat]) (h1::[Maybe Nat]) :: Bool
type instance AdditiveAnd False False h h = False 
type instance AdditiveAnd False True h0 h1 = When (h0 :<: h1) False
type instance AdditiveAnd True False h0 h1 = When (h1 :<: h0) False 
type instance AdditiveAnd True True h0 h1 = True 


{---------------------------------------------------------------------

Final tagless HOAS linear lambda calculus.

---------------------------------------------------------------------}

--
-- Linear types
--
newtype a :-<>: b = Lolli {unLolli :: a -> b}
data a :*: b = Tensor a b
data One = One
type a :&: b = (a, b)
type Top = ()
data a :+: b = Inl a | Inr b
data Zero 
newtype Bang a = Bang {unBang :: a}


type LinVar repr (vid::Nat) a = 
     forall (v::Nat) (i::[Maybe Nat]) (o::[Maybe Nat]) . (Consume vid i o) => 
     repr v False i o a

type RegVar repr a = 
     forall (v::Nat) (i::[Maybe Nat]) . repr v False i i a

class Lin (repr :: Nat -> Bool -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where
    llam :: (VarOk tf var) => 
            (LinVar repr vid a -> repr (S vid) tf (Just vid ': hi) (var ': ho) b) -> 
            repr vid tf hi ho (a :-<>: b)
    (<^>) :: repr vid tf0 hi h (a :-<>: b) -> repr vid tf1 h ho a -> 
             repr vid (Or tf0 tf1) hi ho b

    (!) :: repr vid tf h h a -> repr vid False h h (Bang a)
    letBang :: repr vid tf0 hi h (Bang a) -> 
               (RegVar repr a -> repr vid tf1 h ho b) -> 
               repr vid (Or tf0 tf1) hi ho b

    lam :: (RegVar repr a -> repr vid tf hi ho b) -> repr vid tf hi ho (a -> b)
    (<$>) :: repr vid tf0 hi ho (a -> b) -> repr vid tf1 ho ho a -> repr vid tf0 hi ho b

    one :: repr vid False h h One
    letOne :: repr vid tf0 hi h One -> repr vid tf1 h ho a -> 
              repr vid (Or tf0 tf1) hi ho a

    (<*>) :: repr vid tf0 hi h a -> repr vid tf1 h ho b -> 
             repr vid (Or tf0 tf1) hi ho (a :*: b)
    letStar :: (VarOk tf1 var0, VarOk tf1 var1) => 
               repr vid tf0 hi h (a :*: b) -> 
               (LinVar repr vid a -> 
                  LinVar repr (S vid) b -> 
                    repr (S (S vid)) 
                         tf1 
                         (Just vid ': Just (S vid) ': h) 
                         (var0 ': var1 ': ho) c
               ) -> 
               repr vid (Or tf0 tf1) hi ho c
                    
    top :: repr vid True h h Top

    (<&>) :: repr vid tf0 hi h0 a -> repr vid tf1 hi h1 b -> 
             repr vid (AdditiveAnd tf0 tf1 h0 h1) hi (Intersect h0 h1) (a :&: b)
    pi1 :: repr vid tf hi ho (a :&: b) -> repr vid tf hi ho a 
    pi2 :: repr vid tf hi ho (a :&: b) -> repr vid tf hi ho b 

    inl :: repr vid tf hi ho a -> repr vid tf hi ho (a :+: b)
    inr :: repr vid tf hi ho b -> repr vid tf hi ho (a :+: b)
    letPlus :: (VarOk tf1 var1, VarOk tf2 var2) => 
               repr vid tf0 hi h (a :+: b) ->
               (LinVar repr vid a -> repr (S vid) tf1 (Just vid ': h) (var1 ': ho1) c) ->
               (LinVar repr vid b -> repr (S vid) tf2 (Just vid ': h) (var2 ': ho2) c) ->
               repr vid (Or tf0 (AdditiveAnd tf1 tf2 ho1 ho2)) hi (Intersect ho1 ho2) c

    abort :: repr vid tf hi ho Zero -> repr vid True hi ho a


type Defn tf a = forall repr vid h . 
                 (Lin repr, Intersect h h ~ h, (h :<: h) ~ True) => 
                 repr vid tf h h a
defn :: Defn tf a -> Defn tf a
defn x = x

-- show Evaluator for full LLC.
newtype R (vid::Nat) (tf::Bool) (hi::[Maybe Nat]) (ho::[Maybe Nat]) a = R {unR :: a}

instance Lin (R :: Nat -> Bool -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where
    llam f = R $ Lolli $ \x -> unR (f (R x))
    f <^> x = R $ unLolli (unR f) (unR x)

    (!) = R . Bang . unR
    letBang x f = R $ unR $ f' (unR x)
      where f' (Bang x) = f (R x)

    lam f = R $ \x -> unR (f (R x))
    f <$> x = R $ unR f (unR x)

    x <*> y = R $ Tensor (unR x) (unR y)
    letStar xy f = R $ unR $ f' (unR xy)
      where f' (Tensor x y) = f (R x) (R y)

    one = R One
    letOne x y = R $ unR $ (\One -> y) $ unR x

    top = R ()

    x <&> y = R $ (unR x, unR y)
    pi1 = R . fst . unR
    pi2 = R . snd . unR

    inl = R . Inl . unR
    inr = R . Inr . unR
    letPlus xy fInl fInr = case unR xy of
                             Inl x -> R $ unR $ fInl (R x)
                             Inr y -> R $ unR $ fInr (R y)

    abort x = R $ error "abort"


eval :: R Z tf '[] '[] a -> a
eval = unR

type MyBool = One :+: One

tt = inl one :: Defn False MyBool
ff = inr one :: Defn False MyBool

cond :: Defn False (MyBool :-<>: ((a :&: a) :-<>: a))
cond = llam $ \b -> llam $ \tf -> letPlus b (\x -> letOne x (pi1 tf)) (\x -> letOne x (pi2 tf))
cond1 = defn $ llam $ \b -> llam $ \tf -> letPlus b (\x -> letOne x (pi1 tf)) (\x -> letOne x (pi2 tf))


--bad :: Defn False ((a :-<>: (a :-<>: b)) :-<>: (a :-<>: b))
-- bad = defn $ llam $ \f -> llam $ \x -> f <^> x <^> x

--good :: Defn False ((a :-<>: (a :-<>: b)) :-<>: (a -> b))
good = defn $ llam $ \f -> lam $ \x -> f <^> x <^> x

main = do
  putStrLn $ (eval $ cond <^> tt <^> (lam (\x -> x) <&> 
                                      abort (undefined :: Defn True Zero))
             ) "I was applied to a function."
  putStrLn "ok"
```
 
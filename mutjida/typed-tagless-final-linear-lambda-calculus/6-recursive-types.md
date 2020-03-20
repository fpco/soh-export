# 6 Recursive Types

## Adding Recursive Types

Up to this point, the language is neat-- at least there are some interesting types to play with-- but it is difficult to use for computation since there is no way to define recursive data types. Although recursive types are relatively straightforward, it is an interesting exercise to add them to a typed tagless-final interpreter; it also results in a language rich enough to write toy programs. This section is mostly an adaption of section 6.4 of Frank Pfenning's [notes](http://www.cs.cmu.edu/~fp/courses/15816-f01/handouts/linfp.pdf).

The goal is to be able to define, and use, types such as
```
MyNat = One :+: MyNat

MyList a = One :+: (a :*: MyList a)
```
but even Haskell can't deal with such unrestricted recursion at the type level. Instead, I have to use an explicit type-level fixpoint, and include terms for manipulating such types:
```haskell
-- show Type-level fixpoint
newtype Mu a = Mu {unMu :: a (Mu a)}
-- show Terms for manipulating Mu types
class LinRec (repr :: Nat -> Bool -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where

    wrap :: repr vid tf i o (a (Mu a)) -> repr vid tf i o (Mu a)

    unwrap :: repr vid tf i o (Mu a) -> repr vid tf i o (a (Mu a))
```
I have introduced a new class, `LinRec`, in order to show off the extensibility of tagless-final encodings.

I can then represent my recursive type via `Mu`
```haskell
type MyNat = Mu ((:+:) One)

zero :: DefnRec False MyNat
zero = wrap (inl one)

succ :: DefnRec False (MyNat :-<>: MyNat)
succ = llam $ \x -> wrap (inr x)
```
The fact that GHC accepts the `MyNat` declaration is a bit of an anomaly since Haskell disallows partially applied type synonyms; so a similar trick can't be employed for `MyList a`.

In order to implement `MyList a`, I am forced to use a newtype to untie the knot:
```haskell
newtype MyListF a lst = MLF {unMLF :: One :+: (a :*: lst)}

type MyList a = Mu (MyListF a)
```
However, this is problematic since my language constructs, `wrap` and `unwrap`, only have one wrapper/unwrapper (`Mu`/`unMu`), but I now need to stick another wrapper/unwrapper inside the `repr` type, i.e. I need to use `MLF` to get from `One :+: (a :*: lst)` to `MyListF a lst` before I can use `Mu` to get `MyList a`. I could reflect `MLF` and `unMLF` into the object language:
```haskell
class LinRec (repr :: Nat -> Bool -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where
    wrap :: repr vid tf i o (a (Mu a)) -> repr vid tf i o (Mu a)
    unwrap :: repr vid tf i o (Mu a) -> repr vid tf i o (a (Mu a))
    
    wrapMLF :: repr vid tf i o (One :+: (a :*: lst)) -> repr vid tf i o (MyListF a lst)
    unwrapMLF :: repr vid tf i o (MyListF a lst) -> repr vid tf i o (One :+: (a :*: lst))
    
nil :: DefnRec False (MyList a)
nil = wrap $ wrapMLF $ inl one

cons :: DefnRec False (a :-<>: (MyList a :-<>: MyList a))
cons = llam (\x -> llam (\xs -> wrap $ wrapMLF $ inr $ x <*> xs))
```
But this is a poor solution as it requires adding an extra wrap/unwrap for (almost) every recursive type I wish to define; which means I have effectively lost my general recursive type. 
Though I can't get around the need for newtypes to declare my recursive types, I can get by with one wrap (unwrap) by generalizing the type to take the extra wrapper/unwrapper.
```haskell
wrap :: (b -> a (Mu a)) -> repr vid tf i o b -> repr vid tf i o (Mu a)
unwrap :: (a (Mu a) -> b) -> repr vid tf i o (Mu a) -> repr vid tf i o b

nil = wrap MLF $ inl one
```

Finally, I will add a term-level fixpoint construction so that I can write recursive functions to compute with recursive types; this is not really necessary as I could just use Haskell's own built-in term level fixpoints, but I will include it for completeness.
```haskell
class LinRec (repr :: Nat -> Bool -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where
    wrap :: (b -> a (Mu a)) -> repr vid tf i o b -> repr vid tf i o (Mu a)

    unwrap :: (a (Mu a) -> b) -> repr vid tf i o (Mu a) -> repr vid tf i o b

    fix :: ((forall vid h . repr vid False h h a) -> repr vid tf h h a) -> repr vid tf h h a 
```
Note that recursive functions cannot depend upon linear variables.

As with all the constructs in this interpreter, a concrete instance for evaluation of the terms is trivial:
```haskell
instance LinRec (R :: Nat -> Bool -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where
    wrap f = R . Mu . f . unR 
    unwrap f = R . f . unMu . unR 
    fix f = f (R $ unR $ fix f)
```

Also, as before, I will add a type synonym, `DefnRec`, to give the type checker some hints to allow better type inference. `DefnRec` will be identical to the previous `Defn` except that it will require an additional `LinRec` constraint.
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

import Prelude hiding(succ)

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


newtype Mu a = Mu {unMu :: a (Mu a)}

class LinRec (repr :: Nat -> Bool -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where
    wrap :: (b -> a (Mu a)) -> repr vid tf i o b -> repr vid tf i o (Mu a)
    unwrap :: (a (Mu a) -> b) -> repr vid tf i o (Mu a) -> repr vid tf i o b
    fix :: ((forall vid h . repr vid False h h a) -> repr vid tf h h a) -> repr vid tf h h a 

instance LinRec (R :: Nat -> Bool -> [Maybe Nat] -> [Maybe Nat] -> * -> *) where
    wrap f = R . Mu . f . unR 
    unwrap f = R . f . unMu . unR 
    fix f = f (R $ unR $ fix f)

-- show Hints for the type checker.
type DefnRec tf a = 
    forall repr vid h . 
    (Lin repr, LinRec repr, Intersect h h ~ h, (h :<: h) ~ True) => 
    repr vid tf h h a
defnRec :: DefnRec tf a -> DefnRec tf a
defnRec x = x


-- show Nat examples
type MyNat = Mu ((:+:) One)

instance Show MyNat where
    show (Mu (Inl _)) = "z"
    show (Mu (Inr n)) = "s "++show n

zero :: DefnRec False MyNat
zero = wrap id (inl one)

succ :: DefnRec False (MyNat :-<>: MyNat)
succ = llam $ \x -> wrap id (inr x)

nat1 = succ <^> zero
nat2 = succ <^> nat1
nat3 = succ <^> nat2
nat4 = succ <^> nat3

plus :: DefnRec False (MyNat :-<>: (MyNat :-<>: MyNat))
plus = fix $ \p -> llam $ \x -> llam $ \y -> 
       letPlus (unwrap id x) 
               (\x -> letOne x y) 
               (\x -> succ <^> (p <^> x <^> y))

mult :: DefnRec False (MyNat :-<>: (MyNat -> MyNat))
mult = fix $ \m -> llam $ \x -> lam $ \y -> 
       letPlus (unwrap id x) 
               (\x -> letOne x zero) 
               (\x -> plus <^> (m <^> x <$> y) <^> y)


-- show List examples
newtype MyListF a lst = MLF {unMLF :: One :+: (a :*: lst)}
type MyList a = Mu (MyListF a)

instance Show a => Show (MyList a) where
    show (Mu (MLF (Inl _))) = "nil"
    show (Mu (MLF (Inr (Tensor x xs)))) = show x ++ ":" ++ show xs

nil :: DefnRec False (MyList a)
nil = wrap MLF (inl one)

cons :: DefnRec False (a :-<>: (MyList a :-<>: MyList a))
cons = llam $ \x -> llam $ \xs -> wrap MLF (inr (x <*> xs))

rev :: DefnRec False (MyList a :-<>: (MyList a :-<>: MyList a))
rev = fix $ \r -> llam $ \l -> llam $ \k -> 
      letPlus (unwrap unMLF l) 
              (\x -> letOne x k) 
              (\xs -> letStar xs (\x xs -> r <^> xs <^> (cons <^> x <^> k)))

-- Example using Haskell's recursion.
rev' :: DefnRec False (MyList a :-<>: (MyList a :-<>: MyList a))
rev' = llam $ \l -> llam $ \k -> 
       letPlus (unwrap unMLF l) 
              (\x -> letOne x k) 
              (\xs -> letStar xs (\x xs -> rev' <^> xs <^> (cons <^> x <^> k)))

l0 :: DefnRec False (MyList MyNat)
l0 = cons <^> zero <^> (cons <^> nat1 <^> (cons <^> nat2 <^> nil))


main = do
  putStrLn $ show $ eval $ mult <^> nat2 <$> (plus <^> nat3 <^> nat4)
  putStrLn $ show $ eval $ rev <^> l0 <^> nil
  putStrLn $ show $ eval $ rev' <^> (cons <^> nat3 <^> l0) <^> nil
  putStrLn "ok"
```

# 1 deBruijn Typed Tagless-Final Interpreter

## First-order, deBruijn Index Based Interpreter

This section is my own presentation, and explanation, of [Oleg's LLC
interpreter](http://okmij.org/ftp/tagless-final/course/LinearLC.hs).
First, I will change the presentation of the language to use deBruijn indices:
```haskell
-- show Grammar for LLC with deBruijn Indices
m ::= z | s m | llam m | m ^ m
```

and slightly modify the algorithmic typing rules above to use an
ordered list for the linear context (actually an ordered list of `Maybe`
types, where `Just` is implicit and `_` is `Nothing`), instead of a
set:

```
------------------------ (lvar-z)     
A, D \ _, D  |-  z :: A            


Di \ Do  |- m :: A
--------------------------- (lvar-s)
B, Di \ B, Do  |-  s m :: A


A, Di \ _, Do  |-  m :: B  
----------------------------- (-<>I)
Di \ Do  |-  lam m :: A -<> B


Di \ D  |-  m :: A -<> B     D \ Do  |-  n :: A
----------------------------------------------- (-<>E)
            Di \ Do  |-  m ^ n :: B
```
Note that in rule `lvar-s`, `B` can be either `_` or some type `A`.

I will use well-typed haskell terms:
```haskell
m :: repr hi ho a
```
to represent the judgement:
```
Di \ Do  |-  m :: A
```

The following is a direct transcription of the above typing rules into Haskell types.

```active haskell
-- /show
{-# LANGUAGE 
    NoMonomorphismRestriction,
    FlexibleInstances,
    MultiParamTypeClasses,
    TypeOperators
 #-}    
 -- show
-- type level Maybe for representing elements of the linear context
newtype F a = F a
data U = Used

class Linear repr where
    z     :: repr (F a, h) (U, h) a
    s     :: repr hi ho a -> repr (any, hi) (any, ho) a
    llam  :: repr (F a, hi) (U, ho) b  -> repr hi ho (a -> b)
    (<^>) :: repr hi h (a -> b) -> repr h ho a -> repr hi ho b

t0 = llam $ llam $ s z <^> z 
-- inferred type of t0 :: Linear repr => repr h h ((a -> b) -> a -> b)

-- following won't type check
-- t1 = llam $ llam $ s z <^> z <^> z 

main = putStrLn "ok"
```
Note that the return type of `llam` contains a Haskell function. Although the syntax uses deBruijn notation, linear functions are being represented by Haskell functions; so the `llam` method of any instance of `Linear` will have to create a Haskell function. 
Another interesting observation about the code above is that _Haskell's type checker is enforcing linearity_ for us; as of yet, there is no instance for the class `Linear`. 

In order to have an instance for `Linear` which lets us
evaluate LLC expressions, we need a suitable type, call it
`R`, to concretely represent our LLC terms. Since the language uses deBruijn indices, we will need to have an
explicit representation of the linear context around at runtime.

```active haskell
-- /show
{-# LANGUAGE 
    NoMonomorphismRestriction,
    FlexibleInstances,
    MultiParamTypeClasses,
    TypeOperators 
 #-}

-- type level Maybe for representing elements of the linear context
newtype F a = F a
data U = Used

class Linear repr where
    z   :: repr (F a, h) (U, h) a
    s   :: repr hi ho a -> repr (x, hi) (x, ho) a
    llam :: repr (F a, hi) (U, ho) b  -> repr hi ho (a -> b)
    (<^>) :: repr hi h (a -> b) -> repr h ho a -> repr hi ho b

-- show

-- Typed and tagless evaluator
--   object term ==> metalanguage value

newtype R hi ho a = R {unR :: hi -> (a, ho)}

instance Linear R where
    z     = R $ \(F x, h) -> (x, (Used, h))

    s v   = R $ \(any, hi) -> 
                  let (x, ho) = unR v hi
                  in (x, (any, ho))

    -- !!! Following does not work since vo is not in scope !!!
    -- 
    llam e = R $ \hi -> (f hi, vo)
     where f hi x = let (v, vo) = unR e (F x,hi)
                    in v

    m1 <^> m2 = R $ \hi -> let (v1, h)  = unR m1 hi
                               (v2, ho) = unR m2 h
                           in (v1 v2, ho)    
```

As noted above, this code doesn't type check since the output linear
context needed by `llam` is unknown; it won't be computed until the
function is actually applied. However, there is a
way to get around this problem; the input context and the type of the
output context (which is statically determined) are enough to generate
the actual output context. A type class can
be used to generate an output context from an input context and the known output context type. 

```haskell
class HiHo hi ho where
    hiho :: hi -> ho

instance HiHo () () where
    hiho = id

instance HiHo hi ho => HiHo (F a , hi) (F a , ho) where
    hiho (x, hi) = (x, hiho hi)

instance HiHo hi ho => HiHo (U , hi) (U , ho) where
    hiho (x, hi) = (x, hiho hi)

instance HiHo hi ho => HiHo (F a, hi) (U, ho) where
    hiho (x, hi) = (Used, hiho hi)
```

In order to let `llam` use `hiho`, its type-level linear contexts must be exposed. However, I don't want to add these to `Linear` since neither `z` nor `<^>` mention exactly two type-level linear contexts; therefore, I will add another class, `LinearLam`,
just for `llam`.

```haskell
class Linear repr where
    z   :: repr (F a , h) (U , h) a
    s   :: repr hi ho a -> repr (x , hi) (x , ho) a
    (<^>) :: repr hi h (a -> b) -> repr h ho a -> repr hi ho b

class LinearLam repr hi ho where
    llam :: repr (F a , hi) (U , ho) b  -> repr hi ho (a -> b)
```

I can now write correct instances of `Linear` and `LinearLam` for `R` as well as an evaluator for terms of type `R`:

```active haskell
-- /show
{-# LANGUAGE 
    NoMonomorphismRestriction,
    FlexibleInstances,
    MultiParamTypeClasses,
    TypeOperators 
 #-}

-- type level Maybe for representing elements of the linear context
newtype F a = F a
data U = Used


class HiHo hi ho where
    hiho :: hi -> ho
instance HiHo () () where
    hiho = id
instance HiHo hi ho => HiHo (F a , hi) (F a , ho) where
    hiho (x, hi) = (x, hiho hi)
instance HiHo hi ho => HiHo (U , hi) (U , ho) where
    hiho (x, hi) = (x, hiho hi)
instance HiHo hi ho => HiHo (F a, hi) (U, ho) where
    hiho (x, hi) = (Used, hiho hi)


class Linear repr where
    z   :: repr (F a , h) (U , h) a
    s   :: repr hi ho a -> repr (x , hi) (x , ho) a
    (<^>) :: repr hi h (a -> b) -> repr h ho a -> repr hi ho b

class LinearLam repr hi ho where
    llam :: repr (F a , hi) (U , ho) b  -> repr hi ho (a -> b)


-- show
-- Typed and tagless evaluator
--   object term ==> metalanguage value

newtype R hi ho a = R {unR :: hi -> (a, ho)}

instance Linear R where
    z     = R $ \(F x, h) -> (x, (Used, h))

    s v   = R $ \(any, hi) -> 
                  let (x, ho) = unR v hi
                  in (x, (any, ho))

    m1 <^> m2 = R $ \hi -> let (v1, h)  = unR m1 hi
                               (v2, ho) = unR m2 h
                           in (v1 v2, ho)    

instance HiHo hi ho => LinearLam R hi ho where
    llam e = R $ \hi -> (f hi, hiho hi)
       where f hi x = let (v, _) = unR e (F x, hi)
                      in v

-- The implementation of lam shows that the value of lam, which is
-- f hi, is the closure of the (input) environment in which
-- lam was produced.

eval :: R () () a -> a
eval e = fst $ unR e ()                 -- Eval in the empty environment

t = llam $ llam $ s z <^> z

main = do 
  putStrLn $ eval (t <^> llam z) "I was passed to a real function."
  putStrLn "ok"
```



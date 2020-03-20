# Algebras and Coalgebras

# Algebra/Coalgebra

## Algebra

If you have an arbitrary covariant `Functor`, `T a` you can consider the class of functions

    alg :: T X -> X

for some specific type `X` called the *carrier*. The choice of carrier and the function `alg` together are called an `algebra` and are a recipe for creating `X`es. For instance, the functor

    data T x = T (Either () x)

has an algebra

    alg :: T [()] -> [()]
    alg (T (Left ()))  = []
    alg (T (Right ls)) = () : ls

which demonstrates how to recursively create a list of units. Or it has the algebra

    alg :: T Int -> Int
    alg (T (Left ())) = 0
    alg (T (Right n)) = succ n

which demonstrates how to create natural numbers, or lenths of the lists before. It also has the algebra

    data Fix f = In { out :: f (Fix f) }

    alg :: T (Fix T) -> Fix T
    alg = In    

which shows that we can construct fixed points of `T` from the functor `T` itself in a very compact way.

The upshot is that a functor like `T` provides a signature of functions in a universal algebra to construct elements of the carrier type.

## Coalgebra

If you have an arbitrary covariant `Functor`, `T a` you can consider the class of functions

    coalg :: X -> T X

for some specific type `X` called the *carrier*. The choice of carrier and the function `coalg` together are called a `coalgebra` and are a recipe for observing `X`es. For instance, the functor

    data T x = T (Either () x)

has a coalgebra

    coalg :: [()] -> T [()]
    coalg []     = T (Left ())
    coalg (x:xs) = T (Right xs)

which demonstrates how to check a list of units to see if it is empty. Or it has the coalgebra

    coalg :: Int -> T Int
    coalg 0 = T (Left ())
    coalg n = T (Right (pred n))

which demonstrates how to count down the natural numbers, or the lengths of the lists before. It also has the coalgebra

    data Fix f = In { out :: f (Fix f) }

    coalg :: Fix T -> T (Fix T)
    coalg = out

which shows that we can observe fixed points of `T` via the functor `T` itself in a very compact way.

# Construction/Observation

## Functors in normal forms

Clearly, these concepts are duals and both arise from the nature of `Functor`s of some kind.

We can write functors in an algebraic form inherited from the algebraic form of ADTs. For instance, `T` from above

    data T x = T (Either () x)

is compactly written as

    T X = 1 + X

and 

    data T x = T ((), x)

is compactly written as

    T X = 1 * X

since `Either` is a canonical sum type, `(,)` a canonical product type, and `()` a unit. If we are dealing with nice, distributive functors where we have

    A + (B * C) <-> (A + B) * (A + C)

then we can always write them in disjunctive normal form

    T X = T1 X + T2 X + T3 X + T4 X + ...

where the `Tn` subfunctors are each sum-free or conjunctive normal form

    T X = T1 X * T2 X * T3 X * T4 X * ...

where the subfunctors are each product-free. These simplified forms tell us a lot about what `alg` or `coalg` must be doing. 

## Construction/Observation

Consider `alg`. For disjunctive normal form its first step must be to branch on one of many sub `alg`ebras corresponding to the particular branch in the sum.

    -- pseudo-Haskell
    alg tx = case tx of
      t1@T1{} -> alg_1 t1
      t2@T2{} -> alg_2 t2
      ...

while ach of the `alg_n`s are no longer able to be conditional of the structure of their arguments, they must create a new carrier object with only the information from the chosen branch.

    alg_1 (T1 a b c d) = ...

If we give them names then we might think of them as the various recursive construction functions for the carrier `T`. Specifically, we can think of the list constructors `nil :: [a]` and `cons :: a -> [a] -> [a]` as two subalgebras on the disjunctive normal form functor

    data T a x = 1 + (a, x)

    alg :: T a [a] -> [a]
    alg ()      = nil
    alg (a, as) = cons a as

Consider `coalg`. For conjunctive normal form its first step must be to distribute its work to many sub `coalg`ebras

    -- psuedo-Haskell
    coalg t = (coalg_1 t, coalg_2 t, coalg_3 t)

each of which returning a component of the an observation from the carrier, though they may choose to return useless information like `()`. Specifically, we can think of the stream observers `head :: Stream a -> a` and `tail :: Stream a -> Stream ` as two subcoalgebras on the conjuctive normal form functor

    {- data Stream a = Cons a (Stream a) -}

    data T a x = (a, x)

    coalg :: Stream a -> T a (Stream a)
    coalg s = (head s, tail s)
# Intro

# From Scratch

This is an account of my journey through the conception and implementation of a framework for Generated Algebras in Haskell.

Furthermore, this is a *live* account, in the sense that mistakes will be recorded (if any are made) and then corrected. 

Finally, this will be concise, and hopefully showcase the power of haskell and clear semantics.

Here we go.

# Generated Algebras

A few weeks ago, somebody asked me how to implement generated algebras in
haskell.

I remember roughly what a Generated Algebra is, but to clarify, let us look at wikipedia:

## Wikipedia

Finitely generated algebra
From Wikipedia, the free encyclopedia

In mathematics, a finitely generated algebra is an associative algebra A over a field K where there exists a finite set of elements a1,…,an of A such that every element of A can be expressed as a polynomial in a1,…,an, with coefficients in K. If it is necessary to emphasize the field K then the algebra is said to be finitely generated over K . Algebras that are not finitely generated are called infinitely generated. Finitely generated commutative algebras are basic objects of consideration in modern algebraic geometry, where they correspond to affine algebraic varieties; for this reason, these algebras are also referred to as (commutative) affine algebras.


...

Because we don't care about the *finite* part, we can ignore *polynomials*, and just think of it like this:


kI are in K
aI are in A
``` haskell
-- show elements of the generated algebra look like:

k1 * a1 + k2 * a2 + ... + kI * aI + ...
-- /show

-- show we can represent this as a list of pairs:

[(k1,a1),(k2,a2),...,(kI,aI),...]
-- /show
```
I must say that the notation used by the mathematicians is a little bit confusing; even though they *write* an element of the generated algebra as a sum of products, they don't want anybody to actually try to compute these sums and products.

Here is a way to understand this:

``` active haskell
-- we use different symbols because * and + are already taken.

k `mult` a = [(k,a)]
term1 `plus` term2 = term1 ++ term2

main = do print $ ("k1" `mult` "a1") `plus` ("k2" `mult` "a2")
```
So basically, there is no difference between them writing `+` and `*` and us using a list of pairs.

Furthermore, if we knew in advance in what order the `a`s would appear, we could represent an elements of the generated algebra simply by a list of ordered `k`s, and by convention know which `a` goes where.

We, however, will not go down that road, because I expect that many `k`s will be zeroes most of the time.

Therefore, we can restrict ourselves to a list of pairs where the `k` of each pair is non-zero. This is called a *sparse representation*. To make sure that no problems occur, we can write a function that removes any term with a `k = 0`:

``` haskell

--suppose we have this function
isZero :: K -> Bool

-- then:
clean :: [(K,A)] -> [(K,A)]
clean list = filter (not . isZero . fst) list
```

# The task
Basically, we must implement a multiplication and an addition on our Generated Algebra.

We can specify this requirement in Haskell like this:

``` haskell

class SimpleAlgebra s where
    add :: s -> s -> s
    mult :: s -> s -> s
    
class Mult a where
    mMult :: a -> a -> a
    
class Field k where
    isZero :: k -> Bool
    fAdd :: k -> k -> k
    fMult :: k -> k -> k

instance (Mult a, Field k) => SimpleAlgebra [(k,a)] where
 ...

```

In the next part, we will clean up the requirements, and begin to implement.
# Bound

[`bound`](http://hackage.haskell.org/package/bound) provides a powerful but simple toolbox for dealing with capture-avoiding substitution in your code. It lets you use classes you already know: `Monad`, `Foldable` and `Traversable` to manipulate your syntax tree, and it factors out issues of name capture into a reusable monad transformer named `Scope`.

What I want to do today is help motivate the design of the [`bound`](http://hackage.haskell.org/package/bound) library by talking a bit about alternatives. This may serve, with a few digressions, as a bit of a crash course on capture-avoiding substitution, for those who don't deal with this issue on a day-to-day basis. This presentation is based on a talk I gave on [`bound`](http://hackage.haskell.org/package/bound), but it has been upgraded to work with the School of Haskell tools for easier online consumption. In particular, I've tried to make the code fragments executable, and greatly expand on the content to cover things in greater depth. The [original slides](http://www.slideshare.net/ekmett/bound-making-de-bruijn-succ-less) are also available, but were designed with me speaking over them in mind.

What is in a Name?
==================

We use names in lots of contexts, but any program that deals with names has to deal with a number of issues such as capture avoidance and deciding alpha equivalance... and others that will come up as we go.

If you go to write a syntax tree, the dumbest thing that can possibly work would be something like:

```active haskell
type Name = String

data Exp 
  = Var Name 
  | App Exp Exp
  | Lam Name Exp
  deriving (Eq,Show,Read)

main = do
  print $ Var "x"
  print $ Lam "x" (Var "x")
  print $ Lam "x" (Lam "y" (Var "x"))
```

Two problems we want to address in almost any syntax tree are:

1.) Capture Avoidance

  Blindly substituting 
   
```haskell
Lam "x" (Var "y") 
```
  
  into
  
```haskell
Lam "y" (Var "z")
```
  
  for `z` would yield 
  
```haskell
Lam "y" (Lam "x" (Var "y"))
```
  
  which now causes the formerly free variable `y` to now reference the `y` bound by the outer lambda.

2.) Alpha Equivalence

```haskell
Lam "x" (Var "x")
```

  and

```haskell
Lam "y" (Var "y")
```

  both mean the same thing and it'd be nice to be able to check this easily, make them hash the 
  same for common sub-expression elimination, etc.

There is a cottage industry of solutions to the naming problem:

e.g.

* Naïve substitution
* [The Barendregt convention](http://www.amazon.com/Calculus-Semantics-Studies-Foundations-Mathematics/dp/0444875085)
* [HOAS](http://en.wikipedia.org/wiki/Higher-order_abstract_syntax)
* Weak HOAS/[PHOAS](http://adam.chlipala.net/papers/PhoasICFP08/)
* ["I am not a Number: I am a Free Variable!"](http://www.cs.ru.nl/~james/RESEARCH/haskell2004.pdf)
* [The Locally Nameless Representation](http://www.chargueraud.org/research/2009/ln/main.pdf)
* [Unbound](http://hackage.haskell.org/package/unbound), which mixes Barendregt with Locally Nameless
* Locally Nameless Syntax with De Bruijn Indices

I'm not looking to address all of these here, just a few, to showcase issues with the different points in the design space, and then to try to offer up a nice point in the design space (`bound`) that combines many of the advantages with few of the disadvantages.

### Naïve Substitution

Pros:

  1.) The syntax trees are pretty and easy to read.

  2.) Easy to use to get started

Cons:

  1.) It is easy even for experts to make mistakes.

  2.) Alpha equivalence checking is tedious

  3.) It is comically slow.

```active haskell
import Data.List (union, span, (\\))

type Name = String

data Exp 
  = Var Name 
  | App Exp Exp
  | Lam Name Exp
  deriving (Eq,Show,Read)

-- show
freeVars :: Exp -> [Name]
freeVars (Var x) = [x]
freeVars (App a b) = freeVars a `union` freeVars b 
freeVars (Lam n x) = freeVars x \\ [n]

allVars :: Exp -> [Name]
allVars (Var x) = [x]
allVars (App a b) = allVars a `union` allVars b
allVars (Lam n x) = allVars x

subst :: Name -> Exp -> Exp -> Exp
subst x s b = sub vs0 b where
  sub _ e@(Var v)
    | v == x = s
    | otherwise = e
  sub vs e@(Lam v e')
    | v == x = e
    | v `elem` fvs = Lam v' (sub (v':vs) e'')
    | otherwise = Lam v (sub vs e') where
    v' = newId vs
    e'' = subst v (Var v') e'
  sub vs (App f a) = sub vs f `App` sub vs a
  fvs = freeVars s
  vs0 = fvs `union` allVars b
  
newId :: [Name] -> Name
newId vs = head (names \\ vs)

names :: [Name]
names = [ [i] | i <- ['a'..'z']] ++ [i : show j | j <- [1..], i <- ['a'..'z'] ]
-- /show

-- show and we can see that this deals with capture avoidance
main = print $ subst "z" (Lam "x" (Var "y")) (Lam "y" (Var "z"))
-- /show
```

This code is adapted from Lennart Augustsson's excellent [λ-calculus cooked four ways](http://www.augustsson.net/Darcs/Lambda/top.pdf), which provides a nice crash course on name binding for evaluation.

[Edited *August 27 2013*: The code fragment above was updated based on a conversation with Marc André Ziegert to deal with an issue where in the <code>v `elem` fvs</code> case under `sub` it wasn't properly updating the set of `allVars b` when it introduced `v'` into scope. I told you naïve substitution was tricky!]

### The Barendregt Convention

This one is a serious contender. It is what GHC uses.

Pros:

  1.) The [Secrets of the Glasgow Haskell Compiler inliner](http://research.microsoft.com/en-us/um/people/simonpj/Papers/inlining/) paper by [Simon and Simon](http://www.youtube.com/watch?v=IDhz_mVcVCQ) describes a technique they call "the Rapier," which can make this really fast. 

Cons:

  1.) Easy even for experts to screw up.

  2.) Alpha equivalence is still tedious.

  3.) Need a globally unique variable supply. e.g. [concurrent-supply](http://hackage.haskell.org/package/concurrent-supply)

  4.) The obvious implementation technique chews through a scarily large number of variable IDs! Without the Rapier there is a lot of "administrative" renaming going on.

### Higher-Order Abstract Syntax (HOAS)

HOAS cheats and borrows substitution from the host language.

```active haskell
-- show
data Exp a
  = Var a
  | Lam (Exp a -> Exp a)
  | App (Exp a) (Exp a)
-- /show
main = putStrLn "It typechecks, but a Show instance here is hard!"
```
Pros:

  1.) It provides ridiculously fast substitution by comparison to other techniques.

  2.) Meijer and Hutton's ["Bananas in Space"](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.64.4921&rep=rep1&type=pdf), Fegaras and Sheard's ["Revisiting Catamorphisms over Datatypes with Embedded Functions (or, Programs from Outer Space)](http://citeseer.ist.psu.edu/viewdoc/summary?doi=10.1.1.36.2763) and Weirich and Washburn's ["Boxes Go Bananas: Encoding Higher-order Abstract Syntax with Parametric Polymorphism"](http://www.seas.upenn.edu/~sweirich/papers/itabox/icfp-published-version.pdf) each describe forms of catamorphism that can be used on these to work under binders. 
  
Cons:

  1.) It doesn't work in theorem provers like Coq or Agda, because `Exp` occurs in both positive and negative position, causing it to fail the positivity check.

  2.) It is quite hard to work under binders. You often have to invert your control flow for passes and think "inside out."
 
  3.) You have to deal with exotic terms that do bad things like inspect the expression they are given, so it is in some sense "too big."

  4.) Alpha equivalence is still tedious.
  
  5.) In ["Rotten Bananas"](http://comonad.com/reader/2008/rotten-bananas/) on [comonad.com](http://comonad.com) I go through the issues with general recursion in this approach.

Variants such as Weak HOAS/[PHOAS](http://adam.chlipala.net/papers/PhoasICFP08/)/[Weirich and Washburn](http://www.seas.upenn.edu/~sweirich/papers/itabox/icfp-published-version.pdf) exist to address some of these issues (e.g. working in Coq/Agda, restoring positivity, ruling out exotic terms) at the expense of other problems (e.g. losing the notion of catamorphism or general recursion).


### De Bruijn Indices

> M'colleague Bob Atkey once memorably described the capacity to put up with De Bruijn indices as a Cylon
> detector, the kind of reverse Turing Test that the humans in Battlestar Galactica invent, the better to
> recognize one another by their common inadequacies. He had a point
>
> *-Conor McBride, "I am not a number, I am a classy hack"*

We can split variables into bound and free:

```active haskell
-- show
{-# LANGUAGE DeriveFunctor, DeriveFoldable, DeriveTraversable #-}
import Data.Foldable
import Data.Traversable

data Exp a 
  = Free a
  | Bound !Int
  | App (Exp a) (Exp a)
  | Lam (Exp a)
  deriving (Eq,Show,Read,Functor,Foldable,Traversable)
-- /show
main = putStrLn "It typechecks, but it is boring."
```

We could define combinators to bind names and instantiate them

```haskell
abstract :: Eq a => a -> Exp a -> Exp a
instantiate :: Exp a -> Exp a -> Exp a
```

but let's adopt Conor McBride's less error prone convention instead:

```active haskell
{-# LANGUAGE DeriveFunctor, DeriveFoldable, DeriveTraversable #-}
import Data.Foldable
import Data.Traversable
-- show
newtype Scope f a = Scope (f a) deriving (Eq,Show,Read,Functor,Foldable,Traversable)
data Exp a 
  = Free a
  | Bound !Int
  | App (Exp a) (Exp a)
  | Lam (Scope Exp a)
-- /show
  deriving (Eq,Show,Read,Functor,Foldable,Traversable)
main = putStrLn "It typechecks, and is even slightly interesting."
```

With Conor's convention, `abstract` and `instantiate` actually have useful types that prevent you from doing bad things. Renaming `Bound` to `B` and `Free` to `F`, then adapting the approach he takes in his excellent functional pearl with James McKinna ["I am not a Number -- I am a Free Variable"](http://www.cs.ru.nl/~james/RESEARCH/haskell2004.pdf) to our running example to try to keep Conor's somewhat absurd sense of humor intact we get:

```active haskell
{-# LANGUAGE DeriveFunctor, DeriveFoldable, DeriveTraversable #-}
import Data.Foldable
import Data.Traversable

newtype Scope f a = Scope (f a) deriving (Eq,Show,Read,Functor,Foldable,Traversable)
data Exp a 
  = F a
  | B !Int
  | App (Exp a) (Exp a)
  | Lam (Scope Exp a)
  deriving (Eq,Show,Read,Functor,Foldable,Traversable)
-- show
abstract :: Eq a => a -> Exp a -> Scope Exp a
abstract me expr = Scope (letmeB 0 expr) where
  letmeB this (F you) | you == me = B this
                      | otherwise = F you
  letmeB this (B that)            = B that
  letmeB this (App fun arg)       = letmeB this fun `App` letmeB this arg
  letmeB this (Lam (Scope body))  = Lam $ Scope $ letmeB (succ this) body

instantiate :: Exp a -> Scope Exp a -> Exp a
instantiate what (Scope body) = what'sB 0 body where
  what'sB this (B that) | this == that = what
                        | otherwise    = B that
  what'sB this (F you)                 = F you
  what'sB this (App fun arg)           = what'sB this fun `App` what'sB this arg
  what'sB this (Lam (Scope body))      = Lam $ Scope $ what'sB (succ this) body
-- /show
main = putStrLn "It typechecks, and is even slightly interesting."
```

We could even make a monad that does capture avoiding substitution for `Exp`, but it is an awkward one-off experience.

Pros:

  1.) `Scope`, `abstract`, and `instantiate` made it harder to screw up walking under binders.

  2.) Alpha equivalence is just `(==)` due to the power of De Bruijn indices.

  3.) We _can_ make a `Monad` for `Exp` that does _capture avoiding_ substitution.

  4.) We can use `Traversable` and `Foldable` to find free variables and close terms.

    Using these we can define combinators like
  
```active haskell
import Data.Foldable as Foldable
import Data.Traversable
  
-- show
closed :: Traversable f => f a -> Maybe (f b)
closed = traverse (const Nothing)

isClosed :: Foldable f => f a -> Bool
isClosed = Foldable.all (const False)
-- /show
    
main = putStrLn "The types check out."
```

    to check for closed terms, and we can use `Exp Void` directly as a closed term.
  
Cons:

  1.) We `succ` a lot in `letmeB` and `what'sB`.

  2.) Illegal terms such as `Lam (Scope (B 2))` still exist.

  3.) We have to define one-off versions of `abstract` and `instantiate` for each expression type we come up with. This means this is a design pattern, not a library.

  4.) The `Monad` for `Exp` is similarly a one-off deal.

### Bird and Paterson, Part 1

We can turn to Bird and Paterson's encoding of De Bruijn indices in terms of polymorphic recursion from the first half of [De Bruijn notation as a nested datatype](http://www.cs.uwyo.edu/~jlc/courses/5000_fall_08/debruijn_as_nested_datatype.pdf).

```active haskell
{-# LANGUAGE DeriveFunctor, DeriveFoldable, DeriveTraversable #-}
import Data.Foldable
import Data.Traversable

-- show
data Exp a
  = Var a
  | App (Exp a) (Exp a)
  | Lam (Exp (Maybe a))
  deriving (Functor, Foldable, Traversable)
  
instance Monad Exp where
  return = Var
  Var a >>= f = f a
  App l r >>= f = App (l >>= f) (r >>= f)
  Lam xs >>= f = undefined -- go for it!
-- /show

main = print "Did you finish (>>=)?"
```

Pros:

  1.) This eliminates the illegal terms from Con #2 above once and for all, and we can write a one-off `Monad` for it.

  2.) We have the `Maybe` to mark our extra variables, so we can't forget to deal with our bound term at any point in our recursion.

Cons:

  1.) We still `succ` a lot, except now we call `succ` `Just`.

  2.) We still need to write one-off `abstract` and `instantiate` combinators.

  3.) The `Monad` for `Exp` is a one-off deal.

  4.) They lean somewhat unnecessarily on `RankNTypes` to manipulate these expressions in terms of their folds, when the `Monad` and other classes can all be defined within the confines of [Haskell 98](http://www.haskell.org/onlinereport/).
    

### Attempt #1

We can modify Bird and Paterson's approach in a few ways, to steal some of the desirable properties from McBride's "I am not a Number; I'm a Free Variable" by noticing that Scope forms a Monad transformer!

```active haskell
{-# LANGUAGE DeriveFunctor, DeriveFoldable, DeriveTraversable #-}
import Control.Monad
import Control.Monad.Trans
import Data.Foldable
import Data.Functor
import Data.Traversable

-- show
newtype Scope f a = Scope { runScope :: f (Maybe a) }
  deriving (Functor, Foldable, Traversable)

instance Monad f => Monad (Scope f) where
  return = Scope . return . Just
  Scope m >>= f = undefined -- go for it!

instance MonadTrans Scope where
  lift = Scope . liftM Just
-- /show

main = print "Scope scopes out, but did you finish (>>=)?"
```

The astute observer will recognize that this form of `Scope` is just [`MaybeT`](http://hackage.haskell.org/packages/archive/transformers/0.3.0.0/doc/html/Control-Monad-Trans-Maybe.html) from [`transformers`](http://hackage.haskell.org/package/transformers).

We can finally even define `abstract` and `instantiate` from Conor's approach once and for all, completely independently of our expression type. Sadly, they find themselves now devoid of humor.

```haskell
abstract :: (Functor f, Eq a) => a -> f a -> Scope f a
abstract x xs = Scope (fmap go xs) where
  go y = y <$ guard (x /= y)
  
instantiate :: Monad f => f a -> Scope f a -> f a
instantiate x (Scope xs) = xs >>= go where
  go Nothing = x
  go (Just y) = return y
```

This is starting to feel like a library, rather than design pattern.

Moreover, with that in hand, we can revisit the definition of `Exp` and define our `Monad` by borrowing from `Scope`'s expression-type-agnostic definition.

```active haskell
{-# LANGUAGE DeriveFunctor, DeriveFoldable, DeriveTraversable #-}
import Control.Monad
import Control.Monad.Trans
import Data.Foldable
import Data.Functor
import Data.Traversable

newtype Scope f a = Scope { runScope :: f (Maybe a) }
  deriving (Functor, Foldable, Traversable)

instance Monad f => Monad (Scope f) where
  return = Scope . return . Just
  Scope m >>= f = Scope $ m >>= maybe (return Nothing) (runScope . f)
 
instance MonadTrans Scope where
  lift = Scope . liftM Just

abstract :: (Functor f, Eq a) => a -> f a -> Scope f a
abstract x xs = Scope (fmap go xs) where
  go y = y <$ guard (x /= y)
  
instantiate :: Monad f => f a -> Scope f a -> f a
instantiate x (Scope xs) = xs >>= go where
  go Nothing  = x
  go (Just y) = return y

-- show
data Exp a
  = Var a
  | App (Exp a) (Exp a)
  | Lam (Scope Exp a)
  deriving (Functor, Foldable, Traversable)
  
instance Monad Exp where
  return = Var
  Var a >>= f    = f a
  App l r >>= f  = App (l >>= f) (r >>= f)
  Lam body >>= f = Lam (body >>= lift . f)
-- /show

main = putStrLn "Now we're getting somewhere!"
```

Pros:

  1.) We were able to factor out `Scope`, `abstract` and `instantiate` into code we can write once and package up.
  
  2.) We've gained the benefits of McBride's conventions, as `abstract` and `instantiate` are really nice to reason about.
  
Cons:

  1.) This is still going to `succ` just as much as our earlier De Bruijn solutions. Using `lift` to embed into `Scope` has to traverse all of the leaves to mangle them with a `Just`.
  
  2.) `Eq` `Ord`, `Show`, and `Read` are now non-trivial to write due to polymorphic recursion.

### Bird and Paterson, Part 2

Bird and Paterson did write the other half of their paper for a reason, though. In the other half they showed we can encode a _generalized_ De Bruijn index notion using a different polymorphic recursion pattern.

```haskell
data Exp a
  = Var a
  | App (Exp a) (Exp a)
  | Lam (Exp (Maybe (Exp a)))
```

The astute observer will now note that this is looking a bit less like `MaybeT`. So what happened?

Normally when you work with De Bruijn indices you `succ` only the variables down at the leaves.

What Bird and Paterson's generalized De Bruijn form allows you to do is `Just` née `succ` whole trees at a time!

If we adopt my `MonadTrans` variant of McBride's `Scope` as before, and generalize Bird and Paterson's approach, sticking to Haskell 98. We get very close to the final solution actually encoded in `bound`.

```active haskell
{-# LANGUAGE DeriveFunctor, DeriveFoldable, DeriveTraversable #-}
import Control.Monad
import Control.Monad.Trans
import Data.Foldable
import Data.Functor
import Data.Maybe
import Data.Traversable

-- show
newtype Scope f a = Scope { runScope :: f (Maybe (f a)) }
  deriving (Functor, Foldable, Traversable)
  
instance Monad f => Monad (Scope f) where
  return = Scope . return . Just . return
  Scope e >>= f = Scope $ e >>= \v -> case v of
    Nothing -> return Nothing
    Just ea -> ea >>= runScope . f
    
instance MonadTrans Scope where
  lift = Scope . return . Just
  
abstract :: (Monad f, Eq a) => a -> f a -> Scope f a
abstract a e = Scope (liftM k e) where
  k b = return b <$ guard (a /= b)

instantiate :: Monad f => f a -> Scope f a -> f a
instantiate k (Scope e) = e >>= fromMaybe k
-- /show
main = putStrLn "Almost there!"
```

The key difference realized by Bird and Paterson's design is that now, lifting an expression that does not have our bound term in it into our new `Scope`, no longer requires touching every leaf in the expression. In fact it is `O(1)`!

`Scope` is still a `Monad` transformer, but it is a new one!

This is a salvageable representation, but I want to take one more step before we turn it into a library, before we talk about Pros and Cons.

### Bound: Generalized Generalized De Bruijn

Often we need to deal with simultaneous substitution of several variables. e.g. all of the variables bound by a pattern, all of the variables bound by a lambda, many variables bound by recursive let

`De Bruijn` even as generalized above still only lets me `abstract` or `instantiate` a single variable at a time.

What I want relates to `EitherT` as the above monad relates to `MaybeT`.

Finally, I'm going to alpha rename `Either`, because really when we _do_ finally get to show you these syntax trees, they are going to be bad enough without meaningless `Left` and `Right` names cluttering up my tree. Let's use `B` for bound (pronounced `zero`) and `F` for free (pronounced `succ`) in homage to Conor's conventions above.

```active haskell
{-# LANGUAGE DeriveFunctor, DeriveFoldable, DeriveTraversable #-}
import Control.Monad
import Control.Monad.Trans
import Data.Foldable
import Data.Functor
import Data.Maybe
import Data.Traversable

-- show Var
data Var b a 
  = B b 
  | F a 
  deriving (Eq,Ord,Show,Read,Functor,Foldable,Traversable)

instance Monad (Var b) where
  return = F
  F a >>= f = f a
  B b >>= _ = B b
-- /show

-- show Scope
newtype Scope b f a = Scope { runScope :: f (Var b (f a)) }
  deriving (Functor,Foldable,Traversable)
  
instance Monad f => Monad (Scope b f) where
  return = Scope . return . F . return
  Scope e >>= f = Scope $ e >>= \v -> case v of
    B b  -> return (B b)
    F ea -> ea >>= runScope . f
    
instance MonadTrans (Scope b) where
  lift = Scope . return . F
-- /show

-- show Abstraction and Instantiation
abstract :: Monad f => (a -> Maybe b) -> f a -> Scope b f a
abstract f e = Scope (liftM k e) where
  k y = case f y of
    Just z  -> B z
    Nothing -> F (return y)

instantiate :: Monad f => (b -> f a) -> Scope b f a -> f a
instantiate k e = runScope e >>= \v -> case v of
  B b -> k b
  F a -> a
-- /show

main = putStrLn "It typechecks, therefore it must be correct!"
```

From here on out we can actually use the real `bound` library.

Pros:

  1.) Because we can now `succ`/`F` a whole tree, we don't have to pay the usual De Bruijn performance tax. 
      We get `O(1)` lifting, and when we `instantiate` we can skip past the whole `F`'d tree.
  
  2.) We use `Foldable` and `Traversable` to extract information about free variables and do
      variable -> variable substitution.
  
  3.) We can use the `MonadTrans` instance for `Scope b` to facilitate the construction of `Exp`. 
      We do this so often that we turn the `>>= lift . f` idiom into a combinator (`>>>=`) 
      that we'll talk about further when we talk about patterns.

    This is a complete example syntax tree example that supports the use of `Foldable`/`Traversable`/`Monad`
    to extract information about the free variables, and do capture-avoiding substitution!

```active haskell
-- /show
{-# LANGUAGE DeriveFunctor, DeriveFoldable, DeriveTraversable #-}
import Data.Foldable
import Data.Traversable
import Bound

data Exp a
 = Var a
 | App (Exp a) (Exp a)
 | Lam (Scope () Exp a)
 | Let [Scope Int Exp a] (Scope Int Exp a)
 deriving (Functor,Foldable,Traversable)
  
instance Monad Exp where
  return = Var
  Var a >>= f = f a
  App l r >>= f = App (l >>= f) (r >>= f)
  Lam s >>= f = Lam (s >>>= f)
  Let xs b >>= f = Let (map (>>>= f) xs) (b >>>= f)
-- /show
main = putStrLn "That's it!"
```
  
  4.) We can now deal with simultaneous substitution. To make things interesting in the example above we've extended the example with a simultaneous binding for all of the variables in a recursive let!

Cons:

  1.) I still haven't shown you how to get `Eq`, `Ord`, `Read`, `Show` such that `Eq` and `Ord` respect alpha-equivalence.
  
### Loose Ends

To work around the polymorphic recursion in `Scope`, we turn to my fairly boring and generically named `prelude-extras` package. This package provides higher-rank versions of typeclasses from the `Prelude`.

For example

```haskell
class Eq1 t where
  (==#) :: Eq a => t a -> t a -> Bool
  (/=#) :: Eq a => t a -> t a -> Bool
```

along with `Eq2`, `Ord1`, `Ord2`, `Show1`, `Show2`, `Read1` and `Read2`. 

I told you it was boring.

There has also been some discussion on the libraries mailing list of randomly renaming these things and putting (a subset?) of these in `transformers` to permit Haskell 98 `Show` instances things like `IdentityT` and `WriterT`.

Now, what we can do is use the fact that these classes have default definitions and the instances for `Scope` are defined in terms of `Eq1`, `Ord1`, `Show1`, and `Read1` for the base data type, to obtain the final solution!

```active haskell
-- show
{-# LANGUAGE DeriveFunctor, DeriveFoldable, DeriveTraversable #-}
import Bound
import Control.Applicative
import Control.Monad
import Data.Foldable
import Data.Traversable
import Prelude.Extras

data Exp a 
  = Var a 
  | App (Exp a) (Exp a) 
  | Lam (Scope () Exp a)
  deriving (Eq,Ord,Show,Read,Functor,Foldable,Traversable)

instance Eq1 Exp
instance Ord1 Exp
instance Show1 Exp
instance Read1 Exp
instance Applicative Exp where 
  pure = Var
  (<*>) = ap

instance Monad Exp where
  return = Var
  Var a   >>= f = f a
  App x y >>= f = App (x >>= f) (y >>= f)
  Lam e   >>= f = Lam (e >>>= f)
-- /show
whnf :: Exp a -> Exp a
whnf (App f a) = case whnf f of
  Lam b -> whnf (instantiate1 a b)
  f'    -> App f' a
whnf e = e

lam :: Eq a => a -> Exp a -> Exp a
lam v b = Lam (abstract1 v b)

nf :: Exp a -> Exp a
nf e@Var{}   = e
nf (Lam b) = Lam $ toScope $ nf $ fromScope b
nf (App f a) = case whnf f of
  Lam b -> nf (instantiate1 a b)
  f' -> nf f' `App` nf a

main = putStrLn "It compiles"
```

and we can finally show syntax tree terms again.

The instances of `Eq` and `Ord` for `Scope` in particular are careful to compare only up to alpha-equivalence by quotienting out the placement of any internal `F` levels in the tree as if they'd all been pushed out to the leaves.

We can use `abstract1` and `instantiate1`, which are analogous to the `abstract` and `instantiate` we were using before we generalized our generalized De Bruijn index representation to define things like smart constructors

```haskell
lam :: Eq a => a -> Exp a -> Exp a
lam v b = Lam (abstract1 v b)
```

and evaluation strategies such as computing weak head normal form:

```haskell
whnf :: Exp a -> Exp a
whnf (App f a) = case whnf f of
  Lam b -> whnf (instantiate1 a b)
  f'    -> App f' a
whnf e = e
```

One thing the `f (Maybe a)` representation was good for was for walking under binders. `bound` offers the round trip through this representation as `fromScope` and `toScope`. Now, if we need to walk under a lambda, we can!

```haskell
nf :: Exp a -> Exp a
nf e@V{}   = e
nf (Lam b) = Lam $ toScope $ nf $ fromScope b
nf (App f a) = case whnf f of
  Lam b -> nf (instantiate1 a b)
  f' -> nf f' `App` nf a
```

### Bound.Class

Finally, it is worth commenting on the generality of `(>>>=)` as it points to the class for which this package is named.

In addition to `Scope` and `Var`, the `bound` package provides the `Bound` class:

```haskell
class Bound t where
  (>>>=) :: Monad f => t f a -> (a -> f b) -> t f b
  default (>>>=) :: (MonadTrans t, Monad f) => t f a -> (a -> f b) -> t f b
  m >>>= f = m >>= lift . f
```

An instance of `Bound` is required to form a left module over all monads. That is to say it should satisfy the following two laws that were identified by Andrea Vezzosi:

1.) The first such law is analogous to the first monad law:

```haskell
m >>>= return ≡ m
```
    
2.) The second such law is an associativity law: 

```haskell
m >>>= (λ x → k x >>= h) ≡ (m >>>= k) >>>= h
```
    
Trivially, any valid `MonadTrans` instance satisfies these laws, hence the default definition.

This extra flexibility is used in the [`bound`](https://github.com/ekmett/bound) [`examples/`](https://github.com/ekmett/bound/tree/master/examples) folder to deal with complex pattern matching or other binding/telescoping structures.

### What's Next?

You've just taken a crash course on name capture and been shown how `bound` deals with common isses and makes it easy to work with De Bruijn indices without ever thinking about De Bruijn indices.

Hopefully, you want to learn more. Perhaps after a slight break to let this settle. If so, I can offer you three further avenues for exploration off the top of my head.

First, I'd like to note that there are a few middlingly-complex examples in the [`bound`](https://github.com/ekmett/bound) [`examples/`](https://github.com/ekmett/bound/tree/master/examples) folder to explore and use as a template. Notably:

1.) [`Simple.hs`](https://github.com/ekmett/bound/blob/master/examples/Simple.hs) continues in the vein 
    we've taken here but fleshes out recursive let and sticks to Haskell 98 religiously.
2.) 
  [`Deriving.hs`](https://github.com/ekmett/bound/blob/master/examples/Deriving.hs) takes over from 
  where we're leaving off with here and uses
    
```haskell
{-# LANGUAGE DeriveFunctor, DeriveFoldable, DeriveTraversable #-}
```
  
  to reduce boilerplate, and importantly adds pattern matching. This showcases the need for the `Bound`
  class for dealing with patterns, case alternatives, and similar module structures.
    
3.) [`Overkill.hs`](https://github.com/ekmett/bound/blob/master/examples/Overkill.hs) offers a trip down the rabbit hole of type safety. It shows how strong the type guarantees _can_ get, by using custom kinds to index into its patterns and improves the safety of pattern matching and let binding over and above the `Deriving.hs` approach, but at the expense of a great deal more code!

Second, it is also possible to derive a higher order version of `bound` that can deal with strongly typed EDSLs as well. I've included a worked example version of it here under a spoiler tag simply for completeness, including an example of what could be turned into a network serializable EDSL replete with local variable bindings and lambdas, but it isn't for the faint of heart. It is also probably buggy and is completely untested. If you fix any issues with it, please feel free to email me!

@@@
```active haskell

{-# LANGUAGE GADTs, Rank2Types, KindSignatures, ScopedTypeVariables, TypeOperators, DataKinds, PolyKinds, MultiParamTypeClasses, FlexibleInstances, TypeFamilies, DoRec, ExtendedDefaultRules #-}

import Control.Applicative
import Control.Category
import Control.Comonad
import Control.Monad.Fix
import Control.Monad (ap)
import Data.Functor.Identity
import Data.Typeable
import Data.Monoid
import Data.Unique
import System.IO.Unsafe
import Unsafe.Coerce
import Prelude hiding ((.),id)

infixl 1 >>>-, >>-

type Nat f g = forall x. f x -> g x

class HFunctor t where
  hmap :: Nat f g -> Nat (t f) (t g)

class HFunctor t => HTraversable t where
  htraverse :: Applicative m => (forall x. f x -> m (g x)) -> t f a -> m (t g a)

class HFunctor t => HMonad t where
  hreturn :: Nat f (t f)
  (>>-)   :: t f a -> Nat f (t g) -> t g a

infixr 1 -<<
(-<<) :: HMonad t => Nat f (t g) -> Nat (t f) (t g)
f -<< m = m >>- f

class HBound s where
  (>>>-) :: HMonad t => s t f a -> Nat f (t g) -> s t g a

class HMonadTrans s where
  hlift :: HMonad t => Nat (t f) (s t f)

data Var b f a where
  B :: b a -> Var b f a
  F :: f a -> Var b f a

instance HFunctor (Var b) where
  hmap _ (B b) = B b
  hmap f (F a) = F (f a)

instance HTraversable (Var b) where
  htraverse _ (B b) = pure (B b)
  htraverse f (F a) = F <$> f a

instance HMonad (Var b) where
  hreturn   = F
  B b >>- _ = B b
  F a >>- f = f a

newtype Scope b t f a = Scope { unscope :: t (Var b (t f)) a }

instance HFunctor t => HFunctor (Scope b t) where
  hmap f (Scope b) = Scope (hmap (hmap (hmap f)) b)

instance HTraversable t => HTraversable (Scope b t) where
  htraverse f (Scope b) = Scope <$> htraverse (htraverse (htraverse f)) b

instance HMonad t => HMonad (Scope b t) where
  hreturn = Scope . hreturn . F . hreturn
  Scope e >>- f  = Scope $ e >>- \v -> case v of
    B b -> hreturn (B b)
    F ea -> ea >>- unscope . f

instance HMonadTrans (Scope b) where
  hlift = Scope . hreturn . F

instance HBound (Scope b) where
  Scope m >>>- f = Scope (hmap (hmap (>>- f)) m)

data Equal a b where
   Refl :: Equal a a

instance Category Equal where
  id = Refl
  Refl . Refl = Refl

abstract :: HMonad t => (forall x. f x -> Maybe (b x)) -> Nat (t f) (Scope b t f)
abstract f = Scope . hmap (\y -> case f y of
  Just b -> B b
  Nothing -> F (hreturn y))

instantiate :: HMonad t => Nat b (t f) -> Nat (Scope b t f) (t f)
instantiate k (Scope e) = e >>- \v -> case v of
  B b -> k b
  F a -> a

data Ix :: [*] -> * -> * where
  Z :: Ix (a ': as) a
  S :: Ix as b -> Ix (a ': as) b

data Vec :: (* -> *) -> [*] -> * where
  HNil :: Vec f '[]
  (:::) :: f b -> Vec f bs -> Vec f (b ': bs)

infixr 5 :++, :::

type family (:++) (as :: [*]) (bs :: [*]) :: [*]
type instance '[] :++ bs = bs
type instance (a ': as) :++ bs = a ': (as :++ bs)

happend :: Vec f as -> Vec f bs -> Vec f (as :++ bs)
happend HNil bs = bs
happend (a ::: as) bs = a ::: happend as bs

hsingleton :: f a -> Vec f '[a]
hsingleton x = x ::: HNil

instance HFunctor Vec where
  hmap _ HNil = HNil
  hmap f (x ::: xs) = f x ::: hmap f xs

instance HTraversable Vec where
  htraverse _ HNil = pure HNil
  htraverse f (x ::: xs) = (:::) <$> f x <*> htraverse f xs

class EqF f where
  (==?) :: f a -> f b -> Maybe (Equal a b)

data Lit t where
  Integer :: Integer -> Lit Integer
  Double  :: Double -> Lit Double
  String :: String -> Lit String

instance EqF Lit where
  Integer a ==? Integer b = Just Refl
  ...

value :: Lit a -> a
value (Integer i) = i
value (Double d) = d
value (String s) = s

class Literal a where
  literal :: a -> Lit a

instance Literal Integer where
  literal = Integer

instance Literal String where
  literal = String

data Remote :: (* -> *) -> * -> * where
  Var :: f a -> Remote f a
  Lit :: Lit a -> Remote f a
  Lam :: Scope (Equal b) Remote f a -> Remote f (b -> a)
  Let :: Vec (Scope (Ix bs) Remote f) bs -> Scope (Ix bs) Remote f a -> Remote f a
  Ap  :: Remote f (a -> b) -> Remote f a -> Remote f b

lam_ :: EqF f => f a -> Remote f b -> Remote f (a -> b)
lam_ v f = Lam (abstract (v ==?) f)
 
lit :: Literal a => a -> Remote f a
lit = Lit . literal

instance HFunctor Remote where
  hmap f (Var a)    = Var (f a)
  hmap _ (Lit t)    = Lit t
  hmap f (Lam b)    = Lam (hmap f b)
  hmap f (Let bs b) = Let (hmap (hmap f) bs) (hmap f b)
  hmap f (Ap x y)   = Ap (hmap f x) (hmap f y)

instance HTraversable Remote where
  htraverse f (Var a)    = Var <$> f a
  htraverse _ (Lit t)    = pure $ Lit t
  htraverse f (Lam b)    = Lam <$> htraverse f b
  htraverse f (Let bs b) = Let <$> htraverse (htraverse f) bs <*> htraverse f b
  htraverse f (Ap x y)   = Ap <$> htraverse f x <*> htraverse f y

data MyF a where
  Mean :: MyF (Double -> Double)

instance HMonad Remote where
  hreturn        = Var
  Var a    >>- f = f a
  Lit t    >>- _ = Lit t
  Lam b    >>- f = Lam (b >>>- f)
  Let bs b >>- f = Let (hmap (>>>- f) bs) (b >>>- f)
  Ap x y   >>- f = Ap (x >>- f) (y >>- f)

(!) :: Vec f v -> Ix v a -> f a
(b ::: _)  ! Z   = b
(_ ::: bs) ! S n = bs ! n

eval :: Remote Identity a -> a
eval (Var w) = extract w
eval (Lit i) = value i
eval (Lam b) = \a -> eval (instantiate (\Refl -> Var (Identity a)) b)
eval (Let bs b) = eval (instantiate (vs !) b) where vs = hmap (instantiate (vs !)) bs
eval (Ap x y) = eval x (eval y)

closed :: HTraversable t => t f a -> Maybe (t g a)
closed = htraverse (const Nothing)

newtype V (a :: *) = V Unique

instance EqF V where
  V a ==? V b
    | a == b    = Just (unsafeCoerce Refl)
    | otherwise = Nothing

lam :: (Remote V a -> Remote V b) -> Remote V (a -> b)
lam f = unsafePerformIO $ do
  x <- fmap V newUnique
  return $ Lam $ abstract (x ==?) $ f $ Var x

data Binding a = V a := Remote V a

rhs :: Binding a -> Remote V a
rhs (_ := a) = a

data Bindings = forall bs. Bindings (Vec Binding bs)

elemIndex :: Vec Binding bs -> V a -> Maybe (Ix bs a)
elemIndex HNil              _ = Nothing
elemIndex ((x := r) ::: xs) y = case x ==? y of
  Just Refl -> Just Z
  Nothing   -> S <$> elemIndex xs y

instance Monoid Bindings where
  mempty = Bindings HNil
  Bindings xs `mappend` Bindings ys = Bindings (happend xs ys)

-- Allow the use of DoRec to define let statements

newtype Def a = Def { runDef :: IO (a, Bindings) }

instance Functor Def where
  fmap f (Def m) = Def $ do
    (a,w) <- m
    return (f a, w)

instance Applicative Def where
  pure = return
  (<*>) = ap

instance Monad Def where
  return a = Def $ return (a, mempty)
  Def m >>= f = Def $ do
    (a, xs) <- m
    (b, ys) <- runDef (f a)
    return (b, xs <> ys)

instance MonadFix Def where
  mfix m = Def $ mfix $ \ ~(a, _) -> runDef (m a)

def :: Remote V a -> Def (Remote V a)
def v@Var{} = Def $ return (v, mempty) -- this thing already has a name
def r = Def $ do
  v <- fmap V newUnique
  return (Var v, Bindings (hsingleton (v := r)))

let_ :: Def (Remote V a) -> Remote V a
let_ (Def m) = unsafePerformIO $ do
    (r, Bindings bs) <- m
    return $ Let (hmap (abstract (elemIndex bs) . rhs) bs)
                 (abstract (elemIndex bs) r)

```
@@@

Third, for a much larger "industrial scale" example you can explore our work-in-progress [compiler for Ermine](https://github.com/ermine-language/ermine) that uses [`bound`](https://github.com/ekmett/bound) \(and [`lens`](https://github.com/ekmett/lens)\) extensively for manipulating its [`Term`](http://ermine-language.github.io/ermine/Ermine-Syntax-Term.html) language, [`Type`](http://ermine-language.github.io/ermine/Ermine-Syntax-Type.html) system, [`Kind`](http://ermine-language.github.io/ermine/Ermine-Syntax-Kind.html) system and the [`Core`](http://ermine-language.github.io/ermine/Ermine-Syntax-Core.html) language it spits out as a witness during type checking. 

Happy Hacking!

-[Edward Kmett](mailto:ekmett@gmail.com)

August 19 2013

P.S. Apparently the proper Dutch convention with names is to use [Nicolaas Govert de Bruijn](http://en.wikipedia.org/wiki/Nicolaas_Govert_de_Bruijn) when writing out a full name, but to capitalize De Bruijn when using the surname in isolation.
# Imperative OOP Ceremony: Design Patterns Pt. 1

The naming of `Reader`, `Writer` and `State` is not very intuitive. For starters, a `Reader` is a **function**, a `Writer` is a **tuple** and a `State` is a **function** that returns a **tuple**. The magic lies in the implementation of `Reader`, `Writer` and `State` as *monads*.

* The `Reader` function binds to an operator that "reads" both the function's argument and its return value.
* The `Writer` tuple is a state-value pair that binds to a function that takes the value element and creates a new state-value pair. The implementation of `>>=` "writes" the new state element to the original state element.
* The `State` function takes a state and returns a state-value pair. It binds to an operator that "reads" both the new state element and the new value element.

## Reader

Since a `Reader` is a mere **function**, its type is basically `->`. The following two type signatures are equivalent:

* `a -> b`
* `(->) a b`

For the sake of readability, we will define a type synonym `R` for `->`. The next two type signatures are equivalent to the former two:

* ``a `R` b``
* `R a b`

``` active haskell
{-# LANGUAGE TypeSynonymInstances #-}

import Prelude hiding (Monad (..), (.))
x.f = f(x)

class Monad m where
  (>>=)  :: m a -> (a -> m b) -> m b
  return :: a -> m a

-- show
type R = (->)
instance Monad (R domain) where
  return(x) = \(_) -> x
  g >>= f = \(x) -> g(x).f!(x)
    where (!) = ($)

reader = (*3) >>= (-)

main = print(reader(42))
-- /show
```

@@@
**NOTE**: One may be tempted to simply write `g(x).f(x)` instead of `g(x).f!(x)`. This would still yield a consistent `Reader` (with the arguments of `f` flipped), yet it would be incompatible with the signature `m a -> (a -> m b) -> m b` of `>>=`.
@@@

## Writer

Since a `Writer` is a mere **tuple**, its type is basically `(,)`. (`(a, b)` is syntactic sugar for `(,) a b`.) For the sake of readability, we will define a type synonym `W` for `(,)`, making the following type signatures equivalent:

* `(a, b)`
* `(,) a b`
* `W a b`

In order for a state value to be "writeable" it must be a `Monoid`, i.e. it must implement

* `<>` (or `mappend`), an operator that appends one `Monoid` to another `Monoid` and
* `mempty`, an "empty" `Monoid`.

``` active haskell
{-# LANGUAGE TypeSynonymInstances #-}

import Data.Monoid
import Prelude hiding (Monad (..), (.))
x.f = f(x)

class Monad m where
  (>>=)  :: m a -> (a -> m b) -> m b
  return :: a -> m a

-- show
type W = (,)
instance Monoid this => Monad (W this) where
  return(x) = (mempty, x)
  (this, x) >>= f = (this <> this', x')
    where (this', x') = f(x)

write(x) = ("inc " ++ show(x) ++ "! ", x + 1)

main = print $ (mempty, 42) >>= write >>= write >>= write 
-- /show
```

@@@
**NOTE**: We might have come up with a `Writer` value-state pair `(x, this)` rather than a `Writer` state-value pair `(this, x)`, but such an implementation would be incompatible with the type signature `m a -> (a -> m b) -> m b` of `>>=`. Besides, whereas `(->)` is already an instance of `Monad` implementing the `Reader` pattern, `(,)` has no implementation as a `Monad` instance whatsoever.
@@@

## State

A `State` is combination of `Reader` and `Writer`. It's a `Reader` function that takes a state and returns a `Writer` state-value pair.

Let's define a type synonym `S this a` for `R this (W this a)`, i.e. `this -> (this, a)`.

@@@
**NOTE**: Since `R` is already an instance of `Monad`, and we can't make `S` an instances of `R`, too. But we would fail to implement a `Monad` instance for `S this` anyway, because we can't unify `S this`, a partial applied type synonym, with a `Monad` instance `m`. We therefore can't implement `>>=` (nor `return`), failing to unify the type signatures `m a -> (a -> m b) -> m b` and `S this a -> (a -> S this b) -> S this b`.
@@@

``` active haskell
import Data.Monoid
import Prelude hiding ((.), Monad(..))
x.f = f(x)

type R = (->)
type W = (,)

-- show
type S this a = R this (W this a)
-- "instance Monad (S this a) where"
return :: a -> S this a
return(x) = \(this) -> (this, x)
(>>=) :: S this a -> (a -> S this b) -> S this b
start >>= f = \(this) ->
  let (this',  x') = this.start
  in   this'.f(x')

start = \(this) -> (this ++ "!", 0)
append(x) = \(this) -> (this ++ ".", length(this) - x)

main = do {
  print(start $ "hi");
  print(start >>= append $ "hi");
  print(start >>= append >>= append $ "hi");
}
-- /show
```

In the next tutorial we shall reimplement `Reader`, `Writer` and `State` as `newtype` instances instead of type synonyms (`type`).
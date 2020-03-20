# Imperative OOP Ceremony: Design Patterns Pt. 2

In the last tutorial we saw primitive implementations of the `Reader`, `Writer` and `State` patterns. Since their instantiation as monads was quite hacky, we shall reimplement them in this tutorial as new types of their own rather than type synonyms for `->`, `(,)` and `this -> (this, a)`.

A `newtype` is difficult to read, though, due to the boxing and un-boxing boilerplate.

## Reader

In order to implement a `Reader` instance of `Monad` and a `reader` example, we have to write a lot of boilerplate for a `newtype`. Instead of `(*3) >>= (-)` we have to wrap `(*3)` up as a `Reader` and make `(-)` a function that returns a `Reader`. In order to make `Reader(*3) >>= \(x) -> Reader((-) x)` a normal function again (which it basically is), we have to unwrap it using the `Reader` accessor `runReader`. These two functions are equivalent:

* `(*3) >>= (-)`
* `runReader $ Reader(*3) >>= \(x) -> Reader((-) x)`

``` active haskell
import Prelude hiding (Monad(..), (.))
x.f = f(x)

class Monad m where
  (>>=) :: m a -> (a -> m b) -> m b
  return :: a -> m a

newtype Reader domain codomain = Reader { runReader :: domain -> codomain }
instance Monad (Reader domain) where
  return(x) = Reader $ \(_) -> x
  Reader g >>= f = Reader $ \(x) -> g(x).f!(x)
    where (!) = runReader

reader = runReader $ Reader(*3) >>= \(x) -> Reader((-) x)

main = print(reader(42))
```

## Writer

Please note that Haskell's `Writer` tuple is a value-state pair instead of a state-value pair.

``` active haskell
import Data.Monoid
import Prelude hiding (Monad(..), (.))
x.f = f(x)

class Monad m where
  (>>=) :: m a -> (a -> m b) -> m b
  return :: a -> m a

-- show
newtype Writer this a = Writer { runWriter :: (a, this) } deriving (Show)
instance Monoid this => Monad (Writer this) where
  return(x) = Writer(x, mempty)
  Writer(x, this) >>= f = Writer(x', this <> this')
    where Writer (x', this') = f(x)

write(x) = Writer(x + 1, "inc " ++ show(x) ++ "! ")

main = print(Writer(42, mempty) >>= write >>= write >>= write)
-- /show
```

## State

Also `State` is implemented using a value-state pair instead of a state-value pair.

``` active haskell
import Prelude hiding (Monad(..), (.))
x.f = f(x)

class Monad m where
  (>>=) :: m a -> (a -> m b) -> m b
  return :: a -> m a

-- show
newtype State this a = State { runState :: this -> (a, this) }
instance Monad (State s) where
  return(x) = State $ \(this) -> (x, this)
  start >>= f = State $ \(this) -> 
    let (x', this') = this.(start!)
    in   this'.(f(x')!)
      where (!) = runState

start = State $ \(this) -> (0, this ++ "!")
append(x) = State $ \(this) -> (length(this) - x, this ++ ".")

main = do {
  print(start `runState` "hi");
  print(start >>= append `runState` "hi");
  print(start >>= append >>= append `runState` "hi");
}
-- /show
```

In the next tutorial we shall see how to combine monads like `IO` with a "pattern" monad like `Reader`, `Writer` and `State`.
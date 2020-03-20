# Imperative OOP Ceremony: Fluent Interface Pt. 2

## Programmable Objects

In the last tutorial we created a Fluent Interface for a type `Shape` consisting only of the method `doubleShapeArea` and we enforced it by making the constructor `Circle` and deconstructor (accessor) `radius` private.

We then implemented a public `make` method, a generic constructor (factory) that doesn't leak implementation details of Fluent Interface objects (in our case: the `radius` property of `Circle` objects).

``` active haskell
import Prelude hiding ((.))
x.f = f x
data Shape a = Circle { radius :: a } deriving (Show)
-- show
-- Fluent.hs
class Fluent object where
  make ::  a -> object(a)

-- Geometry.hs
instance Fluent Shape where
  make = Circle

doubleShapeArea(Circle(r)) = Circle(r * sqrt(2)) -- private Fluent Interface

-- Main.hs
circ = make(2) :: Shape(Double)

main = do {
  print(circ);
  print(circ.doubleShapeArea); -- application of exported Fluent Interface
}
```

We are going to add a second method `bind`, a generic deconstructor (accessor) of Fluent Interface objects which applies a function that must return a Fluent Interface object. By doing so we make our Fluent Interface *programmable* without leaking implementation details of objects that implement it.


``` haskell
-- Fluent.hs
class Fluent object where
  make ::  a -> object(a)
  bind :: (a -> object(b)) -> object(a) -> object(b)

-- Geometry.hs
instance Fluent Shape where
  make = Circle
  bind(remake) = \(Circle(r)) -> remake(r)
```

We can now implement our own `doubleShapeArea'` in terms of `make` and `bind`, with `bind` as a method that calls a "bindable" function `doubleShapeAreaBindable`.


``` active haskell
import Prelude hiding ((.))
x.f = f x
data Shape a = Circle { radius :: a } deriving (Show)

class Fluent object where
  make ::  a -> object(a)
  bind :: (a -> object(b)) -> object(a) -> object(b)
instance Fluent Shape where
  make = Circle
  bind(remake) = \(Circle(r)) -> remake(r)
-- show
-- Geometry.hs
doubleShapeArea(Circle(r)) = Circle(r * sqrt(2)) -- private Fluent Interface

-- Main.hs
doubleShapeAreaBindable(r) = make  (r * sqrt(2)) -- customized Fluent Interface
doubleShapeArea' = bind(doubleShapeAreaBindable)

circ = make(2) :: Shape(Double)

main = do {
  print(circ);
  print(circ.doubleShapeArea);  -- application of exported Fluent Interface
  print(circ.doubleShapeArea'); -- application of programmed Fluent Interface
}
```

As we can see, `bind` takes care of deconstructing `Circle` objects and the "bindable" function `doubleShapeAreaBindable` does the rest, i.e. reconstruct a new `Circle` object by means of the factory method `make`.

It is worth noting that a trivial deconstruction-reconstruction "cycle" with `bind` and `make` leaves the programmable object unchanged.

``` active haskell
import Prelude hiding ((.))
x.f = f x
data Shape a = Circle { radius :: a } deriving (Show)

class Fluent object where
  make ::  a -> object(a)
  bind :: (a -> object(b)) -> object(a) -> object(b)
instance Fluent Shape where
  make = Circle
  bind(remake) = \(Circle(r)) -> remake(r)

circ = make(2) :: Shape(Double)

-- show
main = do {
  print(circ);
  print(circ.bind(make))
}
```

In FP lingo, a programmable object is called *Monad*. Monads are powerful enough to model computations. In the next tutorial we shall see how, and why ultimately a Programmable Fluent Interface is also referred to as *Programmable Semicolons* and why Haskell's `make` is in fact called `return`.
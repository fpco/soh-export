# Imperative OOP Ceremony: Fluent Interface Pt. 1

## Method Chaining

A Fluent Interface is a chain of object-modifications that return the modified object on each chained modification.

``` active haskell
import Prelude hiding ((.))
x.f = f x

data Shape = Circle { radius :: Float } deriving (Show)
doubleShapeArea(Circle r) = Circle(r * sqrt(2))
area(Circle r) = r^2 * pi
circ = Circle(2)

main = do {
  print(circ);
  print(circ.doubleShapeArea);
  print(circ.doubleShapeArea.doubleShapeArea);

  print(circ.area);
  print(circ.doubleShapeArea.area);
  print(circ.doubleShapeArea.doubleShapeArea.area);
}
```

As we can see, `doubleShapeArea` can be chained as often as we want because it returns a (modified) `Shape` object.


## Private Constructors, Private Accessors

In order to **enforce** our Fluent Interface, we make the constructor `Circle` and the "deconstructor" `radius` private. Otherwise anyone who imports our module could write their own `doubleShapeArea`.

``` haskell
-- Geometry.hs
module Geometry (Shape, doubleShapeArea, circ) where

data Shape = Circle { radius :: Float }
circ = Circle(2)
doubleShapeArea(Circle r) = Circle(r * sqrt(2))

-- Main.hs
module Main where

import Geometry -- without Circle, but with circ

main = do {
  print(circ);
  print(circ.doubleShapeArea);
}
```

In the export list of our module `Geometry` we have included only `Shape` but not `Circle` and `radius`, thereby making `Shape` public, leaving `Circle` and `radius` private. A user of `Geometry` now relies on imported `Circle` objects like `circ` and methods like `doubleShapeArea`. In other words, we have successfully enforced our Fluent Interface, but our module has become very inflexible, as it doesn't allow for constructing `Circle` objects.

## Factories

We can easily implement and export a factory method `makeCircle` which constructs a `Circle` object but since it's technically not a constructor, it can't be used in pattern matching in order to access `radius`. In return we get a more flexible module with a Fluent Interface that is still enforced.

``` haskell
-- Geometry.hs
module Geometry (Shape, doubleShapeArea, makeCircle) where

data Shape = Circle { radius :: Float }
makeCircle = Circle
doubleShapeArea(Circle r) = Circle(r * sqrt(2))

-- Main.hs
module Main where

import Geometry -- without Circle, but with makeCircle
circ = makeCircle(2)

main = do {
  print(circ);
  print(circ.doubleShapeArea);
}
```

## A Fluent Interface

With the Factory Pattern in place, we have implemented a Fluent Interface. Let's write an interface (FP lingo: *type class*) for Fluent Interfaces. Apparently, all we need is a factory method like `makeCircle` that returns an object. Let's call it `make` then.

``` active haskell
import Prelude hiding ((.))
x.f = f x

doubleShapeArea(Circle(r)) = Circle(r * sqrt(2))
-- show
class Fluent object where
  make :: Float -> object

instance Fluent Shape where
  make = Circle

data Shape = Circle { radius :: Float } deriving (Show)

circ = make(2) :: Shape

main = do {
  print(circ.doubleShapeArea);
}
-- /show
```

## Polymorphism

As we can see, this `Fluent` interface requires an implementation of a `make` method that takes a `Float` as an argument. We can get to a more generic solution with an additional type variable `a` instead of `Float`. (`object` is a type variable as well.) By doing so, we make `Shape` *polymorphic*, (Now we can instantiate `Circle` objects that have a `Double` precision `radius`.)

``` active haskell
import Prelude hiding ((.))
x.f = f x

doubleShapeArea(Circle(r)) = Circle(r * sqrt(2))
-- show
class Fluent object where
  make :: a -> object(a)

instance Fluent Shape where
  make = Circle

data Shape a = Circle { radius :: a } deriving (Show)

circ = make(2) :: Shape(Double)

main = do {
  print(circ.doubleShapeArea);
}
-- /show
```

In the next tutorial we shall add even more flexibility to Fluent Interfaces by making them *programmable*.
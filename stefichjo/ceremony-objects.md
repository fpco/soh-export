# Imperative OOP Ceremony: Classes

## Abstract Classes

In Haskell, a class always extends an abstract class, which has to be declared *before* the concrete class.

``` active haskell
import Prelude hiding ((.))
x.f = f x
-- show
data Shape = Circle {
  radius :: Float
}

circ = Circle 12

main = do {
  print(circ.radius);
}
-- /show
```

As we can see, the abstract class `Shape` is declared by the keyword `data`. So `data Shape = Circle` "means" `abstract class Shape {}` and `class Circle extends Shape {}` combined. In FP lingo, we will refer to `Shape` as the **type** of `circ` rather than to `circ` as an object that extends the abstract class `Shape`.

## Concrete Classes

All concrete classes that extend the same abstract class must be declared together.

``` active haskell
import Prelude hiding ((.))
x.f = f x
-- show
data Shape = Circle     { radius  :: Float }
           | Rectangle  { width   :: Float
                        , height  :: Float }

circ = Circle 12
rect = Rectangle 16 9

main = do {
  print(circ.radius^2 * pi);
  print(rect.width * rect.height);
}

-- /show
```

We can observe, that `Circle` and `Rectangle` are not only the names of the concrete classes `Circle {...}` and `Rectangle { ... }` but also of their constructors `Circle ...` and `Rectangle ... ...`. In FP lingo, we will refer to `Circle` and `Rectangle` as the **constructors** of `circ` and `rect` rather than to `circ` and `rect` as instances of the concrete classes `Circle` and `Rectangle`.

(An abstract class that is extended by multiple concrete classes is called an *algebraic data type* in FP lingo, hence the keyword `data`. In Scala, concrete classes that extend the same abstract class are called *case classes*.)

Because objects are immutable like values, they may be referred to as *value objects* in DDD. In FP lingo, objects are *values*, too.

## Overloading

Methods are defined outside the class definition. Through their type they remain associated with their class, though.

``` active haskell
{-# LANGUAGE DeriveDataTypeable #-}
import Data.Data
import Prelude hiding ((.))
x.f = f x
getConstructor(x) = x.toConstr.show
data Shape = Circle     { radius  :: Float }
           | Rectangle  { width   :: Float
                        , height  :: Float } deriving (Data, Typeable)

circ = Circle 12
rect = Rectangle 16 9

-- show
areaCircle(this) = this.radius^2 * pi
areaRectangle(this) = this.width * this.height

main = do {
  print(circ.areaCircle);
  print(rect.areaRectangle);
}
-- /show
```

Methods are always associated with the **abstract** class, which means that that `areaCircle` and `areaRectangle` are in fact members of the abstract class `Shape`. This allows for writing a single `area` method associated with `Shape`. All we have to do is to distinguish between the concrete classes `Circle` and `Rectangle`.

``` active haskell
{-# LANGUAGE DeriveDataTypeable #-}
import Data.Data
import Prelude hiding ((.))
x.f = f x
getConstructor(x) = x.toConstr.show
data Shape = Circle     { radius  :: Float }
           | Rectangle  { width   :: Float
                        , height  :: Float } deriving (Data, Typeable)

circ = Circle 12
rect = Rectangle 16 9

-- show
area(this) = if this.getConstructor == "Circle"
             then this.radius^2 * pi
             else if this.getConstructor == "Rectangle"
                  then this.width * this.height
                  else undefined

main = do {
  print(circ.area);
  print(rect.area);
}
-- /show
```

This looks quite cluttered and contains an awkward `undefined` because Haskell has no naked `if-then` expressions. Let's try a case switch instead of this nested if-then-else.


``` active haskell
{-# LANGUAGE DeriveDataTypeable #-}
import Data.Data
import Prelude hiding ((.))
x.f = f x
getConstructor(x) = x.toConstr.show
data Shape = Circle     { radius  :: Float }
           | Rectangle  { width   :: Float
                        , height  :: Float } deriving (Data, Typeable)

circ = Circle 12
rect = Rectangle 16 9

-- show
area(this) = case this.getConstructor of
  "Circle"     -> this.radius^2 * pi
  "Rectangle"  -> this.width * this.height

-- /show
main = do {
  print(circ.area);
  print(rect.area);
}
```

Constructor-based case switches are called *pattern matching* in FP lingo and they are supported out of the box, without `getClass` ceremony:

``` active haskell
import Prelude hiding ((.))
x.f = f x
data Shape = Circle     { radius  :: Float }
           | Rectangle  { width   :: Float
                        , height  :: Float }

circ = Circle 12
rect = Rectangle 16 9

-- show
area(this) = case this of
  c@(Circle r)       -> c.radius^2 * pi
  r@(Rectangle w h)  -> r.width * r.height

-- /show
main = do {
  print(circ.area);
  print(rect.area);
}
```

We're declaring inline `r`, `w` and `h`, the arguments of `Circle` and `Rectangle`, but we don't use them. Instead, we use `c` and `r` to access `radius`, `width` and `height`. Let's do some refactoring and access `r`, `w` and `h` directly.

``` active haskell
import Prelude hiding ((.))
x.f = f x
data Shape = Circle     { radius  :: Float }
           | Rectangle  { width   :: Float
                        , height  :: Float }

circ = Circle 12
rect = Rectangle 16 9

-- show
area(this) = case this of
  Circle r       -> r^2 * pi
  Rectangle w h  -> w * h

-- /show
main = do {
  print(circ.area);
  print(rect.area);
}
```

The most idiomatic way of pattern matching in Haskell is to overload a method for each pattern.

``` active haskell
import Prelude hiding ((.))
x.f = f x
-- show
data Shape = Circle     { radius  :: Float }
           | Rectangle  { width   :: Float
                        , height  :: Float }

circ = Circle 12
rect = Rectangle 16 9

area(Circle r)       = r^2 * pi
area(Rectangle w h)  = w * h

main = do {
  print(circ.area);
  print(rect.area);
}
-- /show
```

In the next tutorial we shall see how to serialize an instance of `Shape` by means of an interface.
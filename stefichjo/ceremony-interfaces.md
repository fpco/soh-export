# Imperative OOP Ceremony: Interfaces

## Interfaces

An interface defines functionalities shared by a group of **abstract** classes. Since abstract classes are called *types* in FP lingo, interfaces are referred to as *type classes* (or *classes* for short). Therefore, if we want to make `Shape` of the previous tutorial implement `Show` (an interface that allows for serializing objects to Strings via the method `show`), we write `class Show` and `instance Show Shape` instead of `interface Show` and `abstract class Shape implements Show` in Haskell.

``` active haskell
import Prelude hiding ((.))
x.f = f x
-- show
data Shape = Circle    { radius  :: Float }
           | Rectangle { width   :: Float
                       , height  :: Float }

instance Show Shape where
  show(Circle r)      = "Circle {radius = " ++ show r ++ "}"
  show(Rectangle w h) = "Rectangle {width = " ++ show w ++ ", height = " ++ show h ++ "}"

circ = Circle 12
rect = Rectangle 16 9

main = do {
  print(circ.show);
  print(rect.show);
}
-- /show
```

Haskell provides a default implementation for `Show`. If we want to use it, we have to use the `deriving` keyword:

``` active haskell
import Prelude hiding ((.))
x.f = f x
-- show
data Shape = Circle    { radius  :: Float }
           | Rectangle { width   :: Float
                       , height  :: Float } deriving (Show)

circ = Circle 12
rect = Rectangle 16 9

main = do {
  print(circ.show);
  print(rect.show);
}
-- /show
```

It is worth noting that objects implementing the `Show` interface can be shown without using the `show` method. Moreover, their string representation *is* a value expression itself. This means that instead of `Rectangle 16 9` we can write out `Rectangle { width = 16.0, height = 9.0 }`.

## Inheritance

Let's define our own interfaces `Size` with a required function `size`. For Shapes, we will simply make it return the length of the String representation that `show` yields.

``` active haskell
import Prelude hiding ((.))
x.f = f x
f `compose` g = \(x) -> f(g(x))

data Shape = Circle    { radius  :: Float }
           | Rectangle { width   :: Float
                       , height  :: Float } deriving (Show)

circ = Circle 12
rect = Rectangle 16 9
-- show
class Size a where
  size :: a -> Int

instance Size Shape where
  size = length `compose` show

main = do {
  print(circ.size);
  print(rect.size);
}
-- /show
```

We can also make interfaces inherit from each other. In FP lingo, we *add a type constraint*. If we want to make the `Size` interface inherit from `Show` we write `class Show a => Size a` instead of `interface Size extends Show`.

``` haskell
class Show a => Size a where
  size :: a -> Int
```

In the next tutorial we shall use interfaces in order to implement Fluent Interfaces. (No pun intended.)
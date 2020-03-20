# Imperative OOP Ceremony: do-Notation Pt. 1

## Method Cascades

In the last tutorial we saw how Programmable Fluent Interfaces can be implemented and that they are referred to as *Monads* in FP lingo. We will now start to use Haskell's built-in `Monad` type class instead of our own `Fluent` interface...

``` haskell
import Prelude hiding ((.))
x.f = f x

data Shape a = Circle { radius :: a } deriving (Show)

instance Monad Shape where
  object >>= remake = remake(object.radius)
  return = Circle
```

...but we will keep on using our own method synonyms `bind` and `make`.

``` haskell
bind = flip(>>=)
make = return
```

Everything should be working as before.

``` active haskell
import Control.Monad
import Control.Applicative
import Data.Functor
import Prelude hiding ((.))
x.f = f x

data Shape a = Circle { radius :: a } deriving (Show)
instance Functor Shape where { fmap = liftM }
instance Applicative Shape where { pure = return; (<*>) = ap }
  
instance Monad Shape where
  object >>= remake = remake(object.radius)
  return = Circle

bind = flip(>>=)
make = return

-- show
doubleShapeArea(r) = make(r * sqrt(2))

circ = make(2) :: Shape(Double)

main = do {
  print(circ);
  print(circ.bind(doubleShapeArea));
}
-- /show
```

Let's add a redundant `.bind(make)` to the method chain (it trivially returns its argument) and wrap both `doubleShapeArea` and `make` in closures in order to reflect the values being passed through the method chain.

``` active haskell
import Control.Monad
import Control.Applicative
import Data.Functor
import Prelude hiding ((.))
x.f = f x

data Shape a = Circle { radius :: a } deriving (Show)
instance Functor Shape where { fmap = liftM }
instance Applicative Shape where { pure = return; (<*>) = ap }
instance Monad Shape where
  object >>= remake = remake(object.radius)
  return = Circle

bind = flip(>>=)
make = return

area(Circle r) = r^2 * pi

doubleShapeArea(r) = make(r * sqrt(2))

circ = make(2) :: Shape(Double)

-- show
main = do {
  print(circ.bind(doubleShapeArea).bind(make));
  print(circ.bind(\(a) -> a.doubleShapeArea).bind(\(b) -> make(b)));
}
-- /show
```

As we can imagine, `a` is bound to `circ.radius`, i.e. `2.0`, and `b` is bound to `radius` of a `Circle` object with twice as much area as `circ`, i.e. a radius of `2.8284271247461903`. Both `a` and `b` exist only inside the closures `\(a) -> a.doubleShapeArea` and `\(b) -> b.make`.

But, if we chose to bind `\(b) -> b.make` to `doubleShapeArea` *inside* the closure `\(a) -> a.doubleShapeArea`, `a` remains available to `make` and we can apply `make` to `a` instead of `b`, thereby ignoring `doubleShapeArea`'s return value.

``` active haskell
import Control.Monad
import Control.Applicative
import Data.Functor
import Prelude hiding ((.))
x.f = f x

data Shape a = Circle { radius :: a } deriving (Show)
instance Functor Shape where { fmap = liftM }
instance Applicative Shape where { pure = return; (<*>) = ap }
instance Monad Shape where
  object >>= remake = remake(object.radius)
  return = Circle

bind = flip(>>=)
make = return

area(Circle r) = r^2 * pi

doubleShapeArea(r) = make(r * sqrt(2))

circ = make(2) :: Shape(Double)

-- show
main = do {
  print(circ.bind(\(a) -> a.doubleShapeArea.bind(\(b) -> make(b))));
  print(circ.bind(\(a) -> a.doubleShapeArea.bind(\(b) -> make(a))));
}
-- /show
```

We can also make some computations with `a` and `b` like `sqrt(a^2 + b^2)`, which yields the radius of a circle that combines the area of two circles with radius `a` and `b`.

``` active haskell
import Control.Monad
import Control.Applicative
import Data.Functor
import Prelude hiding ((.))
x.f = f x

data Shape a = Circle { radius :: a } deriving (Show)
instance Functor Shape where { fmap = liftM }
instance Applicative Shape where { pure = return; (<*>) = ap }
instance Monad Shape where
  object >>= remake = remake(object.radius)
  return = Circle

bind = flip(>>=)
make = return

area(Circle r) = r^2 * pi

doubleShapeArea(r) = make(r * sqrt(2))

circ = make(2) :: Shape(Double)

-- show
main = do {
  print(circ.bind(\(a) -> a.doubleShapeArea.bind(\(b) -> make(sqrt(a^2 + b^2)))));
}
-- /show
```

Let's indent our code to reflect the Method Cascade.

``` active haskell
import Control.Monad
import Control.Applicative
import Data.Functor
import Prelude hiding ((.))
x.f = f x

data Shape a = Circle { radius :: a } deriving (Show)
instance Functor Shape where { fmap = liftM }
instance Applicative Shape where { pure = return; (<*>) = ap }
instance Monad Shape where
  object >>= remake = remake(object.radius)
  return = Circle

bind = flip(>>=)
make = return

area(Circle r) = r^2 * pi

doubleShapeArea(r) = make(r * sqrt(2))

circ = make(2) :: Shape(Double)

-- show
main = do {
  print(
    circ.bind(\(a) ->
      a.doubleShapeArea.bind(\(b) ->
        make(sqrt(a^2 + b^2))
      )
    )
  );
}
-- /show
```

As we can see, with each `bind` we add a new variable to the scope of the Method Cascade.

Side note: I say *variable* instead of *value* because values declared in Method Cascades can be redeclared in an inner closure, thereby *overwriting* a previously declared value of an outer closure. This happens in `circ.bind(\(a) -> a.doubleShapeArea.bind(\(a) -> make(a)))`, where `a` is been bound to the `radius` of `circ` and then to the `radius` of the return value of `doubleShapeArea`.

## do-Notation

Let's change our indentation step by step in order to reflect this pattern. First, we append the closing parenthesis of the Method Cascade to the innermost closure.

``` active haskell
import Control.Monad
import Control.Applicative
import Data.Functor
import Prelude hiding ((.))
x.f = f x

data Shape a = Circle { radius :: a } deriving (Show)
instance Functor Shape where { fmap = liftM }
instance Applicative Shape where { pure = return; (<*>) = ap }
instance Monad Shape where
  object >>= remake = remake(object.radius)
  return = Circle

bind = flip(>>=)
make = return

area(Circle r) = r^2 * pi

doubleShapeArea(r) = make(r * sqrt(2))

circ = make(2) :: Shape(Double)

-- show
circ' =
  circ.bind(\(a) ->
    a.doubleShapeArea.bind(\(b) ->
      make(sqrt(a^2 + b^2))))

main = do {
  print(circ');
}
-- /show
```

Now we seperate the `.bind(\(...) ->` snippets and align the Method Cascade.


``` active haskell
import Control.Monad
import Control.Applicative
import Data.Functor
import Prelude hiding ((.))
x.f = f x

data Shape a = Circle { radius :: a } deriving (Show)
instance Functor Shape where { fmap = liftM }
instance Applicative Shape where { pure = return; (<*>) = ap }
instance Monad Shape where
  object >>= remake = remake(object.radius)
  return = Circle

bind = flip(>>=)
make = return

area(Circle r) = r^2 * pi

doubleShapeArea(r) = make(r * sqrt(2))

circ = make(2) :: Shape(Double)

-- show
circ' =
  circ              .bind(\(a) ->
  a.doubleShapeArea .bind(\(b) ->
  make(sqrt(a^2 + b^2))))

main = do {
  print(circ');
}
-- /show
```

As we can see, Method Cascades that use a Programmable Fluent Interfaces introduce a variable to the scope of each chained method, in our case `a` and `b`, that are extracted from objects that implement the Fluent Interface. It's like writing `a <- circ` and `b <- a.doubleShapeArea`. Haskell uses this pattern extensively and therefore provides syntactic sugar called *do-notation* that allows for using this patterns exactly like that.

``` active haskell
import Control.Monad
import Control.Applicative
import Data.Functor
import Prelude hiding ((.))
x.f = f x

data Shape a = Circle { radius :: a } deriving (Show)
instance Functor Shape where { fmap = liftM }
instance Applicative Shape where { pure = return; (<*>) = ap }
instance Monad Shape where
  object >>= remake = remake(object.radius)
  return = Circle

area(Circle r) = r^2 * pi

doubleShapeArea(r) = return(r * sqrt(2))

circ = return(2) :: Shape(Double)

-- show
circ' = do {
  a <- circ;
  b <- a.doubleShapeArea;
  return(sqrt(a^2 + b^2));
}

main = do {
  print(circ');
}
-- /show
```

The naming of Haskell's built-in `return` (which we called `make`) reflects that Method Cascades that use a Programmable Fluent Interface are powerful enough to model computations.
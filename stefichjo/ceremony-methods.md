# Imperative OOP Ceremony: Methods

## Functions

Let's write our first function. It simply substracts `1` from its argument. (Please don't mind the `'` after `return` for now.)

``` active haskell
-- show
import Prelude hiding ((.))
x.f = f(x)
-- /show
apply(f) = \(x) -> f(x)
return'(a) = a
-- show

function(x) = do {
  return'(x - 1);
}

main = do {
  print(function(42));
}
-- /show
```

## Anonymous Functions

We can also define a function declaring its parameter *after* the equals sign. (A right-hand side argument declaration is surrounded by `\... ->`.) By doing so the expression on the right-hand side of our function declaration becomes a valid function on its own, which can be used inline, i.e. without an identifier.

``` active haskell
import Prelude hiding ((.))
x.f = f(x)
apply(f) = \(x) -> f(x)
return'(a) = a
-- show

function = \(x) -> do {
  return'(x - 1);
}

main = do {
  print(function(42));
  print(42.apply(function));
  print(42.apply(\(x) -> do {return'(x - 1);}));
}
-- /show
```

An anonymous function is a *lambda expression* in FP lingo. (The `\` is supposed to resemble a `λ`.)

## Closures

A closure is a partially applicable function, i.e. a function that accepts a single argument and returns a function, which accepts a single argument and returns a value. (The nesting can be as deep as you like.)

``` active haskell
import Prelude hiding ((.))
x.f = f(x)
apply(f) = \(x) -> f(x)
return'(a) = a
-- show
closure = \(a) -> do {
  return'(
    \(b) -> do {
      return'(b - a);
    }
  );
}

partiallyAppliedClosure(x) = do {
  return'(x.closure(1));
}

main = do {
  print(partiallyAppliedClosure(42));
  print(42.apply(partiallyAppliedClosure));
  print(42.closure(1));
}
-- /show
```

A closure can be made a function that accepts a tuple of arguments and vice versa. This process is called "(un-)currying".

``` active haskell
import Prelude hiding ((.))
x.f = f(x)
apply(f) = \(x) -> f(x)
return'(a) = a
-- show
closure = \(a) -> do {
  return'(
    \(b) -> do {
      return'(b - a);
    }
  );
}
closure' = do { return'(uncurry(closure)); }

function(x, y) = do {
  return'(y - 1);
}
function' = do { return'(curry(function)); }

main = do {
  print(closure'(1, 42));
  print(42.function'(1));
}
-- /show
```

## Operators

At the end of the last tutorial we saw that `x.apply(f)` can be written as ``f `apply` x``. The reason for that is that `apply` is a closure with two arguments, a so-called *operator*. You normally see operators like `-` written between its two arguments like in `42 - 1`. In order to use an operator written with alphanumeric characters like `closure` in infix notation, we have to surround it in backticks: ``1 `closure` 42``. In return, we need to surround `-` in parenthesis in order to treat it like an ordinary closure, i.e. `1.(-)(42)`.

``` haskell
42.closure(1)   -- -> 41
1 `closure` 42  -- -> 41
42 - 1          -- -> 41
1.(-)(42)       -- -> 41
```

As we can guess, `closure` *is* `-` but with its arguments flipped. Since `closure` is alphanumeric and `-` is not, this makes `closure` the alphanumeric equivalent of `-`, provided that we will use an alphanumeric operator like `closure` in object-oriented notation and a non-alphanumeric operator like `-` in infix notation.

``` haskell
42.closure(1)  -- -> 41
42 - 1         -- -> 41
```

Let's give `closure` a propper name, `sub`, and define it in terms of `-`. As we have seen, `sub` *is* `-` but with its arguments flipped, so:


``` haskell
sub = do {
  return'(flip((-)));
}

42 - 1     -- -> 41
42.sub(1)  -- -> 41
```

## Methods

As we have seen in the previous tutorial, we can write a function that takes one argument in object-oriented notation, too. This is the very definition of our `.` operator: `x.f = f(x)`. As we are parenthesis-phile, we introduced `apply` which is yet another convoluted way of function application: ``f `apply` x = f(x)``. Together, `.` and `apply` form *inversion of control*.

``` haskell
x.apply(f) == f(x)
```

It is interesting to note that `f(x)` has parenthesis, whereas `x.f` doesn't. This is not a necessity, but a choice. We could define `f()` which looks nice in `x.f()`, but looks awkward in `f()(x)`. Therefore, we will avoid empty parenthesis from now on.

``` haskell
decrement() = \(x) -> do {
  return'(x.sub(1));
}

dec = \(x) -> do {
  return'(x.sub(1));
}

42.sub(1)       -- -> 41
42.decrement()  -- -> 41 :(
42.dec          -- -> 41 :)
```

## Function Composition

You may have noticed, that `sub = do { return'(flip((-))); }` doesn't contain any variable. This style is called "pointfree" in FP lingo. (We could have defined `sub` as ``b `sub` a = do { return'(a - b); }``, too.)

When define functions in terms of other functions, we often concatenate two functions into one. This is called *function composition*. For example, given the functions `even`, which returns `True` or `False` depending on the argument being even or not, and the function `not`, which inverts `True` and `False`, we can easily define a function `odd`, both in pointfull and pointfree style:

``` active haskell
import Prelude hiding ((.))
x.f = f(x)
return'(a) = a
-- show
f `compose` g = \(x) -> do { return'(f(g(x))); }

oddPointfull(x) = do { return'(not(even(x))); }
oddPointfree = do { return'(not `compose` even); }

main = do {
  print(42.oddPointfull);
  print(42.oddPointfree);
}
-- /show
```

Haskell already has a compose operator of its own, i.e. `.`, which we hid from the `Prelude` in the previous tutorial in order to define object-oriented notation. This is why we just had to define our own `compose` operator.

## Constants

Let's define a constant.

``` active haskell
import Prelude hiding ((.))
x.f = f(x)
-- show
constant = 42

main = do {
  print(constant);
}
-- /show
```

We may have written `constant = do { return'(42); }`, but this doesn't make sense, since `constant` is a constant, and not a function, right? Well, in Haskell there is virtually no difference between a constant and the return value of a function, as the same argument applied to the same function will always yield the same return value. In FP lingo, functions are *referentially transparent*. (This has the benefit of return values being cachable - they can be *memoized* in FP lingo.) Therefore, we can treat functions just like constants and ommit the `do { return'( ... ); }` ceremony.

``` active haskell
import Prelude hiding ((.))
x.f = f(x)
-- show
f `compose` g = \(x) -> f(g(x))

oddPointfull(x) = not(even(x))
oddPointfree = not `compose` even

main = do {
  print(42.oddPointfull);
  print(42.oddPointfree);
}
-- /show
```
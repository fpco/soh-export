# Imperative OOP Ceremony: Parenthesis

## Parenthesis

Parenthesis are used for forming nested expressions, when necessary.

``` haskell
(20 + 1) * 2  -- -> 42
 20 + 1  * 2  -- -> 22
(42) == 42    -- -> True
```

Parenthesis around singleton arguments (or *unary tuples* in FP lingo) are superfluous, which is why they are ommitted when coding in idiomatic Haskell.


``` active haskell
import Data.Char

upperA = toUpper 'a'

main = do {
  print(toUpper('a'));
  print upperA;
}
```

## Void

Haskell has a `Void` type (FP lingo: *Unit*), which is a nullary tuple, `()`, whose only value is `()` (`void`, if you will). Defining functions with `()` as an argument looks nice, but would be quite a pointless ceremony.

``` active haskell
pointless() = "Hello World!"

main = do {
  print(pointless());
}
```

## Argument Vectors

An argument vector is a group of values being passed to a function. (Parenthesis are mandatory for argument vectors.)

``` active haskell
import Prelude hiding ((.))
x.f = f x
-- show
printPair(first, second) = do {
  print first;
  print second;
}

main = do {
  printPair("Hello World!", 42);
}
-- /show
```

In Haskell, argument vectors are values, too. An argument vector holding 2 elements is a pair, one holding 3 elements is a triple, and so on. The values of a pair can be easily extracted with `fst` and `snd`.

``` active haskell
import Prelude hiding ((.))
x.f = f x
-- show
printPair p = do {
  print(p.fst);
  print(p.snd);
}

pair = ("Hello World!", 42)

main = do {
  printPair("Hello World!", 42);
  printPair pair;
}
-- /show
```

Since functions in Haskell are "closurized" by default (FP lingo: *curried*), argument vectors are not being used as such. Instead, they are primitive data structures, a poor man's object, if you will. Therefore, argument vectors are being referred to as mere *tuples* in FP lingo.

``` active haskell
import Prelude hiding ((.))
x.f = f x
-- show
printPair(first, second) = do {
  print first;
  print second;
}

printWith = curry printPair

main = do {
  42.printWith "Hello World!";
}
-- /show
```

As we can see, method notation doesn't require for parenthesis either. (We wrote `42.printWith "Hello World!"`.) When we don't write in OOP notation, we simply put the argument on the left side of the dot operator to the end of our operator expression: `printWith "Hello World!" 42` (or ``"Hello World!" `printWith` 42``), which is the idiomatic way of writing Haskell code. (We could also write `(-) 42 1` instead of `42 - 1`.)

``` active haskell
import Prelude hiding ((.))
x.f = f x
printPair(first, second) = do {
  print first;
  print second;
}

printWith = curry printPair

-- show
main = do {
  42.printWith "Hello World!";
  printWith "Hello World!" 42;
}
-- /show
```

If we wanted to use parenthesis in order to show the order in which (partial) functions apply, they would stick to the left, because `printWith "Hello World!"` is a partially applied closure which takes `42` as its last argument. (In FP lingo, *function application is left associative*.)

``` haskell
printWith "Hello World!" 42 == (printWith "Hello World!") 42
(-) 42 1 == ((-) 42) 1  -- -> 41
```

In the next tutorial we shall finally see some objects. :)
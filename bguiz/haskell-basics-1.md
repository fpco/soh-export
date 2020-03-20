# Haskell Basics #1

## Simple arithmetic

``` active haskell
x = 18 - 7
main = print x
```

Gotcha with negative numbers - the follwing won't work,

``` active haskell
x = 10 * -5
main = print x
```

But this will

``` active haskell
x = 10 * (-5)
main = print x
```
## Boolean logic

``` active haskell
x = True && False
y = False || True
z = 7 == (10 - 3)
main = print (x, y)
```

Notice here that the print statement printed a single value,
but that value was a a pair of parentheses,
containing a comma separated list of values,
in this case, `x`, `y`, and `z`.

This is called a pattern. More on this later.

Comparing different types

``` active haskell
x = "derp" == 1337
main = print x
```
``` active haskell
x = "derp" == "herp"
main = print x
```
As expected, the latter works, but the former fails to compile.

That is because, the Haskell compiler does not know how to compare
(or add, or subtract) a number with a string.

## Functions are values

All the values so far are actually Functions

``` haskell
x = 25
```

Is actually defining a function `x`,
which takes no parameters, and always evaluates to 5.

(Note that I say "evaluates to", not "returns".)

Functions with parameters

``` active haskell
x a b = a + b
main = print $ x 3 4
```

This defines a function `x`,
which takes in two parameters, `a` and `b`,
and evaluates it to the sum of `a` and `b`.

``` active haskell
x (a, b) = a + b
main = print $ x 3 4
```

Notice that in the first definition,
there were no parentheses surrounding `x`'s only parameter -
they are simply not required in Haskell, 
unlike most other programming languages.

This one does not compile, because
this defines a function `x`
which takes in just one parameter, `(a, b)`,
which happens to be a tuple containing two values.

``` active haskell
x (a, b) = a + b
main = print $ x (3, 4)
```

Thus, this would be the correct way to invoke it.



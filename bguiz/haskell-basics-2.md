# Haskell Basics #2

## Calling functions

In Haskell, functions are called using the function name, followed by parameters, all seperated by spaces.

In C/ C++/ Java

``` c++
function fn(x, y, z)
```

In Haskell

``` haskell
fn x y z
```

Notice that the Haskell syntax is a lot more terse than that of C, C++, and Java.

## Calling a function on the result of another function

If you wish to chain functions together, 
there are a couple of different ways you would usually do this

```  active haskell
fn a = a * a
gn a = a + a
x a = fn $ gn a
y a = fn (gn a)
main = print $ (x 5, y 5)
```

The function `fn` returns the square if its input.
The function `gn` returns its input added to itself (or its input times two).
The function `x` calls `gn` on its input,
and takes the result from that and calls `fn` on it.
The function `y` does exactly the same thing as `x`,
merely expressed with a different syntax.

The `$` operator can be thought of as equivalent to
surrounding the remainder of the line (to its right)
in parentheses.

```  active haskell
x = 5
main = print $ x == (x)
```

Note that a tuple with only one value within it
evaluates to that value on its own,
hence why the above works.

## Haskell's baked in functions

Haskell comes with a load of built in functions.
Let's practice calling some of them for fun,
and chaining them together, as we have above.

``` active haskell
min3 a b c = min a $ min b c
max3 a b c = max a (max b c)
main = print $ (min3 10 30 (-3), max3 (-1) (-20) (-99))
```

Haskell's `min` and `max` functions do exactly what
you would think that they do -
find the minimum and maximum of the two inputs given to it.

I have built on top of these, functions that find the minimum of
three inputs instead of two.
If you pay close attention,  you will notice that
I have chained one function call to another 
in the same manner as I did previously with `fn` and `gn`.

##Conditionals

So let us make a function that computes the distance
of the diagonal edge of a right angled triangle:

``` active haskell
pythag a b = sqrt $ a * a + b * b 
main = print $ pythag 3 4
```

We make use of `sqrt` to compute the square root of a number.

What if we decide that triangles cannot have negative lengths for their edges? 

Here, there are three new things in play:
- `if .. then .. else` syntax
  - The basic conditional execution syntax
- `do` block syntax
  - Syntax used in Haskell to denote a sequence of actions
- `error` syntax
  - Used to show that something has gone wrong

The `error` is used to disallow arguments which we consider to be illegal
from being used in this function.
The `do` block is used here, because we want two separate print statements.
Previously, we have simply printed a `tuple`, with one value for each output.
However, doing so means that all outputs need to evaluate before any of it
may print. Separating this into two print statements allows any intermediate
output to be printed, prior to attempting to evaluate the next expression. 

``` active haskell
pythag a b = if a < 0 || b < 0
  then error "Lengths are not allowed to be negative."
  else sqrt $ a * a + b * b 
main = do
  print $ pythag 3 4
  print $ pythag (-3) 4
```

Here the function evaluates to an if statement.
If either of the inputs are negative, error is raised.
(Note that this is not the best example,
let us stay focussed on the `if` for now)
Otherwise, the result is computed as before.

Haskell provides some syntactic sugar,
called `guards`, which allows us to  state the above
in a more concise manner.

``` active haskell
pythag a b 
  | a < 0 || b < 0 = error "Lengths are not allowed to be negative."
  | otherwise      = sqrt $ a * a + b * b 
main = do
  print $ pythag 3 4
  print $ pythag (-3) 4
```

Note how the if statement takes precedence over the
rest of the function.
It is as if it is acting as the contoller for the rest
of the evaluation of the function.
That is because it is.
Haskell is effectively defining two different possible
definitions for the body of this function,
based on its inputs matching a set of conditions.

Let us examine how we would write the equivalent
in C/ C++/ Java

``` c++
function pythag(int a, int b)
{
  if (a < 0 || b < 0)
  {
    throw Exc("Lengths are not allowed to be negative.");
  }
  else
  {
    return sqrt(a * a, b * b);
  }
} 
```

(Assume we have `typedef`ed `Exc` to be the exceptions object we wish to throw)

While this might look very similar,
it exposes a very fundamental difference.

In C/ C++/ Java, however, the function does not evaluate in this sense.
Instead it depends on a a `return` statement to define the possible exit points.

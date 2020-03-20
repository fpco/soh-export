# 2. My First Program

Since the first tutorial was about function application, this one should be about function definition. But defining functions is pretty much all you do in Haskell; that, and defining data structures and types. So I'll just show you the basic syntax, enough to write some simple programs and to understand the code behind the examples from the previous tutorial.

## Function Definition Syntax

The basic syntax of a function definition is pretty simple: function name followed by argument names, then an equal sign, and an expression. For instance, here's the function that squares its argument:
``` active haskell
-- show This is a function definition
sq x = x * x
-- /show
main = print $ 
-- show This is a function call (try editing the argument)
    sq 12
-- /show
```
As with function application, the syntax is very terse: no parentheses, no commas. So if you come up against something like this:
``` haskell
a b c d =
```
you are looking at a definition of a function `a` that takes three arguments, `b`, `c`, and `d` (not that I'm advocating the use of meaningless single letter names).

Here's another function we used in the first tutorial:
``` active haskell
-- show
pyth a b = a * a + b * b
-- /show
main = print $
-- show
    pyth 3 4
-- /show
```
This is probably a good place to mention that function and variable names in Haskell always start with a _lowercase_ letter. We'll also see some uppercase names soon.

## The Main Function

Your program needs to have one function called `main`, which is its entry point. That's where the execution starts and ends. In Haskell, `main` is special in other ways as well -- I'll talk about it more in the next tutorial. For now, it's enough to know that you can do I/O from `main` and that the syntax for doing I/O is slightly different than for regular functions. However, if all your I/O is just a single call to `print`, you can just call it from `main`. Here's a complete example -- the code from the previous snippet:
``` active haskell
pyth a b = a * a + b * b
main = print $ pyth 3 4
```
This is enough I/O to get you going for the time being.

## Function Definition is Equation

The use of the equal sign in function definition is not accidental. The left hand side _is equal_ to the right hand side. What does it mean? It means that you can substitute the function call with the expression that is the right hand side of the function definition (and vice versa). When doing this substitution you have to replace the formal parameters of the function with the actual arguments. If the arguments are expression, you just stick these expressions into the body of the function (make sure there are no name clashes -- if there are, rename variables). For instance, this code:
``` active haskell
pyth a b = a * a + b * b
main = print $ pyth (1 + 2) (4/2)
```
is equivalent to:
``` active haskell
main = print $ (1 + 2) * (1 + 2) + (4/2) * (4/2)
```
Whatever evaluation strategy the compiler or the runtime picks, they will never break this property. In fact you can use it to prove the correctness of your Haskell program with what is called _equational reasoning_. This process is so straightforward that it can often be automated with _theorem provers_ that take Haskell programs as input. I'm not going to get into details here, but I just wanted to point out the reason why Haskell is the preferred language for writing high reliability software, as in financial programs that may lose millions of dollars if they are buggy.

This property of a function definition makes it almost look like a macro, except that it's a very safe and smart macro, not a dumb textual substitution.


## Pattern Matching

Pattern matching plays an important role in Haskell, so I'll give you a little taste of it based on some simple data structures built on tuples. Tuples are these amorphous data structures that let you combine other data in neat packages. The simplest tuple is a pair. Here's a pair of numbers:
``` haskell
(1, -1)
```
You may use it, for instance, to represent a 2-D vector. One of the exercise in the previous tutorial used pairs this way. 

``` active haskell
pyth (a, b) = a * a + b * b
main = print $ pyth (3 * 2, pyth (-1, 8))
```
Function `pyth` takes a pair as an argument. It needs to extract two numbers from it in order to evaluate the result. It does it by pattern matching: `(a, b)` is a _pattern_. You can replace any formal parameter on the left hand side of a function definition with a pattern. 

When you call `pyth` with a pair, this pair is matched with the pattern `(a, b)` and `a` and `b` take up the values that were used when the pair was created. For instance, in this program:
``` active haskell
pyth (a, b) = a * a + b * b
main = print $ pyth (1, -1)
```
we call `pyth` with a pair `(1, -1)` so `a` will take the value 1 and `b` will take -1.

Here's an example where a pair is passed around and then matched:
``` active haskell
pyth (a, b) = a * a + b * b
len vec = sqrt (pyth vec)
main = print $ len (12.6, -3.21)
```
Function `len` doesn't need to unpack its pair argument `vec` because it passes it straight to `pyth`. It is `pyth` that needs to extract 12.6 and -3.21 from it.

Pairs may contain other pairs and pattern matching can get arbitrarily sophisticated. For instance, a segment is a pair of points. Function `lenSeg` calculates the length of a segment:

``` active haskell
pyth (a, b) = a * a + b * b
lenSeg ((x, y), (x', y')) = sqrt $ pyth (x' - x, y' - y)
main = print $ lenSeg ((1, 0.5), (-1, -0.5))
```
Here we have one pattern `((x, y), (x', y'))` that not only matches a pair but also matches its constituent pairs. It extract four values, `x`, `y`, `x'`, and `y'` in one fell swoop. 

So far we've seen homogenous pairs -- both constituents being of the same type. (Although we haven't discussed types yet, you instinctively know that a number like 3.14 has a different type than a pair of numbers, `(1, 0.5)`, or a string `"Hello!"`). They can all be combined into one non-homogenous tuple, like:
``` haskell
(3.14, (1, 0.5), "Hello!")
```
and they can be pattern matched, as in:
``` active haskell
getThePoint (a, (x, y), str) = (x, y)
main = print $ getThePoint (3.14, (1, 0.5), "Hello!")
```
As you can see, a tuple can be returned from a function and printed with `print`.

There is special syntax for a wildcard: the underscore `_` that matches anything (and discards it). With that, the function above can be simplified to:
``` active haskell
getThePoint (_, pt, _) = pt
main = print $ getThePoint (2 * 3.14 * 10, (1, 0.5), "Hello!")
```
Since we don't need to match individual components of the point inside `getThePoint`, matching the whole pair with `pt` is all we need. However with this change there is no longer any constraint on the type of `pt` and you can call `getThePoint` with `(3.14, "Point", "Hello!")` and it will work just fine (try it!). We will talk about polymorphism (that's what it's called) much more in the future.

## Exercises

1.  Define a function `flop` that reverses a pair.
    ``` active haskell
    flop ? = ?
    main = print $ flop (1, "one")
    ```
    @@@ Show solution
    ``` active haskell
    flop (a, b) = (b, a)
    main = print $ flop (1, "one")
    ```
    @@@
2.  Define a function that takes two points (pairs) and returns a segment (a pair of points).
    ``` active haskell
    makeSegment ? = ?
    main = print $ makeSegment (1, 2) (3, 4)
    ```
    @@@ Show solution
    ``` active haskell
    makeSegment p1 p2 = (p1, p2)
    main = print $ makeSegment (1, 2) (3, 4)
    ```
    @@@
3.  Define a function that takes a segment and returns it's center point.
    ``` active haskell
    center ? = ?
    main = print $ center ((1,2), (3, 4))
    ```
    @@@ Show solution
    ``` active haskell
    center ((x, y), (x', y')) = ((x + x')/2, (y + y')/2)
    main = print $ center ((1,2), (3, 4))
    ```
    @@@

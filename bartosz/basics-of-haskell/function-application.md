# 1. Calling functions

[Next](2-my-first-program)


Haskell is a functional language, so function calls and function definitions form a major part of any Haskell program. That's why the syntax for those two constructs is reduced to a bare minimum. Admittedly, this makes Haskell programs hard to read for newcomers. This tutorial deals with the function call syntax  and the next one will cover function definition syntax. Normally, I don't like starting a language course with syntax, but in my experience the "sink or swim" approach of learning basic syntax on the go while solving programming exercises doesn't work very well with Haskell.

## Function Application

What does it mean "bare minimum"? It means that _any series of identifiers is a function call_ or, as we often call it, a _function application_. This:

``` haskell
    a b c d
```

is an application of a function `a` to three arguments `b`, `c`, and `d`. That's it! There is no other interpretation of it. If `a` is not a function, this is not valid Haskell. 

Try it! First run this code (click the `Run` button):

``` active haskell
a b c d =
  "Function a called with arguments " ++ b ++ " " ++ c ++ " " ++ d

b = "b"
c = "c"
d = "d"

main = putStrLn $
-- show (program fragment, to see the rest, press the "Show All" button)
	a b c d
-- /show
```
Then try to change the order of identifiers, so that `b` is the first identifier (just edit the code in place and run it again). The compiler will try to interpret `b` as a function and it will fail because I made `b` a string (which is the synonym for `[Char]`, as we'll see later). 

You may parenthesize function application if you need to. Those two are equivalent:

``` haskell
    f a b
```

and

``` haskell
    (f a b)
```

But you should definitely unlearn the traditional interpretation of this syntax:

``` haskell
    f (a, b)
```

This is still valid Haskell, provided `f` is a function that takes a pair, `(a, b)` (also called a _binary tuple_) as an argument. 

You'll see the full impact of the simplified function syntax when we talk about currying. 

## Precedence

Another important thing to know is that function application has the strongest binding power. I'll demonstrate it in a simple example, but first let's talk a little about arithmetics. No weirdness there: Haskell defines a bunch of infix operators like `+`, `-`, `*`, `/`, etc. It also has numbers, both integral and floating-point.

So how would you interpret this?

``` haskell
    a b * c d
```

As I said, function application has the highest precedence, so this code is equivalent to:

``` haskell
    (a b) * (c d)
```

It is _not_ equivalent to:

``` haskell
    a (b * c) d
```

The former says: apply function `a` to `b` and multiply the result by function `c` applied to `d`. The latter is function `a` applied to two arguments, `b * c` and `d`. (In production code, identifiers usually have more meaningful names. Here, I want to show you pure syntax.)

Can you predict the result of the following code? Here `sq` is a function that squares its argument.

``` active haskell
sq b = b * b
main = print $
-- show
	sq 3+1 
-- /show
```
(If you're surprised by the result, read about precedence again.)

There is one case where precedence rules may be confusing: the unary minus. If you want to apply a function `f` to negative one, you have to use parentheses, as in:

``` haskell
    f (-1)
```

If instead you write:

``` haskell
    f -1
```

the compiler will try to subtract one from `f` -- not a good idea if `f` is a function. But if you make such a mistake, the compiler will easily catch it. This will become obvious when we talk about types.

One more thing: If you knew that `f` and `g` were functions, how would you interpret this?

``` haskell
    f g x
```

According to the rules I just described, `f` must be a function and `g` and `x` its arguments. Yes, a function may take another function as an argument. If you wanted `g` to act on `x` instead, you'd use parentheses:

``` haskell
    f (g x)
```

## Dollar and Dot Notation

When we talked about precedence I said that function application has higher precedence than all other operators. But function application is not really an operator. Or is it? If you stretch your imagination, you may think of the whitespace between the function name and the argument as an operator. This is not strictly true, because in some cases you may elide the space if the parse is unique (try removing all spaces from the solution to Exercise 1 at the bottom of this tutorial). When you learn about currying, you'll see that spaces between arguments play exactly the same role as the space between the function and the argument. 

This "operator whitespace" has the highest precedence and binds to the left. Because of left binding, the application of multiple functions requires multiple parentheses. For instance, if you want to square the square root of a number, you write:

``` active haskell
sq x = x * x
main = print $
-- show
	sq (sqrt (7 + 9)) 
-- /show
```

There is, however, another operator (this time a _real_ operator) for function application, denoted by the dollar sign `$`. Even though it serves the same purpose, in a way it's the opposite of the "operator whitespace." It has the _lowest_ precedence and binds to the right. Because of that, the example above can be rewritten without parentheses:

``` active haskell
sq x = x * x
main = print $
-- show
	sq $ sqrt $ 7 + 9
-- /show
```

Because `$` binds to the right, the square root will be executed first; and because it has the lowest precedence, the addition will be performed _before_ function application. 

Applying a function to the result of another function is called _function composition_ and has its own operator, the dot, `.`. This operator has very high precedence, surpassed only by by that of function application. 

The composition of `sq` with `sqrt` may be written as `sq . sqrt`. This new function, when acting on a number will first take its square root and then square the result, so this will work too:

``` active haskell
sq x = x * x
main = print $
-- show
	(sq . sqrt) $ 7 + 9
-- /show
```

This particular composition is not very interesting, since it doesn't do anything to a number except of possibly losing precision. The function that really does nothing is called the _identity_, `id`. Composing identity with any function doesn't change the behavior of that function. Try it:

``` active haskell
sq x = x * x
main = print $
-- show
	(sqrt . id) 256
-- /show
```

## Conclusion

Function syntax in Haskell might seem weird at first. But consider that Haskell programs are built from functions. In particular, function application is one of the most common operations. Its syntax _should_ be the tersest; just as the most common letter in English, 'e', is encoded as a single 'dot' in the Morse alphabet.

In the next tutorial we'll learn how to _define_ a function in Haskell.

## Exercises

1.  I defined a function `pyth` that takes two numbers and returns the sum of their squares. Add parentheses to the code below to make it compile (don't get scared by unintellegible error messages):

    ``` active haskell
    pyth a b = a * a + b * b
    main = print $
    -- show
        pyth 3 * 2 pyth -1 8
    -- /show
    ```
    @@@ Show answer

    ``` active haskell
    pyth a b = a * a + b * b
    main = print $
    -- show
    	pyth (3 * 2) (pyth (-1) 8)
    -- /show
    ```
    @@@

2.  I also defined a function `pyth'` (yes, you can use apostrophes -- or 'primes', as they are called in mathematics -- in identifiers). `pyth'` takes a tuple of numbers, as in `(a, b)`, and returns the sum of their squares. Add parentheses and commas to code below to make it compile. You may reduce the number of parentheses if you take into account that the comma inside a tuple has lower precedence than arithmetic operators.

    ``` active haskell
    pyth' (a, b) = a * a + b * b
    main = print $
    -- show
        pyth' 3 * 2 pyth' -1 8
    -- /show
    ```
    @@@ Show answer


    ``` active haskell
    pyth' (a, b) = a * a + b * b
    main = print $
    -- show
        pyth' (3 * 2, pyth' (-1, 8))
    -- /show
    ```
    @@@
3.  The `print` function prints its argument, as long as it is convertible to a string. Numbers are convertible to strings. The code below works but looks more like Lisp than Haskell. Try to remove as many parentheses as you can using `$` signs (Hint: With some cleverness, you can get rid of them all).

    ``` active haskell
    pyth a b = a * a + b * b
    main = do
    -- show
        print (sqrt (pyth 3 ((-1) - 3)))
    -- /show
    ```

    @@@ Show answer

    ``` active haskell
    pyth a b = a * a + b * b
    main = 
    -- show
        print $ sqrt $ pyth 3 $ -1 -3
    -- /show
    ```
    @@@





# Root Finding

# What is a root?
A root is a mathematical idea.
Suppose you have a real-valued function. For now, we will take the special case where the function goes from real numbers to real numbers:

``` haskell
f :: Real -> Real
```

Then, a root of f (`root f`) has the following property:

``` haskell
f (root f) = 0
```
For instance, the function 

``` haskell
f x = 1 - x
```
has a root at `x=1`.

so, 

``` haskell

root f = 1
--therefore,
f (root f) = f 1 = 1 - 1 = 0
```

# What is a numerical root?

In a computer, we can only hope to approximate a mathematical root.

In other words, a numerical root of a function f, is a function that takes a tolerance, and gives an approximation to the actual root.

``` haskell

numericalRoot f epsilon = ...
root f = ...
```

because of this restriction, we can only approximate the roots of certain kinds of functions. (for instance, if the function is not continuous, there isn't much we can do)

Also, techniques that work for all continuous functions are in practice much slower that some more advanced techniques, but those advanced techniques require at least that the function be derivable, but also, some special cases will make the method fail.

In a system where the need for speed and correctness is very important, it will often be necessary to build some extra measures to detect failure, and allow the algorithm to decide on the fly which technique to use for a given confidence level. 

We might explore the design of such systems (rather easy to do in Haskell, but pretty messy in, say, C/C++) but for now, we assume that the physicist can visualize the function he is working with, and can manually adapt the code to avoid failure. This assumption becomes absurd in large systems, but you can get a very long way (sometime, a whole career) without having to write such a large system. If you are interested, hold your breath; we will come to those eventually, albeit not necessarily in this course.

# Why should you care?

The ability of finding roots of functions of type `Real -> Real` gives you immense power. 

For instance, you can numerically invert a function in this way (without knowing even the explicit expression for the function). Here is how:

``` haskell
f x = ...

inverseOfF y = root (\x -> f x - y)
```

You can also find extremums of your function. Here is how

``` haskell

f x = ... 
f' = derivative f

extremum f = root (derivative f)
```

There are all sorts of applications, which we'll come to.

# Methods of finding roots.

##Bisection
Consider this simple fact about continuous functions:

Suppose `f` has one and only one root in the interval (a,b), then

``` haskell
f a * f b < 0
```
In other words, if you cross the zero line once while going from a to b, then the sign has changed between a and b. (If you don't see why it is true, make a picture)

The idea of this method is to decide in which sub-interval {(a,middle) or (middle, b)} is the root. The above fact is the test we use. 


Introducing `epsilon` as an absolute tolerance, we can write the full deal:

``` haskell
bisection epsilon f a b = let av = (b + a)/2 
                               --the middle of the interval.
                               fa = f a 
                               --the function at a
                               fav = f av 
                               --the function at av
                           in  if (b-a < epsilon) 
                               --this means we are already close enough. 
                               --In this case, we choose the less biased 
                               --guess, which is av.
                               then av
                               else if (fa * fav < 0)
                                    --this means the root is in the 
                                    --interval (a,av)
                                    then bisection epsilon f a av
                                    else if (fav == 0) 
                                    --this is almost impossible if your 
                                    --initial intervals don't have any 
                                    --relationship to the root, but in 
                                    --practice, it happens.
                                            then av
                                            else bisection epsilon f av b 
                                            --at this point, we know that
                                            --the root is in the interval (av,b)
```

Notice that replacing `b-a < epsilon` by `(b-a)/a < epsilon` shifts epsilon from an absolute error to a relative one.

##Newton-Raphson Method

This method uses the derivative, and basically, it improves the current guess until the tolerance is satisfied.

``` haskell
newton epsilon f f' guess = let newGuess = guess - (f guess / f' guess)
                                err =  abs (newGuess - guess)
                            in if (err < epsilon)
                                  then newGuess
                                  else newton epsilon f f' newGuess

```
`epsilon` is the tolerance
`f` is the function we want to find the root of.
`f'` is the derivative of f. Nothing actually enforces this, but the method works only if f' is indeed the derivative of f. This gives you the freedom to calculate the derivative of f in any way that works.
`guess` is a guess for the root `x_0`

### Note: Fixed-Point Methods.
@@@
This is a fixed-point method; this means that we can define a function:

``` haskell
towardRoot x = x - f x / f' x
```

and this function is such that (if some conditions are satisfied) `towardRoot guess` is closer to the true root than `guess`.

Exercise: rewrite newton in terms of fixedPoint. 

Hint: fixedPoint can be written like this:

``` haskell
fixedPoint towardRoot errorEstimator guess = ...
```

or 
``` haskell
fixedPoint towardRoot epsilon guess = ...
```
@@@



##Secant Method

Here it is:
``` haskell
secant epsilon f guess1 guess0 = let 
    newGuess = guess1 - f guess1 * (guess1 - guess0) / (f guess1 - f guess0)
    err =  abs (newGuess - guess1)
    in if (err < epsilon)
       then newGuess
       else secant epsilon f newGuess guess1 


-- Basically, this is the same as Newton's method, 
-- with the following modification:

f' guess -> ( f guess1 - f guess0 ) / (guess1 - guess0)

```

The reason these methods (newton, and sequent) work is that the algorithm follows the slope of the function as if it were the function itself. As you can see, there are two ways of doing this: the first is to look at the local slope. This method (instead of newton's) could have been called the tangent method. The second way is to take a slope passing through two points of the curve (that is called a secant). The slope you get this way is not exactly local.

When these methods work, they are pretty fast. (if the function is linear, it takes only one step) However, these methods can fail spectacularly if they compute a slope  very close to 0 at some point. (try to see what happens then)

You don't have to know in what interval your root is with these *slope methods*, but they will fail if there are multiple roots.

If you happen to know an interval (more generally, a subset of the real numbers) where your root is, then you can make these methods work even with slopes close to 0 and multiple roots. 


We will come back to these methods when we come to "Making reliable tools".

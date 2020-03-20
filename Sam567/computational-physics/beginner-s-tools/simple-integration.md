# Simple Integration

For the sake of smoothness :), we start by 'formalizing' the notion of definite integration. If you haven't seen my previous tutorial on 'Representation, Operations, and Semantics', be aware that this 'formalization' is an application of the concepts discussed in 'ROS'.

The idea is to take you through my own journey in Computational Physics. In this specific tutorial, you will see how I took the concepts from a Calculus 101 course, and turned it into a 'Computational Idea', which I then implemented in Haskell.

You do not have to do this in Haskell, but I recommend it. Even if you plan to work in 'C' or 'FORTRAN', I believe (that is the reason we are here) Haskell can teach you more about 'Computational Ideas' and their relationship to 'Mathematical Ideas' (and later, we will discuss 'Physical Ideas') than any other languages out there. Also, Haskell allows you to write efficient programs, which is important in Computational Physics. Furthermore, I will teach you (if you keep coming back) what optimization is, and how to do it smartly.

But enough blabla, let's dive in.

#From Exact to Approximate Integration

Let us consider definite integrals.

``` haskell
integral f a b

```

How can we make sense of this concept in a 'computational' manner?

In mathematics, `integral`  is a function that takes a function as input, an interval, and returns the area under the curve. 

In a computational context, `integral` is likely to be an `Operation` rather than a `Representation`. 

So, our task is to find a 'computational idea' where `integral` is an operation.

Exercise: How would *you* solve this problem? How would you represent real numbers? How would you represent a function from the real to the real? How would you represent an interval?

###Solution: Two for the price of One
@@@
These are two representations you might have come up with:

1) if we represent mathematical functions `f:: Real -> Real` as haskell functions `f::Double -> Double`, and every number `x::Real` by a haskell number `x::Double`, then `Integral` is a function with the following type:

``` haskell

integral :: (Double -> Double) -> Double -> Double -> Double

```

By introducing type synonyms, we really start to get a handle on the fundamental notion here:

``` haskell

type Func = Double -> Double

integral :: Func -> Double -> Double -> Double

--If we want to emphasize the fact that a and b form an interval, then we can do this:

type Interval = (Double,Double)

integral :: Func -> Interval -> Double
integral f (a,b) = ...

```

So the integral is an `Operation` which takes a function, an interval, and returns a real number.


2) if we represent a mathematical function `f` by the list of values which it takes on the interval (a,b), and we represent the independent variable `x` by the values for which the function is evaluated, then `Integral` is a function with the following type:

``` haskell 

integral :: [Double] -> [Double] -> Double
integral fs xs = ...

--with the understanding that if f is the mathematical function, then 

fs = map f xs

```

With a few type synonyms:

``` haskell

type Func = [Double]
type Var = [Double]

integral :: Func -> Var -> Double

```

@@@

###Note about higher-order Functions
@@@
In mathematics, functions on functions are called operators (linear operators in cases like `integral`). However, once you accept that functions aren't special, then the word `operator` can be replaced by `function`. In Haskell, there is nothing special about `operators`; they are simply functions, and you define them the same way as functions on data. This single fact is responsible for much of Haskell's power, and many other languages have this feature.
@@@


###Take-away from our first two tries
So you notice that these two notions of integrations are different. (I'm talking about the Solution to the first Exercise)

Clearly, the first integral really corresponds directly to the mathematical idea of an exact (definite) integral.

Given a function `f`, and an interval `(a,b)`, then there is really only one exact integral in math, and one exact integral in the first representation.

However, the second representation offers many representations of the same function. (you are free to choose how many point you use to represent your function)

for instance:

``` haskell

f x = x
a = 0
b = 1

fs1 = [0, 1]      --representing f with 2 points on the interval (a,b)
fs2 = [0, 0.5, 1] --................... 3 ...........................

xs1 = [0, 1]      -- the corresponding variables
xs2 = [0, 0.5, 1]
```
So what is essential here? 

Let us suppose that we always evaluate points at constant increments.

Now, the only freedom we have is the number of points contained on our interval.

Thus, a more useful representation than 1) for approximate integration is 

###A third representation; Now, we talk about approximate integration.

``` haskell

type Func = Double -> Double
data Interval = I (Double,Double) Int

integral :: Func -> Interval -> Double
integral f (I (a,b) n) = ...

```

This is a representation of a discrete interval, which now makes the two representations (2 and 3) equivalent.

However, it is possible to define integration without the constraint of constant increment. 

###More: The second and third representation are compatible
@@@
In the case of non-constant increments, the 3rd rep can change to the following, and become equivalent to the 2nd:

``` haskell

type Func = Double -> Double
type Interval = [Double] --This is a Var in disguise, so let us be more explicit:
type Var = [Double]

integral :: Func -> Var -> Double
integral f xs = ...

```

Incidentally, if one restricts himself to the case of constant increments, then one can modify the 2nd representation to  fit the original 3rd:

``` haskell

type Func = [Double]
data Interval = I (Double,Double) Int

integral :: Func -> Interval -> Double
integral fs (I (a,b) n) = ...
```

So we see that at its core, the approximate definite integral has a structure independent of representation.
@@@



#Simple Rules

OK, now that we really really know what an integral is, let us look at various algorithms for integration.

For the moment (we will come back when we talk about Stochastic Methods), the mathematical form of algorithms to compute integrals has the following form:

``` haskell

integral f xs = sum [f x_i * w x_i | x_i <- xs]
--or
integral' fs xs = sum $ zipWith (*) ws fs
```
However, this form doesn't tell us about `w` or `ws`. Intuitively, you can think of an integral as a sum, where the `ws` are the weights of each of the `fs`.

###Note: why we work with the 2nd rep. from now on.
@@@ 
By the way, the way to go from the 3rd rep. to the 2nd is rather obvious (`fs = map f xs`), whereas the other way around seems troublesome. Therefore, we will work with the 2nd, and then, if we want to implement something for the 3rd, we will first convert the input to the  2nd rep, and then use our implementation for the 2nd. 

Exercise: how to get from an `Interval` to a `Var` in the case of equal increments?

Solution:

``` active haskell
data Interval = I (Double,Double) Int
toXs (I (a,b) n) =map (\x -> a + (b-a)/fromIntegral (n-1) * fromIntegral x) [0..n-1]

main = print toXs (I (1,2) 10)
```
@@@

## Left rule

This is the simplest way to integrate; 
1.   we split the interval (a,b) in (n-1) slices of equal length `h = (b - a)/(n-1)`, 
2.   we approximate the function on each of those intervals by a constant.

###What is `n` again?
@@@
n is the number of points in the interval. If `n = 3`, `a=0`, and `b=1`,
then the three points are `[0, 0.5, 1]`

For this rule, the last point is never used, because we only need one point per interval to approximate a constant. 
@@@

In the case of the Left rule, the first interval will be approximated by `f a`, the second by `f (a + h)`, and so forth. Thus, `f b` is never evaluated.

Formally,

``` haskell

integral f (I (a,b) n) = let h = (b - a)/(n-1)
                             xs = map (\i -> a + i * h) [0..n-2]
                         in sum [h * f x_i | x_i <- xs]
```

To help us later, we want to define `w`, the weight function.

``` haskell
wLeft h = h -- w is a constant, always h ... for the first n - 1 points.

```

This is not quite right; we would like to be explicit about the fact that if we were to include `b` in `xs`, then `w` would be `0`.

``` haskell
--here, number is the position in the sum starting at 1.
wLeft h n number | (number == n) = 0
                 | (number >= 1 && number < n) = h
                 | otherwise = error "number is not in range with n"
                 
```

Therefore, we can have something like:
``` haskell
integral f (I (a,b) n) = let h = (b - a)/(n-1)
                             xs = toXs (I (a,b) n)
                             fs = map f xs
                             ws = map (wLeft h n) [1..n] 
                         in sum $ zipWith (*) fs ws

-- If we already have the list of function values from somewhere else. 
integralOfList fs (I (a,b) n) = let h = (b - a)/(n-1)
                                    ws = map (wLeft h n) [1..n] 
                                in sum $ zipWith (*) fs ws                         
```

We see that the structure is rather independent of the specific choice of `wLeft`.

###Why bother with `integralOfList`? (First Reason)
@@@
First, we could implement `integral` in terms of `integralOfList`. (3rd rep vs. 2nd)

Exercise: rewrite `integral` in terms of `integralOfList`:

```  active haskell
data Interval = I (Double,Double) Int
toXs (I (a,b) n) =map (\x -> a + (b-a)/fromIntegral (n-1) * fromIntegral x) [0..n-1]

wLeft h n number | (number == n) = 0
                 | (number >= 1 && number < n) = h
                 | otherwise = error "number is not in range with n"
                 
integralOfList fs (I (a,b) n) = let h = (b - a)/ fromIntegral (n-1)
                                    ws = map (wLeft h n) [1..n] 
                                in sum $ zipWith (*) fs ws  

integral:: (Double -> Double) -> Interval -> Double
integral f (I (a,b) n) = FIXME

main = print $ integral (\x -> x**2) (I (-1,1) 100) 
```
@@@

###Why bother with `integralOfList`? (Second Reason)
@@@
Sometimes, we don't have a `f`; for instance, someone could give you a table containing `fs`; maybe he did an experiment, or maybe he wrote a big program just to compute `fs`. Now, your job is to integrate. Can you do it? Yes you can; with `integralOfList`, you can.

And if you have `f`, then use this (Solution to the previous exercise):
``` haskell
integral f (I (a,b) n) = let xs = toXs (I (a,b) n)
                             fs = map f xs 
                         in integralOfList fs (I (a,b) n)                    
```
@@@

This method---although simple---is never used, because the error is proportional to `1/n`. Let's do better.

## Trapezoidal rule

Before, we approximated the function locally as a constant. Now, we will do just a tiny bit better, but the error will be proportional to `1/n^2` (still bad, but much better than `1/n`).

The idea is that for each slice of `x`, you approximate `f` by a linear function.

This gives you (on a single interval)

``` haskell
integral f [x_0, x_1] = let h = x_1 - x_0
                        in h * (f x_0 + f x_1)/2                       
```

The name of the rule comes from the formula above (the area of each slice is that of a trapezoid).

Exercise: 1)   Draw a trapezoid. 2) What shape corresponds to the Left rule?

If you carry this rule through for a whole interval (by summing over all slices), you get the following:

``` haskell
integral [f1,f2,..,fN] (I (a,b) n) = let h = (b - a)/ fromIntegral (n - 1)
                                     in h*f1/2 + h*f2 + .. + h*fN/2
```

###Why different weights for the edges?
@@@
When we add the contribution from each interval, every point gets a `h/2`, but the points *inside* (not at the edge) get two contributions (one for each interval they belong to).

To see this more clearly, we could rewrite:
``` haskell
integral [f1,f2,..,fN] (I (a,b) n) = let h = (b - a)/ fromIntegral (n - 1)
                                     in h * (f1 + f2)/2+  h * (f2 + f3)/2  + .. +  h * (f(N-1) + fN)/2 
```
@@@

So basically, we only need to define `w` for the trapezoidal rule in accordance to this, and we are good.

``` haskell
--here, number is the position in the sum starting at 1.
wTrap h n number | (number == 1 || number == n) = h/2
                 | (number > 1 && number < n) = h
                 | otherwise = error "number is not in range with n"
                 
```


Now, the full integral formula:

``` haskell
integralOfList fs (I (a,b) n) = let h = (b - a)/(n-1)
                                    ws = map (wTrap h n) [1..n] 
                                in sum $ zipWith (*) fs ws 
```

The only difference between this one and the one for the Left rule is the weight function.

### Common Patterns?
@@@
This gives us the idea to define a super-function (sorry) which takes the weight function as a parameter.

``` haskell
integralOfList w fs (I (a,b) n) = let h = (b - a)/(n-1)
                                      ws = map (w h n) [1..n] 
                                  in sum $ zipWith (*) fs ws 
                                  
intLeft = integralOfList wLeft
intTrap = integralOfList wTrap
```
If you do not understand partial application (last two lines), then rewrite `intLeft` to be the same as `integralOfListLeft`.
@@@

##Simpson's Rule

*This method only works if the number of points n is an _odd_ number.*

The idea is to approximate the function by a quadratic in each slice of three points; it amounts to this:

``` haskell

f x = f x_0 + 
    (x - x_0) * (f (x_0 + h) - f (x_0 - h))/ (2*h) + 
    (x - x_0)**2 / 2 * (f (x_0 + h) - 2 * f x_0 + f (x_0 - h))/ h**2 +
    O (h**3)

--where ** means exponentiation;
-- and O means `Big-oh notation'. 
-- It means that what is left, 
-- after we expanded the first 
-- Taylor terms, is proportional to `h**3`.
```

This gives you:

``` haskell

integral f [x_0, x_1, x_2] = let h = (x_2 - x_0) / 2
                             in h * (f x_0 + 4 * f x_1 + f x_2)/3 
                        
```
If you are skeptical, try to integrate `f x` as approximated above with your knowledge of mathematical (exact) integration. Remember, a quadratic can be integrated exactly. 

If you carry this rule through for a whole interval (by summing all the slices), you get the following:

``` haskell
integral [f1,f2,..,fN] (I (a,b) n) = let h = (b - a)/ fromIntegral (n - 1)
                                     in (1/3) * h*f1 + 
                                        (4/3) * h*f2 +
                                        (2/3) * h*f3 +
                                        (4/3) * h*f4 +
                                        .. + 
                                        (1/3) * h*fN
```

So basically, we only need to define `w` for the Simpson's rule in accordance to this, and we are good.

``` haskell
--here, number is the position in the sum starting at 1.
wSimp h n number | (number == 1 || number == n) = (1/3)*h
                 | (number > 1 && number < n && even number) = (4/3)*h
                 | (number > 1 && number < n && odd number) = (2/3)*h
                 | otherwise = error "number is not in range with n"
                 
```


The full integral formula is:

``` haskell
intSimp fs (I (a,b) n) = if odd n 
                         then integralOfList wSimp fs (I (a,b) n)
                         else error "The Simpson's rule only works with an odd number of points; the Interval given had an even n."

```



Exercise: Use both the Simpson's rule and the Trapezoidal rule to create an integration which never fails due to `even n`, and has best accuracy.

@@@
``` haskell
--You got it
intSuper fs (I (a,b) n) = if odd n 
                          then integralOfList wSimp fs (I (a,b) n)
                          else integralOfList wTrap fs (I (a,b) n)
                          
--You're a genius. Seriously, that was clever of you.  
intGotcha fs (I (a,b) n) = if odd n 
                           then integralOfList wSimp fs (I (a,b) n)
                           else let h = (b-a) / fromIntegral (n-1)
                                in integralOfList wSimp (init fs) (I (a,b-h) (n-1))
```
@@@

The Simpson's rule makes the error go as `1/n^4`.

#A little Problem just for You

The error function is defined exactly by:

``` haskell

erf x = 2 / sqrt pi * integral (\y -> exp (-y**2)) (0,x)

```

Now, suppose we fix `x=1`, 

I am telling you that: `erf 1 = 0.842700792949715`. 

Your mission---should you choose to accept it---is to use the three methods we saw 
1.  Left 
2.  Trapezoidal 
3.  Simpson 
to _investigate_.

More precisely, for each of these methods, you can define the relative error as a function of `n`:

``` haskell
--this is not actual code; part of your mission is to define relative error properly.

relError n = abs (methodAnswer n - exactAnswer) / exactAnswer

```
Then, you must: 

1.  Compute this function for all three methods for n = 3, 5, 9, 16, 32, 64, 128.

2.  Make a table, or plot your results.

3.  Compare the three methods.

4.  (Very important) Have fun.

5.  Hold your breath for the upcoming Gaussian Quadrature in 'Advanced Integration'.

#Note on Performance.

As we will see later, the Gaussian quadrature is much better than Simpson's rule, but suppose we didn't know about such quadrature; how could we improve the performance of Simpson's rule?

So far, we used lists to represent our function. The compiler does an amazing job for you when you're working with lists, but (this is completely irrelevant now, and we will come back to it when it becomes relevant) lists are not the best representation for things of fixed length. 

Because our integrator doesn't return a list (in fact, it doesn't have to take a list as input to begin with), we could replace lists by some similar representation.

A better representation, which has the same interface (has the same name for similar functions) and operations, is the Vector representation. If you are a performance junky, I challenge you to implement Simpson's rule without using any lists; instead, use <hoogle>Vector</hoogle> from <hoogle>Data.Vector.Unboxed</hoogle>. You can click on the green word for more info on Data.Vector. 

###A few thoughts on Optimization (we will come back to it)
@@@
When building complex programs, it is always better to get something simple working quickly. However, Computational Physics contains many (most) problems where your code needs to be super-fast in order to be useful. 

My take on the subject is the following: 
1.  build the simplest program you can think of, with crystal-clear semantics, 

2.  make sure it does what it's supposed to, 
3.  explain (using meaningful names in your program, and comments) what it does, and how it does it. 
4.  use abstraction and observation to identify the key features of your program. (more on that later)
5.  use optimized representations (Data.Vector is an optimized representation of Data.List; it does the same things, has a few more utilities purely for performance), and optimized libraries written by someone else. However, you must not change the semantics developed at step 1 unless you have an excellent reason (you found even clearer semantics; the original semantics turn out to be incomplete, which means you can't do certain things you wanted to do; the new semantics are as clear, as powerful, and allow for a *dramatic* performance increase).
6.  learn about optimization.
7.  optimize without changing the semantics. At this point, you should have a copy of your code which has the same semantics, but isn't optimized, you can think of it as a backup. Here is why:
    1. to show other people; say something like "this code isn't my actual code, but you can write your program which uses my code AS IF this were the actual code."
    2. to test whether your 'optimized' code is 1) actually much faster 2) actually equivalent in terms of meaning (semantics)
8.  use parallelism, GPUs, FFI.
9.  (if you are good at this step, this step should be inserted between 5. and 6.) Go back to the drawing board, and find a new algorithm, or read the literature for similar problems, or, and this is a sad alternative, understand that your problem can't be solved on today's computers (have you tried super-computers?). At this point, it is time to change your problem of study. If you are emotionally attached to the problem, it might be worthwhile to try setting it up differently. 

You might be surprised at how rare you have to go past step 5 (not so rare in Computational Physics, but still rare). However, you sometimes have to go all the way to step 9. Later, we will learn to predict, at step 1, up to where a certain program is likely to take us. Even later, we might learn to predict to where we'll need to go without writing any code (just by looking at the problem).

Anyway, this is my take on optimization in a nutshell. If you stick to this course, we will eventually revisit this hard and crucial subject, but not until we actually need it.
@@@
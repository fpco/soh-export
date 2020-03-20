# Simple Interpolation

# A simpler problem.

Given a list of points, we want to find a polynomial function `f` such that:

For every point `(x_i,y_i)` in your list, then `f x_i == y_i` is always true. 

Just for fun, let us write the exact same problem using Haskell:

``` haskell
-- show This is given to you
listOfPoints = [(x_1,y_1), .. (x_N,y_N)]
-- /show

-- show The following property must be satisfied
map snd listOfPoints == map (f . fst) listOfPoints
-- /show

```

## An example for a set of three points.

Here is a solution to the previous problem in the special case where the input list contains 3 points.

``` haskell
-- show The input
listOfPoints = [(x1,y1),(x2,y2),(x3,y3)]
-- /show

-- show The solution
f x = y1 * (x - x2) * (x - x3) / ((x1 - x2) * (x1 - x3)) +
      y2 * (x - x1) * (x - x3) / ((x2 - x1) * (x2 - x3)) +
      y3 * (x - x1) * (x - x2) / ((x3 - x1) * (x3 - x2))
-- /show
```

### Exercise 
1.   Evaluate `f x1`, and show that indeed `f x1 = y1`. 
2.   Either figure out why this shows `f x2 = y2`, or do it all over again. 
3.   Same thing for `f x3`.
4.   *Bonus:* Write down a similar function for the case of 4 points.

### Note: Representing a polynomial (and a little story about a pirate in Tibet)
@@@
In this example, the solution was given as a Haskell function. This is fine to get the concept of interpolation, but it is not very useful in practice.

Here is a scenario to illustrate my point.

Long ago, a pirate retired, and decided to hide his gold on the top of a mountain in Tibet.

You have gone to that mountain, and realized that the gold was missing from the top of the mountain. 

Your best hypothesis is that the gold has been carried away through the mountain by an underground stream.

So you walk around the mountain (it's a 2D mountain for now), and record your height at several places. 

With this data, you need to guess where to dig for the gold.

In short, you need to find the local minima of that mountain.

With the tutorial on *Root Finding*, you should be able to estimate those minima by looking for the roots of the derivative of `f`. However, this is a lot of work, and there is a slightly nicer way to do this.

In my tutorial about *Polynomials*, I showed you (check it out if you haven't already) how to represent a polynomial function as a list of coefficients.

In this way, you can derive `f` exactly, and even write down an expression for the roots in the case of (for instance) `f x = a * x**2 + b * x + c`. 

Thus, you can find your gold and steal it too.

The key idea here is that you don't always know what you might be asked to do with the output of a certain program. Therefore, you should aim for the output which contains the biggest amount of information. Notice that this last bit is just wrong if it is impossible to convert from your specific output to the standard one. 
@@@

## The solution for any number of points.

Now, we do the general thing.

By looking at the specific example (3 points), and any other specific examples you might have worked out (4 points), we will try to build a general solution. If you just want the solution, see the hidden content below:  

@@@
``` haskell
interpolationPoly xs ys x = let 
    lamb xi  = product $ map (\xj -> (x-xj)/(xi-xj)) (delete xi xs)                                               
    in sum $ zipWith (*) ys (map lamb xs)
  
```
@@@

### Generalizing a specific solution

Let us rewrite the solution: 

``` haskell

-- show The solution
f x = y1 * (x - x2) * (x - x3) / ((x1 - x2) * (x1 - x3)) +
      y2 * (x - x1) * (x - x3) / ((x2 - x1) * (x2 - x3)) +
      y3 * (x - x1) * (x - x2) / ((x3 - x1) * (x3 - x2))
-- /show

```

We see that it is a sum of various terms. Let us rewrite it as such.

``` haskell

-- show The solution
f x = sum $ [y1 * (x - x2) * (x - x3) / ((x1 - x2) * (x1 - x3))
             , y2 * (x - x1) * (x - x3) / ((x2 - x1) * (x2 - x3))
             , y3 * (x - x1) * (x - x2) / ((x3 - x1) * (x3 - x2))]
-- /show

```
Each term is the product of `y_i` with the rest.

``` haskell
-- show Inputs:
ys
x1, x2, x3
-- /show

-- show The solution
f x = sum $ zipWith (*) ys [(x - x2) * (x - x3) / ((x1 - x2) * (x1 - x3))
                            , (x - x1) * (x - x3) / ((x2 - x1) * (x2 - x3))
                            , (x - x1) * (x - x2) / ((x3 - x1) * (x3 - x2))]
-- /show

```

We can rewrite again:

``` haskell
-- show Inputs:
ys
x1, x2, x3
-- /show

-- show The solution
f x = sum $ zipWith (*) ys [product [(x - x2) / (x1 - x2), (x - x3) / (x1 - x3)]
                            , product [(x - x1) / (x2 - x1), (x - x3) / (x2 - x3)]
                            , product [(x - x1) / (x3 - x1), (x - x2) / (x3 - x2)]]
-- /show

```

Finally, we get the final form:

``` haskell
-- show Inputs:
xs
ys
-- /show

-- show The solution
f x = let 
   lamb xi  = product $ map (\xj -> (x-xj)/(xi-xj)) (delete xi xs)                                               
   in sum $ zipWith (*) ys (map lamb xs)
-- /show  
```

This is called the Lagrange method.

# Better than Interpolation?

So far, we have discussed a global (the whole input contributes to the same function) polynomial interpolation.

Later, we will talk about local interpolation, and even better interpolation.

However, you should realize that interpolation is rather bad if it comes to real data with built-in error.
To tackle real data, we need Data Fitting techniques, and even later, we will talk about machine learning.

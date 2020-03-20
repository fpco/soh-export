# Currying and Partial Application

# First-Class Functions #

Haskellers often say that functions are first-class in Haskell, but if you're new to functional programming, that may not mean anything to you. 

Let's start exploring what a first-class function is with a little bit of code:

``` haskell
two = 2 -- Int, with a value of 2.
double x = x + x -- function that takes a number and returns double its argument.
```

The gist of the idea is that Haskell treats functions as values. `two`'s type is, unsurprisingly, Int. double's type is `(Num a) => a -> a`; don't worry about the `Num a`, it basically means that it can take any numeric value. Since Haskell functions need to be pure, `double`'s nothing more than a way to get one value from another: no I/O, network activity, or changing global values goes on. 

Another unique thing about Haskell is that all functions take just one argument. That sounds weird, especially when you've seen and used functions that take two arguments and work like you'd expect, but it's true. 

``` active haskell
-- show
firstFiveOdds = take 5 [1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
-- /show
main = putStrLn $ "first five odd numbers = " ++ (show $ take 5 [1, 3, 5, 7, 9, 11, 13, 15, 17, 19])
```

``` active haskell
-- show
fourOrThree = 4 > 3
-- /show
main = putStrLn $ "Is 4 > 3? " ++ (show $ 4 > 3)
```

``` active haskell
-- show
greaterOrLess = compare 5 6
-- /show
main = putStrLn $ "Is 5 greater than, less than, or equal to 6? " ++ (show $ compare 5 6)
```


# Currying #

There is nothing special about this code. We're calling these functions with the standard call syntax in Haskell.

To understand Haskell, you need to understand what's going on here. Let's examine how `take` behaves when it's called first. take has two arguments, an `Int` and a list. When Haskell evaluates the line `take 5 [1,3,5,7,9,11,13,15,17,19]`, first it applies 5 to `take`, and that makes a function that takes a list as its argument and returns the first 5 items of the list. Since there's a list right after 5, it immediately runs the function `(take 5)` on that list.

The chain of evaluation goes like this: 

``` haskell
-- before any evaluation: 
firstFiveOdds = take 5 [1, 3, 5, 7, 9, 11, 13, 15, 17, 19]

-- 5 is given to take, and the resulting function is applied to the list:
(take 5) [1, 3, 5, 7, 9, 11, 13, 15, 17, 19]

-- the result of applying (take 5) to the list: 
[1,3,5,7]
```

This sounds strange, but that's how it works. Every function in Haskell works this way; functions that take one argument at a time like this exist in other languages, and in general they're called *curried* functions, after the mathematician Haskell Curry. The language Haskell is named after him, too!

# Partial Application #

Curried functions are actually quite useful. We've seen that when you call a function with only some of its arguments, you get another function back. The technical term for "underloading" a function like this is *partial application*. In the `take 5` example, the new function immediately got its other argument and ran, but we can do other things with partially applied functions. 

For a relatively small example, let's say that you know you're only going to be working with the first five elements of any given list in your program. In that case, you could define a new function like `take5 = take 5`. The function that applying 5 to `take`, which takes a list and returns the first five elements of that list, now has a name, and we can call it like any other function. 

``` active haskell
-- show
take5 = take 5
-- /show
main = putStrLn $ "Let's test take5. take5 [1, 3, 5, 7, 9, 11, 13, 15, 17, 19] = " ++ (show $ take5 [1, 3, 5, 7, 9, 11, 13, 15, 17, 19])
```

`take5` isn't the most useful example, but it's simple and shows that partial application works. You can do this with any function that takes multiple arguments, which makes partial application a powerful tool!
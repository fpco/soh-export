# Haskell Basics #4

## List comprehension

The powers of list comprehension in a functional programming language
are best illustrated when compared alongside their imperative language equivalents

If I wish to create a list of all the odd numbers between 1 and 100

In C++, I could do:

``` c++
int len = 50;
int[len] m;
counter = 0;
for (int i = 1; i <= 100; ++i)
{
  if (i % 2 == 1)
  {
    m[counter] = i;
    ++counter;
  }
}
```

In Haskell I could do:

``` active haskell
m = [ x | x <- [1..100], x `mod` 2 == 1]
main = print $ m
```

The C++ was a lot of code, compared to `[ x | x <- [1..100], x `mod` 2 == 1]`!

Also, notice that I did not have to predict the number of elements that there would be in the final array, as I had to do in C++.

We can even replace the modulo arithmetic with a the built in `odd`.

``` active haskell
m = [ x | x <- [1..100], odd x]
main = print $ m
```

The way this works is everything before the pipe, `|`,
is what is getting added to the list.
Everything after the pipe describes how that gets added.

One could read that in prose as,
"Let the list m be contain elements, where each element is
and integer between 1 and 100, and is odd."

The power, and beauty, of this, is that using list comprehensions,
one feels more like they are expressing what they want,
instead of expressing how to get what they want.

## FizzBuzz

A classic beginners programming exercise is FizzBuzz:
"Write a program that prints out all of the numbers between 1 and 100.
If a number is divisble by 5 print FIZZ instead of that number.
If a number is divisble by 3 print BUZZ instead of that number.
If a number is divisble by both 3 and 5 print FIZZBUZZ instead of that number."

Try writing this in C++, or some other imperative language.
@@@
In C++:
``` c++
for (int i = 1; i <= 100; ++i)
{
  if (i % 15 == 0)
  {
    std::cout <<"FIZZBUZZ" <<std::endl;
  }
  else if (i % 5 == 0)
  {
    std::cout <<"FIZZ" <<std::endl;
  }
  else if (i % 3 == 0)
  {
    std::cout <<"BUZZ" <<std::endl;
  }
  else
  {
    std::cout <<i <<std::endl;
  }
}
```
@@@


Now how would you do the same in Haskell, using list comprehensions?

Firstly, let us just take the list comprehension above,
`[ x | x <- [1..100], odd x]` and add an `if .. then .. else` block to it
just for `FIZZBUZZ`.

``` active haskell
m = [ if (x `mod` 15 == 0) then "FIZZBUZZ" else (show x) | x <- [1..100]]
main = print $ m
```

Note that `show` as used here, converts the integer to a string.
This is necessary because, remember that a list is homogenous -
all of its elements must be of the same type.

Now try adding `FIZZ` into the mix.
Hint: Nest the new `if .. then .. else` block within the `else` block of the existing `else`.

@@@
``` active haskell
m = [ if (x `mod` 15 == 0) then "FIZZBUZZ" else (if (x `mod` 5 == 0) then "FIZZ" else (show x)) | x <- [1..100]]
main = print $ m
```
@@@

Now add `BUZZ` into the mix.

@@@
``` active haskell
m = [ if (x `mod` 15 == 0) then "FIZZBUZZ" else (if (x `mod` 5 == 0) then "FIZZ" else (if (x `mod` 3 == 0) then "BUZZ" else (show x))) | x <- [1..100]]
main = print $ m
@@@

Now that is starting to a look a little unwieldy, isn't it?
All we need to do now is to make it look neater, and we will do that
by splitting into multiple lines.

Exercise:

Define a function `fizzbuzz` which you will use as the filter.
Define this function using guards, as we have previously
in our recursive list functions.

``` active haskell
fizzbuzz x
  -- your code here

m = [ fizzbuzz x | x <- [1..100] ]
main = print $ m
```

@@@
``` active haskell
fizzbuzz x
  | (x `mod` 15 == 0) = "FIZZBUZZ"
  | (x `mod` 5 == 0) = "FIZZ"
  | (x `mod` 3 == 0) = "BUZZ"
  | otherwise = show x

m = [ fizzbuzz x | x <- [1..100] ]
main = print $ m
```

Note that `fizzbuzz` is not a recursive function,
like our `listMax` and `listMin` in our previous  exercises.

Thus it may seem that this list comprehension has been
accomplished through iteration.
If you look under the hood, however, it really is
still recursionm. This happens in the list comprehension (and not in the `fizzbuzz` function). Tail-order recursion, which is compiled into an iteration by the Haskell compiler.

Although not immediately obvious, the Haskell compiler
also would recognise our implementations for
`listMax` and `listMin` as tail-order recursion,
and compile it iteratively too.

This is a compiler optimisation, and thus as functional programmers, should be aware of that this happens. We should not, however, not care about it, and instead just let the compiler do its thing.
@@@

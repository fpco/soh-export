# 3. Pure Functions, Laziness, I/O, and Monads

> Haskell is the world’s finest imperative programming language. <br/>- _Simon Peyton Jones_

## Pure Functions

Unlike in imperative languages, a function in Haskell is _really_ a function, just as mathematicians intended it to be. To distinguish it from cheap imitations, Haskell functions are sometimes called _pure_ functions. Here are the fundamental properties of a pure function:

1. A function returns exactly the same result every time it's called with the same set of arguments. In other words a function has no state, nor can it access any external state. Every time you call it, it behaves like a newborn baby with blank memory and no knowledge of the external world.
2. A function has no side effects. Calling a function once is the same as calling it twice and discarding the result of the first call. In fact if you discard the result of any function call, Haskell will spare itself the trouble and will never call the function. No wonder Haskell has a reputation for laziness (more about it later).

One of the major strengths of Haskell is that the execution of pure functions can be easily parallelized  because they have no side effects. With no side effects there are no data races -- the bane of parallel programming.

However, an astute reader might at this point start doubting the sanity of programming in Haskell. How can a program built from pure functions do anything useful other than heat up the processor during cold winters in Siberia?

## Simple I/O

And yet, even the simplest program in Haskell does more than stir electrons in a CPU.
``` active haskell
main = putStrLn "Hello World!"
```
Run this code and you'll see that it miraculously makes text appear on your screen. A pure function can't do that -- it cannot have side effects!

So what's happening here? Haskell runtime evaluates `main` (it's really just an expression). The trick is that `main` evaluates not to a simple value but to an _action_. The runtime then _executes_ this action. 

So the program itself has no side effects, but the action does. And this so called `IO` action is invoked _after_ the Haskell program has run, so we don't break the functional nature of the program. At least that's the conceptual model -- things are a little more complicated because of Haskell's laziness (see later).

Does `main` have to produce an `IO` action? Yes, it's part of the contract. But just for some cheap thrills, try making something that's not an action, like in this example:

``` active haskell
main = True
```
Don't worry about the verbosity of the compiler error message. Just notice that it contains the symbol `IO`. 

So how did the examples in my previous tutorials work? I always made sure that `main` called functions like <hoogle>putStrLn</hoogle> or <hoogle>print</hoogle>, which return `IO` actions. These function are, in turn, built from simpler functions also returning `IO` actions. If you follow this chain, you'll find that there is only one non-trivial source of IO actions: primitives built into the language. As you'll see later, you can also trivially convert any value into an `IO` action using `return`.

However you can never execute an IO action inside the program -- there just isn't a function that can do it. Once created, an `IO` action keeps percolating up until it ends up in `main` and is executed by the runtime. (You can also discard an `IO` action, but that means it will never be evaluated.)

## Laziness

Haskell takes laziness seriously. No, it doesn't spend most of its time sitting on a couch and munching potato chips. But it will not calculate anything unless it's strictly necessary (or is forced by the programmer -- there's the way to do it). Haskell is so lazy that it won't even evaluate arguments to a function before calling it. Unless proven otherwise, Haskell assumes that the arguments will not be used by the function, so it procrastinates as long as possible.

Let me demonstrate Haskell's laziness through a simple example. There are some expressions that, when evaluated, lead to a runtime error -- definitely not as many as in other languages but still, there are some. Division by zero comes to mind. There is also a built in object that, by definition, can never be evaluated. It's called `undefined`. It's sort of like trying to dereference `null` in Java. Try running this program:
``` active haskell
main = print $ undefined + 1
```
Notice that the compiler doesn't complain that you're trying to add `undefined` to 1. I'll explain this leniency later. The error you're seeing is a runtime error resulting from an attempt to evaluate `undefined`. But try this instead:
``` active haskell
foo x = 1
main = print $ (foo undefined) + 1
```
Here, Haskell calls `foo` but never evaluates its argument `undefined`. You might think that it's an optimization: The compiler sees the definition of `foo` and figures out that `foo` discards its argument. But the result is the same if the definition of `foo` is hidden from view in another module. We haven't talked about modules yet, but just to make this point, here's the same example split between two files:
``` active haskell
{-# START_FILE Foo.hs #-}
-- show
module Foo (foo) where
foo x = 1
{-# START_FILE Main.hs #-}
-- show
import Foo
main = print $ (foo undefined) + 1
```
Later you'll see that Haskell's laziness allows it to deal with infinity: for instance, with infinite list, or with the future that hasn't materialized yet.

Laziness or not, a program needs to be executed at some point.

So what makes a program run? There are several reasons why an expression would have to be evaluated -- the fundamental one being that somebody wants to display its result. So, really, without I/O nothing would ever be evaluated in Haskell.

## Sequencing

Larger IO actions are built from smaller IO actions. The composition of IO actions has one very important property: The order of composition matters. It matters whether you first print "Hello" and then "World" or the other way around. In the world of pure functions and lazy evaluation this is a non-trivial requirement. 

You have to be able to _sequence_ IO actions.

Haskell has special syntax for sequencing. It's called the _do notation_. Here's a simple example:

``` active haskell
main = do
    putStrLn "The answer is: "
    print 43
```
Here we are sequencing two `IO` actions, one returned by `putStrLn` and another returned by `print`. We do this by putting them inside a `do` block. The block is constructed using line breaks and proper indentation. In C-like languages blocks are usually delimited by braces and statements are separated by semicolons. In fact, you can do the same in Haskell, although such notation is rarely used:
``` active haskell
main = do {
    putStrLn "The answer is: ";
    print 43
}
```
The "input" part of the I/O is also easy. Just like in imperative programming, whatever you receive from the user or from a file you _assign_ to a variable and use it later. Like this (enter a line of text and press enter when you run this program):
``` active haskell
main = do 
    str <- getLine
    putStrLn str
```
Although this works as expected, `str` is not really a variable, and the assignment is not really an assignment. Remember, Haskell is a functional language. The line:
``` haskell
    str <- getLine
```
creates an action that, when executed will take the input from the user. It will then pass this input to the rest of the do block (which is also an action) under the name `str` when it (the rest) is executed. In Haskell you never assign to a variable, instead you _bind_ a name to a value. When the action produced by the do block is executed, it binds the name `str` to the value returned by executing the action that was produced by `getLine`. 

You can safely ignore what I just said and imagine that an assignment is possible inside a `do` block. It won't hurt you. But I just want you to know that, unlike in other functional languages, I/O in Haskell is not a hack that breaks the pure functional nature.

In fact the `do` block is not a special hack that only works for `IO` actions. Far from that, `do` blocks are used for sequencing a more general set of _monadic_ operations. `IO` is just one example of a monad, but I will show you many more. You'll have to wait for the formal definition of a monad because it requires the understanding of a few more advanced features of Haskell.

Unlike "object" in object-oriented programming, the "monad" of functional programming is not a word from common vocabulary, and there is no simple intuition that would elicit the aha! from a casual programmer. You need to see quite a few examples before you develop your own intuition. The closest I can describe the idea of a monad is that it has an imperative feel. 

Monadic `do` blocks really look like chunks of imperative code. They also behave like imperative code: You can think of lines of code in the `do` block as _statements_ that are executed one after another. This similarity is no coincidence -- all imperative programming _is_ at its core monadic. An imperative programmer learning Haskell might be as shocked as Molière's Bourgeois Gentleman upon discovering that all his life he'd been speaking prose. In Haskell this "imperative prose" is implemented using "functional poetry." 

The way the actions are glued together is the essence of the Monad. Since the glueing happens between the lines, the Monad is sometimes described as an "overloading of the semicolon." Different monads overload it differently.


## Exercises

1.  Define a function `putStrLn'` using `putStr` and `putChar`; the latter to output the newline, `'\n'`. (Replace `undefined` with actual code).
    ``` active haskell
    putStrLn' str = undefined

    main = do
        putStrLn' "First line"
        putStrLn' "Second line"
    ```
    @@@ Show solution
    ``` active haskell
    putStrLn' str = do
        putStr str
        putChar '\n'
    
    main = do
        putStrLn' "First line"
        putStrLn' "Second line"
    ```
    @@@
1.  Define a function `putQStrLn` that outputs a string surrounded by quotes, `'"'`
    ``` active haskell
    putQStrLn str = undefined

    main = putQStrLn "You can quote me."
    ```
    @@@ Show solution
    ``` active haskell
    putQStrLn str = do
        putChar '"'
        putStr str
        putChar '"'
        putChar '\n'
    
    main = putQStrLn "You can quote me."
    ```
    @@@
3.  Rewrite the previous exercise to take the input string from the user.
    @@@ Show solution
        
    ``` active haskell
    putQStrLn str = do
        putChar '"'
        putStr str
        putChar '"'
        putChar '\n'
    
    main = do
        putStrLn "Enter text:"
        str <- getLine
        putQStrLn str
    ```
    @@@


## Bibliography

1. Simon Peyton-Jones, [Tackling the Awkward Squad: monadic input/output, concurrency, exceptions, and foreign-language calls in Haskell](http://research.microsoft.com/en-us/um/people/simonpj/papers/marktoberdorf/mark.pdf)
2. Eugenio Moggi, [Notions of computation and monads](http://www.disi.unige.it/person/MoggiE/ftp/ic91.pdf).

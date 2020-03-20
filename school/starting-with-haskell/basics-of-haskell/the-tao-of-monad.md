# 3.a The Tao of Monad

## The TAO of Monad

There comes a time in every Haskell tutorial when you have to have a serious talk about monads. If I wait too long, you'll get the impression that Haskell is only good for doing math -- especially after you read the section on pure functions. If I start too early, you may be put off by a lot of theoretical talk that makes no sense without proper context. 

So far I've been trying to be truthful, even if I wasn't telling the whole truth. I could continue this way with monads and get into category theory, morphisms, functors, type theory, and all this theoretical stuff. But I know that not everybody loves higher math.

Or I could start lying. The problem with lying is that Haskell purists would notice and drive me out of town with pitchforks and torches. 

But there is a third way: mystical rather than categorical. I can follow the ancient Chinese sage Lao Tzu who essentially wrote the first Haskell manual called Tao Te Ching. The Tao is supposed to be as elusive as the Monad:
```
The TAO (Monad) that can be told is not the eternal TAO (Monad).
```
The fact that Lao Tzu didn't stop writing after this opening gives me hope that I can do it too: Talk about the Monad without defining it. Starting with this caveat:
```
However I describe the Monad to you, I'll be missing something important.
 -- Bartosz Milewski
```

## Pure Functions

Unlike in imperative languages, a function in Haskell is _really_ a function, just as mathematicians intended it to be. To distinguish it from cheap imitations, Haskell functions are sometimes called _pure_ functions. Here are the fundamental properties of a pure function:

1. A function returns exactly the same result every time it's called with the same set of arguments. In other words a function has no state, nor can it access any external state. Every time you call it, it behaves like a newborn baby with blank memory and no knowledge of the external world.
2. A function has no side effects. Calling a function once is the same as calling it twice and discarding the result of the first call. In fact if you discard the result of any function call, Haskell will spare itself the trouble and never call the function. No wonder Haskell has a reputation for laziness (more about it later).

One of the major strengths of Haskell is that the execution of pure functions can be easily parallelized  because they have no side effects. With no side effects there are no data races -- the bane of parallel programming.

However and astute reader might at this point start doubting the sanity of programming in Haskell. How can a program built from pure functions do anything useful other than heat up the processor during cold winters in Siberia?

## Simple I/O

And yet, even the simplest program in Haskell does more than stir electrons in a CPU.
``` active haskell
main = putStrLn "Hello World!"
```
Run this code and you'll see that it miraculously makes text appear on your screen. A pure function can't do that -- it can't have side effects!

Here's the somewhat mystical explanation of how this is possible:

1. All functions in Haskell are pure, including `main`
2. The runtime calls `main`, which produces a _monadic action_.
3. This monadic action, when given the Universe as input, produces a new modified Universe.
4. We continue living in this new Universe

In the case of the Hello World program, the Universe is modified by displaying the text "Hello World!" on your screen. 

So the Universe we are living in is being constantly modified by Haskell programs. That's why being a Haskell programmer feels like being the Master of the Universe. 

I should probably mention that this is not how Haskell I/O is implemented in real life. But in most cases the implementation behaves just like this mystical ideal. And that's the conceptual model you should be using when reasoning about Haskell programs.

The monad that deals with actions operating on the Universe is called the _IO monad_. It's just one of many monads, but it's pretty important because of the special role of the `main` function. This function _must_ evaluate to an IO monadic object. If you don't believe me, try this for instance: 

``` active haskell
main = True
```
Don't worry about the meaning of the compiler error message. Just notice that it contains the symbol `IO` -- that's the IO monad I've been talking about. 

So how did my previous examples work? I just made sure that `main` called functions like <hoogle>putStrLn</hoogle> or <hoogle>print</hoogle>, which return monadic IO actions, which were then returned from `main`. These function are, in turn, built from simpler functions also returning IO actions. If you follow this chain, you'll find that there is only one source of IO actions: primitives built into the language. In Haskell, you can't create an IO action from scratch, no matter how hard you try. Also, you can't execute an IO action inside the program. For that you have to return it from `main`. So an IO action is truly indescribable. 

Now you might be thinking that monads are about actions. So let me get the record straight:

```
Monad that is about actions, is not the eternal Monad.
 -- Bartosz Milewski
```
## The TE of Laziness

TE means _virtue_. We usually don't think of laziness as virtue. Spending hours in a couch in front of a TV drinking beer and munching junk food can hardly be considered a good thing. But let's see what Tao Te Ching has to say about it:
```
Those highest in TE take no action
And don't need to act.
```
Haskell takes laziness seriously. It will not calculate anything unless it's strictly necessary (of if forced by the programmer). Haskell is so lazy that it won't even evaluate arguments to a function before calling it. Unless proven otherwise, Haskell assumes that the arguments will not be used by the function, so why bother.

Let me demonstrate Haskell's laziness through a simple example. There are some expressions that, when evaluated, lead to a runtime error -- definitely not as many as in other languages but still, there are some. Division by zero comes to mind. There is also a built in object that, by definition, can never be evaluated. It's called `undefined`. It's sort of like `null` in Java. Try running this program:
``` active haskell
main = print $ undefined + 1
```
Notice that the compiler doesn't complain that you're trying to add `undefined` to 1. I'll explain it this leniency later. But when this program is run, it terminates with a runtime error because it tries to evaluate `undefined`. But try this instead:
``` active haskell
foo x = 1
main = print $ (foo undefined) + 1
```
Here, Haskell calls `foo` but never evaluates its argument. You might think that it's an optimization: The compiler sees the definition of `foo` and figures out that `foo` discards its argument `x`. But the result is the same if the definition of `foo` is hidden from view in another module. We haven't talked about modules, but just to make this point, here's the same example split between two files:
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
Later you'll see that Haskell's laziness allows it also to deal with the Infinite, for instance, infinite list, or with the future that hasn't materialized yet.

Laziness or not, a program needs to be executed at some point.
```
Teaching without words,
Benefit witout action --
Few in this world can attain this
```
So what makes a program run? There are several reasons why an expression would have to be evaluated -- the fundamental one being that somebody wants to display its result. So, really, without I/O nothing would ever be evaluated (which would lead to tremendous energy savings).

## Sequencing

```
Big things are built from smaller things. We call this composability.
 -- Bartosz Milewski
```

As I mentioned earlier, larger IO actions are built from smaller IO actions. The composition of IO actions has one very important property: The order of composition matters. In the world of pure functions and lazy evaluation this is a significant requirement. 

You have to be able to _sequence_ IO actions.

Haskell has special syntax for sequencing monadic actions. It's called the _do notation_. Here's a simple example using the IO monad:

``` active haskell
main = do
    putStrLn "The answer is: "
    print 43
```
Here we are sequencing two monadic actions, one returned by `putStrLn` and another returned by `print`. We do this by putting them inside the `do` block. The block is constructed using line breaks and proper indentation. In C-like languages blocks are usually delimited by braces and separated by semicolons. In fact, you can do the same in Haskell, although such notation is rarely used:
``` active haskell
main = do {
    putStrLn "The answer is: ";
    print 43;
}
```
The "input" part of the I/O should also be easy, right? Whatever you receive from the user or from a file you just assign to a variable and use it later. Like this (you'll have to enter a line of text and press enter when you run this program):
``` active haskell
main = do 
    str <- getLine
    putStrLn str
```
Although this works as expected, `str` is not really a variable, and the assignment is not really an assignment. Remember, Haskell is a functional language. The line:
``` haskell
    str <- getLine
```
creates an action that, when executed will take the input from the user. It will then pass this input to the rest of the do block (which is also an action) when it (the rest) is executed. `str` is just a name we give this input, so we can use it in subsequent actions. In Haskell you never assign a variable, instead you _bind_ a name to a value. When the action produced by the do block is executed, it binds the name `str` to the value returned by executing the action that was produced by `getLine`. You can safely ignore what I just said and imagine that an assignment is possible inside a `do` block. It won't hurt you. But I want you to know that, unlike in other functional languages, I/O in Haskell is not a hack that breaks the pure functional nature.

Monadic `do` blocks really look like chunks of imperative code! They also behave like imperative code: think of those lines of code as _statements_ that are executed one after snother. This similarity is no coincidence -- all imperative programming _is_ at its core monadic. An imperative programmer learning Haskell might be as shocked as Molière's Bourgeois Gentleman upon discovering that all his life he's been speaking prose. In Haskell this "imperative prose" is implemented using "functional poetry." In the case of the IO monad, this is more of a Taoist statement, since IO action are "pure" functions operating on the Universe. The story passed from generation to generation goes like that:

1. Each I/O function produces a monadic IO action that takes a Universe as input and returns another Universe as output (sometimes with a piece of data attached to it -- as with input actions).
2. The `do` block glues together these actions in such a way that the Universe produced by one action becomes the input to the next action. 

So the sequencing is a result of hidden data dependency -- data being the Universe in this case.

The way the actions are glued together is the essence of the Monad. Since the glueing happens between the lines, the Monad is sometimes described as an "overloading of the semicolon." Different monads overload it differently.

## Conclusion

So is the Monad really about sequencing? Pretty much so, although it is wise to hedge one's bets:
```
Monad that is about sequencing is not the eternal monad
 -- Bartosz Milewski
```
You will learn much more about monads in the following tutorials. You'll also see many more examples and eventually you'll develop a very good intuition for it.

## Exercises

1.  Define a function `putStrLn'` using `putStr` and `putChar`; the latter to output the newline, `'\n'`. (Replace `undefined` with actual code).
    ``` active haskell
    putStrLn' str = undefined

    main = putStrLn' "A whole line of text!"
    ```
    @@@ Show solution
    ``` active haskell
    putStrLn' str = do
        putStr str
        putChar '\n'
    
    main = putStrLn' "A whole line of text!"
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

Unless otherwise attributed, the quotations come from these two books:

1. Lao Tzu, [Tao Te Ching](http://www.amazon.com/Tao-Te-Ching-Stephen-Mitchell/dp/0060812451), Stephen Mitchell, translator. This translation is easier to read, but it contains Mitchell's own interpretations.
2. Lao Tzu, [Tao Te Ching](http://www.amazon.com/Tao-Te-Ching-Lao-Tzu/dp/1590305469), Stephen Addiss and Stanley Lombardo, translators. This translation is closer to the original, but is very terse and, at times, cryptic.

I also took my spiritual guidance from this article:

1. Eugenio Moggi, [Notions of computation and monads](http://www.disi.unige.it/person/MoggiE/ftp/ic91.pdf).


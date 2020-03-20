# Fun with lazy evaluation and I/O actions

# Lazy evaluation

Haskell doesn't evaluate most expressions until it absolutely has to, which has interesting consequences.  Among them is the
ability to create what would be an infinite data structure in most other languages:

``` active haskell
numbers = [1 .. ]

ones = 1 : ones

main = do
    print (take 10 numbers)
    print (take 10 ones)
```

Here's a fun one:

```active haskell
fibonacci = 1 : 1 : zipWith (+) fibonacci (tail fibonacci)

main = do
    print (take 10 fibonacci)
```

and another way to write it that might be clearer:

``` active haskell
{-# LANGUAGE ParallelListComp #-}
fibonacci = 1:fib1
fib1 = 1:fib2
fib2 = [ xn + xnp 
       | xn <- fibonacci
       | xnp <- fib1 ]

main = do
    print (take 10 fibonacci)
```

## Explanation of parallel list comprehensions and LANGUAGE pragmas

- The `{-# LANGUAGE ... #-}` line is a specially formatted comment called a *pragma* that enables certain features in GHC that aren't part of the official Haskell standard yet.

- The list comprehension there is *parallel* which means that each `xn` pulled from the list `fibonacci` is paired up with one `xnp` pulled from the list `fib1`.  This useful notation isn't part of the original Haskell standard, so GHC only accepts it if you include the pragma. 

- You can also do list comprehnsions that do a cross product: `[... | x <- xs, y <- ys]` will produce a list item for each `x` paired with each possible `y`.

Here's an example

``` active haskell
{-# LANGUAGE ParallelListComp #-}

parComp = [ (x, y) 
          | x <- [1 .. 7] 
          | y <- ["a","b","c","d","e","f","g"] ]
          
crossComp = [ (x, y) 
            | x <- [1 .. 7], 
              y <- ["a","b","c","d","e","f","g"] ]

main = do
    print parComp
    print crossComp
```


# Side effects, order of actions, and the IO monad

Haskell programs are run via graph reduction, and parts of the expression graph are evaluated in no particular order, in analogy to arithmetic.  So for example:

``` haskell
n = (1 + 2) * (3 + 4)
```

You get the same result whether you reduce `(1+2)` to `3` first then reduce `(3+4)` to `7`, or if you do `(3+4)` first.

Which should leave you wondering how all the `print` statements that we've been using always generate output in the correct order:

``` active haskell
main = do
    print "this"
    print "that"
```
The output is magically always the same: `"this"` followed by `"that"`.

Clearly these `do` blocks are doing some kind of special magic, but what is it?

The answer relies on the fact that the structure of the expression graph still imposes an order of operations:  In the `(1+2)*(3+4)` example, the multiplication `*` can't happen until the results of both additions have been evaluated.  This means that if we want two actions (=functions with physical side effects) to happen in a particular order, we have to set things up so that the second one can't be evaluated until the first yields a result.

Which leads to the following idea: a program consisting of `print "this"` followed by `print "that"` needs to compile into some sort of structure so that `print "that"` can't be evaluated without some return value from `print "this"`.  So for example, we might try passing around world state objects, something like this: 

``` haskell
print message world = ...
```
where the `...` first ensures that the `world` is evaluated, then does whatever primitive operations are required to do the printing and, only after those operations are complete, returns a pair `(retVal, newWorld)`.

Some terminology: A function `f x` is *strict* in its argument `x` if the value of `f x` cannot be determined without the value of `x` being fully determined.  Integer arithmetic is strict in all of its arguments (except for a few special cases like `0 * x`).  A function can also be partially strict.  For example, the `length` function is strict in the nodes of the list you give it, but not on the elements contained in those nodes.  So the compiler is designed to treat I/O primitive operations as strict in their `world` argument.  Chaining functions by passing around new world states is how I/O actions will be forced to run in sequence, and the `retVal` is so that I/O actions can return values such as the contents of a file, success vs. failure etc.

That means that sequences of calls to `print`  might look something like this under the hood:

``` haskell
main world0 =
    let
        (retVal1, world1) = print "this" world0
        (retVal2, world2) = print "that" world1
     in (retVal2, world2)
```

To get the final value out of `main`, the compiler has to evaluate `world2`, which requires evaluating `print "that"`, which requires evaluating `world1`, which requires evaluating `print "this"`.  You do have to have some help from the compiler and run-time so that primitive I/O actions impose evaluation constraints on their `world` parameter, but given that, sequences of several I/O actions happen in the right order automatically.

The magic `do` syntax is interpreted as follows:

``` haskell
do
    action1
    action2
    action3 ...
```
becomes
``` haskell
action1 >> action2 >> action3 ...
```
where `>>` is a sequencing operator.  It takes two I/O actions and returns a new action that forces them to happen in sequence.  Its definition is roughly:
``` haskell
(a1 >> a2) world0 =
    let
        (retVal1, world1) = a1 world0
        (retVal2, world2) = a2 world1
    in
        (retVal2, world2)
```

The `>>` suffices for sequencing output, but what about reading input, or doing something other than discarding the return value from an action?  That's done by passing the return value to the next function along with the world state.  Here's how to open and write to a file:

``` haskell
import System.IO

main = do
    h <- openFile "message.txt" WriteMode
    hPutStrLn h "Greetings earthlings"
    hClose h
```

The `<-` arrow notation in a `do` block is translated from
``` haskell
do
    x <- action1
    action2 x a b c
```
into a call to the bind operator `>>=`
``` haskell
action1 >>= \x -> action2 x a b c
```
and the definition of `>>=` is roughly
``` haskell
(generatingAction >>= receivingAction) world0 =
    let
        (retVal1, world1) = generatingAction world0
        (retVal2, world2) = receivingAction retVal1 world1
    in
        (retVal2, world2)
```
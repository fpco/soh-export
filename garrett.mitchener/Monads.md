# Monads

# Class Monad

Class `Monad` is defined as follows:
``` haskell
-- from Control.Monad that comes with GHC
class  Monad m  where
    -- | Sequentially compose two actions, passing any value produced
    -- by the first as an argument to the second.
    (>>=)       :: forall a b. m a -> (a -> m b) -> m b
    -- | Sequentially compose two actions, discarding any value produced
    -- by the first, like sequencing operators (such as the semicolon)
    -- in imperative languages.
    (>>)        :: forall a b. m a -> m b -> m b
        -- Explicit for-alls so that we know what order to
        -- give type arguments when desugaring

    -- | Inject a value into the monadic type.
    return      :: a -> m a
    -- | Fail with a message.  This operation is not part of the
    -- mathematical definition of a monad, but is invoked on pattern-match
    -- failure in a @do@ expression.
    fail        :: String -> m a

    {-# INLINE (>>) #-}
    m >> k      = m >>= \_ -> k
    fail s      = error s
```

The two key functions are `>>=` and `return`.  Note that `>>`, which sequences two actions, is given a default definition at the bottom.  Instances of `Monad` don't have to implement `>>` if that default definition is okay, which it usually is.

# Reminder about do notation

``` haskell
stuff x y z = do
    a <- anAction x
    b <- anotherAction y z
    let c = nifty a b
    d <- if c == 0
        then narfle a
        else burble b
    return (a,b,c,d)
```
translates into
``` haskell
stuff x y z =
    anAction x >>= (\a ->
        anotherAction y z >>= (\b ->
            let
                c = nifty a b
            in (if c == 0 then narfle a else burble b) >>= (\d ->
                return (a,b,c,d)
                ) -- end of \d -> ...
            ) -- end of \b -> ...
        ) -- end of \a -> ...
```

# What return does and doesn't do

We've seen the `IO` monad's bind operator already:
``` haskell
instance Monad IO where
    action1 >>= action2 = ...
        -- perform action1, yielding a result x
        -- perform (action2 x)
    ...
```
but `return` is new.  Note the type, `return :: a -> m a`, so for the `IO` monad, what `return` does is produce `x`, sort of as if it had been read from a file or some other input source, but without actually doing anything to the real world.  Here's one reason you might need `return`:
``` haskell
analyzeFile :: FilePath -> IO String

analyzeFile path = do
    h <- openFile path ReadMode
    result <- ...
    hClose h
    return result
    
main = do
    result <- analyzeFile "/etc/stuff"
    ...
```
That is, you want to output something from an action, but you have to perform other actions after the construction of the result but before you consider your action complete.  Note that unlike in Python, C, C++, Java, and so many other languages, `return` is *not* a keyword, it doesn't generally cause exectution to exit from a procedure early, and you don't need it for a function that isn't a monadic action.

# About the unit type and value ()

Haskell functions have to result in some value.  There is no `void` as in C and Java.  However, sometimes an action has nothing useful to produce.  Haskell's convention is the unit type `()` which suggests a 0-element tuple.  The type of `()` is also called `()`.  Here's a standard monadic conditional function:
``` haskell
when :: Monad m => Bool -> m () -> m ()
when t action =
    if t
        then action
        else return ()
```
If the given test value `t` is `True`, perform the action, otherwise, do nothing.  And since the result of `when` may be an action that does nothing and produces nothing, the action you pass it to do when `t == True` must also produce nothing. (Both branches of an `if` must have the same type.)

# Famous monad

What follows is a rough sketch of several other monads that are built in to the Haskell standard library.  They aren't exactly as I've presented here: I've left out a lot of complications because this article is about the concepts rather covering every tiny nasty little detail.  Look at the modules `Control.Monad.*` for full details and references to research papers and other articles.

## The State monad

### Threading state

Since you generally create new data structures rather than modifying them, it's common to have stuff like this in Haskell:
``` haskell

doTheWork x y z =
    let
        thing0 = ...
        (a, thing1) = doStuff x thing0
        (b, thing2) = doMoreStuff y thing1
        (c, thing3) = doEvenMoreStuff z thing2
    in
        (..., thing3)
```
Common examples include adding things to a `Data.Set` or other container, and random number generation.

### An easy random number generator

So, let's write a linear congruential generator.  This isn't an especially good generator--Mersenne twister would be better for example--but it'll serve as an example.
``` active haskell
import Data.Word

data LCGState = LCGState Word32 Word32 Word32 Word32
    deriving (Read, Show, Eq, Ord)

-- This is apparently what glibc uses
makeSeed x = LCGState (2 ^ 31) 1103515245 12345 x

nextState :: LCGState -> LCGState
nextState (LCGState m a c x1) = LCGState m a c x2
    where x2 = (a * x1 + c) `mod` m

-- | Pull a random number between 0 and 1
nextRandomDouble :: LCGState -> (Double, LCGState)
nextRandomDouble state0 =
    let
        LCGState m a c x = state0
        state1 = nextState state0
    in
        ((fromIntegral x) / (fromIntegral m), state1)

s0 = makeSeed 6502
(y1, s1) = nextRandomDouble s0
(y2, s2) = nextRandomDouble s1
(y3, s3) = nextRandomDouble s2

main = do
    print s0
    print [y1, y2, y3]
```
It would be nice not to have to keep threading the random seed through all those calculations.  It turns out that this type of calculation is handled nicely as a state monad.  We want to end up writing stuff like this:
``` haskell
pickThree = do
    y1 <- pullRandomDouble
    y2 <- pullRandomDouble
    y3 <- pullRandomDouble
    return [y1, y2, y3]
```

### Implementing a state monad for the random generator

This is how to implement a state monad:
``` active haskell
-- This is a wrapper around a transition function
-- that takes a s(tate) and returns a r(esult) and a new s(tate)
data State s r = State (s -> (r, s))

-- Running a state object on an s is easy, just apply the wrapped function
runState (State f) s = f s

instance Monad (State s) where
    
    -- introduce the result without modifying s
    return result = State (\s -> (result, s))
    
    -- compose two transition functions,
    -- threading the result of the first into the second
    (State fTrans) >>= mkTrans = State hTrans
        where
        hTrans s0 =
            let
                (r1, s1) = fTrans s0
                State gTrans = mkTrans r1
                (r2, s2) = gTrans s1
            in
                (r2, s2)
```
So now we can do random number generation like this:
``` active haskell
import Data.Word

data LCGState = LCGState Word32 Word32 Word32 Word32
    deriving (Read, Show, Eq, Ord)

-- This is apparently what glibc uses
makeSeed x = LCGState (2 ^ 31) 1103515245 12345 x

nextState :: LCGState -> LCGState
nextState (LCGState m a c x1) = LCGState m a c x2
    where x2 = (a * x1 + c) `mod` m

-- | Pull a random number between 0 and 1
nextRandomDouble :: LCGState -> (Double, LCGState)
nextRandomDouble state0 =
    let
        LCGState m a c x = state0
        state1 = nextState state0
    in
        ((fromIntegral x) / (fromIntegral m), state1)



-- This is a wrapper around a transition function
-- that takes a s(tate) and returns a r(esult) and a new s(tate)
data State s r = State (s -> (r, s))

-- Running a state object on an s is easy, just apply the wrapped function
runState (State f) s = f s

instance Monad (State s) where
    
    -- introduce the result without modifying s
    return result = State (\s -> (result, s))
    
    -- compose two transition functions,
    -- threading the result of the first into the second
    (State fTrans) >>= mkTrans = State hTrans
        where
        hTrans s0 =
            let
                (r1, s1) = fTrans s0
                State gTrans = mkTrans r1
                (r2, s2) = gTrans s1
            in
                (r2, s2)
pullRandomDouble :: State LCGState Double
pullRandomDouble = State nextRandomDouble

pickThree :: State LCGState [Double]
pickThree = do
    y1 <- pullRandomDouble
    y2 <- pullRandomDouble
    y3 <- pullRandomDouble
    return [y1, y2, y3]

s0 = makeSeed 6502
(ys, sNext) = runState pickThree s0

main = do
    print ys
```
The GHC library includes an implementation of the state monad very much like the one here.  See the `Control.Monad.State` module.

## Exercises

Use this workspace:
``` active haskell
--show Imports
import Data.Word
import Control.Monad
--/show

--show Random number generator
data LCGState = LCGState Word32 Word32 Word32 Word32
    deriving (Read, Show, Eq, Ord)

-- This is apparently what glibc uses
makeSeed x = LCGState (2 ^ 31) 1103515245 12345 x

nextState :: LCGState -> LCGState
nextState (LCGState m a c x1) = LCGState m a c x2
    where x2 = (a * x1 + c) `mod` m

-- | Pull a random number between 0 and 1
nextRandomDouble :: LCGState -> (Double, LCGState)
nextRandomDouble state0 =
    let
        LCGState m a c x = state0
        state1 = nextState state0
    in
        ((fromIntegral x) / (fromIntegral m), state1)
--/show


--show State monad
-- This is a wrapper around a transition function
-- that takes a s(tate) and returns a r(esult) and a new s(tate)
data State s r = State (s -> (r, s))

-- Running a state object on an s is easy, just apply the wrapped function
runState (State f) s = f s

instance Monad (State s) where
    
    -- introduce the result without modifying s
    return result = State (\s -> (result, s))
    
    -- compose two transition functions,
    -- threading the result of the first into the second
    (State fTrans) >>= mkTrans = State hTrans
        where
        hTrans s0 =
            let
                (r1, s1) = fTrans s0
                State gTrans = mkTrans r1
                (r2, s2) = gTrans s1
            in
                (r2, s2)
--/show

--show More friendly random number generation

-- | Generate a random number uniformly between 0 and 1
pullRandomDouble :: State LCGState Double
pullRandomDouble = State nextRandomDouble

-- Your job, see below:

pullRandomBool = undefined

pullDieToss = undefined

pullDieTosses = undefined

--/show

--show Test cases
pickThree :: State LCGState [Double]
pickThree = do
    y1 <- pullRandomDouble
    y2 <- pullRandomDouble
    y3 <- pullRandomDouble
    return [y1, y2, y3]

s0 = makeSeed 6502
(ys, sNext) = runState pickThree s0

main = do
    print ys
--/show
```

### Flip a coin (Bernouli random variables)
Write a monadic function `pullRandomBool :: Double -> State LCGState Double` that takes a number q between 0 and 1 and generates `True` with probability q, and `False` with probability (1-q).

### Toss a die
Write a monadic function `pullDieToss :: Int -> State LCGState Int` that takes a number n (= number of sides on the die) and generates a random number between 1 and n, uniformly.

### Many dice
Look up `Control.Monad` on the Haskell web site.  Look for a function called `replicateM`.  Use it to write a function `pullDieTosses :: Int -> Int -> State LCGState [Int]` that takes a number k and a number n, and generates a list of k rolls of a die with n sides.



## The Maybe monad

Recall that the `Maybe` type represents something that might not exist, like a pointer that is allowed to be null:
``` haskell
data Maybe t = Nothing | Just t
```
Maybeness also has monadic semantics, and I like to think of the `>>=` operator as 'feeding' a value to a function.  Feeding `Nothing` to any function results in `Nothing`.  Feeding `Just x` to a function applies it to `x`, and the result might be an actual value `Just y` or `Nothing`.
``` haskell
instance Monad Maybe where
    return x = Just x
    
    Nothing >>= f  = Nothing
    Just x >>= f  = f x
```
Using `Maybe` monadically is a clean way to write a sequence of operations that could fail (resulting in `Nothing`) at any step.

You can also create an error monad, which is very similar but instead of `Nothing`, you return something describing the failure.  You can use that as a means of throwing and catching exceptions, although Haskell can also do that within the IO monad.  See `Control.Monad.Error`.


## The List monad

A list has monadic sematics, too.  If a `Maybe t` is either one value `Just x` or `Nothing`, you can think of a list as zero or more possible values.  Returning a value to the list monad produces a list of that one value.  Binding a list of possible values to a function means applying that function to each possible value, and building one big list out of all the possible results it returns.
``` haskell
instance Monad [] where
    return x = [x]
    
    xs >>= f =
        concat (map f xs)
```
And actually, the list comprehension notation is transformed into monadic binding during compilation:
``` active haskell
ns = [1 .. 5]

squares = [n^2 | n <- ns]

squares2 = do
    n <- ns
    return (n^2)

-- all possible pairings of ns and squares:
pairs = [(n, m) | n <- ns, m <- squares]

-- all possible pairings of ns and squares:
pairs2 = do
    n <- ns
    m <- squares
    return (n, m)
    
main = do
    print ns
    print squares
    print squares2
    print pairs
    print pairs2
```


## The Identity monad

Plain old function application is also a monad, called the identity monad.  All we have to do is wrap it:
``` haskell
data Identity t = Identity t

instance Monad Identity where

    return x = Identity x
    
    (Identity x) >>= f = f x
```
Why would you ever both with that? Well...


## Monad transformers

... it turns out there are ways to convert a monad into a *monad transformer*, which then adds properties to another monad.  So, there's a `StateT` that adds a state item to another monad.  There's `ErrorT` that adds exception handling capabilities.  So in the Haskell standard libary, the plain old `State` monad is defined by applying `StateT` to `Identity` and `Error` is defined by applying `ErrorT` to `Identity`.  There's also a `ListT` that adds backtracking, `ReaderT`, `WriterT`, ... and you can combine them in all sorts of ways.

We don't have time for all of that here.


## ST

The state thread monad `ST` is kind of like `State`: It performs sequences of operations that send state around, but the state itself is an abstraction that is never made available.  In fact, it's essentially an abstraction of the entire machine memory.  The use of `ST` is more like `IO` than `Stat`: It provides a way of sequencing imperative-like operations, such as changing a data structure in place.  However, you can run many `ST` actions in any part of the program, and the actions aren't as strictly ordered as in `IO`. There's normally only one sequence of `IO` actions (the `main` function).  And you can peek at `Control.Monad.ST` and see how GHC implements `IO` in terms of `ST`.

The weirdness of `ST` is a feature called *rank-n types*.  In this case it refers to a phantom unspecified type that represents the abstract world state to be threaded, and because of how rank-*n* types work, each call to `runST` that executes a sequence of `ST` actions gets its own abstract world state.  This bit of madness is needed to make sure that, for example, a mutable array under construction inside `runST` in one place can't be deallocated by another sequence of actions running inside another `runST`.  (Remember that Haskell can evaluate a program in any order, so it's allowed to do some work on one `ST` action, then turn its attention to another.)


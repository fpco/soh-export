# Pipes tutorial

# Overview

Conventional Haskell stream programming forces you to choose only two of the
following three features:

* Effects

* Streaming

* Composability

If you sacrifice *Effects* you get Haskell's pure and lazy lists, which you
can transform using composable functions in constant space, but without
interleaving effects.

If you sacrifice *Streaming* you get `mapM`, `forM` and
"ListT done wrong", which are composable and effectful, but do not return
a single result until the whole list has first been processed and loaded
into memory.

If you sacrifice *Composability* you write a tightly coupled read,
transform, and write loop in `IO`, which is streaming and effectful, but is
not modular or separable.

`pipes` gives you all three features: effectful, streaming, and composable
programming.  `pipes` also provides a wide variety of stream programming
abstractions which are all subsets of a single unified machinery:

* effectful `Producer`s (like generators),

* effectful `Consumer`s (like iteratees),

* effectful `Pipe`s (like Unix pipes), and:

* `ListT` done right.

All of these are connectable and you can combine them together in clever and
unexpected ways because they all share the same underlying type.

`pipes` requires a basic understanding of monad transformers, which you can
learn about by reading either:

* the paper "Monad Transformers - Step by Step",

* chapter 18 of "Real World Haskell" on monad transformers, or:

* the documentation of the `transformers` library.

If you want a Quick Start guide to `pipes`, read
[the documentation in `Pipes.Prelude`](http://hackage.haskell.org/package/pipes-4.1.0/docs/Pipes-Prelude.html)
from top to bottom.

This tutorial is more extensive and explains the `pipes` API in greater
detail and illustrates several idioms.

# Introduction

The `pipes` library decouples stream processing stages from each other so
that you can mix and match diverse stages to produce useful streaming
programs.  If you are a library writer, `pipes` lets you package up
streaming components into a reusable interface.  If you are an application
writer, `pipes` lets you connect pre-made streaming components with minimal
effort to produce a highly-efficient program that streams data in constant
memory.

To enforce loose coupling, components can only communicate using two
commands:

* `yield`: Send output data

* `await`: Receive input data

`pipes` has four types of components built around these two commands:

* `Producer`s can only `yield` values and they model streaming sources

* `Consumer`s can only `await` values and they model streaming sinks

* `Pipe`s can both `yield` and `await` values and they model stream
  transformations

* `Effect`s can neither `yield` nor `await` and they model non-streaming
  components

You can connect these components together in four separate ways which
parallel the four above types:

* `for` handles `yield`s

* `(>~)` handles `await`s

* `(>->)` handles both `yield`s and `await`s

* `(>>=)` handles return values

As you connect components their types will change to reflect inputs and
outputs that you've fused away.  You know that you're done connecting things
when you get an `Effect`, meaning that you have handled all inputs and
outputs.  You run this final `Effect` to begin streaming.

# Producers

`Producer`s are effectful streams of input.  Specifically, a `Producer` is a
monad transformer that extends any base monad with a new `yield` command.
This `yield` command lets you send output downstream to an anonymous
handler, decoupling how you generate values from how you consume them.

The following `stdinLn` `Producer` shows how to incrementally read in
`String`s from standard input and `yield` them downstream, terminating
gracefully when reaching the end of the input:

```haskell
import Control.Monad (unless)
import Pipes
import System.IO (isEOF)

--         +--------+-- A 'Producer' that yields 'String's
--         |        |
--         |        |      +-- Every monad transformer has a base monad.
--         |        |      |   This time the base monad is 'IO'.
--         |        |      |  
--         |        |      |  +-- Every monadic action has a return value.
--         |        |      |  |   This action returns '()' when finished
--         v        v      v  v
stdinLn :: Producer String IO ()
stdinLn = do
    eof <- lift isEOF        -- 'lift' an 'IO' action from the base monad
    unless eof $ do
        str <- lift getLine
        yield str            -- 'yield' the 'String'
        stdinLn              -- Loop
```

`yield` emits a value, suspending the current `Producer` until the value is
consumed.  If nobody consumes the value (which is possible) then `yield`
never returns.  You can think of `yield` as having the following type:

```haskell
yield :: Monad m => a -> Producer a m ()
```

The true type of `yield` is actually more general and powerful.  Throughout
the tutorial I will present type signatures like this that are simplified at
first and then later reveal more general versions.  So read the above type
signature as simply saying: "You can use `yield` within a `Producer`, but
you may be able to use `yield` in other contexts, too."

If you navigate to the
[documentation for `yield`](http://hackage.haskell.org/package/pipes-4.1.0/docs/Pipes.html#v:yield)
you will
see that `yield` actually uses the `Producer'` (with an apostrophe) type
synonym which hides a lot of polymorphism behind a simple veneer.  The
documentation for `yield` says that you can also use `yield` within a
`Pipe`, too, because of this polymorphism:

```haskell
yield :: Monad m => a -> Pipe x a m ()
```

Use simpler types like these to guide you until you understand the fully
general type.

`for` loops are the simplest way to consume a `Producer` like `stdinLn`.
`for` has the following type:

```haskell
--                +-- Producer      +-- The body of the   +-- Result
--                |   to loop       |   loop              |
--                v   over          v                     v
--                --------------    ------------------    ----------
for :: Monad m => Producer a m r -> (a -> Effect m ()) -> Effect m r
```

`for producer body` loops over `producer`, substituting each `yield` in
`producer` with `body`.

You can also deduce that behavior purely from the type signature:

* The body of the loop takes exactly one argument of type `a`, which is
  the same as the output type of the `Producer`.  Therefore, the body of the
  loop must get its input from that `Producer` and nowhere else.

* The return value of the input `Producer` matches the return value of the
  result, therefore `for` must loop over the entire `Producer` and not skip
  anything.

The above type signature is not the true type of `for`, which is actually
more general.  Think of the above type signature as saying: "If the first
argument of `for` is a `Producer` and the second argument returns an
`Effect`, then the final result must be an `Effect`."

If you navigate to the
[documentation for `for`](http://hackage.haskell.org/package/pipes-4.1.0/docs/Pipes.html#v:for)
you will
see the fully general type and underneath you will see equivalent simpler
types.  One of these says that if the body of the loop is a `Producer`, then
the result is a `Producer`, too:

```haskell
for :: Monad m => Producer a m r -> (a -> Producer b m ()) -> Producer b m r
```

The first type signature I showed for `for` was a special case of this
slightly more general signature because a `Producer` that never `yield`s is
also an `Effect`:

```haskell
data X  -- The uninhabited type

type Effect m r = Producer X m r
```

This is why `for` permits two different type signatures.  The first type
signature is just a special case of the second one:

```haskell
for :: Monad m => Producer a m r -> (a -> Producer b m ()) -> Producer b m r

-- Specialize 'b' to 'X'
for :: Monad m => Producer a m r -> (a -> Producer X m ()) -> Producer X m r

-- Producer X = Effect
for :: Monad m => Producer a m r -> (a -> Effect     m ()) -> Effect     m r
```

This is the same trick that all `pipes` functions use to work with various
combinations of `Producer`s, `Consumer`s, `Pipe`s, and `Effect`s.  Each
function really has just one general type, which you can then simplify down
to multiple useful alternative types.

Here's an example use of a `for` loop, where the second argument (the
loop body) is an `Effect`:

```haskell
loop :: Effect IO ()
loop = for stdinLn $ \str -> do  -- Read this like: "for str in stdinLn"
    lift $ putStrLn str          -- The body of the 'for' loop

-- more concise: loop = for stdinLn (lift . putStrLn)
```

In this example, `for` loops over `stdinLn` and replaces every `yield` in
`stdinLn` with the body of the loop, printing each line.  This is exactly
equivalent to the following code, which I've placed side-by-side with the
original definition of `stdinLn` for comparison:

```haskell
loop = do
    eof <- lift isEOF
    unless eof $ do
        str <- lift getLine
        {-hi-}(lift . putStrLn){-/hi-} str
        loop

stdinLn = do
    eof <- lift isEOF
    unless eof $ do
        str <- lift getLine
        {-hi-}yield{-/hi-} str
        stdinLn
```

You can think of `yield` as creating a hole and a `for` loop is one way to
fill that hole.

Notice how the final `loop` only `lift`s actions from the base monad and
does nothing else.  This property is true for all `Effect`s, which are just
glorified wrappers around actions in the base monad.  This means we can run
these `Effect`s to remove their `lift`s and lower them back to the
equivalent computation in the base monad:

```haskell
runEffect :: Monad m => Effect m r -> m r
```

This is the real type signature of `runEffect`, which refuses to accept
anything other than an `Effect`.  This ensures that we handle all inputs and
outputs before streaming data:

```haskell
main :: IO ()
main = runEffect loop
```

... or you could inline the entire `loop` into the following one-liner:

```active haskell
import Control.Monad (unless)
import Pipes
import System.IO (isEOF)

stdinLn :: Producer String IO ()
stdinLn = do
    eof <- lift isEOF
    unless eof $ do
        str <- lift getLine
        yield str
        stdinLn
-- show
-- Try me!
main = runEffect $ for stdinLn (lift . putStrLn)
```

Run the above program and it will loop over standard input and echo every line to
standard output.  If you run the program from the command line instead of School of
Haskell you can also test how the program handles end of input:

```
$ ./echo
Test<Enter>
Test
ABC<Enter>
ABC
<Ctrl-D>
$
```

The final behavior is indistinguishable from just removing all the `lift`s
from `loop`:

```haskell
main = do
    eof <- isEOF
    unless eof $ do
        str <- getLine
        putStrLn str
        main

loop = do
    eof <- {-hi-}lift{-/hi-} isEOF
    unless eof $ do
        str <- {-hi-}lift{-/hi-} getLine
        {-hi-}lift{-/hi-} $ putStrLn str
        main
```

This `main` is what we might have written by hand if we were not using
`pipes`, but with `pipes` we can decouple the input and output logic from
each other.  When we connect them back together, we still produce streaming
code equivalent to what a sufficiently careful Haskell programmer would
have written.

You can also use `for` to loop over lists, too.  To do so, convert the list
to a `Producer` using `each`, which is exported by default from the `Pipes`
module:

```haskell
each :: Monad m => [a] -> Producer a m ()
each as = mapM_ yield as
```

Combine `for` and `each` to iterate over lists using a "foreach" loop:

```active haskell
import Pipes

main = runEffect $ for (each [1..4]) (lift . print)
```

`each` is actually more general and works for any `Foldable`:

```haskell
each :: (Monad m, Foldable f) => f a -> Producer a m ()
```

So you can loop over any `Foldable` container or even a `Maybe`:

```active haskell
import Pipes

main = runEffect $ for (each (Just 1)) (lift . print)
```

# Producers

You might wonder why the body of a `for` loop can be a `Producer`.  Let's
test out this feature by defining a new loop body that `duplicate`s every
value:

```haskell
import Pipes
import qualified Pipes.Prelude as Pipes  -- Pipes.Prelude already has 'stdinLn'
 
duplicate :: Monad m => a -> Producer a m ()
duplicate x = do
    yield x
    yield x

loop :: Producer String IO ()
loop = for Pipes.stdinLn duplicate

-- This is the exact same as:
--
-- loop = for P.stdinLn $ \x -> do
--     yield x
--     yield x
```

This time our `loop` is a `Producer` that outputs `String`s, specifically
two copies of each line that we read from standard input.  Since `loop` is a
`Producer` we cannot run it because there is still unhandled output.
However, we can use yet another `for` to handle this new duplicated stream:

```active haskell
import Pipes
import qualified Pipes.Prelude as Pipes  -- Pipes.Prelude already has 'stdinLn'
 
duplicate :: Monad m => a -> Producer a m ()
duplicate x = do
    yield x
    yield x

loop :: Producer String IO ()
loop = for Pipes.stdinLn duplicate
-- show
main = runEffect $ for loop (lift . putStrLn)
```

Run the above program, which will echo every line from standard input to
standard output twice.

However, are is this really necessary?  Couldn't we have instead written this using a
nested for loop?

```haskell
main = runEffect $
    for Pipes.stdinLn $ \str1 ->
        for (duplicate str1) $ \str2 ->
            lift $ putStrLn str2
```

Yes, we could have!  In fact, this is a special case of the following
equality, which always holds no matter what:

```haskell
s :: Monad m =>      Producer a m ()  -- i.e. Pipes.stdinLn
f :: Monad m => a -> Producer b m ()  -- i.e. duplicate
g :: Monad m => b -> Producer c m ()  -- i.e. lift . putStrLn

for (for s f) g = for s (\x -> for (f x) g)
```

We can understand the rationale behind this equality if we first define the
following operator that is the point-free counterpart to `for`:

```haskell
 (~>) :: Monad m
      => (a -> Producer b m r)
      -> (b -> Producer c m r)
      -> (a -> Producer c m r)
 (f ~> g) x = for (f x) g
```

Using `(~>)` (pronounced "into"), we can transform our original equality
into the following more symmetric equation:

```haskell
f :: Monad m => a -> Producer b m r
g :: Monad m => b -> Producer c m r
h :: Monad m => c -> Producer d m r

-- Associativity
(f ~> g) ~> h = f ~> (g ~> h)
```

This looks just like an associativity law.  In fact, `(~>)` has another nice
property, which is that `yield` is its left and right identity:

```haskell
-- Left Identity
yield ~> f = f

-- Right Identity
f ~> yield = f
```

In other words, `yield` and `(~>)` form a `Category`, specifically the
generator category, where `(~>)` plays the role of the composition operator
and `yield` is the identity.  If you don't know what a `Category` is, that's
okay, and category theory is not a prerequisite for using `pipes`.  All you
really need to know is that `pipes` uses some simple category theory to keep
the API intuitive and easy to use.

Notice that if we translate the left identity law to use `for` instead of
`(~>)` we get:

```haskell
for (yield x) f = f x
```

This just says that if you iterate over a pure single-element `Producer`,
then you could instead cut out the middle man and directly apply the body of
the loop to that single element.

If we translate the right identity law to use `for` instead of (`~>`) we
get:

```haskell
for s yield = s
```

This just says that if the only thing you do is re-`yield` every element of
a stream, you get back your original stream.

These three "for loop" laws summarize our intuition for how `for` loops
should behave and because these are `Category` laws in disguise that means
that `Producer`s are composable in a rigorous sense of the word.

In fact, we get more out of this than just a bunch of equations.  We also
get a useful operator: `(~>)`.  We can use this operator to condense
our original code into the following more succinct form that composes two
transformations:

```haskell
main = runEffect $ for Pipes.stdinLn (duplicate ~> lift . putStrLn)
```

This means that we can also choose to program in a more functional style and
think of stream processing in terms of composing transformations using
`(~>)` instead of nesting a bunch of `for` loops.

The above example is a microcosm of the design philosophy behind the `pipes`
library:

* Define the API in terms of categories

* Specify expected behavior in terms of category laws

* Think compositionally instead of sequentially

# Consumers

Sometimes you don't want to use a `for` loop because you don't want to consume
every element of a `Producer` or because you don't want to process every
value of a `Producer` the exact same way.

The most general solution is to externally iterate over the `Producer` using
the `next` command:

```haskell
next :: Monad m => Producer a m r -> m (Either r (a, Producer a m r))
```

Think of `next` as pattern matching on the head of the `Producer`.  This
`Either` returns a `Left` if the `Producer` is done or it returns a `Right`
containing the next value, `a`, along with the remainder of the `Producer`.

However, sometimes we can get away with something a little more simple and
elegant, like a `Consumer`, which represents an effectful sink of values.  A
`Consumer` is a monad transformer that extends the base monad with a new
`await` command. This `await` command lets you receive input from an
anonymous upstream source.

The following `stdoutLn` `Consumer` shows how to incrementally `await`
`String`s and print them to standard output, terminating gracefully when
receiving a broken pipe error:

```haskell
import Control.Monad (unless)
import Control.Exception (try, throwIO)
import qualified GHC.IO.Exception as G
import Pipes

--          +--------+-- A 'Consumer' that awaits 'String's
--          |        |
--          v        v
stdoutLn :: Consumer String IO ()
stdoutLn = do
    str <- await  -- 'await' a 'String'
    x   <- lift $ try $ putStrLn str
    case x of
        -- Gracefully terminate if we got a broken pipe error
        Left e@(G.IOError { G.ioe_type = t}) ->
            lift $ unless (t == G.ResourceVanished) $ throwIO e
        -- Otherwise loop
        Right () -> stdoutLn
```

`await` is the dual of `yield`: we suspend our `Consumer` until we receive a
new value.  If nobody provides a value (which is possible) then `await`
never returns.  You can think of `await` as having the following type:

```haskell
await :: Monad m => Consumer a m a
```

One way to feed a `Consumer` is to repeatedly feed the same input using
using `(>~)` (pronounced "feed"):

```haskell
--                 +- Feed       +- Consumer to    +- Returns new
--                 |  action     |  feed           |  Effect
--                 v             v                 v  
--                 ----------    --------------    ----------
(>~) :: Monad m => Effect m b -> Consumer b m c -> Effect m c
```

`draw >~ consumer` loops over `consumer`, substituting each `await` in
`consumer` with `draw`.

So the following code replaces every `await` in `stdoutLn` with
`lift getLine` and then removes all the `lift`s:

```active haskell
import Control.Monad (unless)
import Control.Exception (try, throwIO)
import qualified GHC.IO.Exception as G
import Pipes

stdoutLn :: Consumer String IO ()
stdoutLn = do
    str <- await
    x   <- lift $ try $ putStrLn str
    case x of
        Left e@(G.IOError { G.ioe_type = t}) ->
            lift $ unless (t == G.ResourceVanished) $ throwIO e
        Right () -> stdoutLn
-- show
main = runEffect $ lift getLine >~ stdoutLn
```

Run the above program and it will echo standard input to standard input.  The
difference is that this time it checks for a broken output pipe instead of end
of input.

You might wonder why (`>~`) uses an `Effect` instead of a raw action in the
base monad.  The reason why is that `(>~)` actually permits the following
more general type:

```haskell
(>~) :: Monad m => Consumer a m b -> Consumer b m c -> Consumer a m c
```

`(>~)` is the dual of `(~>)`, composing `Consumer`s instead of `Producer`s.

This means that you can feed a `Consumer` with yet another `Consumer` so
that you can `await` while you `await`.  For example, we could define the
following intermediate `Consumer` that requests two `String`s and returns
them concatenated:

```haskell
doubleUp :: Monad m => Consumer String m String
doubleUp = do
    str1 <- await
    str2 <- await
    return (str1 ++ str2)

-- more concise: doubleUp = (++) <$> await <*> await
```

We can now insert this in between `lift getLine` and `stdoutLn` and see
what happens:

```active haskell
import Control.Monad (unless)
import Control.Exception (try, throwIO)
import qualified GHC.IO.Exception as G
import Pipes

stdoutLn :: Consumer String IO ()
stdoutLn = do
    str <- await
    x   <- lift $ try $ putStrLn str
    case x of
        Left e@(G.IOError { G.ioe_type = t}) ->
            lift $ unless (t == G.ResourceVanished) $ throwIO e
        Right () -> stdoutLn

doubleUp :: Monad m => Consumer String m String
doubleUp = do
    str1 <- await
    str2 <- await
    return (str1 ++ str2)
-- show
main = runEffect $ lift getLine >~ doubleUp >~ stdoutLn
```

If you run the above example it will repeatedly request two lines of input and
output them concatenated.

`doubleUp` splits every request from `stdoutLn` into two separate requests
and returns back the concatenated result.

We didn't need to parenthesize the above chain of `(>~)` operators, because
`(>~)` is associative:

```haskell
-- Associativity
(f >~ g) >~ h = f >~ (g >~ h)
```

... so we can always omit the parentheses since the meaning is unambiguous:

```haskell
f >~ g >~ h
```

Also, `(>~)` has an identity, which is `await`!

```haskell
-- Left identity
await >~ f = f

-- Right Identity
f >~ await = f
```

In other words, `(>~)` and `await` form a `Category`, too, specifically the
iteratee category, and `Consumer`s are also composable.

# Pipes

Our previous programs were unsatisfactory because they were biased either
towards the `Producer` end or the `Consumer` end.  As a result, we had to
choose between gracefully handling end of input (using `stdinLn`) or
gracefully handling end of output (using `stdoutLn`), but not both at the
same time.

However, we don't need to restrict ourselves to using `Producer`s
exclusively or `Consumer`s exclusively.  We can connect `Producer`s and
`Consumer`s directly together using (`>->`) (pronounced "pipe"):

```haskell
(>->) :: Monad m => Producer a m r -> Consumer a m r -> Effect m r
```

This returns an `Effect` which we can run:

```active haskell
import Pipes
import qualified Pipes.Prelude as Pipes  -- Pipes.Prelude also provides 'stdoutLn'

main = runEffect $ Pipes.stdinLn >-> Pipes.stdoutLn
```

This program is more declarative of our intent: we want to stream values
from `stdinLn` to `stdoutLn`.  The above "pipeline" not only echoes
standard input to standard output, but also handles both end of input and
broken pipe errors.

`(>->)` is "pull-based" meaning that control flow begins at the most
downstream component (i.e. `stdoutLn` in the above example).  Any time a
component `await`s a value it blocks and transfers control upstream and
every time a component `yield`s a value it blocks and restores control back
downstream, satisfying the `await`.  So in the above example, `(>->)`
matches every `await` from `P.stdoutLn` with a `yield` from `stdinLn`.

Streaming stops when either `stdinLn` terminates (i.e. end of input) or
`stdoutLn` terminates (i.e. broken pipe).  This is why `(>->)` requires
that both the `Producer` and `Consumer` share the same type of return value:
whichever one terminates first provides the return value for the entire
`Effect`.

Let's test this by modifying our `Producer` and `Consumer` to each return a
diagnostic `String`:

```active haskell
import Control.Applicative ((<$))  -- (<$) modifies return values
import Pipes
import qualified Pipes.Prelude as P
import System.IO

main = do
    hSetBuffering stdout NoBuffering
    str <- runEffect $
        ("End of input!" <$ P.stdinLn) >-> ("Broken pipe!" <$ P.stdoutLn)
    hPutStrLn stderr str
```

If you run this program on the command line you can trigger both termination
scenarios:

```
$ ./echo2
Test<Enter>
Test
<Ctrl-D>
End of input!
$ ./echo2 | perl -e 'close STDIN'
Test<Enter>
Broken pipe!
$
```

You might wonder why `(>->)` returns an `Effect` that we have to run instead
of directly returning an action in the base monad.  This is because you can
connect things other than `Producer`s and `Consumer`s, like `Pipe`s, which
are effectful stream transformations.

A `Pipe` is a monad transformer that is a mix between a `Producer` and
`Consumer`, because a `Pipe` can both `await` and `yield`.  The following
example `Pipe` is analogous to the Prelude's `take`, only allowing a fixed
number of values to flow through:

```haskell
import Control.Monad (replicateM_)
import Pipes
import Prelude hiding (take)

--              +--------- A 'Pipe' that
--              |    +---- 'await's 'a's and
--              |    | +-- 'yield's 'a's
--              |    | |
--              v    v v
take ::  Int -> Pipe a a IO ()
take n = do
    replicateM_ n $ do                     -- Repeat this block 'n' times
        x <- await                         -- 'await' a value of type 'a'
        yield x                            -- 'yield' a value of type 'a'
    lift $ putStrLn "You shall not pass!"  -- Fly, you fools!
```

You can use `Pipe`s to transform `Producer`s, `Consumer`s, or even other
`Pipe`s using the same `(>->)` operator:

```haskell
(>->) :: Monad m => Producer a m r -> Pipe   a b m r -> Producer b m r
(>->) :: Monad m => Pipe   a b m r -> Consumer b m r -> Consumer a m r
(>->) :: Monad m => Pipe   a b m r -> Pipe   b c m r -> Pipe'   a c m r
```

For example, you can compose `take` after `stdinLn` to limit the number
of lines drawn from standard input:

```active haskell
import Control.Monad (replicateM_)
import Pipes
import qualified Pipes.Prelude as Pipes
import Prelude hiding (take)

take ::  Int -> Pipe a a IO ()
take n = do
    replicateM_ n $ do
        x <- await
        yield x
    lift $ putStrLn "You shall not pass!"
-- show
maxInput :: Int -> Producer String IO ()
maxInput n = Pipes.stdinLn >-> take n

main = runEffect $ maxInput 3 >-> Pipes.stdoutLn
```

... or you can pre-compose `take` before `stdoutLn` to limit the number
of lines written to standard output:

```active haskell
import Control.Monad (replicateM_)
import Pipes
import qualified Pipes.Prelude as Pipes
import Prelude hiding (take)

take ::  Int -> Pipe a a IO ()
take n = do
    replicateM_ n $ do
        x <- await
        yield x
    lift $ putStrLn "You shall not pass!"
-- show
maxOutput :: Int -> Consumer String IO ()
maxOutput n = take n >-> Pipes.stdoutLn

-- Exact same behavior
main = runEffect $ Pipes.stdinLn >-> maxOutput 3
```

Those both gave the same behavior because `(>->)` is associative:

```haskell
(p1 >-> p2) >-> p3 = p1 >-> (p2 >-> p3)
```

Therefore we can just leave out the parentheses:

```active haskell
import Control.Monad (replicateM_)
import Pipes
import qualified Pipes.Prelude as Pipes
import Prelude hiding (take)

take ::  Int -> Pipe a a IO ()
take n = do
    replicateM_ n $ do
        x <- await
        yield x
    lift $ putStrLn "You shall not pass!"
-- show
-- Exact same behavior
main = runEffect $ Pipes.stdinLn >-> take 3 >-> Pipes.stdoutLn
```

`(>->)` is designed to behave like the Unix pipe operator, except with less
quirks.  In fact, we can continue the analogy to Unix by defining `cat`
(named after the Unix `cat` utility), which reforwards elements endlessly:

```haskell
cat :: Monad m => Pipe a a m r
cat = forever $ do
    x <- await
    yield x
```

`cat` is the identity of `(>->)`, meaning that `cat` satisfies the
following two laws:

```haskell
-- Useless use of 'cat
cat >-> p = p

-- Forwarding output to 'cat' does nothing
p >-> cat = p
```

Therefore, `(>->)` and `cat` form a `Category`, specifically the category of
Unix pipes, and `Pipe`s are also composable.

A lot of Unix tools have very simple definitions when written using `pipes`:

```active haskell
import Control.Monad (forever)
import Pipes
import qualified Pipes.Prelude as Pipes  -- Pipes.Prelude provides 'take', too
import Prelude hiding (head)

head :: Monad m => Int -> Pipe a a m ()
head = Pipes.take

yes :: Monad m => Producer String m r
yes = forever $ yield "y"

main = runEffect $ yes >-> head 3 >-> Pipes.stdoutLn
```

This prints out 3 `y`s, just like the equivalent Unix pipeline:

```
$ yes | head -3
y
y
y
$
```

This lets us write "Haskell pipes" instead of Unix pipes.  These are much
easier to build than Unix pipes and we can connect them directly within
Haskell for interoperability with the Haskell language and ecosystem.

# ListT

`pipes` also provides a
["ListT done right" implementation](http://www.haskell.org/haskellwiki/ListT_done_right).
This differs from the implementation in `transformers` because this `ListT`:

* obeys the monad laws, and

* streams data immediately instead of collecting all results into memory.

The latter property is actually an elegant consequence of obeying the monad
laws.

To bind a list within a `ListT` computation, combine `Select` and `each`:

```haskell
import Pipes

pair :: ListT IO (Int, Int)
pair = do
    x <- Select $ each [1, 2]
    lift $ putStrLn $ "x = " ++ show x
    y <- Select $ each [3, 4]
    lift $ putStrLn $ "y = " ++ show y
    return (x, y)
```

You can then loop over a `ListT` by using `every`:

```haskell
every :: Monad m => ListT m a -> Producer a m ()
```

So you can use your `ListT` within a `for` loop:

```active haskell
import Pipes

pair :: ListT IO (Int, Int)
pair = do
    x <- Select $ each [1, 2]
    lift $ putStrLn $ "x = " ++ show x
    y <- Select $ each [3, 4]
    lift $ putStrLn $ "y = " ++ show y
    return (x, y)
-- show
-- Try me!
main = runEffect $ for (every pair) (lift . print)
```

    ... or a pipeline:

```active haskell
import Pipes
import qualified Pipes.Prelude as Pipes

pair :: ListT IO (Int, Int)
pair = do
    x <- Select $ each [1, 2]
    lift $ putStrLn $ "x = " ++ show x
    y <- Select $ each [3, 4]
    lift $ putStrLn $ "y = " ++ show y
    return (x, y)
-- show
main = runEffect $ every pair >-> Pipes.print
```

Note that `ListT` is lazy and only produces as many elements as we request:

```active haskell
import Pipes
import qualified Pipes.Prelude as Pipes

pair :: ListT IO (Int, Int)
pair = do
    x <- Select $ each [1, 2]
    lift $ putStrLn $ "x = " ++ show x
    y <- Select $ each [3, 4]
    lift $ putStrLn $ "y = " ++ show y
    return (x, y)
-- show
-- Try me!
main = runEffect $ for (every pair >-> Pipes.take 2) (lift . print)
```

You can also go the other way, binding `Producer`s directly within a
`ListT`.  In fact, this is actually what `Select` was already doing:

```haskell
Select :: Producer a m () -> ListT m a
```

This lets you write crazy code like:

```haskell
import Pipes
import qualified Pipes.Prelude as Pipes

input :: Producer String IO ()
input = Pipes.stdinLn >-> Pipes.takeWhile (/= "quit")

name :: ListT IO String
name = do
    firstName <- Select input
    lastName  <- Select input
    return (firstName ++ " " ++ lastName)
```

Here we're binding standard input non-deterministically (twice) as if it
were an effectful list:

```active haskell
import Pipes
import qualified Pipes.Prelude as Pipes

input :: Producer String IO ()
input = Pipes.stdinLn >-> Pipes.takeWhile (/= "quit")

name :: ListT IO String
name = do
    firstName <- Select input
    lastName  <- Select input
    return (firstName ++ " " ++ lastName)
-- show
main = runEffect $ every name >-> Pipes.stdoutLn
```

Here is an example session using the above program:

```
> Daniel<Enter>
> Fischer<Enter>
Daniel Fischer
> Wagner<Enter>
Daniel Wagner
> quit<Enter>
> Donald<Enter>
> Stewart<Enter>
Donald Stewart
> Duck<Enter>
Donald Duck
> quit<Enter>
> quit<Enter>
```

Notice how this streams out values immediately as they are generated, rather
than building up a large intermediate result and then printing all the
values in one batch at the end.

# Tricks

`pipes` is more powerful than meets the eye so this section presents some
non-obvious tricks you may find useful.

Many pipe combinators will work on unusual pipe types and the next few
examples will use the `cat` pipe to demonstrate this.

For example, you can loop over the output of a `Pipe` using `for`, which is
how `map` is defined:

```haskell
map :: Monad m => (a -> b) -> Pipe a b m r
map f = for cat $ \x -> yield (f x)

-- Read this as: For all values flowing downstream, apply 'f'
```

This is equivalent to:

```haskell
map f = forever $ do
    x <- await
    yield (f x)
```

You can also feed a `Pipe` input using `(>~)`.  This means we could have
instead defined the `yes` pipe like this:

```haskell
yes :: Monad m => Producer String m r
yes = return "y" >~ cat

-- Read this as: Keep feeding "y" downstream
```

This is equivalent to:

```haskell
yes = forever $ yield "y"
```

You can also sequence two `Pipe`s together.  This is how `drop` is
defined:

```haskell
drop :: Monad m => Int -> Pipe a a m r
drop n = do
    replicateM_ n await
    cat
```

This is equivalent to:

```haskell
drop n = do
    replicateM_ n await
    forever $ do
        x <- await
        yield x
```

You can even compose pipes inside of another pipe:

```haskell
customerService :: Producer String IO ()
customerService = do
    each [ "Hello, how can I help you?"                -- Begin with a script
         , "Hold for one second."
         ]
    Pipes.stdinLn >-> Pipes.takeWhile (/= "Goodbye!")  -- Now continue with a human
```

Also, you can often use `each` in conjunction with (`~>`) to traverse nested
data structures.  For example, you can print all non-`Nothing` elements
from a doubly-nested list:

```active haskell
import Pipes

main = runEffect $ (each ~> each ~> each ~> lift . print) [[Just 1, Nothing], [Just 2, Just 3]]
```

Another neat thing to know is that 'every' has a more general type:

```haskell
every :: (Monad m, Enumerable t) => t m a -> Producer a m ()
```

`Enumerable` generalizes `Foldable` and if you have an effectful container
of your own that you want others to traverse using `pipes`, just have your
container implement the `toListT` method of the `Enumerable` class:

```haskell
class Enumerable t where
    toListT :: Monad m => t m a -> ListT m a
```

You can even use `Enumerable` to traverse effectful types that are not even
proper containers, like `MaybeT`:

```active haskell
import Control.Monad (guard)
import Control.Monad.Trans.Maybe
import Pipes
import qualified Pipes.Prelude as Pipes

input :: MaybeT IO String
input = do
    str <- lift getLine
    guard (str /= "Fail")
    return str

main = runEffect $ every input >-> Pipes.stdoutLn
```

# Conclusion

This tutorial covers the concepts of connecting, building, and reading
`pipes` code.  However, this library is only the core component in an
ecosystem of streaming components.  Derived libraries that build immediately
upon `pipes` include:

* `pipes-concurrency`: Concurrent reactive programming and message passing

* `pipes-parse`: Minimal utilities for stream parsing

* `pipes-safe`: Resource management and exception safety for @pipes@

These libraries provide functionality specialized to common streaming
domains.  Additionally, there are several libraries on Hackage that provide
even higher-level functionality, which you can find by searching under the
"Pipes" category or by looking for packages with a `pipes-` prefix in
their name.  Current examples include:

* `pipes-network`/`pipes-network-tls`: Networking

* `pipes-zlib`: Compression and decompression

* `pipes-binary`: Binary serialization

* `pipes-attoparsec`: High-performance parsing

* `pipes-aeson`: JSON serialization and deserialization

Even these derived packages still do not explore the full potential of
`pipes` functionality, which actually permits bidirectional communication.
Advanced `pipes` users can explore this library in greater detail by
studying the documentation in
[the `Pipes.Core` module](http://hackage.haskell.org/package/pipes-4.1.0/docs/Pipes-Core.html)
to learn about the symmetry of the underlying `Proxy` type and operators.

To learn more about `pipes`, ask questions, or follow `pipes` development,
you can subscribe to
[the `haskell-pipes` mailing list](https://groups.google.com/forum/#!forum/haskell-pipes)
or you can
[mail the list directly](mailto:haskell-pipes@googlegroups.com).
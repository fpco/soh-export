# 4. The Santa Claus problem

I want to show you a complete, runnable concurrent program using STM. A
well-known example is the so-called Santa Claus problem<sup>10</sup>, originally due to
Trono [14]:

> Santa repeatedly sleeps until wakened by either all of his nine reindeer, back from their holidays, or by a group of three of his ten elves.
If awakened by the reindeer, he harnesses each of them to his sleigh,
delivers toys with them and finally unharnesses them (allowing them
to go off on holiday). If awakened by a group of elves, he shows each
of the group into his study, consults with them on toy R&D and
finally shows them each out (allowing them to go back to work).
Santa should give priority to the reindeer in the case that there is
both a group of elves and a group of reindeer waiting.

Using a well-known example allows you to directly compare my solution with
well-described solutions in other languages. In particular, Trono’s paper gives
a semaphore-based solution which is partially correct; Ben-Ari gives a solution
in Ada95 and in Ada [1]; Benton gives a solution in Polyphonic C# [2].

## 4.1 Reindeer and elves

The basic idea of the STM Haskell implementation is this. Santa makes one
“`Group`” for the elves and one for the reindeer. Each elf (or reindeer) tries to join
its `Group`. If it succeeds, it gets two “`Gate`s” in return. The first `Gate` allows
Santa to control when the elf can enter the study, and also lets Santa know
when they are all inside. Similarly, the second `Gate` controls the elves leaving
the study. Santa, for his part, waits for either of his two `Group`s to be ready, and
then uses that `Group`’s `Gate`s to marshal his helpers (elves or reindeer) through
their task. Thus the helpers spend their lives in an infinite loop: try to join
a group, move through the gates under Santa’s control, and then delay for a
random interval before trying to join a group again.

Rendering this informal description in Haskell gives the following code for an
elf<sup>11</sup>:

``` haskell
elf1 :: Group -> Int -> IO ()
elf1 group elf_id = do
    (in_gate, out_gate) <- joinGroup group
    passGate in_gate
    meetInStudy elf_id
    passGate out_gate
```

The `elf` is passed its `Group`, and an `Int` that specifies its elfin identity. This
identity is used only in the call to `meetInStudy`, which simply prints out a
message to say what is happening<sup>12</sup>

``` haskell
meetInStudy :: Int -> IO ()
meetInStudy id = putStr ("Elf " ++ show id ++ " meeting in the study\n")
```

@@@ Try it...
``` active haskell
meetInStudy :: Int -> IO ()
meetInStudy id = putStr ("Elf " ++ show id ++ " meeting in the study\n")

main = meetInStudy 3
```
@@@

The elf calls `joinGroup` to join its group, and `passGate` to pass through each
of the gates:

``` haskell
joinGroup :: Group -> IO (Gate, Gate)
passGate  :: Gate -> IO ()
```

The code for reindeer is identical, except that reindeer deliver toys rather than
meeting in the study:

``` haskell
deliverToys :: Int -> IO ()
deliverToys id = putStr ("Reindeer " ++ show id ++ " delivering toys\n")
```

@@@ Try it...

``` active haskell
deliverToys :: Int -> IO ()
deliverToys id = putStr ("Reindeer " ++ show id ++ " delivering toys\n")

main = deliverToys 4
```
@@@

Since `IO` actions are first-class, we can abstract over the common pattern, like
this:

``` haskell
helper1 :: Group -> IO () -> IO ()
helper1 group do_task = do
    (in_gate, out_gate) <- joinGroup group
    passGate in_gate
    do_task
    passGate out_gate
```

The second argument of `helper1` is an `IO` action that is the helper’s task, which
the helper performs between the two `passGate` calls. Now we can specialise
`helper1` to be either an elf or a reindeer:

``` haskell
elf1, reindeer1 :: Group -> Int -> IO ()
elf1      gp id = helper1 gp (meetInStudy id)
reindeer1 gp id = helper1 gp (deliverToys id)
```

@@@ Try it...
In order to test `helper1` we will jump ahead just a bit by defining dummy `Group` and `Gate`:

``` active haskell
data Group = MkGroup
data Gate  = MkGate

joinGroup :: Group -> IO (Gate, Gate)
joinGroup group = do putStr "joinGroup\n"
                     return (MkGate, MkGate)

passGate :: Gate -> IO ()
passGate gate = putStr "passGate\n"

meetInStudy :: Int -> IO ()
meetInStudy id = putStr ("Elf " ++ show id ++ " meeting in the study\n")

deliverToys :: Int -> IO ()
deliverToys id = putStr ("Reindeer " ++ show id ++ " delivering toys\n")

helper1 :: Group -> IO () -> IO ()
helper1 group do_task = do
    (in_gate, out_gate) <- joinGroup group
    passGate in_gate
    do_task
    passGate out_gate

elf1, reindeer1 :: Group -> Int -> IO ()
elf1 gp id = helper1 gp (meetInStudy id)
reindeer1 gp id = helper1 gp (deliverToys id)

main = do
    elf1 MkGroup 3
    reindeer1 MkGroup 4

```

@@@

## 4.2 Gates and Groups

The first abstraction is a `Gate`, which supports the following interface:

``` haskell
newGate     :: Int -> STM Gate
passGate    :: Gate -> IO ()
operateGate :: Gate -> IO ()
```

A `Gate` has a fixed _capacity_, _n_, which we specify when we make a new `Gate`, and a
mutable _remaining capacity_. This remaining capacity is decremented whenever
a helper calls `passGate` to go through the gate; if the remaining capacity is
zero, `passGate` blocks. A `Gate` is created with zero remaining capacity, so that
no helpers can pass through it. Santa opens the gate with `operateGate`, which
sets its remaining capacity back to _n_.

Here, then, is a possible implementation of a `Gate`:

``` haskell
data Gate = MkGate Int (TVar Int)

newGate :: Int -> STM Gate
newGate n = do
    tv <- newTVar 0
    return (MkGate n tv)

passGate :: Gate -> IO ()
passGate (MkGate n tv)
  = atomically (do n_left <- readTVar tv
                   check (n_left > 0)
                   writeTVar tv (n_left-1))

operateGate :: Gate -> IO ()
operateGate (MkGate n tv) = do
    atomically (writeTVar tv n)
    atomically (do n_left <- readTVar tv
                   check (n_left == 0))
```

The first line declares `Gate` to be a new _data type_, with a single _data constructor_
`MkGate`<sup>13</sup>. The constructor has two _fields_: an `Int` giving the gate capacity, and
a `TVar` whose contents says how many helpers can go through the gate before
it closes. If the `TVar` contains zero, the gate is closed.

The function `newGate` makes a new `Gate` by allocating a `TVar`, and building a
`Gate` value by calling the `MkGate` constructor. Dually, `passGate` uses pattern-matching to take apart the `MkGate` constructor; then it decrements the contents
of the `TVar`, using `check` to ensure there is still capacity in the gate, as we
did with `withdraw` (Section 3.4). Finally, `operateGate` first opens the `Gate`
by writing its full capacity into the `TVar`, and then waits for the `TVar` to be
decremented to zero.

A `Group` has the following interface:

``` haskell
newGroup   :: Int -> IO Group
joinGroup  :: Group -> IO (Gate,Gate)
awaitGroup :: Group -> STM (Gate,Gate)
```

Again, a `Group` is created empty, with a specified capacity. An elf may join
a group by calling `joinGroup`, a call that blocks if the group is full. Santa
calls `awaitGroup` to wait for the group to be full; when it is full he gets the
`Group`’s gates, and the `Group` is immediately re-initialised with fresh `Gate`s, so
that another group of eager elves can start assembling.

Here is a possible implementation:

``` haskell
data Group = MkGroup Int (TVar (Int, Gate, Gate))

newGroup n = atomically (do g1 <- newGate n; g2 <- newGate n
                            tv <- newTVar (n, g1, g2)
                            return (MkGroup n tv))
```

Again, `Group` is declared as a fresh data type, with constructor `MkGroup` and
two fields: the `Group`’s full capacity, and a `TVar` containing its number of empty
slots and its two `Gate`s. Creating a new `Group` is a matter of creating new `Gate`s,
initialising a new `TVar`, and returning a structure built with `MkGroup`.

The implementations of `joinGroup` and `awaitGroup` are now more or less determined by these data structures:


``` haskell
joinGroup (MkGroup n tv)
  = atomically (do (n_left, g1, g2) <- readTVar tv
                   check (n_left > 0)
                   writeTVar tv (n_left-1, g1, g2)
                   return (g1,g2))

awaitGroup (MkGroup n tv) = do
    (n_left, g1, g2) <- readTVar tv
    check (n_left == 0)
    new_g1 <- newGate n; new_g2 <- newGate n
    writeTVar tv (n,new_g1,new_g2)
    return (g1,g2)
```

Notice that `awaitGroup` makes new gates when it re-initialises the `Group`. This
ensures that a new group can assemble while the old one is still talking to Santa
in the study, with no danger of an elf from the new group overtaking a sleepy
elf from the old one.

@@@ Try it...

Let's test the functionality so far:

``` active haskell
import Control.Concurrent.STM
import Control.Concurrent

-- capacity, TVar ()
data Group = MkGroup Int (TVar (Int, Gate, Gate))

newGroup :: Int -> IO Group
newGroup n = atomically (do g1 <- newGate n
                            g2 <- newGate n
                            tv <- newTVar (n, g1, g2)
                            return (MkGroup n tv))

joinGroup :: Group -> IO (Gate,Gate)
joinGroup (MkGroup n tv)
  = atomically (do (n_left, g1, g2) <- readTVar tv
                   check (n_left > 0)
                   writeTVar tv (n_left-1, g1, g2)
                   return (g1,g2))

awaitGroup :: Group -> STM (Gate,Gate)
awaitGroup (MkGroup n tv) = do
    (n_left, g1, g2) <- readTVar tv
    check (n_left == 0)
    new_g1 <- newGate n; new_g2 <- newGate n
    writeTVar tv (n,new_g1,new_g2)
    return (g1,g2)

-----
data Gate = MkGate Int (TVar Int)

newGate :: Int -> STM Gate
newGate n = do
    tv <- newTVar 0
    return (MkGate n tv)

passGate :: Gate -> IO ()
passGate (MkGate n tv)
  = atomically (do n_left <- readTVar tv
                   check (n_left > 0)
                   writeTVar tv (n_left-1))

operateGate :: Gate -> IO ()
operateGate (MkGate n tv) = do
    atomically (writeTVar tv n)
    atomically (do n_left <- readTVar tv
                   check (n_left == 0))
----
meetInStudy :: Int -> IO ()
meetInStudy id = putStr ("Elf " ++ show id ++ " meeting in the study\n")

deliverToys :: Int -> IO ()
deliverToys id = putStr ("Reindeer " ++ show id ++ " delivering toys\n")

helper1 :: Group -> IO () -> IO ()
helper1 group do_task = do
    (in_gate, out_gate) <- joinGroup group
    passGate in_gate
    do_task
    passGate out_gate

elf1, reindeer1 :: Group -> Int -> IO ()
elf1 gp id = helper1 gp (meetInStudy id)
reindeer1 gp id = helper1 gp (deliverToys id)

main = do
    grp <- newGroup 1    -- group of capacity 1
    forkIO (elf1 grp 13) -- elf 13 wants to talk
    (in_gate, out_gate) <- atomically (awaitGroup grp) -- Santa waits
    operateGate in_gate  -- let the elf in
    operateGate out_gate -- let the elf out
```

@@@

Reviewing this section, you may notice that I have given some of the `Group`
and `Gate` operations `IO` types (e.g. `newGroup`, `joinGroup`), and some `STM` types
(e.g. `newGate`, `awaitGroup`). How did I make these choices? For example,
`newGroup` has an `IO` type, which means that I can never call it from within an
`STM` action. But this is merely a matter of convenience: I could instead have
given `newGroup` an `STM` type, by omitting the `atomically` in its definition. In
exchange, I would have had to write `atomically (newGroup n)` at each call
site, rather than merely `newGroup n`. The merit of giving `newGate` an `STM` type
is that it is more composable, a generality that `newGroup` did not need in this
program. In contrast, I wanted to call `newGate` inside `newGroup`, and so I gave
`newGate` an `STM` type.

In general, when designing a library, you should give the functions `STM` types
wherever possible. You can think of `STM` actions as Lego bricks that can be
glued together, using `do ...`, `retry`, and `orElse`, to make bigger `STM` actions.
However, as soon as you wrap a block in `atomically`, making it an `IO` type, it
can no longer be combined atomically with other actions. There is a good reason
for that: a value of `IO` type can perform arbitrary, irrevocable input/output
(such as `launchMissiles`).

It is therefore good library design to export `STM` actions (rather than `IO` actions)
whenever possible, because they are composable; their type advertises that they
do no irrevocable effects. The library client can readily get from `STM` to `IO` (using
`atomically`), but not vice versa.

Sometimes, however, it is _essential_ to use an `IO` action. Look at `operateGate`.
The two calls to `atomically` cannot be combined into one, because the first
has an externally-visible side effect (opening the gate), while the second blocks
until all the elves have woken up and gone through it. So `operateGate` must
have an `IO` type.

## 4.3 The main program

We will first implement the outer structure of the program, although we have
not yet implemented Santa himself. Here it is.

``` haskell
main = do
    elf_group <- newGroup 3
    sequence_ [ elf elf_group n | n <- [1..10] ]
          
    rein_group <- newGroup 9
    sequence_ [ reindeer rein_group n | n <- [1..9] ]
          
    forever (santa elf_group rein_group)
```

The first line creates a `Group` for the elves with capacity 3. The second line is
more mysterious: it uses a so-called _list comprehension_ to create a list of `IO` actions and calls `sequence_` to execute them in sequence. The list comprehension
`[e|x<-xs]` is read “the list of all _e_ where _x_ is drawn from the list _xs_”. So the
argument to `sequence_` is the list

``` haskell
[elf elf_group 1, elf elf_group 2, ..., elf elf_group 10]
```

Each of these calls yields an `IO` action that spawns an elf thread. The function
`sequence_` takes a list of `IO` actions and returns an action that, when performed,
runs each of the actions in the list in order<sup>14</sup>:

``` haskell
sequence_ :: [IO a] -> IO ()
```

An `elf` is built from `elf1`, but with two differences. First, we want the elf to
loop indefinitely, and second, we want it to run in a separate thread:

``` haskell
elf :: Group -> Int -> IO ThreadId
elf gp id = forkIO (forever (do elf1 gp id
                                randomDelay))
```

The `forkIO` part spawns its argument as a separate Haskell thread (Section 3.1).
In turn, `forkIO`’s argument is a call to `forever`, which runs _its_ argument repeatedly (compare the definition of `nTimes` in Section 3.1):

``` haskell
forever :: IO () -> IO ()
-- Repeatedly perform the action
forever act = do
    act
    forever act
```

@@@ Try it...
This is a test version of `forever` that doesn't (thankfuly!) go forever:
``` active haskell
forever :: IO () -> IO ()
-- Repeatedly perform the action
forever act = forever' act 10
  where -- cheating here to make it stop eventually
    forever' :: IO () -> Int -> IO ()
    forever' act 0 = return ()
    forever' act n = do act
                        forever' act (n - 1)

main = forever (putStr "Are we there yet?\n")
```

@@@

Finally the expression `(elf1 gp id)` is an `IO` action, and we want to repeat
that action indefinitely, followed each time by a random delay:

``` haskell
randomDelay :: IO ()
-- Delay for a random time between 1 and 1,000,000 microseconds
randomDelay = do waitTime <- getStdRandom (randomR (1, 1000000))
                 threadDelay waitTime
```

@@@ Try it...

Run this program several times to experience random delays of up to 5 seconds:

``` active haskell
import System.Random
import Control.Concurrent

randomDelay :: IO ()

randomDelay = do
    waitTime <- getStdRandom (randomR (1, 5000000))
    threadDelay waitTime

randomOp = do
    n <- getStdRandom (randomR (0, 3))
    return (["+", "-", "*", "/"] !! n)

main = do
    putStr "All our operators are currently busy...\n"
    randomDelay
    op <- randomOp
    putStr ("Operator (" ++ op ++ ") here, how may I help you?\n")
```

@@@

The rest of the main program should be self-explanatory. We make nine reindeer
in the same way that we made ten elves, except that we call `reindeer` instead
of `elf`:

``` haskell
reindeer :: Group -> Int -> IO ThreadId
reindeer gp id = forkIO (forever (do reindeer1 gp id
                                     randomDelay))
```

The code for `main` finishes by re-using `forever` to run `santa` repeatedly. All
that remains is to implement Santa himself.

## 4.4 Implementing Santa

Santa is the most interesting participant of this little drama, because he makes
choices. He must wait until there is _either_ a group of reindeer waiting, _or_ a
group of elves. Once he has made his choice of which group to attend to, he
must take them through their task. Here is his code:

``` haskell
santa :: Group -> Group -> IO ()
santa elf_gp rein_gp = do
    putStr "----------\n"
    (task, (in_gate, out_gate)) <- atomically (orElse
                     (chooseGroup rein_gp "deliver toys")
                     (chooseGroup elf_gp "meet in my study"))
    putStr ("Ho! Ho! Ho! let’s " ++ task ++ "\n")
    operateGate in_gate
              -- Now the helpers do their task
    operateGate out_gate
  where
    chooseGroup :: Group -> String -> STM (String, (Gate,Gate))
    chooseGroup gp task = do gates <- awaitGroup gp
                             return (task, gates)
```

The choice is made by the `orElse`, which first attempts to choose the reindeer (thereby giving them priority), and otherwise choosing the elves. The
`chooseGroup` function does an `awaitGroup` call on the appropriate group, and
returns a pair consisting of a string indicating the task (delivering toys or meeting in the study) and the gates that Santa must operate to take the group
through the task. Once the choice is made, Santa prints out a message and
operates the two gates in sequence.

This implementation works fine, but we will also explore an alternative, more
general version, because `santa` demonstrates a very common programming pattern. The pattern is this: a thread (Santa in this case) makes a choice in one
atomic transaction, followed by one or more further consequential transactions.
Another typical example might be: take a message from one of several message
queues, act on the message, and repeat. In this case, the consequential action
was very similar for elves and reindeer — in both cases, Santa had to print a
message and operate two gates. But that would not work if Santa should do
very different things for elves and reindeer. One approach would be to return
a boolean indicating which was chosen, and dispatch on that boolean after the
choice; but that becomes inconvenient as more alternatives are added. Here is
another approach that works better:

``` haskell
santa :: Group -> Group -> IO ()
santa elf_gp rein_gp = do
    putStr "----------\n"
    choose [(awaitGroup rein_gp, run "deliver toys"),
            (awaitGroup elf_gp, run "meet in my study")]
  where
    run :: String -> (Gate,Gate) -> IO ()
    run task (in_gate,out_gate) = do
        putStr ("Ho! Ho! Ho! let’s " ++ task ++ "\n")
        operateGate in_gate
        operateGate out_gate
```

The function `choose` is like a guarded command: it takes a list of pairs, waits
until the first component of a pair is ready to “fire”, and then executes the
second component. So `choose` has this type<sup>15</sup>:

``` haskell
choose :: [(STM a, a -> IO ())] -> IO ()
```

The guard is an `STM` action delivering a value of type `a`; when the `STM` action
is ready (that is, does not retry), `choose` can pass the value to the second
component, which must therefore be a function expecting a value of type `a`.
With this in mind, `santa` should be easy reading. He uses `awaitGroup` to
wait for a ready `Group`; the `choose` function gets the pair of `Gate`s returned by
`awaitGroup` and passes it to the `run` function. The latter operates the two gates
in succession – recall that `operateGate` blocks until all the elves (or reindeer)
have gone through the gate.

The code for `choose` is brief, but a little mind-bending:

``` haskell
choose :: [(STM a, a -> IO ())] -> IO ()
choose choices = do
    act <- atomically (foldr1 orElse actions)
    act
  where
    actions :: [STM (IO ())]
    actions = [ do val <- guard
                   return (rhs val)
              | (guard, rhs) <- choices ]
```

First, it forms a list, `actions`, of `STM` actions, which it then combines with
`orElse`. (The call `foldr1` ⊕ [_x_<sub>1</sub>, . . . , _x_<sub>n</sub>] returns _x_<sub>1</sub> ⊕ _x_<sub>2</sub> ⊕ . . . ⊕ _x_<sub>n</sub>.) Each
of these `STM` actions itself returns an `IO` action, namely _the thing to be done
when the choice is made_. That is why each action in the list has the cool type
`STM (IO ())`. The code for `choose` first makes an atomic choice among the
list of alternatives, getting the action `act`, with type `IO (`) in return; and then
performs the action `act`. The list `actions` is defined in the `where` clause by
taking each pair `(guard,rhs)` from the list `choices`, running the guard (an
`STM` action), and returning the `IO` action gotten by applying the `rhs` to the
`guard`’s return value.

## 4.5 Compiling and running the program

I have presented _all_ the code for this example. If you simply add the appropriate
import statements at the top, you should be good to go<sup>16</sup>:

``` haskell
module Main where
import Control.Concurrent.STM
import Control.Concurrent
import System.Random
```
Try it:

``` active haskell
import Control.Concurrent.STM
import Control.Concurrent
import System.Random

main = do
    elf_gp <- newGroup 3
    sequence_ [ elf elf_gp n | n <- [1..10]]
    rein_gp <- newGroup 9
    sequence_ [ reindeer rein_gp n | n <- [1..9]]
    forever (santa elf_gp rein_gp)
  where
    elf      gp id = forkIO (forever (do elf1 gp id
                                         randomDelay))
    reindeer gp id = forkIO (forever (do reindeer1 gp id
                                         randomDelay))

santa :: Group -> Group -> IO ()
santa elf_group rein_group = do
    putStr "----------\n"
    choose [(awaitGroup rein_group, run "deliver toys"), 
            (awaitGroup elf_group,  run "meet in my study")]
  where
    run :: String -> (Gate,Gate) -> IO ()
    run task (in_gate,out_gate) = do 
        putStr ("Ho! Ho! Ho! let's " ++ task ++ "\n")
        operateGate in_gate
        operateGate out_gate

helper1 :: Group -> IO () -> IO ()
helper1 group do_task = do
    (in_gate, out_gate) <- joinGroup group
    passGate in_gate
    do_task
    passGate out_gate

elf1, reindeer1 :: Group -> Int -> IO ()
elf1      group id = helper1 group (meetInStudy id)
reindeer1 group id = helper1 group (deliverToys id)


meetInStudy id = putStr ("Elf " ++ show id ++ " meeting in the study\n")
deliverToys id = putStr ("Reindeer " ++ show id ++ " delivering toys\n")

---------------
data Group = MkGroup Int (TVar (Int, Gate, Gate))

newGroup :: Int -> IO Group
newGroup n = atomically (do g1 <- newGate n
                            g2 <- newGate n
                            tv <- newTVar (n, g1, g2)
                            return (MkGroup n tv))

joinGroup :: Group -> IO (Gate,Gate)
joinGroup (MkGroup n tv) 
  = atomically (do (n_left, g1, g2) <- readTVar tv
       		       check (n_left > 0) 
       		       writeTVar tv (n_left-1, g1, g2)
       		       return (g1,g2))

awaitGroup :: Group -> STM (Gate,Gate)
awaitGroup (MkGroup n tv) = do
    (n_left, g1, g2) <- readTVar tv
    check (n_left == 0) 
    new_g1 <- newGate n
    new_g2 <- newGate n
    writeTVar tv (n,new_g1,new_g2)
    return (g1,g2)

---------------
data Gate  = MkGate Int (TVar Int)

newGate :: Int -> STM Gate
newGate n = do
    tv <- newTVar 0
    return (MkGate n tv)

passGate :: Gate -> IO ()
passGate (MkGate n tv) 
  = atomically (do n_left <- readTVar tv
  	               check (n_left > 0)
  	               writeTVar tv (n_left-1))

operateGate :: Gate -> IO ()
operateGate (MkGate n tv) = do
    atomically (writeTVar tv n)
    atomically (do n_left <- readTVar tv
                   check (n_left == 0))

----------------
forever :: IO () -> IO ()
-- Repeatedly perform the action
forever act = forever' act 10
  where -- cheating here to make it stop eventually
    forever' :: IO () -> Int -> IO ()
    forever' act 0 = return ()
    forever' act n = do
        act
        forever' act (n - 1)

randomDelay :: IO ()
-- Delay for a random time between 1 and 1000,000 microseconds
randomDelay = do
    waitTime <- getStdRandom (randomR (1, 1000000))
    threadDelay waitTime

choose :: [(STM a, a -> IO ())] -> IO ()
choose choices = do 
    to_do <- atomically (foldr1 orElse stm_actions)
    to_do
  where
    stm_actions :: [STM (IO ())]
    stm_actions = [ do val <- guard
                       return (rhs val)
                  | (guard, rhs) <- choices ] 

```

@@@ Note...
Since you can run this program directly from within the FP Complete version of this article, the following description might be of less interest than it was in the original publication. 
@@@

To compile the code, use the Glasgow Haskell Compiler, GHC<sup>17</sup>:

``` 
$ ghc Santa.hs -package stm -o santa
```

Finally you can run the program:

```
$ ./santa
----------
Ho! Ho! Ho! let’s deliver toys
Reindeer 8 delivering toys
Reindeer 7 delivering toys
Reindeer 6 delivering toys
Reindeer 5 delivering toys
Reindeer 4 delivering toys
Reindeer 3 delivering toys
Reindeer 2 delivering toys
Reindeer 1 delivering toys
Reindeer 9 delivering toys
----------
Ho! Ho! Ho! let’s meet in my study
Elf 3 meeting in the study
Elf 2 meeting in the study
Elf 1 meeting in the study
...and so on...
```
» [Next: Reflections on Haskell](5-reflections-on-haskell).

---

&nbsp;<sup>10</sup> My choice was influenced by the fact that I am writing these words on 22 December.

&nbsp;<sup>11</sup> I have given this function a suffix “1” because it only deals with one iteration of the elf,
whereas in reality the elves re-join the fun when they are done with their task. We will define `elf` in Section 4.3.

&nbsp;<sup>12</sup> The function `putStr` is a library function that calls `hPutStr stdout`.

&nbsp;<sup>13</sup> A data type declaration is not unlike a C `struct` declaration, with `MkGate` being the
structure tag.

&nbsp;<sup>14</sup> The type `[IO a]` means “a list of values of type `IO a`”. You may also wonder about the
underscore in the name `sequence_` : it’s because there is a related function `sequence` whose
type is `[IO a] -> IO [a]`, that gathers the results of the argument actions into a list. Both
`sequence` and `sequence_` are defined in the `Prelude` library, which is imported by default.

&nbsp;<sup>15</sup> In Haskell, the type `[`_ty_`]` means a list whose elements have type _ty_. In this case `choose`’s
argument is a list of pairs, written _(ty<sub>1</sub>,ty<sub>2</sub>)_; the first component of the pair has type `STM a`,
while the second is a function with type `a->IO ()`.

&nbsp;<sup>16</sup> You can get the code online at  [http://research.microsoft.com/~simonpj/papers/stm/Santa.hs.gz](http://research.microsoft.com/~simonpj/papers/stm/Santa.hs.gz)

&nbsp;<sup>17</sup>GHC is available for free at [http://haskell.org/ghc](http://haskell.org/ghc)

---

[1] Mordechai Ben-Ari. How to solve the Santa Claus problem. _Concurrency:
Practice and Experience_, 10(6):485–496, 1998.

[2] Nick Benton. Jingle bells: Solving the Santa Claus problem in Polyphonic
C#. Technical report, Microsoft Research, 2003.

[14] JA Trono. A new exercise in concurrency. _SIGCSE Bulletin_, 26:8–10, 1994.

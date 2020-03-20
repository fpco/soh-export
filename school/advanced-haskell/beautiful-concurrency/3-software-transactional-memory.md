# 3. Software Transactional Memory

Software Transactional Memory is a promising new approach to the challenge of
concurrency, as I will explain in this section. I shall explain STM using Haskell,
the most beautiful programming language I know, because STM fits into Haskell
particularly elegantly. If you don’t know any Haskell, don’t worry; we’ll learn
it as we go.

## 3.1 Side effects and input/output in Haskell

Here is the beginning of the code for transfer in Haskell:
``` haskell
transfer :: Account -> Account -> Int -> IO ()
-- Transfer ’amount’ from account ’from’ to account ’to’
transfer from to amount = ...
```

The second line of this definition, starting “`--`”, is a comment. The first line
gives the _type signature_ for `transfer`. This signature says that `transfer` takes
as its arguments<sup>2</sup> two values of type `Account` (the source and destination accounts), and an `Int` (the amount to transfer), and returns a value of type `IO ()`.
This result type says “`transfer` returns an action that, when performed, may
have some side effects, and then returns a value of type `()`”. The type `()`, pronounced “unit”, has just one value, which is also written `()`; it is akin to `void`
in C. So `transfer`’s result type `IO ()` announces that its side effects constitute
the only reason for calling it. Before we go further, we must explain how side
effects are handled in Haskell.

A “side effect” is anything that reads or writes mutable state. Input/output is
a prominent example of a side effect. For example, here are the signatures of
two Haskell functions with input/output effects:

``` haskell
hPutStr  :: Handle -> String -> IO ()
hGetLine :: Handle -> IO String
```

We call any value of type `IO t` an “action”. So `(hPutStr h "hello")`<sup>3</sup> is an
action that, when performed, will print `"hello"` on handle<sup>4</sup> `h` and return the
unit value. Similarly, `(hGetLine h)` is an action that, when performed, will
read a line of input from handle `h` and return it as a `String`. We can glue
together little side-effecting programs to make bigger side-effecting programs
using Haskell’s “`do`” notation. For example, `hEchoLine` reads a string from the
input and prints it:

``` haskell
hEchoLine :: Handle -> IO String
hEchoLine h = do
    s <- hGetLine h
    hPutStr h ("I just read that " ++ s)
    return s
```

@@@ Try it...

Here's a small program that calls the function `hEchoLine` and sets up data for it. Run it!

``` active haskell
{-# START_FILE main.hs #-}
import System.IO

hEchoLine :: Handle -> IO String
hEchoLine h = do
    s <- hGetLine h
    hPutStr h ("I just read that " ++ s)
    return s
        
main = do
    h <- openFile "test.txt" ReadWriteMode
    str <- {-hi-}hEchoLine h{-/hi-}
    hClose h
    str1 <- readFile "test.txt"
    hPutStr stdout str1

{-# START_FILE test.txt #-}
Haskell rules!
```
First, we `import` the appropriate library, in this case `System.IO`, which contains the definitions of `hGetLine`, `hPutStr`, and a few others used in `main`. Function `main` makes the call to `hEchoLine` with the appropriate argument -- a file handle, `h`. 
We get the handle by calling `openFile` with two arguments: the name of the file and the file mode -- `ReadWriteMode` in this case. Finally, we read the whole modified file using `readFile` and send the result to the standard output, `stdout`, for display.

Notice that we are discarding the result of `hEchoLine`: `str`. If you want to display it, change the last line of `main`.

The file `test.txt` is provided too (feel free to edit its content and run the program again).

@@@

The notation:
``` haskell
do
   a1
   ...
   an 
```
constructs an action by gluing together the
smaller actions _a<sub>1</sub> . . . a<sub>n</sub>_ in sequence. So `hEchoLine h` is an action that, when
performed, will first perform `hGetLine h` to read a line from `h`, naming the result `s`. Then it will perform `hPutStr` to print `s`, preceded<sup>5</sup> by “`I just read: `”.
Finally, it returns the string `s`. This last line is interesting, because `return` is
not a built-in language construct: rather, it is a perfectly ordinary function with
type

``` haskell
return :: a -> IO a
```
The action `return v`, when performed, returns `v` without having caused any
side effects<sup>6</sup>. This function works on values of any type, and we indicate this
by using a type variable `a` in its type.

Input/output is one important sort of side effect. Another is the act of reading
or writing a mutable variable. For example, here is a function that increments
the value of a mutable variable:

``` haskell
incRef :: IORef Int -> IO ()
incRef var = do
    val <- readIORef var
    writeIORef var (val+1)

```

@@@ Try it...

Here is a small test program calling `incRef`:

``` active haskell
import System.IO
import Data.IORef

incRef :: IORef Int -> IO ()
incRef var = do
    val <- readIORef var
    writeIORef var (val+1)
                
main = do
    var <- newIORef 42
    {-hi-}incRef var{-/hi-} 
    val <- readIORef var
    hPutStr stdout (show val)

```
This time `main` creates a new `IORef` with the initial value `42`, passes it to our function `incRef`, reads the new value from `IORef`, and displays it. To display a number we convert it to a string using the function `show`.

@@@

Here, `incRef var` is an action that first performs `readIORef var` to read the
value of the variable, naming its value `val`, and then performs `writeIORef`
to write the value `(val+1)` into the variable. The types of `readIORef` and
`writeIORef` are as follows:

``` haskell
readIORef  :: IORef a -> IO a
writeIORef :: IORef a -> a -> IO ()
```

A value of type `IORef t` should be thought of as a pointer, or reference, to a
mutable location containing a value of type `t`, a bit like the type `(t *)` in C.
In the case of `incRef`, the argument has type `IORef Int` because `incRef` only
applies to locations that contain an `Int`.

So far I have explained how to build big actions by combining smaller ones
together — but how does an action ever actually get performed? In Haskell,
the whole program defines a single `IO` action, called `main`. To run the program
is to perform the action `main`. For example, here is a complete program:

``` active haskell
import System.IO

-- show
main :: IO ()
main = do
    hPutStr stdout "Hello"
    hPutStr stdout " world\n"
```
          
This program is a sequential program, because the `do`-notation combines `IO`
actions in sequence. To construct a concurrent program we need one more
primitive, `forkIO`:

``` haskell
forkIO :: IO a -> IO ThreadId
```

The function `forkIO`, which is built into Haskell, takes an `IO` action as its
argument, and spawns it as a concurrent Haskell thread. Once created, it is run
concurrently with all the other Haskell threads, by the Haskell runtime system.
For example, suppose we modified our main program thus<sup>7</sup>:

``` active haskell
import System.IO
import Control.Concurrent

-- show
main :: IO ()
main = do 
    forkIO (hPutStr stdout "Hello")
    hPutStr stdout " world\n"
```

@@@ Note...
In the first line of main, we could instead have written `tid <- forkIO (hPutStr ...)`,
to bind the `ThreadId` returned by `forkIO` to `tid`. However, since we do not use the returned
`ThreadId`, we are free to discard it by omitting the “`tid <-`” part.

``` active haskell
import System.IO
import Control.Concurrent

main :: IO ()
main = do
    tid <- forkIO (hPutStr stdout "Hello")
    hPutStr stdout " world\n"
```
@@@

Now the two `hPutStr` actions would run concurrently. Which of them would
“win” (by printing its string first) is unspecified. Haskell threads spawned by
`forkIO` are extremely lightweight: they occupy a few hundred bytes of memory,
and it is perfectly reasonable for a single program to spawn thousands of them.

Gentle reader, you may by now be feeling that Haskell is a very clumsy and
verbose language. After all, our three-line definition of `incRef` accomplishes
no more than `x++` does in C! Indeed, in Haskell side effects are extremely explicit and somewhat verbose. However, remember first that Haskell is primarily
a _functional_ language. Most programs are written in the functional core of
Haskell, which is rich, expressive, and concise. Haskell thereby gently encourages you to write programs that make sparing use of side effects.

Second, notice that being explicit about side effects reveals a good deal of useful
information. Consider two functions:

``` haskell
f :: Int -> Int
g :: Int -> IO Int
```

From looking only at their types we can see that `f` is a pure function: it has
no side effects. Given a particular `Int`, say `42`, the call `(f 42)` will return the
same value every time it is called. In contrast, `g` has side effects, and this is
apparent in its type. Each time `g` is performed it may give a different result —
for example it may read from `stdin`, or modify a mutable variable — even if its
argument is the same every time. This ability to make side effects explicit will
prove very useful in what follows.

Lastly, actions are first-class values: they may be passed as arguments as well
as returned as results. For example, here is the definition of a (simplified) `for`
loop function, written entirely in Haskell rather than being built in:

``` haskell
nTimes :: Int -> IO () -> IO ()
nTimes 0 do_this = return ()
nTimes n do_this = do
    do_this
    nTimes (n-1) do_this
```

This recursive function takes an `Int` saying how many times to loop, and an action `do_this`; it returns an action that, when performed, performs the `do_this`
action `n` times. Here is an example of a use of `nTimes` to print “`Hello`” 10 times:

``` haskell
main = nTimes 10 (hPutStr stdout "Hello\n")
```

@@@ Try it...
``` active haskell
import System.IO

nTimes :: Int -> IO () -> IO ()
nTimes 0 do_this = return ()
nTimes n do_this = do 
    do_this
    nTimes (n-1) do_this

main = nTimes 10 (hPutStr stdout "Hello\n")
```
We are calling `nTimes` with `10` and the action returned by `(hPutStr stdout "Hello\n")` that prints "Hello" to the standard input.
@@@

In effect, by treating actions as first-class values, Haskell supports _user-defined
control structures_.

This chapter is not the place for a full introduction to Haskell, or even to side
effects in Haskell. A good starting point for further reading is the tutorial
“_Tackling the awkward squad_” [9].

## 3.2 Transactions in Haskell

Now we can return to our `transfer` function. Here is its code:

``` haskell
transfer :: Account -> Account -> Int -> IO ()
-- Transfer ’amount’ from account ’from’ to account ’to’
transfer from to amount
  = atomically (do  deposit to amount
                    withdraw from amount)
```

The inner `do`-block should by now be fairly self-explanatory: we call `deposit`
to deposit amount in `to`, and `withdraw` to withdraw amount from account `from`.
We will write these auxiliary functions in a moment, but first look at the call
to `atomically`. It takes an action as its argument, and performs it atomically.
More precisely, it makes two guarantees:

> **Atomicity:** the effects of `atomically act` become visible to another thread
all at once. This ensures that no other thread can see a state in which
money has been deposited in to but not yet withdrawn from from.

> **Isolation:** during a call `atomically act`, the action `act` is completely unaffected by other threads. It is as if act takes a snapshot of the state of the
world when it begins running, and then executes against that snapshot.

Here is a simple execution model for `atomically`. Suppose there is a single,
global lock. Then `atomically act` grabs the lock, performs the action `act`,
and releases the lock. This implementation brutally ensures that no two atomic
blocks can be executed simultaneously, and thereby ensures atomicity.

There are two problems with this model. First, it does not ensure isolation
at all: while one thread is accessing an `IORef` inside an atomic block (holding
the Global Lock), there is nothing to stop _another_ thread writing the same
`IORef` directly (i.e. outside `atomically`, without holding the Global Lock),
thereby destroying the isolation guarantee. Second, performance is dreadful,
because every atomic block would be serialised even if no actual interference
was possible.

I will discuss the second problem shortly, in Section 3.3. Meanwhile, the first
objection is easily addressed with the type system. We give `atomically` the
following type:

``` haskell
atomically :: STM a -> IO a
```

The argument of atomically is an action of type `STM a`. An `STM` action is like
an `IO` action, in that it can have side effects, but the range of side effects for `STM`
actions is much smaller. The main thing you can do in an `STM` action is read
or write a transactional variable, of type `(TVar a)`, much as we could read or
write `IORefs` in an `IO` action<sup>8</sup>.
``` haskell
readTVar  :: TVar a -> STM a
writeTVar :: TVar a -> a -> STM ()
```

`STM` actions can be composed together with the same `do`-notation as `IO` actions
— the `do`-notation is overloaded to work on both types, as is `return`<sup>9</sup>. Here,
for example, is the code for `withdraw`:

``` haskell
type Account = TVar Int

withdraw :: Account -> Int -> STM ()
withdraw acc amount = do
    bal <- readTVar acc
    writeTVar acc (bal - amount)
```

We represent an `Account` by a transactional variable containing an `Int` for the
account balance. Then withdraw is an `STM` action that decrements the balance
in the account by `amount`.

To complete the definition of `transfer` we can define `deposit` in terms of
`withdraw`:

``` haskell
deposit :: Account -> Int -> STM ()
deposit acc amount = withdraw acc (- amount)
```

@@@ Try it...

``` active haskell
import System.IO
import Control.Concurrent.STM

type Account = TVar Int

withdraw :: Account -> Int -> STM ()
withdraw acc amount = do
    bal <- readTVar acc
    writeTVar acc (bal - amount)

deposit :: Account -> Int -> STM ()
deposit acc amount = withdraw acc (- amount)

transfer :: Account -> Account -> Int -> IO ()
-- Transfer ’amount’ from account ’from’ to account ’to’
transfer from to amount 
    = atomically (do deposit to amount
                     withdraw from amount)

showAccount :: Account -> IO Int
showAccount acc = atomically (readTVar acc)

main = do
    from <- atomically (newTVar 200)
    to   <- atomically (newTVar 100)
    transfer from to 50
    v1 <- showAccount from
    v2 <- showAccount to
    putStrLn $ (show v1) ++ ", " ++ (show v2)
```
We are uning `newTVar` to create two `TVar`s representing two accounts, `from` and `to`.
@@@

Notice that, `transfer` ultimately performs four primitive read/write actions: a
read and then write on account `to`, followed by a read and then write on account
`from`. These four actions execute atomically, and that meets the specification
given at the start of Section 2.

The type system neatly prevents us from reading or writing a `TVar` outside of a
transaction. For example, suppose we tried this:

``` haskell
bad :: Account -> IO ()
bad acc = do
    hPutStr stdout "Withdrawing..."
    withdraw acc 10
```

@@@ Try it...
This program won't compile:

``` active haskell
import System.IO
import Control.Concurrent.STM

type Account = TVar Int

withdraw :: Account -> Int -> STM ()
withdraw acc amount = do
    bal <- readTVar acc
    writeTVar acc (bal - amount)

bad :: Account -> IO ()
bad acc = do
    hPutStr stdout "Withdrawing..."
    withdraw acc 10

main = do
    acc <- atomically (newTVar 200)
    bad acc 
    hPutStr stdout "\nDone!\n"
```
@@@

This program is rejected because the `hPutStr` is an `IO` action, while the
withdraw is an `STM` action, and the two cannot be combined in a single do
block. If we wrap a call to `atomically` around the `withdraw`, all is well:

``` haskell
good :: Account -> IO ()
good acc = do 
    hPutStr stdout "Withdrawing..."
    atomically (withdraw acc 10)
```

@@@ Try it...

This program compiles and runs:

``` active haskell
import System.IO
import Control.Concurrent.STM

type Account = TVar Int

withdraw :: Account -> Int -> STM ()
withdraw acc amount = do
    bal <- readTVar acc    
    writeTVar acc (bal - amount)

good :: Account -> IO ()
good acc = do
    hPutStr stdout "Withdrawing..."
    {-hi-}atomically{-/hi-} (withdraw acc 10)

main = do
    acc <- atomically (newTVar 200)
    good acc 
    hPutStr stdout "\nDone!\n"
```
@@@

## 3.3 Implementing transactional memory

The guarantees of atomicity and isolation that I described earlier should be all
that a programmer needs in order to use STM. Even so, I often find it helpful
to have a reasonable implementation model to guide my intuitions, and I will
sketch one such implementation in this section. But remember that this is just
_one_ possible implementation. One of the beauties of the STM abstraction is
that it presents a small, clean interface that can be implemented in a variety of
ways, some simple and some sophisticated.

One particularly attractive implementation is well established in the database
world, namely _optimistic execution_. When `(atomically act)` is performed, a
thread-local _transaction log_ is allocated, initially empty. Then the action `act`
is performed, without taking any locks at all. While performing `act`, each call
to `writeTVar` writes the address of the `TVar` and its new value into the log; it
does not write to the `TVar` itself. Each call to `readTVar` first searches the log
(in case the `TVar` was written by an earlier call to `writeTVar`); if no such record
is found, the value is read from the `TVar` itself, and the `TVar` and value read
are recorded in the log. In the meantime, other threads might be running their
own atomic blocks, reading and writing `TVars` like crazy.

When the action `act` is finished, the implementation first _validates_ the log and,
if validation is successful, _commits_ the log. The validation step examines each
`readTVar` recorded in the log, and checks that the value in the log matches the
value currently in the real `TVar`. If so, validation succeeds, and the commit step
takes all the writes recorded in the log and writes them into the real `TVars`.

These steps are performed truly indivisibly: the implementation disables interrupts, or uses locks or compare-and-swap instructions — whatever is necessary
to ensure that validation and commit are perceived by other threads as completely indivisible. All of this is handled by the implementation, however, and
the programmer does not need to know or care how it is done.

What if validation fails? Then the transaction has had an inconsistent view of
memory. So we abort the transaction, re-initialise the log, and run `act` all over
again. This process is called _re-execution_. Since none of `act`’s writes have been
committed to memory, it is perfectly safe to run it again. However, notice that
it is crucial that act contains no effects _other than_ reads and writes on `TVars`.
For example, consider

``` haskell
atomically (do x <- readTVar xv
               y <- readTVar yv
               if x>y then launchMissiles
                      else return () )
```

where `launchMissiles :: IO ()` causes serious international side-effects.
Since the atomic block is executed without taking locks, it might have an inconsistent view of memory if other threads are concurrently modifying `xv` and
`yv`. If that happens, it would be a mistake to launch the missiles, and only _then_ discover that validation fails so the transaction should be re-run. Fortunately,
the type system prevents us running `IO` actions inside `STM` actions, so the above
fragment would be rejected by the type checker. This is another big advantage
of distinguishing the types of `IO` and `STM` actions.

@@@ Try it...
``` active haskell
import System.IO
import Control.Concurrent.STM

launchMissiles :: IO ()
launchMissiles = hPutStr stdout "Zzzing!"

main = do 
    xv <- atomically (newTVar 2)
    yv <- atomically (newTVar 1)
    atomically (do x <- readTVar xv
                   y <- readTVar yv
                   if x > y then launchMissiles
                            else return () )

```

@@@

## 3.4 Blocking and choice

Atomic blocks as we have introduced them so far are utterly inadequate to
coordinate concurrent programs. They lack two key facilities: _blocking_ and
_choice_. In this section I’ll describe how the basic STM interface is elaborated
to include them in a fully-modular way.

Suppose that a thread should _block_ if it attempts to overdraw an account (i.e.
withdraw more than the current balance). Situations like this are common in
concurrent programs: for example, a thread should block if it reads from an
empty buffer, or when it waits for an event. We achieve this in STM by adding
the single function `retry`, whose type is

``` haskell
retry :: STM a
```

Here is a modified version of withdraw that blocks if the balance would go
negative:

``` haskell
limitedWithdraw :: Account -> Int -> STM ()
limitedWithdraw acc amount = do
    bal <- readTVar acc
    if amount > 0 && amount > bal
    then retry
    else writeTVar acc (bal - amount)
```
@@@ Try it...

``` active haskell
import System.IO
import Control.Concurrent.STM
import Control.Concurrent

type Account = TVar Int

limitedWithdraw :: Account -> Int -> STM ()
limitedWithdraw acc amount = do
    bal <- readTVar acc
    if amount > 0 && amount > bal
    then retry
    else writeTVar acc (bal - amount)

delayDeposit acc amount = do
    hPutStr stdout "Getting ready to deposit money...hunting through pockets...\n"
    threadDelay 3000000
    hPutStr stdout "OK! Depositing now!\n"
    atomically ( do bal <- readTVar acc
                    writeTVar acc (bal + amount) )

main = do
    acc <- atomically (newTVar 100)
    forkIO (delayDeposit acc 1)
    hPutStr stdout "Trying to withdraw money...\n"
    atomically (limitedWithdraw acc 101)
    hPutStr stdout "Successful withdrawal!\n"
```
We are forking a thread that calls `delayDeposit`, which waits for 3 seconds before depositing the amount. In the meanwhile `limitedWithraw` blocks because of insufficient funds. Soon after the deposit from the other thread goes through, `limitedWithdraw` succeeds.

@@@

The semantics of `retry` are simple: if a `retry` action is performed, the current
transaction is abandoned and retried at some later time. It would be correct to
`retry` the transaction immediately, but it would also be inefficient: the state of
the account will probably be unchanged, so the transaction will again hit the
`retry`. An efficient implementation would instead block the thread until some
other thread writes to `acc`. How does the implementation know to wait on `acc`?
Because the transaction read `acc` on the way to the `retry`, and that fact is
conveniently recorded in the transaction log.

The conditional in `limitedWithdraw` has a very common pattern: check that a
boolean condition is satisfied and, if not, `retry`. This pattern is easy to abstract
as a function, `check`:

``` haskell
check :: Bool -> STM ()
check True  = return ()
check False = retry
```

Now we can use `check` to re-express `limitedWithdraw` a little more neatly:

``` haskell
limitedWithdraw :: Account -> Int -> STM ()
limitedWithdraw acc amount = do
    bal <- readTVar acc
    check (amount <= 0 || amount <= bal)
    writeTVar acc (bal - amount)
```

@@@ Try it...

The same program using `check`, which is actually a library function:

``` active haskell
import System.IO
import Control.Concurrent.STM
import Control.Concurrent

type Account = TVar Int

limitedWithdraw :: Account -> Int -> STM ()
limitedWithdraw acc amount = do
    bal <- readTVar acc
    check (amount <= 0 || amount <= bal)
    writeTVar acc (bal - amount)

delayDeposit acc amount = do
    threadDelay 3000000
    hPutStr stdout "Depositing right now!\n"
    atomically ( do bal <- readTVar acc
                    writeTVar acc (bal + amount) )

main = do
    acc <- atomically (newTVar 100)
    forkIO (delayDeposit acc 1)
    hPutStr stdout "Withdrawing... Hope the deposit has cleared...\n"
    atomically (limitedWithdraw acc 101)
    hPutStr stdout "Oh, phew!\n"
```

@@@

We now turn our attention to _choice_. Suppose you want to withdraw money
from account A if it has enough money, but if not then withdraw it from account
B? For that, we need the ability to choose an alternative action if the first one
retries. To support choice, STM Haskell has one further primitive action, called
`orElse`, whose type is

``` haskell
orElse :: STM a -> STM a -> STM a
```

Like `atomically` itself, `orElse` takes actions as its arguments, and glues them
together to make a bigger action. Its semantics are as follows. The action
`(orElse a1 a2)` first performs `a1`; if `a1` retries (i.e. calls `retry`), it tries `a2`
instead; if `a2` also retries, the whole action retries. It may be easier to see how
`orElse` is used:

``` haskell
limitedWithdraw2 :: Account -> Account -> Int -> STM ()
-- (limitedWithdraw2 acc1 acc2 amt) withdraws amt from acc1,
-- if acc1 has enough money, otherwise from acc2.
-- If neither has enough, it retries.
limitedWithdraw2 acc1 acc2 amt
  = orElse (limitedWithdraw acc1 amt) (limitedWithdraw acc2 amt)
```

@@@ Try it...

``` active haskell
import System.IO
import Control.Concurrent.STM
import Control.Concurrent

type Account = TVar Int

limitedWithdraw :: Account -> Int -> STM ()
limitedWithdraw acc amount = do
    bal <- readTVar acc
    check (amount <= 0 || amount <= bal)
    writeTVar acc (bal - amount) 

showAcc name acc = do
    bal <- atomically (readTVar acc)
    hPutStr stdout (name ++ ": $")
    hPutStr stdout (show bal ++ "\n")

limitedWithdraw2 :: Account -> Account -> Int -> STM ()
-- (limitedWithdraw2 acc1 acc2 amt) withdraws amt from acc1,
-- if acc1 has enough money, otherwise from acc2.
-- If neither has enough, it retries.
limitedWithdraw2 acc1 acc2 amt
  = orElse (limitedWithdraw acc1 amt) (limitedWithdraw acc2 amt)
  
delayDeposit name acc amount = do
    threadDelay 3000000
    hPutStr stdout ("Depositing $" ++ show amount ++ " into " ++ name ++ "\n")
    atomically ( do bal <- readTVar acc
                    writeTVar acc (bal + amount) )

main = do
    acc1 <- atomically (newTVar 100)
    acc2 <- atomically (newTVar 100)
    showAcc "Left pocket" acc1
    showAcc "Right pocket" acc2
    forkIO (delayDeposit "Right pocket" acc2 1)
    hPutStr stdout "Withdrawing $101 from either pocket...\n"
    atomically (limitedWithdraw2 acc1 acc2 101)
    hPutStr stdout "Successful withdrawal!\n"
    showAcc "Left pocket" acc1
    showAcc "Right pocket" acc2
```

We use a helper function `showAcc` to display the contents of accounts before and after the withdrawal. We have two accounts, `acc1` and `acc2`, both with insufficient funds for the `limitedWithdraw2` to succeed immediately. However, when the background thread deposits $1 into `acc2`, the call succeeds.

@@@

Since the result of `orElse` is itself an `STM` action, you can feed it to another call
to `orElse` and so choose among an arbitrary number of alternatives.

## 3.5 Summary so far

In this section I have introduced all the key transactional memory operations
supported by STM Haskell. They are summarised in Figure 1. 

![STM.PNG](https://www.fpcomplete.com/media/db5dbe9d-7461-45c1-b1d2-ca6da66be69e.png)

This figure
includes one operation that has not so far arisen: `newTVar` is the way in which
you can create new `TVar` cells, and we will use it in the following section.

» [Next: The Santa Claus problem](4-the-santa-claus-problem).

---

&nbsp;<sup>2</sup> You may think it odd that there are three function arrows in this type signature, rather
than one. That’s because Haskell supports _currying_, which you can find described in any
book about Haskell (e.g. [13]), or on [Wikipedia](http://en.wikipedia.org/wiki/Currying). For the purposes of this paper, simply treat
all the types except the final one as arguments.

&nbsp;<sup>3</sup> In Haskell we write function application using simple juxtaposition. In most languages you
would write `hPutStr(h,"hello")`, but in Haskell you write simply `(hPutStr h "hello")`.

&nbsp;<sup>4</sup> A `Handle` in Haskell plays the role of a file descriptor in C: it says which file or pipe to
read or write. As in Unix, there are three pre-defined handles, `stdin`, `stdout`, and `stderr`.

&nbsp;<sup>5</sup> The `(++)` operator concatenates two strings.

&nbsp;<sup>6</sup> The `IO` type indicates the _possibility_ of side effects, not the certainty!

&nbsp;<sup>7</sup> In the first line of `main`, we could instead have written `tid <- forkIO (hPutStr ...)`,
to bind the `ThreadId` returned by `forkIO` to `tid`. However, since we do not use the returned
`ThreadId`, we are free to discard it by omitting the “`tid <-`” part.

&nbsp;<sup>8</sup> The nomenclature is inconsistent here: it would be more consistent to use either `TVar` and
`IOVar`, or `TRef` and `IORef`. But it would be disruptive to change at this stage; for better or
worse we have `TVar` and `IORef`.

&nbsp;<sup>9</sup> This overloading of `do`-notation and `return` is not an ad-hoc trick to support `IO` and
`STM`. Rather, `IO` and `STM` are both examples of a common pattern, called a _monad_ [15], and
the overloading is achieved by expressing that common pattern using Haskell’s very general
_type-class_ mechanism [16, 10].

---

[9] Simon Peyton Jones. Tackling the awkward squad: monadic input/output,
concurrency, exceptions, and foreign-language calls in Haskell. In CAR
Hoare, M Broy, and R Steinbrueggen, editors, _Engineering Theories of
Software Construction, Marktoberdorf Summer School 2000_, NATO ASI
Series, pages 47–96. IOS Press, 2001.

[10] Simon Peyton Jones, Mark Jones, and Erik Meijer. Type classes: an exploration of the design space. In J Launchbury, editor, _Haskell workshop_,
Amsterdam, 1997.

[13] SJ Thompson. _Haskell: the craft of functional programming._ Addison
Wesley, 1999.

[15] PL Wadler. The essence of functional programming. In _20th ACM Sym-
posium on Principles of Programming Languages (POPL’92)_, pages 1–14.
ACM, Albuquerque, January 1992.

[16] PL Wadler and S Blott. How to make ad-hoc polymorphism less ad hoc.
In _Proc 16th ACM Symposium on Principles of Programming Languages,
Austin, Texas_. ACM, January 1989.



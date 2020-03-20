# 2. A simple example: bank accounts

Here is a simple programming task.

> Write a procedure to transfer money from one bank account to another. To keep things simple, both accounts are held in memory: no interaction with databases is required. The procedure must operate correctly in a concurrent program, in which many threads may call
transfer simultaneously. No thread should be able to observe a state in which the money has left one account, but not arrived in the other (or vice versa).

This example is somewhat unrealistic, but its simplicity allows us to focus on
what is new: the language Haskell (Section 3.1), and transactional memory
(Sections 3.2 onwards). But first let us briefly look at the conventional approach.

## 2.1 Bank accounts using locks

The dominant technology for coordinating concurrent programs today is the use
of _locks_ and _condition variables_. In an object-oriented language, every object
has an implicit lock and the locking is done by _synchronised methods_, but the
idea is the same. So one might define a class for bank accounts something like
this:

```
class Account {
  Int balance;
  synchronized void withdraw( int n ) {
    balance = balance - n; }
  void deposit( int n ) {
    withdraw( -n ); }
}
```

We must be careful to use a synchronized method for `withdraw`, so that we do
not get any missed decrements if two threads call `withdraw` at the same time.
The effect of `synchronized` is to take a lock on the account, run `withdraw`, and
then release the lock.

Now, here is how we might write the code for transfer:

```
void transfer( Account from, Account to, Int amount ) {
  from.withdraw( amount );
  to.deposit( amount ); }
```

This code is fine for a sequential program, but in a concurrent program another
thread could observe an intermediate state in which the money has left account
`from`, but not arrived in `to`. The fact that both methods are `synchronized`
does not help us at all. Account `from` is first locked and then unlocked by the
call to method `withdraw`, and then to is locked and unlocked by `deposit`. In
between the two calls, the money is (visibly) absent from both accounts.

In a finance program, that might be unacceptable. How do we fix it? The usual
solution would be to add explicit locking code thus:

```
void transfer( Account from, Account to, Int amount ) {
  from.lock(); to.lock();
    from.withdraw( amount );
    to.deposit( amount );
  from.unlock(); to.unlock(); }
```

But this program is fatally prone to deadlock. In particular, consider the (unlikely) situation in which another thread is transferring money in the opposite
direction between the same two accounts. Then each thread might get one lock
and then block indefinitely waiting for the other.

Once recognised – and the problem is not always so obvious – the standard fix is
to put an arbitrary global order on the locks, and to acquire them in increasing
order. The locking code would then become

```
if from < to
  then { from.lock(); to.lock(); }
  else { to.lock(); from.lock(); }
```

That works fine when the full set of required locks can be predicted in advance,
but that is not always the case. For example, suppose `from.withdraw` is implemented by transferring money out of account `from2` if `from` does not have
enough funds. We don’t know whether to acquire `from2`’s lock until we have
read from, and by then it is too late to acquire the locks in the “right” order.
Furthermore, the very existence of `from2` may be a private matter that should
be known by `from`, but not by `transfer`. And even if `transfer` did know about
`from2`, the locking code must now take three locks, presumably by sorting them
into the right order.

Matters become even more complicated when we want to _block_. For example,
suppose that `transfer` should block if `from` has insufficient funds. This is
usually done by waiting on a _condition variable_, while simultaneously releasing
`from`’s lock. It gets much trickier if we want to block until there are sufficient
funds in `from` and `from2` considered together.

## 2.2 Locks are bad

To make a long story short, today’s dominant technology for concurrent programming – locks and condition variables – is fundamentally flawed. Here are
some standard difficulties, some of which we have seen above:

> **Taking too few locks.** It is easy to forget to take a lock and thereby end up with two threads that modify the same variable simultaneously.

> **Taking too many locks.** It is easy to take too many locks and thereby inhibit concurrency (at best) or cause deadlock (at worst).

> **Taking the wrong locks.** In lock-based programming, the connection between a lock and the data it protects often exists only in the mind of the programmer, and is not explicit in the program. As a result, it is all too easy to take or hold the wrong locks.

> **Taking locks in the wrong order.** In lock-based programming, one must be careful to take locks in the “right” order. Avoiding the deadlock that can otherwise occur is always tiresome and error-prone, and sometimes extremely difficult.

> **Error recovery** can be very hard, because the programmer must guarantee that no error can leave the system in a state that is inconsistent, or in which locks are held indefinitely.

> **Lost wake-ups and erroneous retries.** It is easy to forget to signal a condition variable on which a thread is waiting; or to re-test a condition after a wake-up.

But the fundamental shortcoming of lock-based programming is that _locks and
condition variables do not support modular programming_. By “modular programming” I mean the process of building large programs by gluing together
smaller programs. Locks make this impossible. For example, we could not use
our (correct) implementations of `withdraw` and `deposit` unchanged to implement transfer; instead we had to expose the locking protocol. Blocking and
choice are even less modular. For example suppose we had a version of `withdraw`
that blocks if the source account has insufficient funds. Then we would not be
able to use `withdraw` directly to withdraw money from A or B (depending on
which has sufficient funds), without exposing the blocking condition — and even
then it’s not easy. This critique is elaborated elsewhere [7, 8, 4].

» [Next: Software Transactional Memory](3-software-transactional-memory).

---

[4] Tim Harris, Simon Marlow, Simon Peyton Jones, and Maurice Herlihy.
Composable memory transactions. In _ACM Symposium on Principles and
Practice of Parallel Programming (PPoPP’05)_, June 2005.

[7] Edward A. Lee. The problem with threads. _IEEE Computer_, 39(5):33–42,
May 2006.

[8] J. K. Ousterhout. Why threads are a bad idea (for most purposes), January 1996. Invited Talk, USENIX Technical Conference.

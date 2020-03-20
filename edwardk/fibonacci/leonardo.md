# Part 1: Leonardo Random Access Lists

Chris Okasaki wrote about many data structures in his excellent book ["Purely Functional Data Structures"](http://www.amazon.com/Purely-Functional-Structures-Chris-Okasaki/dp/0521663504).

Today I have the rare opportunity to talk about (invent?) one that he didn't. It doesn't achieve anything
new in this space, but it is an interesting rearrangement of parts folks already knew for other things.

# Leonardo Numbers

The "Leonardo" numbers are given by [1,1,3,5,9,15,25,41...](https://oeis.org/A001595).

This arises from a recurrence:

```haskell
leonardo 0 = 1
leonardo 1 = 1
leonardo n = leonardo (n - 2) + leonardo (n - 1) + 1
```

This is just a slight mutation of the usual Fibonacci recurrence, where we add an extra 1 at each step.

(Fibonacci was named Leonardo Bonacci.)

We can read the recurrence formula forwards:

```haskell
next i j = i + j + 1
```

or backwards

```haskell
prev i j = j - i - 1
```

You can take multiple steps given a sequence of Leonardo numbers. Given `...g,h,i,j..` you can compute `g` from `i` and `j` directly, just by expanding:

```haskell
prev2 i j = prev (prev i j) i
```

into

```haskell
prev2 i j = 2*i - j
```

We can compute the n_th_ Leonardo number in `O(log n)` time, just like the Fibonacci numbers, or any similar linear recurrence relation.

They are in fact, intimately related to the Fibonacci numbers:

```haskell
leonardo n = 2 * fibonacci (n+1) - 1
```

We know `leonardo 0 = leonardo 1 = 1`, but after that the positive Leonardo numbers are unique.

By the recurrence, `leonardo (-1) = prev 1 1 = -1`. We can continue to define negative Leonardo numbers analogous to negative Fibonacci numbers.

Regardless, to exploit any of the recurrences above, to walk forward or backwards from some current Leonardo number, it is highly beneficial to
keep around the previous (or next) one. You can think of this pair as a cursor through the list of Leonardo numbers.

Dijkstra has a lot more to say on the subject of Leonardo numbers in [EWD797](http://www.cs.utexas.edu/users/EWD/transcriptions/EWD07xx/EWD797.html).

# Building a Data Structure

Okasaki, Knuth and others teach us to think of data structures from number systems or recurrences like this and I'm definitely not the first to play around with using the Leonardo numbers in this fashion.

The first use of Leonardo numbers in the shape of a data structure that I am aware of is the heap shape in Dijkstra's "smoothsort," defined in [EWD796a](http://www.cs.utexas.edu/users/EWD/transcriptions/EWD07xx/EWD796a.html) from which I first learned the name of the sequence. We'll revisit that in a bit.

A Leonardo number adds one to the previous two Leonardo numbers. When we recast this as a data structure, it sounds like a biased tree.

But the property of Leonardo numbers that grabs my attention is that you have two trees "plus one new element". This sounds a lot like the structure of skew binary.

So this invited the question: 

Can you build an efficient random access list structure using the Leonardo numbers, rather than skew binary?

It turns out the answer is yes.

# Lopsided Trees

We'll need some lopsided binary trees

```haskell
data Tree a = Bin a (Tree a) (Tree a) | Tip a deriving Show
```

We'll internally maintain the invariant that all of our trees are some Leonardo number in size, with the left and right children having the previous 2 Leonardo numbers worth of children respectively. I'll leave it as an exercise to the folks in the crowd who love dependent types to enforce this invariant in their code, and just assume this invariant from here out.


E.g. a tree of size 5 has a left child of size 1, and a right child of size 3:

```
  1     
  +---,
  |   |
  2   3
      +----, 
      |    | 
      4    5
```

while a tree of size 9 has a left child of size 3 and a right child of size 5:


```
  1
  +-------+
  |       |
  2       5
  +---,   +---,
  |   |   |   |
  3   4   6   7
              +---, 
              |   | 
              8   9 
```

You can show that the height of the tree grows logarithmically with the number of elements despite the skew.

We'll delegate tracking the sizes of the trees to the next part.

# Growing a Spine

We need a "spine" for the structure, analogous to the skew binary random access list, but now cluttered with a couple of extra numbers.

If given a Leonardo number n, if `leonardo i = n`, with i >= 0, I'll say that n has index i. Since 1 occurs twice, 1 has both index 0 and 1.

```haskell
data Leonardo a = Cons !Int !Int (Tree a) (Leonardo a) | Nil deriving Show
```

Each tree we cons onto the spine will be associated with some Leonardo number, and we'll keep the previous Leonardo number as well to facilitate walking the trees and merging nodes later. We maintain the invariant that the sequence of Leonardo numbers we use has not just a series of increasing indices, but other than possibly the two trees present that have the smallest indices, there are no Leonardo numbers with with adjacent indices in the spine.

This prevents the representation from being ambiguous, forcing us to use a tree of size 5 instead of the sequence 1,1,3.

I now leave as an exercise for the reader to show that this operation preserves the invariant above.

```haskell
cons :: a -> Leonardo a -> Leonardo a
cons a (Cons i j bs (Cons j' k cs zs))
  | j == j' = Cons k (next j k) (Bin a bs cs) zs
cons a rs = Cons 1 1 (Tip a) rs
```

I cheat a bit, and use the (1,1) digit twice to save a little work, rather than start with (1,-1).

**Exercise:** Why does this work?

This yields the following progression:

```haskell
1
1,1
3
1,3
5
1,5
1,1,5
3,5
9
1,9
1,1,9
3,9
1,3,9
5,9
15
1,15
...
```

We can compute the size of the random access list in log time. 

**Exercise:** Why?

```haskell
size :: Leonardo a -> Int
size (Cons _ i _ as) = i + size as
size Nil = 0
```

We can now write a routine that indexes into a given Leonardo random access list in log time:

```haskell
(!) :: Leonardo a -> Int -> a
Nil ! i = undefined
Cons j k a as ! i
  | i < k     = go i (prev j k) j a
  | otherwise = as ! (i - k)
  where
    go 0 _ _ (Tip a) = a
    go 0 _ _ (Bin a _ _) = a
    go i j k (Bin _ l r)
      | i <= j    = go (i-1)  (prev2 j k) (prev j k) l
      | otherwise = go (i-j-1) (prev j k) j r
```

Like I showed with skew-binary random access lists for [on-line lowest common ancestor search](https://www.fpcomplete.com/user/edwardk/online-lca), you should be able to compute how to `drop n` elements from a list in `O(log n)` time. This should work as a drop-in replacement for skew-binary random access lists in that algorithm as well.

The spine is actually quite a bit shorter than the spine of a skew-binary random access list in terms of constant factors in exchange for the trees being correspondingly taller. (Dijkstra ended EWD797 on this note.)

Consequently, this pretty much works as a drop in replacement for a skew-binary random access list.

The code is available all together in a short [gist](https://gist.github.com/ekmett/036e28ff705cb27e9de6).

# Simplifying the Spine

It is somewhat ugly that we store two coefficients in each node in the spine. There is a scheme by Dijkstra that manages to track everything
with a simple bit vector for which Leonardo numbers are present in the tree and a single pair of adjacent Leonardo numbers for where to start counting. He uses this in smoothsort to argue for it really being a fully in-place algorithm. On the other hand, it'd considerably complicate `cons`, above.

I leave this as an exercise for the reader as well.

# Why Care?

Gerth Stølting Brodal pointed out [in a paper with Gabriel Moruz](http://dl.acm.org/citation.cfm?id=1276254) that balancing your tree wasn't actually optimal on real hardware. We have branch predictions, cache effects, etc. Consequently, the costs of going left vs. going right aren't the same!

(He also has [slides available](http://www.cs.au.dk/~gerth/slides/cphstl06.pdf).)

In particular, he showed empirically a bias to one side can speed things up considerably due to these effects. Something like 30% or so was the optimal balance, gaining up to 15% performance over a "real" balanced binary tree.

The ratio between consecutive Leonardo numbers approaches to the golden ratio φ and `1/(1+φ) ~ 38%` at least gets us in that ballpark.

Is there a nice, cheap, recurrence that gets us closer and retains the nice property that `cons` is also _O(1)_?

-[Edward Kmett](mailto:ekmett@gmail.com)

April 27, 2015
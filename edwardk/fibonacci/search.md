# Part 2: Open-Ended Fibonacci Search

[Last time](https://www.fpcomplete.com/user/edwardk/fibonacci/leonardo), I built a form of random access list with _O(1)_ cons using the Leonardo numbers and emphasized that the slight skew in the tree could be a good thing.

Now I want to do some searching, but after all, I'm a functional programmer and everybody knows all we do all day is play with ways to write `fib`, so we'll start by building an industrial strength version of the `fib` function to get a feel for how the Fibonacci numbers fit together, and then maybe we can turn it into something with which we can perform efficient searches.

# Integral Domains

To avoid lying, let's stop and define the notion of an integral domain, which is any non-zero commutative ring in which there are no non-zero zero divisors; if `a*b = 0` then either `a = 0`, or `b = 0`.

```haskell
class Num a => IntegralDomain a
```

```haskell
instance IntegralDomain Integer
instance IntegralDomain Rational
instance IntegralDomain Float
instance IntegralDomain Double
```

The latter only pass the no non-zero zero divisors test if you ignore underflow, and I'd probably be beaten up for pretending they are a ring. For the sake of the code at hand we're okay with them, but delete these instances if they make you uncomfortable! I'm not using them.

# Homogeneous Linear Recurrences

I mentioned last time that you could compute Leonardo and Fibonacci numbers in logarithmic time using the fact that they were linear recurrence relationships. Bill Gosper and Richard Schroeppel called this technique the ["Fast Fibonacci Transform"](http://www.inwap.com/pdp10/hbaker/hakmem/recurrence.html) in HAKMEM, a collection of number theoretic computing tricks from the MIT AI Lab from back in the early 70s. 

Now, I confess, I lied a little bit by saying this. Technically you need the fact that you have a _homogeneous_ linear recurrence relationship.

```haskell
F(N) = X*F(N-1) + Y*(F(N-2))
```

Note how no additional constant is being added in. While the Leonardo recurrence doesn't meet this criterion, it is fortunate that the related recurrence

```haskell
leo n = leonardo n + 1 = leonardo (n-1) + leonardo (n-2) + 2 = leo (n-1) + leo (n-2)
```

does. In fact, `leo` satisfies the same recurrence as in the Fibonacci numbers, and after plugging in some constants we find that

```haskell
leo n = 2 * fib (n+1)
```

which gives rise to the formula for Leonardo numbers in terms of Fibonacci numbers that I gave last time:

```haskell
leonardo n = 2 * fib (n+1) - 1
```

Dijkstra talks more about this specific relationship between the Fibonacci and Leonardo sequences in [EWD797](http://www.cs.utexas.edu/users/EWD/transcriptions/EWD07xx/EWD797.html).

# R[φ]

The HAKMEM writeup gives a form of multiplication for an arbitrary homogeneous linear recurrence relationship, but we only need the Fibonacci recurrence today. We can take the number type described in HAKMEM and turn it directly into a Haskell data type.

I'm interested in numbers of the form `aφ+b`, where `φ ~ 1.6180339887...` is the golden ratio. 

```haskell
φ = (1+√5)/2
```

Another fancier way to think of it formally is that we're working in `Z[x] mod x^2 - x - 1` and the polynomial `x^2 - x - 1` has two roots: `(1+√5)/2` and `(1-√5)/2`.

Addition and subtraction are done pointwise as in complex arithmetic.

We can work through multiplication fairly directly by exploiting the interesting fact that

```haskell
φ^2 = φ+1
```

so

```haskell
(aφ+b)(cφ+d) = ac(φ+1) + (ad+bc)φ + bd = (ac+ad+bc)φ + (ac+bd) = (a(c+d)+bc)φ + (ac+bd)
```

Turning this into a data type `Fib r` where the data constructor `Fib a b` represents `aφ + b` in the ring extension `r[φ]` we get:

```haskell
data Fib a = Fib a a deriving
  (Show, Functor, Foldable, Traversable, Eq)

instance IntegralDomain a => Num (Fib a) where
  Fib a b + Fib c d = Fib (a + c) (b + d)
  Fib a b * Fib c d = Fib (a*(c + d) + b*c) (a*c + b*d)
  Fib a b - Fib c d = Fib (a - c) (b - d)
  negate (Fib a b) = Fib (negate a) (negate b)
  abs x = x
  signum _ = Fib 0 1 -- meh
  fromInteger n = Fib 0 (fromInteger n)
```

Why an `IntegralDomain`? At the least we rely on the ability to commute multiplication to get this definition. In practice, we'd probably just presume `Num` has whatever properties we need, and skip this requirement. We should otherwise be able to weaken it to a `CommutativeRing` without lying.

I'm not too happy with the `abs`/`signum` in there, but then nobody is really happy with them in Haskell, and these
pass the only law we require in Haskell for them, that `abs a * signum a = a`, hence `abs = id`, `signum = 1` should always be admissable. Doing better requires an `Ord a`, and rules out potentially many good uses for this ring extension.

If `a` was an integral domain then so is `Fib a`:

```haskell
instance IntegralDomain a => IntegralDomain (Fib a)
```

We can represent

```haskell
φ = Fib 1 0
```

without any loss of precision in this representation without doing symbolic computation. 

If we know more about `a`, we can extend this ring to a field. In our simplistic numerical tower:

```haskell
class (IntegralDomain a, Fractional a) => Field a
instance (IntegralDomain a, Fractional a) => Field a
```

so

```haskell
instance (IntegralDomain a, Fractional a) => Fractional (Fib a) where
  recip (Fib a b) = Fib (-a/d) ((a+b)/d) where
    d = b*b + a*b - a*a
  fromRational r = Fib 0 (fromRational r)  
```

is just a claim that this is a field extension.

But, it turns out we don't need any of that `Fractional` nonsense for `fib`. After all Fibonacci numbers, even the negative ones, are whole numbers! In this ring, φ has an interesting property: It is a "unit" of this ring, which is to say it has an inverse, even if we don't have inverses for general elements. We can generate it algebraically from the same simple claim we used to figure out multiplication

```haskell
φ^2 = φ+1
```

by multiplying on the left by `φ^(-1)`

```haskell
φ = φ^(-1) * φ^2 = φ^(-1) * (φ+1) = 1 + φ^(-1)
```

so

```haskell
φ^(-1) = φ - 1
```

We can check our work by using the formula for `recip`. If we fix the argument to φ, we get:

```haskell
recip φ = recip (Fib 1 0) = Fib (-1 / -1) (1/ -1) = Fib (-1 * -1) (1 * -1) = Fib 1 (-1)
```

The only division we needed was by `-1`, which is a unit in the integers as `-1 * -1 = 1`. Being a unit means we can replace division by -1 with multiplication by its inverse (also `-1`), which we know exists in our ring.

As a further aside, we can also represent `√5` without requiring us to work over anything more than an integral domain.

```haskell
√5 = 2*φ - 1
```

Clearly, `phi`, `1`, `recip phi`, and `√5` all have a very complicated relationship status.

# Putting R[φ] in Order

You can implement `Ord` in a manner compatible with the answers given by more traditional numeric types, but it is a bit tricky.

First we check for a quick exit if both results are consistent. It is only when the sign `a` and `b` differ in `aφ + b` that we have a problem, which is bigger? `aφ` or `b`?

We can write a recursive solution that computes repeated remainders, `gcd` style, but we actually hit literally the worst case possible for the usual `gcd` algorithm for `Fib 1 (-1) ^ n`. This is [Lamé's Theorem](http://www.cut-the-knot.org/blue/LamesTheorem.shtml) showing up in the wild, and these numbers show up below!

```haskell
instance (IntegralDomain a, Ord a) => Ord (Fib a) where
  compare (Fib a b) (Fib c d) = case compare a c of
    LT | b <= d    -> LT
       | otherwise -> go compare (a-c) (b-d)
    EQ -> compare b d
    GT | b >= d    -> GT
       | otherwise -> go (flip compare) (a-c) (b-d)
   where
     go k e f = k (sq (e+2*f)) (5*sq e)
     sq x = x*x
```

So instead, in the above instance I convert to `aφ + b` to `e√5 + f` and compare the squares instead. This works nicely even when `a` is `Double` and is capable of representing `√5` (more or less) on its own.

With this `Ord` instance in hand we could redefine the notion of `abs` and `signum` in `Num` above to be compatible with the real number line, but to do so, you'd have to give up instances that aren't in `Ord`, for example `Fib (Complex Double)`. This just goes to show that `abs`/`signum` being placed directly in `Num` was a bad idea.

**Exercise:** Why?

Note that even this comes at a price. We have to similarly complicate the `Eq` instance, replacing the automatically derived one we start with with

```haskell
instance (IntegralDomain a, Ord a) => Eq (Fib a) where
  x == y = compare x y == EQ
```

**Exercise:** Why?

With that in mind you'd probably want to make a separate data type if you actually cared about this ordering anyways. =(


Nothing stops us from wiring this type up with instances for use with `linear` as a vector space:

```haskell
instance Applicative Fib where
  pure a = Fib a a
  Fib a b <*> Fib c d = Fib (a c) (b d)

instance Monad Fib where
  return a = Fib a a
  Fib a b >>= f = Fib a' b' where
    Fib a' _ = f a
    Fib _ b' = f b

instance MonadZip Fib where
  mzipWith f (Fib a b) (Fib c d) = Fib (f a c) (f b d)
  munzip (Fib (a,b) (c,d)) = (Fib a c, Fib b d)

instance Additive Fib where
  zero = Fib 0 0
  (^+^) = (+)
  (^-^) = (-)
```

and it may wind up in there at some point if I get bored.

# The Fast Fibonacci Transform

If we're extending a ring/field with φ, why did I call it `Fib`?

Well, with one more observation, we can now write the efficient notion of `fib`.

Let's take our multiplication formula

```haskell
  Fib a b * Fib c d = Fib (a*(c + d) + b*c) (a*c + b*d)
```

and consider the effect of multiplying by φ:

```haskell
Fib a b * φ = Fib a b * Fib 1 0 = Fib (a*(1+0)+b*1) (a*1 + b*0) = Fib (a+b) a
```

Multiplying by φ shifts `a` to the right, and adds the previous `b`. This is the same structure as the
"cursor" we used last time when the number to the right is the previous Fibonacci number!

This gives us

```haskell
Fib (fib n) (fib (n-1)) * φ = Fib (fib (n+1)) (fib n)
```

(The above formula can be rearranged to see why the ratio between consecutive Fibonacci numbers tends to φ in the limit.)


Of course, we can compute `(^)` using peasant exponentiation, by repeated squaring rather than
working one factor at a time. This is what Lennart's version of `(^)` which is used by GHC today
does for us already, so we don't need to write it.

With that:

```haskell
getPhi :: Fib a -> a
getPhi (Fib a _) = a 

-- | Compute the nth Fibonacci number in O(log n) 
fib :: IntegralDomain a => Integer -> a
fib n
  | n >= 0 = getPhi (Fib 1 0 ^ n)
  | otherwise = getPhi (Fib 1 (-1) ^ negate n)
```

This has the benefit of nicely extending to negative Fibonacci, unlike the usual boring definitions people tend to write. Moreover, if we don't `getPhi` at the end then we get both the desired Fibonacci number and the preceding Fibonacci number, which is precisely what we need in order to move around in the sequence with `Fib` as our "cursor"!

As mentioned earlier, this same technique can be used to jump quickly to any element of any homogeneous linear recurrence in `O(log n)` time and we can navigate from there, so we're not limited to doing this with Fibonacci numbers. You can jump around in any recurrence you like.

# Fibonacci Search

Remember why I got started here? Oh yeah, there was something about searching.

[Fibonacci search](http://en.wikipedia.org/wiki/Fibonacci_search_technique) was invented back in the 50s by [Jack Kiefer](http://en.wikipedia.org/wiki/Jack_Kiefer_(statistician)). It is traditionally defined in terms of a closed space we want to search.

To search using the Fibonacci sequence we slightly modify the binary-tree like nature of binary search to use Fibonacci numbers instead.

_E.g._ given 100 elements, you find the first Fibonacci number >= 100 and use that as your high end, and start below at 0.

If we have a Fibonacci number of elements, then instead of dividing in half, we can divide it into biased halves with sizes based on the two previous Fibonacci numbers. You can pick if you want the smaller on the left or the right and test the "midpoint" like traditional binary search. This is a fun programming exercise.

# Open-Ended Fibonacci Search

But what about unbounded search? 

An "upwardly closed predicate" `p` on the natural numbers is a predicate `p :: Natural -> Bool` for which there exists `n`, such that `p n` holds, and for all k, `p k` implies `p (succ k)`.

Given an upwardly closed predicate, open-ended binary searching proceeds by repeatedly squaring a power of 2, until it finds an upper bound which passes the predicate, then binary searching within the resulting interval.

But with the machinery above it is easy to see that we can do this same thing now with repeated squaring of φ to get results of form `aφ + b`, where `a = fib(2^i)`, `b = fib(2^i - 1)` until `p a` holds, then we have the two consecutive Fibonacci numbers `b` and `a`, where `b` is the size of one of the two branches we want to cut `a` into, and `a-b` is the other, and we know the predicate holds at `a`.

From there we're well equipped to start a traditional Fibonacci search.

# Ripping Out All The Math

Once we inline all the arithmetic needed for repeated squaring into `bound` and merge all the manipulations of the cursors which are effectively just multiplications by `φ^-1`, and assuming I haven't made the standard undergraduate mistake of screwing up binary search, we get a completely self-contained implementation:

```haskell
search :: (Natural -> Bool) -> Natural
search p
  | p 0 = 0
  | otherwise = bound 0 1
  where
    bound !a !b
     | p b = go 0 a b
     | bb <- b*b = bound (a*a+bb) (bb+2*a*b)
    -- the answer lies in the interval (l,l+k], i,j,k are consecutive Fibonacci numbers.
    go !l !j !k
      | k == 1    = l + 1
      | p m       = go l (j-i) i
      | otherwise = go m i j
      where
        m = l + i
        i = k - j
```

With that, things like

```haskell
>>> search (>12389012380128301283012381203912380192830123801283120931203)
12389012380128301283012381203912380192830123801283120931204
```

are effectively instantaneous.

_NB:_ This version wastes a tiny bit of effort. When we finish with `bound` we'd know the predicate `p` failed for `fib (2^(i-1))`, not just `fib 0`, but the interval size between `fib (2^(i-1))` and `fib (2^i)` has no nice expression in terms of Fibonacci numbers. We could break `go` up into two functions with one for use as long as your candidate range includes this better lower bound, or we could just clutter things up with an additional test to avoid these redundant probes. 

We could also just bias the result upward by `fib(2^(i-1))` and search a window starting from there relying on our bias to avoid heavily searching the known hits at the top. The best choice really depends on the cost of the predicate and how that cost grows as the argument to it increases.

# Further Thoughts

By repeatedly squaring the index of the Fibonacci number in question we're shooting through the list of Fibonacci numbers quickly, so we'll find a bound pretty fast. Approximately every 5th index adds a digit to the result but we're doubling the index every time. Is it worth using / possible to effectively use a slower growth rate to lower the initial top bound? e.g. if we could grow by 1.5x, we'd expect a huge win in reducing the cost of the descent part of the algorithm. We should have time to spend on walking up a bit more slowly. Is there some different scheme that would let us ascend some fraction of the way rather than doubling the index every time?

# Why Care?

This is a biased search algorithm, after all.

Well, if our predicate spends a lot of time rummaging around inside of arrays, we know [binary search is a pathological case for caches](http://www.pvk.ca/Blog/2012/07/30/binary-search-is-a-pathological-case-for-caches/). Paul Khuong showed that a traditional binary search if you had a power of 2 worth of elements was pretty awful in terms of its use of the _k_-way set associative caches we actually have in our CPUs and that at the very least you should bias your search. Here we're able to skew a little more for little effort. This argument winds up resoundingly similar to the case made by Gerth Stølting Brodal in the paper I linked last time about skewed binary trees, but for different reasons.

But even if we aren't rummaging through an array, there are some benefits to Fibonacci search. In the implementation above, I put the smaller tree on the left, so it will tend to favor checking smaller elements. If the cost of testing the predicate on a larger number is higher than testing it on a smaller number, then this biases in the correct direction. 

Compared to an open-ended binary search, open-ended Fibonacci search tries to test smaller numbers with lower dispersion in a biased fashion in exchange for testing more numbers, but we're starting to see that this can be a good thing.

-[Edward Kmett](mailto:ekmett@gmail.com)

April 27, 2015
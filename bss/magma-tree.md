# Algebraic Terraforming: Trees from Magma

At ICFP 2009, Guy Steele critizied the FP community on their use of lists instead of trees.  Trees parallelize better with balanced trees being the ideal case, and a divide-and-conquer approach can often achive good speedup over a more iterative approach.

## Why Choose Lists?

Lists are a very simple recursive, parameterized data type.  They have a trivial base case, and only one recursive case.  Their popularity has much to do with their simplicity: pedantically, they are accessible to a wider audience; academically, they can serve as a concrete example for a generic approach without taking time away from the development of the generic approach; industrially, they are used both due to their lack of time investment (e.g. during prototyping) and reliabilty (i.e. they are unlikely to be the source of your latest test failure).  But are they the simplest?

## All the Algebra You Need to Know (For This Post)

First, some terminology.  A monoid is a set of objects, an associative (binary) operation on those objects, and a particular, known object that is "neutral" with respect to the operation.  A semigroup is simpler; they are almost monoids but do not require the neutral element.  A [magma](http://en.wikipedia.org/wiki/Magma_(algebra)) is simpler still; they are almost semigroups but do not require the operation to be associative.  You've probably used monoids without knowing they were monoids; you may have even used some of the universal properties of monoids without realizing they applied to all monoids.

A [free object](http://ncatlab.org/nlab/show/free+object) over a set forgets everything about that set except some universal properties, specified by the word following free.  For example, the free monoid over Integers forgets unique factorization, unique representation in every base, the GCD function, and everything else about the Integers except: they are a set of objects, there is an associative (binary) operation on Integers, and there is a "neutral" Integer; precisely the universal properties of monoids.


## Lists viewed Algebraically

Now, it turns out that `[a]` is a free monoid of values of type `a`.  Our first pass at constructing a free monoid might look something like this

``` haskell
data FreeMonoid a
    = Neutral
    | Single a
    | Operation (FreeMonoid a) (FreeMonoid a)
```

In our free monoid we have a representation of the netural element, a representation of any value from the underlying set, and a representation of applying an operation.  Unfortunately, this way of building a free monoid lacks canonicity.  That is, there are multiple ways to apply these constructors that give equivalent objects.  The representation of `(a * b) * c` looks like `Operation (Operation (Single a) (Single b)) (Single c)` and the represenation of `a * (b * c)` looks like `Operation (Single a) (Operation (Single a) (Single b))` but in a monoid `(a * b) * c = a * (b * c)`!  We'd like the representation of equivalent things to be the same.

### Ensuring associativity

First, let's have our representation always group to the right.  That should address the current issue.  This simplest way to do that is to change our `Operation` constructor so that it's left argument is always "small" i.e., it never has a nested `Operation` constructor.  That will make things a bit more complicated, but we'll clean this up shortly.

``` haskell
data FreeMonoid a
    = Atom (FMAtom a)
    | OpAssoc (FMAtom a) (FreeMonoid a)

data FMAtom a
    = Neutral
    | Single a

op :: FreeMonoid a -> FreeMonoid a -> FreeMonoid a
op (Atom h)      t = OpAssoc h t
op (OpAssoc h m) t = OpAssoc h (op m t)
```

Now, we can show that `(a * b) * c` and `a * (b * c)` give the same representation, using equational reasoning:

```
  -- (a * b) * c
  op (op (Atom a)     b) c
= op (OpAssoc  a      b) c
= OpAssoc      a  (op b  c)
= op     (Atom a) (op b  c)
  -- a * (b * c)
```

With that change, it's no longer necessary to be explicit with parentheses, at least as long as we are talking about out free monoid, but we still have canonicity problems.  If we call our neutral element `e`, then `e * a * e = a` but again we still generate different representations.

### Ensuring neutrality

We can generally add or remove our neutral element from expressions without affecting equality, so it might be a good idea to have all our canonical representations have the same number of neutral elements.  Still calling our neutral element `e`, there's one expression (namely `e`) that requires at least one neutral element.  So, we make our new representation always have *exactly* one neutral element.  Again, this is a change in the type of our `OpAssoc` constructor; we'll now require the left argument to have no neutral elements and the right argument to have exactly one.  We also lose the `Atom` constructor from our main type--not enough neutral elements (zero, instead of the required one)--but re-introduce the `Neutral` constructor into our main type.

``` haskell
data FreeMonoid a
    = OpAssoc (FMAtom a) (FreeMonoid a)
    | Neutral

data FMAtom a = Single a
```

Wait just a minute, that's wasteful.  `FMAtom a` and `a` are really the same thing.  Let's try that again.

``` haskell
data FreeMonoid a
    = OpAssoc a (FreeMonoid a)
    | Neutral

single :: a -> FreeMonoid a
single s = OpAssoc s Neutral

op :: FreeMonoid a -> FreeMonoid a -> FreeMonoid a
op Neutral t       = t
op (OpAssoc h m) t = OpAssoc h (op m t)
```

That should clear up our canonicity problems.  Reasoning:

```
  op Neutral (single a) -- e * a
= single a              -- a
= OpAssoc a Neutral
= OpAssoc a (op Neutral Neutral)
= op (OpAssoc a Neutral) Neutral
= op (single a) Neutral -- a * e
```

A few simple renames: FreeMonoid -> [], OpAssoc -> :, Neutral -> Nil, and op -> (++), and it's clear that our free monoid is just your standard (singly-linked, cons) list.  If we did our associations to the left we would have gotten a singly-linked *snoc* list.

As an aside, non-empty list with elements of type `a` are a free semigroup over `a`.  Again, with the choice of associativity giving chosing the cons or snoc representation.

## Trees viewed Algebraically

Turns out that binary trees with values of type `a` at the leaves is a free magma over `a`.  Here's a first attempt at building a free magma:

``` haskell
data FreeMagma a
    = Single a
    | Operation (FreeMagma a) (FreeMagma a)
```

In our free magma, we have the representation of any value from the underlying set, and a representation of applying an operation.  Now in the free monoid case, our first representation had some problems.  But they were caused by two things: the neutral element and associativity.  When moving from discussing monoids to semigroups, we lost the neutral element, so there's no work to do there.  When moving from discussing semigroups to magmas, we also lost associatity, so there's no work to do there.  In fact, this is a perfectly valid free magma.

If you don't see the trees for the forest yet, try renaming: FreeMagma -> Tree, Single -> Leaf, Operation -> Branch.  

## For Simplicity Choose Trees

We started from a simpler algebraic object (a magma instead of a monoid), translated it directly, did no additional work, and got trees.  Compare this with the several steps needed to derive lists from the definition of a monoid, and I think you'll agree that trees are actually simpler than lists!  Next time you are choosing a data structure for your tutorial, paper, or project, if simplicity is your primary motivator, choose trees.

More complex trees form the foundation of many high-performance persistent data sturctures, arise naturaly when parsing and evaluating, and are key in many wire protocols (e.g. JSON-RPC or REST).  With associative operations, even the lowly free magma can still be used as an intermediate, non-canonical representation to delay reassociation which may improve preformance.
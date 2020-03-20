# From Zero to Comp Data, Part 2: Base Functors

In the introduction I opened with the question "What are recursive
data types, *really*?". Let's start to break that down here.

We'll first explore one mathematically natural definition---though
there are other perspectives, especially for Haskell's data
types---that a recursive data type is an initial `F`-algebra
for some *base functor* `F`.

Without trying to define that notion yet, let's just take it as a
guide that base functors, whatever they are, are important.

## Base functors

Base functors let us extract from a recursive data type a
non-recursive piece. We'll later see that the remaining recursive bits
are generic and it is this non-recursive piece that forms the nature
of a recursive data type. You might say that base functors are
simple lenses which refract essential recursion into the various
shapes we need.

Immediately, here are some example recursive data types and their
base functors.

``` haskell
-- numbers
data Nat    = Zero | Succ Nat
data NatF a = Zero | Succ a

instance Functor NatF where
  fmap f Zero     = Zero
  fmap f (Succ a) = Succ (f a)

-- lists
data List  b   = Nil | Cons b (List a)
data ListF b a = Nil | Cons b a

instance Functor (ListF b) where
  fmap f Nil = Nil
  fmap f (Cons b a) = Cons b (fmap f a)

-- n-ary trees
data Tree  b   = Branch b [Tree b]
data TreeF b a = Branch b [a]

instance Functor (TreeF b) where
  fmap f (Branch b as) = Branch b (map f as)
```

For these simple patterns, the idea ought to be obvious enough. We add
an extra type parameter, here called `a`, which replaces the
recursive name in each data type. Note that technically
`TreeF` is still recursive since it uses `[]`, a
recursive data type in itself. We could technically extend the base
functor notion to remove all recursiveness, but for simplicity we'll
just ignore it for now.

There is one complexity when we consider `ListF` and
`TreeF`, though, in that we're used to them being functors
over the type parameter `b`. While it's not yet clear why,
we've given them `Functor` instances over the new
parameter. This is a clear mismatch from the data type the each
base is supposed to somehow represent. We'll want that original
`Functor` behavior later, though, so we'll create a new
typeclass for data types which could be functors on two type
parameters.

```haskell
-- a functor on two type arguments
class Bifunctor f where
  bimap :: (b -> b') -> (a -> a') -> f b a -> f b' a'
  
instance Bifunctor TreeF where
  bimap f g (Branch b as) = Branch (f b) (map g as)
```

It's worth noting that `Bifunctor` could be extended to
`Trifunctor` and `Quadrafunctor` and so on---while
Haskell makes it appear that parameter position is important,
`Bifunctor` makes it clear that with types like
`TreeF` we can "lift" functions up into any of the parameter
slots. When we do, each lifting simply acts independently.

### When is a functor not a `Functor`?

At this point, it's a good idea to take a short digression into a kind
of parallel univese of `Functor`s and examine the friction between
mathematical functors and Haskell `Functor`
instances. Consider the `Reader` functor which represents
the computation of a value `a` in the context of a value
`b`.

```haskell
data Reader b a = Reader (b -> a)

instance Functor (Reader b) where
  -- transform our result by *composing* a function
  fmap f (Reader g) = Reader (f . g)
```

This is clearly a `Functor`, but it has an extra type
parameter like `TreeF` above. Is it also a
`Bifunctor`?

The answer is no, it cannot be. It is instead what's known
mathematically as a "contravariant functor", which is like a functor
but *backwards*.

```haskell
class Contravariant f where
  --           a-to-b becomes... b-to-a?
  contramap :: (a -> b) -> (f b -> f a)
```

So, a `Contravariant` functor is "a functor that reverses
arrows". Generally, all data types are either covariant
`Functor`s, `Contravariant` functors, or both in
each of their arguments. We can talk fluently about these variances if
we use the notation `+` to represent a covariant position,
`-` to represent a contravariant position, and `?`
to represent an "impartial" position. In this notation we can annotate
some functors

```haskell
TreeF  b a   ~~~>    TreeF  + +
ListF  b a   ~~~>    ListF  + +
Reader b a   ~~~>    Reader - +

data Const b a = Const b -- ignores its second parameter
~~~> Const + ?
```

When we write these, they're called a signature. The notation also
suggest a method for determining the signature of any type. We begin
by labeling result types as covariant (`+`), input types as
contravariant (`-`), and unused types as `?`.

```haskell
data F1 a b c d e f g = F1 (a -> c -> d | a -> b -> d )
~~~> F1 - - - + ? ? ?
```

Then, we just *multiply* the variances for compound types, like so

```haskell
data F2 a b c d e = F2 (  (a -> b) -> c | F1 d d d e e e e )
                  = F2 ( -(- -> +) -> + | F1 - - - + ? ? ? )
                  = F2 (  (+ -> -) -> + | F1 - - - + ? ? ? )
~~~> F2 + - + - +
```

where `?` multiplied by `-` or `+` is
just `-` or `+` respectively.

Since `Reader` has signature `Reader - +`, we can
make it a `Functor`, we can make "flipped reader" a
`Contravariant` functor, or we can make `Reader` itself a
`Profunctor`, which is the name for a functor of signature `- +`.

```haskell 
data FlippedReader a b = FlippedReader (b -> a)

instance Contravariant (FlippedReader a) where
  -- precompose our function, making the reader take a new kind of argument
  contramap f (FlippedReader g) = FlippedReader (f . g)

class Profunctor f where
  promap :: (b -> b') -> (a -> a') -> (f b' a -> f b a')
  
instance Profunctor Reader where
  promap f g (Reader h) = Reader (g . h . f)
```

## Recap

We've examined recursive data types and taken note of a way to
simplify recursive datatypes into their deflated base
functors. Next we'll reinflate base functors to recapture recursive
data types, and begin to explore what that separation gives us.

We've also explored the notion of a functor a bit to learn about
variance signatures. Variance signatures are useful tool for
describing functors and functors of different signatures will be
important later.

### Questions

1. Can we form base functors for more exotic data types? Consider

    `
    data X  c b   = X1 c | X2 (X c b -> b)
    `

    What is `X`'s base functor? What is its signature?

2. What is the base functor of `Tree b` eliminating both
   the `Tree` recursion and the `[]` recursion?

3. What happens if a type parameter is both contravariant and
   covariant? How can we extend the notation and method to handle this
   case?

4. Since `Reader` cannot be a `Bifunctor`
   instance, but it's still a mathematical functor, what would a
   typeclass a functor like `Reader` look like?


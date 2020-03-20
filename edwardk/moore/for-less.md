# Moore for Less

I was playing around while stuck on a plane this morning, and realized a few things that had previously escaped me about [Moore machines](http://en.wikipedia.org/wiki/Moore_machine).

In previous posts, I've talked about the notion of an (infinite) Moore machine.

Here we have a machine where each state `Moore a b` has a label `b`, and given an input `a` we transition along an edge to a new state. Unlike a traditional Moore machine, we may well have an infinite number of states, which removes all those pesky limitations on what you can recognize with such a machine.

```haskell
data Moore a b = Moore b (a -> Moore a b)
```

In this form it can be seen to be an `Cofree` comonad.

```haskell
data Cofree f a = Cofree a (f (Cofree f a))
```

We can see that `Moore a b` is isomorphic to `Cofree ((->) a) b`.

Soul of a Nu Machine
--------------------

An equivalent definition of a Moore machine that [I've covered before](https://www.fpcomplete.com/user/edwardk/cellular-automata/part-2) is to switch to an explicit 'state' type.

We can derive that definition from the more direct definition above by one of several different means.

Probably the most straightforward way to do so is to exploit the fact that `Cofree f a = Fix (Compose ((,) a) f)`, where `Fix f` is the greatest fixed point of `f`, and look at the two different ways to encode the greatest fixed point in Haskell.

The usual definition of `Fix` is given by:

```haskell
data Fix f = In { out :: f (Fix f) }
```

This definition exploits the fact that the least and greatest fixed points are the same, but we can also just use the direct definition as the greatest fixed point, which we get when we write down the definition of an anamorphism, which we can use to build a member of the greatest fixed point of `f`:

``` haskell
ana :: Functor f => (s -> f s) -> s -> Nu f
```

and just take its type signature as the definition of `Nu`.

```haskell
data Nu f where
  Nu :: (s -> f s) -> s -> Nu f

ana = Nu 
```

`Nu` represents the greatest fixed point in a much more portable way than `Fix`. `Fix` only really works as a greatest fixed point due to laziness in Haskell. In a strict language these two defintions are not equivalent.

Substituting `Nu` into `Cofree` instead of `Fix` we get:

```
Cofree f b =
Nu (Compose ((,) b) f) =
∃s. (s -> Compose ((,) b) f s, s) =
∃s. (s -> (b, f s), s) =
∃s. (s -> b, s -> f s, s)
```

In the particular case of a Moore machine:

```
Moore a b = 
Cofree ((->) a) b =
∃s. (s -> b, s -> a -> s, s)
```

This corresponds to the data type:

```haskell
data Moore a b where
  Moore :: (s -> b) -> (s -> a -> s) -> s -> Moore a b
```

I've talked about this data type before, as has [Gabriel Gonzalez](http://www.haskellforall.com/2013/08/composable-streaming-folds.html). I have it packaged up in [my `folds` package](http://hackage.haskell.org/package/folds), and he has a version of it in [his `foldl` package](http://hackage.haskell.org/package/foldl).

Distributive Functors, Represent!
---------------------------------

Now for something new: In category theory a representable functor `f` is a functor for which there exists an object `x` such that we can equip our functor with a natural isomorphism between `f a` and `(x -> a)`. It describes all of the arrows out of some object `x`.

I have long had this definition split across two packages.

We first have the "haskell 98" `distributive` package, which provides

```haskell
class Functor g => Distributive g where
  distribute :: Functor f => f (g a) -> g (f a)
  collect :: Functor f => (a -> g b) -> f a -> g (f b)
```

`Distributive` is effectively a co-`Traversable`. Given that the package is Haskell 98 / 2010, it can't supply us `x`. For every `Distributive` functor such an `x` should exist though. In the `adjunctions` package, you can get your hands on it with:

```haskell
class Distributive f => Representable f where
  type Rep f
  tabulate :: (Rep f -> a) -> f a
  index :: f a -> Rep f -> a
```

Here, `x = Rep f`, and `tabulate` and `index` are inverses.

Cofree Anyone?
--------------

There is an obvious representation for `((->) x)`, as `(x -> a)` is clearly isomorphic to `(x -> a)`!

```haskell
instance Representable ((->) x) where
  type Rep ((->) x) = x
  tabulate = id
  index = id
```

With this we can just randomly embellish the definition of a cofree comonad by explicitly choosing `s` to be the representation of _some_ representable functor, and just not telling me what it is.

```haskell
data Cofree f a where
  Cofree :: Representable k => k a -> k (f (Rep k)) -> Rep k -> Cofree f a

instance Functor (Cofree f) where
  fmap f (Cofree k u s) = Cofree (fmap f k) u s

instance Comonad (Cofree f) where
  extract (Cofree k _ s) = index k s
  duplicate (Cofree k u s) = Cofree (tabulate (Cofree k u)) u s
```

Representable Machines
----------------------

Now we're equipped to play with a similarly modified definition of a Moore machine.

```haskell
data Moore a b where
  Moore :: Representable k => k b -> k (a -> Rep k) -> Rep k -> Moore a b
```

Above and beyond what we can say about `Cofree` in general, we know something else in the `Moore` case, namely that both our unknown representable functor `k` and `(->) a` are `Distributive`, so we can freely interchange them in the definition above. This yields the following definition, in terms of which all subsequent instances are defined:

```haskell
data Moore a b where
  Moore :: Representable k => k b -> (a -> k (Rep k)) -> Rep k -> Moore a b
```

Since the `u` argument isn't used at all in the definition of the `Comonad` for `Cofree f` and is just silently passed along, these instances for `Moore a` work either way.

```haskell
instance Functor (Moore a) where
  fmap f (Moore k u b) = Moore (fmap f k) u b

instance Comonad (Moore a) where
  extract (Moore k _ s)   = index k s
  duplicate (Moore k u s) = Moore (tabulate (Moore k u)) u s
  extend f (Moore k u s)  = Moore (tabulate (f . Moore k u)) u s
```

Due to the `Representable ((->) x)` instance no power is lost, but now some choices of `k` might be suitable for memoization, acting as a trie to hold onto the results rather than recomputing them each time they are asked.

Doing Two Things at Once
------------------------

While I've put the `Comonad` for `Moore` to good use in previous posts, much of the original motivation for using `Moore` was to give us the ability to describe how to fuse together multiple passes over the data.

To derive the `Applicative` for our new machine we'll need suitable functors to use to memoize all of our states. Rather than using functions, let's look for some things that are better behaved.

The next simpler instance after the naïve function instance above is:

```haskell
instance Representable Identity where
  type Rep Identity = ()
  tabulate f = Identity (f ())
  index (Identity a) () = a
```
  
Next, given two representable functors, their composition is also representable:

```haskell
instance (Representable f, Representable g) => Representable (Compose f g) where
  type Rep (Compose f g) = (Rep f, Rep g)
  index (Compose fg) (i,j) = index (index fg i) j
  tabulate = Compose . tabulate . fmap tabulate . curry
```

So let's put these instances to work:

```haskell
instance Applicative (Moore a) where
  pure a = Moore (Identity a) (\_ -> Identity ()) ()
  Moore kf uf sf <*> Moore ka ua sa =
    Moore (Compose ((<$> ka) <$> kf))
          (\x -> Compose $ (\y -> (,) y <$> ua x) <$> uf x)
          (sf, sa)

instance ComonadApply (Moore a) where
  (<@>) = (<*>)
```

This is just the definition we used to use for the `foldl`-style `Moore` machine, but now instead of using functions from our state, we just use representable functors that have our states as their representations.

Finally, as a small but useful aside, a `Moore` machine is a `Profunctor`, so we can map contravariantly over the inputs as well as covariantly over the results.

```haskell
instance Profunctor Moore where
  dimap f g (Moore k u s) = Moore (g <$> k) (u . f) s
```

Stepping Lightly
----------------

So how do we run the machine?

We can feed our machine in one of two ways. We can define

```haskell
step1 :: a -> Moore a b -> b
step1 a (Moore k u s) = index k (index (u a) s)
```

and rely on the ability to `extend (step a)` to change out all of the labels on our machine. This unfortunately builds up and tears down a whole representable functor worth of data.

On the other hand, we can simply move the start state and get a whole new machine:

```haskell
step :: a -> Moore a b -> Moore a b
step a (Moore k u s) = Moore k u (index (u a) s)
```

The choice between `step a` and `extend (step1 a)` is indistinguishable to the outside observer except in terms of performance. While `step` isn't a Cokleisli arrow, it is much faster.

Running a Tab
-------------

That said, we don't have to run the machine one step at a time!

Dan Piponi once wrote an article on recognizing a regular language with a monoid. What he did was build a data type to represent the tabulation of transitions in his DFA.

The `a -> f (Rep f)` in the body of our Moore machine suggests what such a tabulation might look like, generically. It takes `a` to a structure that contains references to all the new states.

```haskell
newtype Tab f = Tab { getTab :: f (Rep f) }

instance Representable f => Monoid (Tab f) where
  mempty = Tab $ tabulate id
  mappend (Tab fs) (Tab gs) = Tab (index gs <$> fs)
```

You can view this as a form of `Endo (Rep f)` that happens to be able to memoize the results of each argument to the function if `f` is sufficiently "nice".

We can now feed our machine a whole `Foldable` container at a time.

```haskell
feed :: Foldable f => f a -> Moore a b -> Moore a b
feed as (Moore k u s) = Moore k u (index (getTab $ foldMap (Tab . u) as) s)
```

Compressive Parsing
-------------------

The next trick is finding the right container type to fold over to make use of the memoized internal states.

For this, I'll turn to an old package of mine, `compressed`, which supplies a rather peculiar `LZ78` container type. LZ78 is a compression scheme by Lempel and Ziv that has a number of nice properties for my purposes.

```haskell
data Token a = Token {-# UNPACK #-} !Int a deriving (Eq, Ord)
data LZ78 a
  = Cons {-# UNPACK #-} !(Token a) (LZ78 a)
  | Nil
```
  
The idea is this: 

1.) You start with a dictionary that maps integers to a list of values and which contains a single entry that maps 0 to the empty string.

2.) Now you receive (or generate) a series of `(Int, value)` pairs, where each `Int` represents an existing slot in the dictionary, and the value represents something you want to `snoc` onto the end of it to make a fresh dictionary entry.

More advanced versions of this scheme collect old entries, but we can define a particularly naive LZ78 encoder / decoder very easily.

We can encode using a variety of different constraint types and times depending on how we represent the dictionary during construction.

In *O(n<sup>2</sup>)* we can construct an LZ78 stream using a list internally, but no more than an `Eq` constraint on `a`.

```haskell
encodeEq :: Eq a => [a] -> LZ78 a
encodeEq = go [] 1 0 where
  go _ _ _ [] = Nil
  go _ _ p [c] = Cons (Token p c) Nil
  go d f p (c:cs) = let t = Token p c in case List.lookup t d of
    Just p' -> go d f p' cs
    Nothing -> Cons t (go ((t, f):d) (succ f) 0 cs)
```

With a `Map` our time upgrades to *O(n log n)* with an `Ord` constraint.

```haskell
encodeOrd :: Ord a => [a] -> LZ78 a
encodeOrd = go Map.empty 1 0 where
  go _ _ _ [] = Nil
  go _ _ p [c] = Cons (Token p c) Nil
  go d f p (c:cs) = let t = Token p c in case Map.lookup t d of
    Just p' -> go d f p' cs
    Nothing -> Cons t (go (Map.insert t f d) (succ f) 0 cs)
```

We can also turn to a `HashMap` if we have `Hashable` inputs.

```haskell
encode :: (Hashable a, Eq a) => [a] -> LZ78 a
encode = go HashMap.empty 1 0 where
  go _ _ _ [] = Nil
  go _ _ p [c] = Cons (Token p c) Nil
  go d f p (c:cs) = let t = Token p c in case HashMap.lookup t d of
    Just p' -> go d f p' cs
    Nothing -> Cons t (go (HashMap.insert t f d) (succ f) 0 cs)
```

But regardless of how it was constructed, we can `decode` with `Foldable`.

```haskell
instance Foldable LZ78 where
  foldMap f = go (Seq.singleton mempty) mempty where
    go _ m Nil = m
    go s m (Cons (Token w c) ws) = m `mappend` go (s |> v) v ws where
      v = Seq.index s w `mappend` f c
```

The key here is that the decompression scheme never actually _looks_ at the values it decodes, so it is possible to decompress directly in any target `Monoid` you want.

When you do so you'll gain some sharing of intermediate values.

Other compression schemes may also be useful depending on your application.

The Story So Far
----------------

We can open up such a machine and borrow its internal type of tabulations to generate anything that can be generated by such a machine in parallel or incrementally.

It is possible to run a `Representable` `Moore` machine directly on compressed inputs and pay proportionally to the size of the compressed data, not the decompressed data. 


Representability and Adjunctions
--------------------------------

In category theory we have the notion of an adjunction.

Given two functors `F : D -> C`, and `G : C -> D`, when `F a -> b` is naturally isomorphic to `a -> G b` we describe this situation in one of several equivalent ways, we say that `F -| G`, `G` is right adjoint to `F`, or `F is left adjoint to G`, and if the categories matter, sometimes we'll write `F -| G :: C -> D`. 

Different authors have slightly different conventions on the latter and may give the signature for F instead of G.

* The left (or right) adjoint of a functor is unique up to isomorphism if it exists at all.

* All adjunctions `F -| G :: C -> Hask` have the property that `G` is representable and [`F ()` represents `G`](http://en.wikipedia.org/wiki/Representable_functor#Left_adjoint)

* Since adjoints are unique the fact that the right adjoint is isomorphic to `(F () -> a)` lets us go back across the `(,) (F ()) -| (->) (F ())` adjunction and use the uniqueness of adjoints to see `F a` is isomorphic to `(,) (F ()) a`. In other words, `F` contains exactly one `a`. On top of that, every left adjoint `F :: Hask -> Hask` looks like `F = (,) x` for some `x`.

* [Representations of representable functors are unique up to isomorphism.](http://en.wikipedia.org/wiki/Representable_functor#Uniqueness)

With all of these constraints, if we write down a class describing adjunctions from `Hask -> Hask` it is rather poorly inhabited! All instances of this class are isomorphic (for some `s`) to the canonical `(,) s -| (->) s` adjunction that gives rise to the `State` monad and `Store` comonad!

```haskell
class (Functor f, Representable g) => f -| g where
  leftAdjunct :: (f a -> b) -> a -> g b
  rightAdjunct :: (a -> g b) -> f a -> b
```

First consider that if `f` and `g` are `Representable` then `Product f g` is isomorphic to `(->) (Either (Rep f) (Rep g))` and we can also look at a couple of our recently explored instances:

* `Identity` is isomorphic to `(->) ()`

* If `f` and `g` are `Representable` then `Compose f g` is isomorphic to `(->) (Rep f, Rep g)`

If you squint at these a bit the representation looks a lot like the "logarithm" of the data type in question. Exponents become products, products become sums, etc. Conor McBride is fond of calling representable functors "Napierian" and using `Log` instead of `Rep`.

Since every adjunction gives us a representable functor (the right adjoint), and every representable functor is the right adjoint in some adjunction, and all of this stuff is the same up to isomorphism, we could rephrase everything we just wrote in terms of an adjunction from `Hask -> Hask`.

We get a couple of options (all of equivalent expressive power) for how to arrange things:

```haskell
Moore :: (f -| g) => g b -> f (a -> g (f ())) -> Moore a b
Moore :: (f -| g) => g b -> f (g (a -> f ())) -> Moore a b
Moore :: (f -| g) => f (g b) -> (a -> g (f ())) -> Moore a b
```

I think the last one of those is the most interesting. When `f -| g`, then `f·g` is a comonad and `g·f` is a monad. Here our Moore machine is pairing up some comonad with a function that gives a monadic action that is intimately related to that comonad.

Since all adjunctions from `Hask -> Hask` look like the representable cases we've already explored we haven't yet gained anything from this, but we could work with adjunctions that visit other categories than `Hask`.

I'm My Own Grandpa
------------------

There is a nice adjunction from `Hask -> Hask`<sup>op</sup> that we use in Haskell a great deal.

Using backwards arrows to denote arrows in `Hask`<sup>op</sup>, any such adjunction would look like a statement that `f a <- b` is isomorphic to `a -> g b`. `f` and `g` here are contravariant functors, and if we turn this around we get that `b -> f a` is isomorphic to `a -> g b`.

There is such an adjunction:

```haskell
(-> r) -| (-> r)
```

When composed in one direction we get a monad in `Hask`, when composed the other way around we get a comonad in `Hask`<sup>op</sup>. This is why `Cont r` has no comonadic equivalent. It is its own inverse.

`(b -> a -> r)` be isomorphic to `(a -> b -> r)`, and this is witnessed by `flip` in both directions.

Less Like Moore
---------------

Now, we can mechanically massage that last definition of a Moore machine to go round trip through `Hask`<sup>op</sup>.

```haskell
Wat :: (b -> r -> r) -> a -> (() -> r) -> r
```

The `() ->` in there adds no value, so we can apply it to `()` to get more or less Rich Hickey's notion of a [transducer](http://clojure.org/transducers).

```haskell
type Transducer a b = forall r. (b -> r -> r) -> (a -> r -> r)
```

Hickeys's transducers derive from the type signature of `foldl`:

```haskell
>>> :t foldl
foldl :: (b -> a -> b) -> b -> [a] -> b
>>> :t foldl.foldl
foldl.foldl :: (b -> a -> b) -> b -> [[a]] -> b
...
```

and you can convert from that form to this form with a couple of judiciously placed `flip`s.

So in this sense a `Transducer` is a "generalized Moore machine". The generalization here is powerful enough to allow the transducer to emit multiple `b`s per `a`.

Representing Transducers
------------------------

But now we have functions to and from some arbitrary `r` in our `Transducer` definition and we can replay this same motivating trick we used to exploit representations on that definition as well.

```haskell
type Transducer a b = forall f. Representable f => (b -> f (Rep f)) -> a -> f (Rep f)
```

which is equivalent to

```haskell
type Transducer a b = forall f. Representable f => (b -> Tab f) -> a -> Tab f
```

In fact we can apply such a transducer to a Moore machine to get one that turns each `a` into potentially several `b`s, and makes that whole chain of transitions in the state diagram at once.

```haskell
transduce :: Moore b c -> Transducer a b -> Moore a c
transduce (Moore k u s) t = Moore k (getD #. t (D #. u)) s
```

Finally, if you work for all tabulations of a function, nothing stops you from working for a monoid through `Endo m`, so you might as well just go to:

```haskell
type Transducer a b = forall f. Monoid m => (b -> m) -> a -> m
```

but this is just what `lens` calls a `Fold`

```haskell
type Fold a b = forall f. (Contravariant f, Applicative f) => (b -> f b) -> a -> f a
```

If you are (legally) both `Contravariant` and `Functor` then your argument must be phantom, and using `contramap` and `fmap` you can freely change it to anything you want, so this is isomorphic to the last definition.

Open Thoughts
-------------

Similar changes can be applied to a [coiterative comonad generated by a comonad](http://hackage.haskell.org/package/free-4.12.1/docs/Control-Comonad-Trans-Coiter.html), which looks very similar to the `Mealy` machine even if it has wildly different semantics. But given that coincidence, what does such a `Comonad` mean for a `Mealy` machine that has a `Monoid` on its input type? How would such a machine have to work? What does it do?

Just like we ultimately massaged the transducer into a form where it was obvious we could make the same machinery work for any `Monoid` once it supported tabulated endomorphisms, can do find a series of direct generalizations that take us from a `Moore` machine to one that uses an intermediate `Monoid`? Either like the `M` machine in `folds` or using a Monoid action to update the state instead of accepting just any new state.

We built a tabulation of a deterministic not-necessarily-finite automaton. What about a non-deterministic automaton? For that we can make a set of representations using the trie we get for a representable functor `f`.

```haskell
newtype Set f = Set { getSet :: f Bool }

instance Representable f => Monoid (Set f) where
  mempty = Set $ tabulate (const False)
  mappend (Set as) (Set bs) = Set $ tabulate $ \i -> index as i || index bs i
  
singleton :: (Representable f, Eq (Rep f)) => Rep f -> Set f
singleton i = Set $ tabulate (i==)

insert :: (Representable f, Eq (Rep f)) => Rep f -> Set f -> Set f
insert i (Set is) = Set $ tabulate $ \j -> index is j || (i==j)

contains :: Set f -> Rep f -> Bool
contains (Set is) i = index is i 
```

but it'd probably be better to use a real "Set" in practice for most applications. You need something like `Foldable f` as well as `Representable` in order to make the `Monoid` for

```haskell
newtype N f = N { getN :: f (Set f) }
```

This would start to limit the domain to at-most-countably-infinite automata.

Given that can we define a nice compiler that takes a regular expression builds an NFA state replete with the appropriate functor as it goes, and then converts it to a DFA?

The compressive parsing technique provided by `LZ78` above works for lots of monoids, not just this one.

For instance, we can modify the code in [Efficient Parallel and Incremental Parsing of Practical Context-Free Languages](http://www.cse.chalmers.se/~bernardy/PP.pdf) to work with a `Monoid` rather than the notion of a sequence algebra they use there. (A sequence-algebra can be converted to `Monoid` by using a finger-tree to peel off one symbol of work on one side.)

This would let us parse context-free languages using this same machinery.

The main body of code here is available in this [gist](https://gist.github.com/ekmett/0b9d00c621d34a352e57)

(Also, the code above probably needs a couple of tweaks. Notably, it should use a strict pair for the internal state.)

-[Edward Kmett](mailto:ekmett@gmail.com)
May 28, 2015
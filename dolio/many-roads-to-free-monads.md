# Many Roads to Free Monads

Free monads are handy constructs to have in one's bag of tricks, and have become fairly popular of late. This article attempts to explain their basis in category theory, as well as some interesting/amusing connections that may not have been spelled out elsewhere.

### Free as in Monoids

Free monads arise as a standard sort of construction in category theory. Suppose we have two categories. The first, `D` may be any category, and the second, `C` has objects that are like objects of `D`, but equipped with extra structure or properties. Finally, suppose there is a functor `U : C -> D` that takes the structured objects to the corresponding plain objects, and embeds structure-preserving morphisms into the collection of plain morphisms. This `U` is often called a forgetful functor (although there is no way of formally specifying what 'forgetful' really means). If there exists a functor `F : D -> C` that is left adjoint to `U`:

```
Hom(F A, B) ~= Hom(A, U B)
```

then it is said to generate 'free' objects with structure. In other words, homomorphisms from the free structure over `A` into `B` correspond to 'plain' morphisms from `A` to the 'plain' part of `B`.

A common example takes `C` to be the category of monoids and `D` to be the category of sets. The free monoid functor takes sets `A` to the monoid `([A], (++), [])`. And the adjunction says:

```
Hom(([A], (++), []), (M, mappend, mempty)) ~= A -> M
```

That is, the monoid homomorphisms from the free monoid over `A` to some other monoid `M` are in correspondence with ordinary functions from `A` to `M`. Another way of saying this is that all the behavior of such a homomorphism is determined by the behavior on singleton lists, which makes sense, because the rest of the behavior is determined by the rules for a homomorphism:

``` haskell
h [] = mempty
h (l1 ++ l2) = h l1 `mappend` h l2
```

### Algebras of a Monad

Time for a quick digression. Whenever we have an adjunction `F -| U`, we can get a monad `UF`. In the case of the above adjunction involving monoids, we get the free monoid monad, or list monad as it's usually known in Haskell. In category theory, monads are typically used for algebra. First, one defines the algebras of a monad `T`. These are pairs `(A, a : T A -> A)` (call the first component the 'carrier' and the second the 'action') that follow the following rules:

```
a . return = id
a . fmap a = a . join
```

So, for the free monoid monad, these are pairs `(M, fold : [M] -> M)`, such that:

```
forall (m :: M). fold [m] = m
forall (mss :: [[M]]). fold (map fold ms) = fold (concat ms)
```

One also defines homomorphisms between these algebras, which are maps on the carriers that follow the following rule:

```
h :: (A, a : T A -> A) -> (B, b : T B -> B)
h :: A -> B

h . a = b . fmap h
```

Now, if one looks closely, it turns out that algebras of the free monoid monad are monoids, and the algebra homomorphisms are monoid homomorphisms. And this is a general pattern: if there is a free-forgetful adjunction `F -| U : C -> D` involving categories of algebraic structures, then the category of algebras of the monad `UF` is equivalent to the category `C`.

### Free Monads

Free monads are actually quite analogous to free monoids, except instead of categories of monoids and sets, we use categories of monads and endofunctors. So, there is a forgetful functor:

``` haskell
U :: Monad -> Endofunctor
U (T, join, return) = T
```

And the free monad functor can be given as a Haskell data type, if we're interested in endofunctors that can be represented in Haskell:

``` haskell
data Free f a = Pure a | Roll (f (Free f a))
```

Which is rather like having 0-or-more copies of `f` applied to `a` (instead of 0-or-more elements of a set . The required adjunction similarly says that monad morphisms from `Free F A` to some other monad `M` correspond to natural transformations from `F` to `M`, which is again saying that such morphisms are determined by the 'singleton' pieces of the free monad, and the rest is determined by the structure-preserving rules.

### Another type of algebra

Above we talked about algebras of a monad. However, there is a weaker type of algebra that we talk about more often in Haskell: algebras of a functor. These look much like monad algebras, except there is no structure to the functor (just being an endofunctor), so there are no rules for the 'action' to follow. The algebras of an endofunctor `F` are simply pairs `(A, a : F A -> A)`, and algebra homomorphisms are morphisms on the carriers that commute with the actions.

Data types are given semantics in terms of 'initial F-algebras.' The monad algebras earlier could express generators and equations, but for data types, we need only the former. It is the unit and join rules that enforce the equations of the monad algebras, so we can cover everything needed for data types by saying that they are F-algebras, where `F` specifies generators for the constructors of the type. Being initial ensures that there is in some sense just the structure generated by `F`, and that no structure of `F` has been lost, which is what we want out of a data type.

Do note that in the case of a monad `T`, there is a difference between functor algebras of `T` and monad algebras of `T`. For instance, lists are the free monoid monad. Monad algebras are monoids, but functor algebras aren't necessarily. The initial algebra of the list functor has unadorned rose trees as values:

``` haskell
data Rose = Node { children :: [Rose] }
```

### Free Algebras

However, instead of looking at initial F-algebras, we can look at a connection between the category of F-algebras and the category of types. There is an obvious forgetful functor, which just gives the carrier of the algebra (and algebra homomorphisms are already functions on the carriers---just ones that satisfy a particular property). So we can talk about free F-algebras, such that:

```
Hom((Free F A, ???), (B, b)) ~ A -> B
```

It was pointed out to me recently that this is exactly the same `Free` as above, and `???` is actually `Roll`. This adjunction says that every algebra homomorphism above is determined only by a choice of embedding `A` into `B`; the rest of the structure is determined by rules for homomorphisms (`b` must be applied for all the `F` layers in the `Free F A`, after using the function to create a `Free F B`).

We remarked earlier that every adjunction `F -| U` gives a monad `UF`. And in this case, that monad is the free monad over `F`. This is also a free-forgetful adjunction to a category of algebras (F-algebras), so we can get that the category of F-algebras is equivalent to the category of `Free F` monad algebras. So, above we had two notions of algebras for the list functor. It turns out these can both be expressed as monad algebras: monoids are monad algebras of `[]`, while the rose-tree like things are monad algebras of `Free []`.

One final aside is that there is a well known theorem that "left adjoints preserve colimits" (and right adjoints, limits). `Free F` is a left adjoint, so it will take an initial object `0` to the initial F-algebra.

### General versus specific cases

I was asked about the free algebra stuff recently with regard to why we get the same data type in both cases. The answer, I think, can be explained as part of a general pattern in category theory. Basically, both free monads and the free F-algebra are trying to accomplish the same thing.

We noted earlier that monads and their algebras are used to characterize algebras with generators and equations, and that functors and their algebras can cover generators, but not equations. The category of F-algebras, and free F-algebras do this, but we also saw that we can do both with the free monad. The free monad just takes any functor, and completes it into a monad that enforces no equations, but uses the given functor for the generators, and then monad algebras of `Free F` are just the same as functor algebras of `F`. It is just two ways of achieving the same result.

However, the way we specified free monads used an adjunction `Free -| U` between a category of monads and one of endofunctors. For such an adjunction to exist, it is required that there be a free monad for _all_ `F`. This is true in Haskell, because it is 'nice' in many ways. But in a general categorical setting, it is possible for that to fail, so that there is no free monad functor. However, in such a case, there may still be specific choices of `F` such that free `F` algebras exist. So our second adjunction would work (for some `F`), but the first wouldn't. Those free algebra functors would still give rise to a monad, and one could call it the free monad over `F`, but it would be for particular points, so to speak.

One could actually take this down another level. We also specified free F-algebras via an adjunction, but this assumes that there is such a construction for all choices of `A`. It is possible for that not to be true, but for there to be specific choices of `A` for which a type like `Free F A` exists. We could construct a diagram involving `A + F _`, such that a colimit of the diagram would have the right properties, for instance. But there might not be such a colimit for all `A`.

### A new slogan

So, above we have shown how free monads are related to free algebras, and noted that F-algebras are `Free F` monad-algebras. We also saw similarities between monoids and monads. There is, of course, a famous line that, "monads are monoids in the category of endofunctors." Let's play a bit more with these similarities.

Earlier we went through how the category of monoids is equivalent to the category of free monoid monad algebras. But, our slogan above just said that monads are themselves like monoids. So, we should be able to make a similar statement about categories of monads.

Again, the free monoid monad comes from the adjunction `F -| U`. `UF` forms a monad, and this is the monad used to characterize the algebraic structure. But, we now have `Free -| U` as an adjunction between a category of monads and a category of endofunctors. So, `U . Free` is itself a monad _on endofunctors_:

``` active haskell
{-# LANGUAGE RankNTypes #-}
{-# LANGUAGE TypeOperators #-}
import Control.Applicative
import Control.Monad
-- show
type f :-> g = forall a. f a -> g a

data Free f a = Pure a | Roll (f (Free f a))

instance Functor f => Functor (Free f) where
  fmap g (Pure a) = Pure $ g a
  fmap g (Roll f) = Roll $ fmap g <$> f

instance Functor f => Monad (Free f) where
  return = Pure
  Pure x >>= g = g x
  Roll f >>= g = Roll $ (>>= g) <$> f
  
eta :: Functor f => f :-> Free f
eta = Roll . fmap Pure

mu :: Functor f => Free (Free f) :-> Free f
mu (Pure a) = Pure a
mu (Roll f) = join $ mu <$> f

-- /show
main = putStrLn "typechecks"
```

The typical naming convention seems to be to call this the 'free &lt;X&gt; monad', where &lt;X&gt; is the richer type of algebraic structure. Since the algebraic structure is monads, this is the free monad monad. And, the category we went through to get it is the (a) category of monads, so the category of monad-algebras is equivalent to this category of monads. This leads to the excellent slogan (with some punctuation to help):

> Monads are just free-monad monad monad-algebras. What's the problem?
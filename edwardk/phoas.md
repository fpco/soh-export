# PHOAS For Free

Back in 2008 I wrote an article entitled ["Rotten Bananas"](http://comonad.com/reader/2008/rotten-bananas/) about how to convert between the different forms of catamorphisms over "exponential" data types that arise when we go to work with Higher Order Abstract Syntax (HOAS). 

Today I want to go back and revisit that post in light of my current understanding of profunctors from working on [`lens`](http://hackage.haskell.org/package/lens).

My goal today is to go through and reformulate Parametric HOAS slightly differently by using profunctors to tease apart the positive and negative occurences of the type variable.

By doing so, we can show the connection between Fegaras and Sheard's catamorphism and the free monad that laid so close to the surface in the original formulation, and we can derive a variant on Weirich and Washburn's efficient catamorphism by using an alternate encoding of the free monad that I've already [blogged about before](http://comonad.com/reader/2011/free-monads-for-less-2/) in my series on ["Free Monads for Less"](http://comonad.com/reader/2011/free-monads-for-less/).

Folding Invariants
==================

If we take the base functor for an expression type that has both positive and negative occurences of a variable, there isn't much we can do with our standard tools. It isn't even a `Functor`, so it can't be `Foldable` or `Traversable`, `Applicative` or a `Monad`.

```haskell
data ExpF a
  = App a a
  | Lam (a -> a)
```

To work with it, you can define a rather unsatisfying

```haskell
class Invariant f where
  invmap :: (a -> b) -> (b -> a) -> f a -> f b
  
instance Invariant ExpF where
  invmap ab ba (App x y) = App (ab x) (ab y)
  invmap ab ba (Lam aa)  = Lam (ab.aa.ba)
```

I described this in my 2008 post as `ExpFunctor`, based on a similar operation in Weirich and Washburn. It is packaged up in the `invariant` package on Hackage as `Invariant`, with a method named `invmap`, if you find you  really want to use it.

One of my goals today is to show that we can get away without it!

Erik Meijer and Graham Hutton showed in ["Bananas in Space"](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.64.4921&rep=rep1&type=pdf) that you can define a catamorphism over that data type if you can _also_ define a corresponding anamorphism that serves as its inverse. E.g. to define a pretty printer for this type with Meijer/Hutton's catamorphism you also need a parser. ಠ__ಠ

```haskell
newtype Mu f = In { out :: f (Mu f) } 

cata :: Invariant f => (f a -> a) -> (a -> f a) -> Mu f -> a
cata f g (In x) = f (invmap (cata f g) (ana f g) x)

ana :: Invariant f => (f a -> a) -> (a -> f a) -> a -> Mu f
ana f g x = In (invmap (ana f g) (cata f g) (g x))
```

[Fegaras and Sheard](http://citeseer.ist.psu.edu/viewdoc/summary?doi=10.1.1.36.2763) showed that if you change the domain of the problem a bit, you can get away with a 'fake' anamorphism, by adding a constructor that you agree never to use, because the anamorphism is only ever used as a right inverse to our catamorphism. They then proceeded to provide a type system for checking this. Adopting the notation from Weirich and Washburn's take on Fegaras and Sheard's catamorphism, this would be:

```haskell
data Rec f a = Place a | Roll (f (Rec f a))

cata :: Invariant f => (f a -> a) -> Rec f a -> a
cata f (Roll x) = f (invmap (cata f) Place x)
cata f (Place x) = x
```

You can clearly see that `Place` forms a right inverse of `cata f`:

```haskell
cata f . Place = id
```

Now you can define smart constructors for the running `Exp` example, using `Roll`.

```haskell
type Exp = Rec ExpF

lam :: (Exp a -> Exp a) -> Exp a
lam f = Roll (Lam f)

app :: Exp a -> Exp a -> Exp a
app x y = Roll (App x y)
```

Weirich and Washburn noted that you can prevent the accidental use of the type parameter `a` by quantifying over it.

Finally, Weirich and Washburn reimplemented `Rec`.

```haskell
type Rec f a = (f a -> a) -> a
```

This made `cata` ridiculously simple:

```haskell
cata :: (f a -> a) -> Rec f a -> a
cata f x = x f
```

But we don't have `Roll` any more, so we need to move the complexity into an explicit `roll` combinator:

```haskell
roll :: Invariant f => f (Rec f a) -> Rec f a
roll x f = f (invmap (cata f) place x)
```

We no longer have `Place` either, but nothing says `(f a -> a) -> a` has to use the supplied function, when we know `a`!

```haskell
place :: a -> Rec f a
place = const
```

Then we can see:

```haskell
type Exp a = Rec ExpF a

lam :: (Exp a -> Exp a) -> Exp a
lam f = roll (Lam f)

app :: Exp a -> Exp a -> Exp a
app x y = roll (App x y)

var :: a -> Exp a
var = place
```

Again, in particular in both the Fegaras-Sheard and Weirich-Washburn forms, you can denote a closed term safely using

```haskell
type ClosedExp = forall x. Exp x
```

The safety of this is really the subject of the "Boxes Go Bananas" paper.

Based on the observations about the price of substitution into free monads by [Janis Voigtländer](http://www.iai.uni-bonn.de/~jv/mpc08.pdf), one should probably favor Weirich and Washburn's construction if you're going to be doing substitution on your HOAS representation -- and if you're not doing substitution, then why are you using HOAS in the first place?!

Weak HOAS
=========

Everything I've mentioned above is a "strong" HOAS variants. You can perform substitution just by passing in the expression you want.

In weak HOAS (and PHOAS) the decision is made to deal with negative occurences differently than in what we've done so far.

Instead of taking a full `Exp a` in negative position, we're just going to take an `a`.

Written monolithically, it would look something like:

```haskell
data Exp a
  = Var a
  | Lam (a -> Exp a)
  | App (Exp a) (Exp a)
```

Then our smart constructors change a bit:

```haskell
lam :: (Exp a -> Exp a) -> Exp a
```

weakens to

```haskell
lam :: (a -> Exp a) -> Exp a
```

We still have both positive and negative occurences of `a`, but we no longer have `Exp` occurring in both positive and negative position.

This is particularly popular in the Coq and Agda communities because it can pass the positivity checker, but unlike strong HOAS can be a bit more painful to work with.

Parametric HOAS
===============

[Adam Chlipala](http://adam.chlipala.net/) does a lot of work with Parametric HOAS, which is based on a weak HOAS variant of Weirich and Washburn's [Boxes Go Bananas: Encoding Higher-Order Abstract Syntax with
Parametric Polymorphism](http://www.seas.upenn.edu/~sweirich/papers/itabox/icfp-published-version.pdf), which I talked about in that post. His [Lambda Tamer](http://ltamer.sourceforge.net/) is based on this model.

One issue is that neither of these really gives us a free `Monad`, due to the fact that the `a` occurs in both positive and negative position and so we can't even derive a `Functor` instance. 

Adam's work doesn't really feel this pinch, because in [Coq](http://coq.inria.fr/) he can [define custom tactics](http://adam.chlipala.net/papers/PhoasICFP08/PhoasICFP08Talk.pdf) for [`ltac`](http://coq.inria.fr/files/adt-30jun09-bruno-ltac.pdf) to use and you generally don't try to factor out this pattern into a library, but when you switch to [Agda](http://wiki.portal.chalmers.se/agda/pmwiki.php) and have to do everything by hand, the lack of common abstractions comes to bite you.

P is for Profunctor
===================

Dan Piponi wrote a nice article on [Profunctors in Haskell](http://blog.sigfpe.com/2011/07/profunctors-in-haskell.html) introducing profunctors to the Haskell community, and they now form the basis of much of `lens`. Liyang HU also has a more [tongue in cheek introduction](https://www.fpcomplete.com/school/pick-of-the-week/profunctors) here on the School of Haskell that he originally presented back in April at a [benkyoukai](http://comonad.com/reader/2013/japanese-workshop-1/) about various libraries of mine in Japan. 

To say something is a `Profunctor` in Haskell is just to say it is contravariant in the first argument and covariant in the second. 

```haskell
class Profunctor p where
  dimap :: (a -> b) -> (c -> d) -> p b c -> p a d
```

This argument order is a bit reversed from how it is usually tackled in category theory, but by lining things up this way we match up with the variances for `Arrow` and `(->)`.

For convenience we also provide class methods for:

```haskell
lmap :: Profunctor p => (a -> b) -> p b c -> p a c
rmap :: Profunctor p => (b -> c) -> p a b -> p a c
```

Notice when I wrote the Weak HOAS definition, I had to keep it monolithic? To factor out `Var` / `Place`, it winds up needing two type parameters, since now both `a` and `Exp a` occur inside the expression.

So, if we go back to the start and separate positive and negative occurences of these, we're left with something like

```haskell
data ExpF a b 
  = App b b 
  | Lam (a -> b)
```

This forms a valid `Profunctor`.

```haskell
instance Profunctor ExpF where
  dimap f g (App x y) = App (g x) (g y)
  dimap f g (Lam h)   = Lam (g.h.f)
```

PHOAS Is Free
=============

Now we can revisit the `Exp` we were talking about in the section on Weak HOAS and we can rip apart the old:

```haskell
data Exp a
  = Var a
  | Lam (a -> Exp a)
  | Abs (Exp a) (Exp a)
```

into the application of a Fegaras and Sheard free-monad-like construction

```haskell
data Rec p a b
  = Place b
  | Roll (p a (Rec p a b))
```

to our base profunctor `ExpF`, so now

```haskell
type Exp = Rec ExpF
```

But now, we're no longer free-monad like, we're actually a free monad!

We can even define this `Monad`, by cribbing the definition from [`free`](http://hackage.haskell.org/package/free).

```haskell
instance Profunctor p => Monad (Rec p a) where
  return = Place
  Place b  >>= f = f b
  Roll bs >>= f = Roll $ rmap (>>= f) bs
```

This `Monad` even performs capture avoiding substitution.

The Fegaras-Sheard catamorphism doesn't change much

```haskell
cata :: Profunctor p => (p a b -> b) -> Rec p a b -> b
cata phi (Place b)  = b
cata phi (Roll bs) = phi (rmap (cata phi) bs)
```

However, it now just turns a `p a`-algebra into a `Rec p a`-algebra, and doesn't need to use `Place` at all!

And we get weak HOAS style smart constructors:

```haskell
lam :: (a -> Exp a b) -> Exp a b
lam f = Roll (Lam f)

app :: Exp a b -> Exp a b -> Exp a b
app x y = Roll (App x y)
```

Now, `var` is the only thing that [crosses the streams](http://www.youtube.com/watch?v=jyaLZHiJJnE) and causes the two type arguments to unify

```haskell
var :: b -> Exp a b
var = return
```

Notice the difference between the type of the argument of `lam` and the signature of `var`. This causes actual expressions that use any variable they bind to wind up with the types agreeing:

```haskell
foo :: Exp a a
foo = lam $ \x -> lam $ \y -> app (var x) (var y)
```

Take The End
============

Now instead of talking about a closed term in terms of quantifying over a single parameter that can only vary with an isomorphism, we take an end over our `Profunctor` instead:

```haskell
type End p = forall x. p x x

iter0 :: Profunctor p => (p a a -> a) -> End (Rec p) -> a
iter0 phi x = cata phi x
```

I intend to write up a post on how to "read" universal properties as types, to help folks see where this definition for `End` comes from.

This was the Free monad I just said you probably don't want to use for substitution, so let's go try the other one.

Boxes Go Bananas For Less
=========================

Recall Weirich and Washburn's `Rec`:

```haskell
type Rec f a = (f a -> a) -> a
```

When we go to turn that into a `Profunctor` directly we get stuck.

We could make

```haskell
newtype Rec p a b = Rec { runRec :: (p a b -> a) -> b }
```

and get the variances right for a `Profunctor`.

```haskell
instance Profunctor p => Profunctor (Rec p) where
  dimap f g (Rec h) = Rec $ \pab2a -> g $ h $ f . pab2a . dimap f g
```

But this doesn't look much like the instance I derived from Fegaras and Sheard's construction, and doesn't give us a Monad!

They could pull this off because `a` and `b` were interchangeable in either direction. We're a bit more constrained.

We could turn to `Codensity` of my weak variant of the Fegaras-Sheard construction, but as I've blogged about before, `Codensity (Free f)` is bigger than it needs to be.

Fortunately, that same series ended showing how you can get there with just Yoneda. Armed with that, we can borrow the Church-Free monad and get a Weirich and Wasburn-style `Profunctor` for `Rec`.

```haskell
data Rec p a b = Rec 
  { runRec :: forall r. 
    (b -> r) -> (p a r -> r) -> r 
  }

instance Profunctor p => Profunctor (Rec p) where
  dimap f g m = Rec $ \kp kf -> runRec m (kp.g) (kf.lmap f)
```

We retain the `Monad` we won by splitting our type parameters in the weak Fegaras-Sheard variant above:

```haskell
instance Profunctor p => Monad (Rec p a) where
  return b = Rec $ \ br _ -> br b
  m >>= f  = Rec $ \kp kf -> 
    runRec m (\a -> runRec (f a) kp kf) kf
```

and we can define the rest of the catamorphism machinery:

```haskell
type End p = forall x. p x x

cata :: Profunctor p => (p a b -> b) -> Rec p a b -> b
cata phi m = runRec m id phi

roll :: Profunctor p => p a (Rec p a b) -> Rec p a b
roll w = Rec $ \kp kf -> kf (rmap (\r -> runRec r kp kf) w)

iter0 :: Profunctor p => (p a a -> a) -> End (Rec p) -> a
iter0 phi x = cata phi x
```

Now we can put together our smart constructors

```haskell
lam :: (a -> Exp a b) -> Exp a b
lam f = roll (Lam f)

app :: Exp a b -> Exp a b -> Exp a b
app x y = roll (App x y)

var :: b -> Exp a b
var = return
```

and we can build expressions

```haskell
foo :: Exp a a
foo = lam $ \x -> lam $ \y -> app (var x) (var y)
```

Conclusion
==========

There is a lot more we can play with here. 

Many expression types will admit a `Strong` instance. Now that we've split the input and output parameters can we perhaps use that or something like it to more easily [manipulate environments](http://www.haskell.org/pipermail/haskell-cafe/2008-November/049473.html)?

Just like with [`bound`](https://www.fpcomplete.com/user/edwardk/bound) we can build an [`indexed`](http://github.com/ekmett/indexed) version of this construction that permits us to write strongly typed EDSLs. That is how PHOAS is usually presented in Coq after all.

There is also probably a lot more to be said about dinaturality here.

I still generally prefer working with [`bound`](http://hackage.haskell.org/package/bound) to working in HOAS these days, because it is much easier to work under lambdas, and it is easier to grab all your free variables using `Foldable` and `Traversable` and harder to make mistakes with. 

That said it is good to finally be able to merge together the notion of Fegaras and Sheard's construction and the standard free monad. 

This also strikes me as a good first step towards being able to turn PHOAS into something that can be encoded usefully in Haskell as a library rather than a design pattern.

[-](https://plus.google.com/u/0/113063331545548237308?rel=author)[-](<a href="https://plus.google.com/u/0/113063331545548237308" rel="publisher">-</a>)[Edward Kmett](mailto:ekmett@gmail.com) 

September 18th, 2013
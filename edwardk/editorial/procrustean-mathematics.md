# Procrustean Mathematics

This post is a bit of a divergence from my norm. 

I'm going to editorialize a bit about mathematics, type classes and the tension between different ways of fitting ideas together, rather than about any one algorithm or data structure.

I apologize in advance for the fact that my examples are written from my perspective, and as I'm writing about them unilaterally my characterization is likely to be unfair. Something in here is likely to offend everyone.

## Generalized Abstract Nonsense

The term "generalized abstract nonsense" was originally coined by [Norman Steenrod](http://en.wikipedia.org/wiki/Norman_Steenrod) as a term of endearment, rather than of denigration. *e.g.* Saying in passing "this is true by abstract nonsense" when referring to a long-winded proof that offers no insight into the domain at hand.

Now, some mathematicians like to refer to category theory as [_generalized abstract nonsense_](http://en.wikipedia.org/wiki/Abstract_nonsense), not out of endearment, but because they do not find immediate value in its application. Among category theorists this view is seen as somewhat daft, as they view category theory as a sort of [Rosetta Stone](http://math.ucr.edu/home/baez/rosetta.pdf) for mapping ideas from one area of mathematics to another -- a _lingua franca_ that lets you express commonalities and cross-cutting concerns across domains.

To me category theory serves as a road map to new domains. I don't know much about [rational tangles](http://en.wikipedia.org/wiki/Tangle_%28mathematics%29), but if I know that with 2 dimensions of freedom, they form a braided monoidal category letting me tie myself in er.. knots, but in 3 or more dimensions they form a symmetric monoidal category, letting me untie all knots.

With this, I can work with them and derive useful results without caring about irrelevant details and dealing with rope burn.

## Centipede Mathematics

Just as some general mathematicians look down with various degrees of seriousness upon generalized abstract nonsense, even some category theorists look down upon what the analyst Antoni Zygmund famously referred to as [_centipede mathematics_](http://ncatlab.org/nlab/show/centipede+mathematics):

> You take a centipede and pull off ninety-nine of its legs and see what it can do.

In this sense working with a `Semigroup` is just working with a neutered `Monoid` that has had its unit removed. The usual critique of "centipede mathematics" is that it lacks taste. 

With such colorful metaphors, it'd be hard to argue otherwise!

The negative view of this practice seems to stem from the era of folks evaluating grant proposals to see whether or not it was likely to lead to interesting research that they could use. With fewer parts to use, it would seem that one would be unlikely to find new results that benefit those solely concerned with the larger mathematical object.

But in many ways, all of [modern abstract algebra](http://en.wikipedia.org/wiki/Abstract_algebra#Modern_algebra) can be seen as an exercise in centipede mathematics.

Mathematicians started with the real numbers, which sit on a line, and, coincidentally, with suitable markers, look an awful lot like a centipede. Starting from there at the turn of the last century, mathematicians kept ripping off legs to get fields, rings, groups, monoids, etc.

## Applied Mathematics and Computer Science

Of course, all of these folks are mathematicians, and many mathematicians famously look down their noses at applied mathematicians. 

Consider the famous claim by G. H. Hardy:

> I have never done anything 'useful'. No discovery of mine has made, or is likely to make, directly or indirectly, for good or ill, the least difference to the amenity of the world.

Mind you that didn't stop folks from later putting his ideas to work in thermodynamics and quantum physics, and the gap between pure mathematics and theoretical physics seems to be narrowing every year.

To me the difference between a mathematician and an applied mathematician is one of focus. 

Often a mathematician will start with an abstraction and try to find things that fit, to help give intuition for folks for the more general concept.

Conversely, an applied mathematician will typically start with a thing, and try to find abstractions that capture its essence, giving rise to insight into its behavior. Mathematics is used to provide a bit of [vaseline for the lens](http://en.wikipedia.org/wiki/Go_motion#Petroleum_jelly), blurring away the parts you don't want to focus on.

We aren't pulling legs off a centipede and seeing if it can go. We're trying to understand the behavior of a spider without gluing an extra 92 legs onto its thorax and then wondering why it lacks the strength to climb back into its web.

While not all centipede mathematicians are applied mathematicians, some really do just want to pull apart their abstractions in a Mengelean fashion and understand why they tick, but the art of applying mathematics is largely an exercise in centipede mathematics. At one extreme, roboticists often pull or [blow the legs off](http://www.washingtonpost.com/wp-dyn/content/article/2007/05/05/AR2007050501009.html) their centipedes literally to see how they'll adjust their gait.

## The View from the Bottom

Of course, all of these folks are mathematicians, so they can look down on the lowly computer scientist, the practitioner of an artform that is delightfully neither truly about computers nor, properly, a science. 

Virtually everything we touch in computer science arose from a form of centipede mathematics. 

Constructive logic is the natural vocabulary of computer science. You obtain it by ripping double-negation out of classical logic and watching the system hobble along. The idealized category of Haskell types `Hask` is effectively used as a constructive analogue to `Set`.

## Theseus and Procrustes

We can, however, turn this perspective around.

In Greek mythology, Procrustes served as the final trial of Theseus during his travels to Athens. He would offer travelers along the road food and a night's lodging in a bed he promised would be a perfect fit.

Upon their repose, he would proceed to set to work on them with a hammer to stretch them to fit, or an axe to cut off any excess length, forcing them to fit the bed.

Worse, Procrustes kept two beds, so no traveler could ever measure up. While Theseus ultimately triumphed over the giant Procrustes, forcing him to fit his own bed by cutting off his head and feet, the concept lives on. In literary analysis, a [Procrustean bed](http://en.wikipedia.org/wiki/Procrustes#Contemporary_usage) is an arbitrary standard to which exact conformity is enforced.

## Procrustean Mathematics

Traditional mathematicians finds themselves often forced into the role of Procrustes with many such theoretical beds at their disposal, they can proceed by cutting off bits or adjoining pieces to satisfy the requirements of whatever mathematical construct in which they want to work.

Conversely, the applied mathematician or computer scientist often finds themself in a situation where they care more about their problem patient, but can't find a bed to fit. Being a bit less psychopathic, they must set aside mathematical taste and adjust the [bed](http://www.youtube.com/watch?v=2q4JDpDSXMw). This is an exercise in centipede mathematics, the bed itself doesn't fit, so they rip parts off of it until it does.

We don't do this because we hate the bed, but because we are concerned for the patient.

The mathematician is looking out for the needs of the abstraction. The applied mathematician or computer scientist is looking out to maximize the fit for a given domain.

## The Operation Was a Success

In a [recent thread](http://www.reddit.com/r/haskell/comments/1ou06l/improving_applicative_donotation/) on Reddit it was suggested that every `Semigroup` should be extended to a `Monoid` and then we could be done with the need for the more refined concept.

We can actually often accomplish this. Consider this `Semigroup`:

```haskell
newtype First a = First { getFirst :: a }

instance Semigroup (First a) where
  m <> _ = m
```

If you really want a `Monoid` you can lay about with Procrustes' hammer and adjoin a unit. 

```haskell
newtype First a = First { getFirst :: Maybe a }

instance Semigroup (First a) where
  First Nothing <> m = m
  m             <> _ = m
  
instance Monoid (First a) where
  mempty = First Nothing

  First Nothing `mappend` m = m
  m             `mappend` _ = m
```

Now our object has grown a bit more complicated, but we can use it with all the usual `Foldable` machinery.

Sometimes the patient may be better off for his extra parts, but you may kill other properties you want along the way, or have to consider impossible cases.

Having `Semigroup` as a more fine-grained constraint does enable us to handle the empty case once and for all, and lets us fold over a `NonEmpty` container with more things and capture the non-empty nature of something via `Foldable1` and it simplifies `First`'s implementation considerably, but requires you to use something like `Option` to lift it into a `Monoid` you can use for a general purpose list.

Here this is simply a matter of taste, but that isn't always the case.

## ... But the Patient Died

If you try to stretch any particular `Comonad` to fit `Alternative`, you have to deal with the fact that

```haskell
extract empty :: a
```

This strongly implies no such beast can exist, and so the `Comonad` must die to make room for the `Alternative` instance. 

We have to give up either `empty` or `extract`.

The type system has taken on the role of the serial killer [Saw](http://en.wikipedia.org/wiki/Saw_%28film%29), sadistically forcing us to choose which of our friends will lose a limb. 

Even if you want to disavow centipede mathematics, you're going to be forced to occasionally put your abstractions on the chopping block, or abandon rigor.

## Haskell

You may be able to upgrade an `Applicative` parser to one that is a `Monad`, perhaps at the cost of parallelism.

Haskell's type system is very good at expressing a few well chosen abstractions. `Monad` used to be the golden hammer of the Haskell community, until we found out that `Applicative` functors exist and are useful for capturing context-free code, where the control flow doesn't vary based on previous results.

`Arrow` was introduced along the way, but later had a smaller `Category` class carved out of it.

Typeclasses in Haskell tend to force us into a small set of bed sizes, because it is relatively bad at code reuse across fine-grained class hierarchies.

Each attempt at refining the class hierarchy carries with it a price that library implementors and users who instantiate the classes must now write more methods. Worse, they must often do so without access to the full gamut of extra laws obtained further down in the class hierarchy, because they don't have a tenable way of offering defaults for superclass methods when they write a subclass.

Even with one of the superclass default proposals, you get no real code reuse for any form of transformer, and the existing default signature mechanism runs "the wrong way" in such a way that it even forces you to put everything in the same module.

The initial arguments against a fine-grained class hierarchy in Haskell arose from the same place as the denigration of centipede mathematics, but they are butressed by the pragmatic concern that there is real pain in an accurate class hierarchy caused by the design of the language.

These are valid concerns!

Arguments in favor of a finer-grained hierarchy arise from a desire to avoid flooding the namespace with redundant operations, and to capture the relationship between things. It arises from caring about the needs of the things you want to be able to reason about, rather than capturing just the examples that happen to measure up to an arbitrary standard.

These are also valid concerns!

## Semigroupoids

My `semigroupoids` package was originally written because I couldn't work with product categories in Haskell, but needed them in code. 

I still can't, due to the presence of `Any` as a distinguished member of every kind in Haskell.

Along the way, I dug up the [_-oid_-ification](http://ncatlab.org/nlab/show/horizontal+categorification) of a `Semigroup`, also known as a [semicategory](http://ncatlab.org/nlab/show/semicategory) to capture the portions of the product category that we _can_ write nicely in Haskell today.

When we look at the Kleisli category of things that are not quite a `Monad`, or the static arrow category of things that are not quite `Applicative`, we wind up with mere a `Semigroupoid` rather than a `Category`.

But where do we find such examples?

Consider the lowly `IntMap`. It cannot be made an instance of `Applicative` or `Monad` directly. 

We have three options to proceed if we want to work with something like `(<*>)` or `(>>=)` on it. 

1. We can clutter the namespace with a completely ad hoc combinator that we can't abstract over.

2. We can try to adjoin a universal default. This means that you have to kill the `Foldable` and `Traversable` instances for it, or deal with the fact that they basically return nonsense. It also means that you either have to give up the ability to delete from the map, or accept the fact that you aren't really modeling `(Int -> Maybe b)` any more.

3. We can engage in a bit of centipede mathematics, ripping off the `pure` and `return` from `Applicative` and `Monad` respectively to get a semi-`Applicative` and a semi-`Monad`, which I unartfully called `Apply` and `Bind`, in `semigroupoids`. Now we've respected the needs of the domain object, at the expense of a finer grained class hierarchy.

In a perfect world, from the perspective of the centipede mathematician, `Apply` and `Bind` would occupy a place of privilege in the class hierarchy.

However, to the Procrustean mathematicians who are only really concerned with `Applicative` and `Monad`, and who can't be bothered to deal with the finer grained hierarchy, such a refinement of the hierarchy merely adds cognitive overhead. They are happy to discard these examples in favor of a simpler, more teachable, meta-theory.

Both of these perspectives are valid.

## Extensible Effects

To unfairly cast Oleg Kiselyov in the role of Procrustes with Edwin Brady as his understudy, we can look at the modeling of [extensible effects](http://lambda-the-ultimate.org/node/4786) in this same light. 

Lawvere theories offer us too small a bed to fit many of the effects we care to model, such as continuation passing. This is why that effect is ignored by Edwin Brady's handling of effects for Idris. They just don't fit.

On the other hand, Oleg offers us a second bed that is much bigger, his `Eff` monad is the result of applying `Codensity` to a Lawvere Theory. Now it's the job of the handler to deal with the impossible cases. 

We're forced to set about with Procrustes' hammer to embed many monads, like `Reader` into a domain that is too large.

```haskell
Codensity ((->) s) a ~ 
forall r. (a -> s -> r) -> s -> r
```

is strong enough to implement all of CPS'd `State`. If you pass it `(,)` for its first argument you get `s -> (a, s)`! It is only by convention and hiding that we can restrict such a reader down to size.

<div align="center" style="padding-bottom: 10px"><img src="http://ekmett.github.io/images/suit.jpg"></div>

This means that the compiler has really no chance of ever optimizing the code properly as it must always live in fear that you could change the environment, even though the handler never will. This forces a single thread of execution through otherwise parallelizable code.

We improve the adjustability of this bed by switching from the `Codensity` construction to an arbitrary right Kan extension, like my old `monad-ran` package did. This lets the bed conform to the shape of `Reader`.

```haskell
Ran Identity ((->) s) a ~ 
Yoneda ((->) s) a ~
forall r. (a -> r) -> s -> r
```

This is just a CPS'd function, when passed `id` we recover `(s -> a)`. It is no longer large enough to model all of `State` and properly captures the features we want.

Yet even this bed is still too small for some patients. The infinite usecases of lazy writer and lazy state monads still cannot be made to fit at all. This destroys many interesting use cases of the [Tardis](http://hackage.haskell.org/package/tardis-0.3.0.0/docs/Control-Monad-Tardis.html). Admittedly they are the kinds of things that tie users in knots. Perhaps like an appendix removal, your patients will not miss those parts.

However, many monads for syntax trees like the transformer used by `bound` cannot be adapted without an _asymptotic_ performance hit, causing you to redo whole calculations every time you want to pattern match on the result.

From the standpoint of the Procrustean mathematician, the extensible effects approach is fairly elegant, it provides a single bed into which many common effects can fit.

The elegance of this approach makes it very appealing!

However, it isn't roomy enough to hold all of the current effects we can capture with monad transformers. Without upgrading to `Ran`, many effects are forced into a model that is too big, where you have to handle many impossible conditions that are merely ruled out by convention. On the other side, the inability to handle a number of cases that we do use in practice is also somewhat of a bad sign.

This is why I can bring myself to view extensible effects as a usful way to think about effects, but I can't view it as a full replacement for the monad transformer approach. 

Monad transformers do pay an `O(n^2)` complexity tax, describing how everything commutes over everything else, but `n` is typically not that big of a number. 

The abilty to handle the extra cases that don't fit the extensible effects approach means I can't bring myself to just relegate them to the waste bin of history. As an applied mathematician / computer scientist, I still need to handle those effects!

Concerns about the need to write `lift . lift . lift`, seem to arise from a particularly awkward style of use that I frankly never see in real code. There are ways to handle this and the multiple-state issues using tools we have, such as lenses. I'll relegate both of these concerns to another post.

## Lenses

The `lens` package is very much an exercise in building a very fine grained set of distinctions and trying to make it so you can say very precisely what constraints you want to impose on each operator.

Alternative designs like `fclabels` capture a different trade-off between factors.

I, personally, find that the ability to express a `Fold` or `Getter` is worth the added complexity of the representation. Your mileage may vary.

`fclabels` forces you to stretch each such thing into a form where you cannot express any laws, and then lays about with Procrustes' axe cutting of a number of abstractions we've found useful at the top end of the lens ecosystem, for Indexed traversals and the like.

It is a pretty clean exercise in Procrustean mathematics, though. If you fit into the abstraction it models comfortably, you'll never feel the bite of the axe.

Even `lens`, with its deep hierarchy, occasionally cuts you with Procrustes' axe. There are some constructions that just don't fit the domain. For instance `lens` offers no tool for validating input -- a `Lens' s a` must accept any such `a`.

There is a tension between these design criteria as with `Comonad` and `Alternative`. Something had to give. 

Lens chooses a different point on the design curve than `fclabels`. The choice we made was to gain a great deal of expressive power and ability to reason about the code with laws in exchange for validation.

## The View from the Middle

It is important to realize that modifying the abstraction/bed or modifying the problem/patient are both options. 

Sometimes you can easily adapt the problem to the mathematical construct, and sometimes the mathematical construct can be easily adapted to the problem.

When we add laws and operatons to an abstraction, we wind up with fewer examples.

If we work parametrically over an abstraction, the weaker the requirements we put on our inputs, the more scenarios we can cover.

Other times one or the other is up against a hard constraint. It is very important to distinguish between the normative concerns of taste and the very real concerns that some times one or the other of these things cannot give, as in the `Comonad/Alternative` case above.

To argue against a straw man, giving up either one of these degrees of freedom unilaterally strikes me as absurd. 

A hundred years ago, nobody cared about the foundations of mathematics, then came Russell and Whitehead, but their encoding was in many ways too dense and full of incredibly fine-grained distinctions.

Competing tensions like this gave us the mathematical world we inhabit today.

The choice of how to balance these factors, to abstract over enough problem domans to be useful without needlessly quibbling over impossibly fine-grained distinctions is really the true role of taste in library design and in mathematics, and tastes vary over time.

-[Edward Kmett](mailto:ekmett@gmail.com)

October 25th, 2013